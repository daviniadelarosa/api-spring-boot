# Arquitectura

> Decisiones técnicas que sostienen el dominio descrito en [`01-domain-design.md`](01-domain-design.md). Cada decisión incluye la alternativa descartada y el motivo, no solo la elección final.

## Decisiones de arquitectura

| Decisión | Elegido | Alternativa descartada | Por qué |
|---|---|---|---|
| Mensajería entre eventos | RabbitMQ | Event Bus interno de Spring (`ApplicationEventPublisher`) | El bus interno es síncrono y vive solo dentro del proceso: si el servicio cae, los eventos pendientes se pierden, y no escala a varias instancias. RabbitMQ persiste mensajes y permite crecer a varios servicios sin rediseñar nada. |
| Mensajería entre eventos | RabbitMQ | Kafka | Kafka está pensado para volúmenes y streaming que este dominio no tiene. Usarlo aquí sería sobre-ingeniería. |
| Tiempo real hacia el cliente | WebSockets | Server-Sent Events (SSE) | SSE es unidireccional (servidor→cliente). Aquí hace falta bidireccionalidad real: ej. un instructor marca "revisando ahora" para evitar que otro instructor duplique trabajo. |
| Notificaciones a Instructor sobre certificados | Informe mensual agregado | Notificación individual por certificado emitido | No todo evento necesita ser en tiempo real. Saber cuándo *no* usar push inmediato es tan parte del diseño como saber cuándo sí. |
| Entrega del certificado | PDF generado + envío por email | Solo notificación WebSocket sin documento | El certificado es un documento que debe persistir y poder reenviarse, no un aviso efímero. El email es canal de *entrega del documento*, no un canal de notificación paralelo — el WebSocket sigue siendo el único aviso en tiempo real de que "tu certificado ya está". |

## Principio rector

El reto de diseño no es el CRUD en sí — es decidir **qué reacciona a qué, cuándo debe ser inmediato y cuándo puede ser agregado**, y modelar eso con una arquitectura de eventos coherente en lugar de lógica condicional dispersa por el código.

Esto se traduce en dos flujos de eventos con productores distintos (ver [`01-domain-design.md`](01-domain-design.md#principio-de-diseño-eventos-transaccionales-vs-eventos-de-scheduler)), reflejados también en la estructura de paquetes ([`03-project-structure.md`](03-project-structure.md)).

## Pendiente de decidir (técnico)

- Qué librería de generación de PDF usar (ej. iText, Apache PDFBox, o plantilla HTML→PDF)
- Qué servicio de envío de email usar (SMTP propio vía Spring Mail, o un proveedor externo tipo SendGrid/Mailgun) y cómo manejar reintentos si el envío falla
- Implementación técnica del modelo de permisos por rol ya cerrado en [`01-domain-design.md`](01-domain-design.md#decisiones-cerradas-para-no-reabrir) — método de Spring Security a usar (`@PreAuthorize`, filtros, etc.)