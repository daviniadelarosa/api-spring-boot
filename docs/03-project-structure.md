# Project structure

> Package organization in Spring, reflecting the domain described in [`01-domain-design.md`](01-domain-design.md) and the decisions in [`02-architecture.md`](02-architecture.md).

## Approach: package by feature, not by technical layer

A classic `controller/ service/ repository/` split works for a simple CRUD, but becomes a junk drawer once multiple related entities, events, and messaging are involved. Instead, each domain is self-contained: opening `submissions/` shows everything related to a Submission — controller, service, repository, events — without jumping across layers to follow one flow.

The transactional/scheduler split is **not** repeated inside every feature folder — it's a cross-cutting concern (who produces the event), so it lives in its own `scheduler/` package instead.

## Structure

```
src/main/java/com/yourcompany/progressapi/
│
├── submissions/
│   ├── SubmissionController.java
│   ├── SubmissionService.java
│   ├── SubmissionRepository.java
│   ├── Submission.java                  (entity)
│   └── events/
│       ├── SubmissionCreatedEvent.java
│       ├── SubmissionViewedEvent.java
│       ├── SubmissionReturnedToDraftEvent.java
│       └── ReviewStartedEvent.java
│
├── inquiries/
│   ├── InquiryController.java
│   ├── InquiryService.java
│   ├── InquiryRepository.java
│   ├── Inquiry.java
│   └── events/
│       ├── InquiryCreatedEvent.java
│       └── InquiryAnsweredEvent.java
│
├── certificates/
│   ├── CertificateController.java
│   ├── CertificateService.java
│   ├── CertificateRepository.java
│   ├── Certificate.java
│   ├── PdfGeneratorService.java          (certificate PDF generation)
│   ├── EmailService.java                 (sends PDF as attachment)
│   └── events/
│       ├── CertificateEligibilityDetectedEvent.java
│       └── CertificateIssuedEvent.java
│
├── groups/                               (Subject + SubjectGroup + enrollment)
│   ├── SubjectController.java
│   ├── SubjectGroupController.java
│   ├── ...Service.java / ...Repository.java
│   ├── Subject.java
│   └── SubjectGroup.java
│
├── students/
│   └── ... (basic Student CRUD, managed by Administration)
│
├── instructors/
│   └── ... (basic Instructor CRUD, managed by Administration)
│
├── scheduler/                            ← where the transactional/scheduler split shows
│   ├── DeadlineApproachingJob.java
│   ├── UnsubmittedDraftJob.java
│   ├── InactivityDetectionJob.java
│   ├── PendingCountJob.java
│   ├── MonthlyReportJob.java
│   └── CertificateEligibilityJob.java
│
├── messaging/                            (RabbitMQ config and infrastructure)
│   ├── RabbitMQConfig.java
│   ├── EventPublisher.java               (publishes any event to the broker)
│   └── EventListener.java                (consumes and routes to WebSocket/email)
│
├── websocket/
│   ├── WebSocketConfig.java
│   └── NotificationWebSocketService.java
│
└── reports/                              (Administration: aggregated views)
    ├── ReportController.java
    └── ReportService.java
```

## Why this, not something else

- **Each domain (`submissions/`, `inquiries/`, `certificates/`) is self-contained** — mirrors the entities already closed in the domain design.
- **`scheduler/` is its own package, separate from the domains** — each job inside touches several domains at once (e.g. `MonthlyReportJob` reads from Submissions, Inquiries, and Certificates to build the report). Forcing it inside a single domain would be artificial.
- **`messaging/` centralizes RabbitMQ** — any domain publishes events without knowing queue/exchange details, just calling `EventPublisher.publish(event)`.
- **`reports/` is separate from `groups/`** — the Administration use case (aggregated views, no access to individual correction detail) is a distinct responsibility from "managing who's in which group".

## Naming reference (Spanish domain terms → English code terms)

| Domain term (docs) | Code term |
|---|---|
| Entrega | Submission |
| Consulta | Inquiry |
| Título / certificado | Certificate |
| Asignatura | Subject |
| GrupoAsignatura | SubjectGroup |
| Alumno | Student |
| Instructor | Instructor |
| Administración | Administration |

## Pending decisions

- Whether `students/` and `instructors/` should merge into `groups/` instead of staying separate