# Estructura del proyecto

> Organización de paquetes en Spring, reflejando el dominio descrito en [`01-domain-design.md`](01-domain-design.md) y las decisiones de [`02-architecture.md`](02-architecture.md).

## Enfoque: organización por dominio/feature, no por capa técnica

La división clásica `controller/ service/ repository/` funciona para un CRUD simple, pero se convierte en un cajón de sastre en cuanto hay varias entidades relacionadas, eventos y mensajería de por medio. En cambio, cada dominio es autocontenido: abrir `entregas/` muestra todo lo relacionado con Entrega — controller, service, repository, eventos — sin tener que saltar entre capas para seguir un solo flujo.

La separación transaccional/scheduler **no se repite dentro de cada carpeta de dominio** — es una cuestión transversal (quién produce el evento), así que vive en su propio paquete `scheduler/`.

## Estructura

```
src/main/java/com/tuempresa/progresoapi/
│
├── entregas/
│   ├── EntregaController.java
│   ├── EntregaService.java
│   ├── EntregaRepository.java
│   ├── Entrega.java                  (entidad)
│   └── events/
│       ├── EntregaRealizadaEvent.java
│       ├── EntregaVistaEvent.java
│       ├── EntregaDevueltaABorradorEvent.java
│       └── RevisionIniciadaEvent.java
│
├── consultas/
│   ├── ConsultaController.java
│   ├── ConsultaService.java
│   ├── ConsultaRepository.java
│   ├── Consulta.java
│   └── events/
│       ├── ConsultaRealizadaEvent.java
│       └── RespuestaConsultaPublicadaEvent.java
│
├── titulos/
│   ├── TituloController.java
│   ├── TituloService.java
│   ├── TituloRepository.java
│   ├── Titulo.java
│   ├── PdfGeneratorService.java        (generación del PDF del certificado)
│   ├── EmailService.java               (envío con adjunto)
│   └── events/
│       ├── TituloElegibilidadDetectadaEvent.java
│       └── TituloEmitidoEvent.java
│
├── grupos/                             (Asignatura + GrupoAsignatura + matriculación)
│   ├── AsignaturaController.java
│   ├── GrupoAsignaturaController.java
│   ├── ...Service.java / ...Repository.java
│   ├── Asignatura.java
│   └── GrupoAsignatura.java
│
├── alumnos/
│   └── ... (CRUD básico de Alumno, gestionado por Administración)
│
├── instructores/
│   └── ... (CRUD básico de Instructor, gestionado por Administración)
│
├── scheduler/                          ← aquí se ve la separación transaccional/scheduler
│   ├── PlazoPorVencerJob.java
│   ├── BorradorSinEnviarJob.java
│   ├── InactividadDetectadaJob.java
│   ├── RecuentoPendientesJob.java
│   ├── InformeMensualJob.java
│   └── TituloElegibilidadJob.java
│
├── messaging/                          (configuración e infraestructura RabbitMQ)
│   ├── RabbitMQConfig.java
│   ├── EventPublisher.java             (publica cualquier evento al broker)
│   └── EventListener.java              (consume y enruta a WebSocket/email)
│
├── websocket/
│   ├── WebSocketConfig.java
│   └── NotificacionWebSocketService.java
│
└── informes/                           (Administración: vistas agregadas)
    ├── InformeController.java
    └── InformeService.java
```

## Por qué esta estructura, no otra

- **Cada dominio (`entregas/`, `consultas/`, `titulos/`) es autocontenido** — refleja directamente las entidades cerradas en el diseño de dominio.
- **`scheduler/` es su propio paquete, separado de los dominios** — cada Job ahí dentro toca varios dominios a la vez (ej. `InformeMensualJob` lee de Entregas, Consultas y Títulos para construir el informe). Meterlo dentro de un solo dominio sería forzado.
- **`messaging/` centraliza RabbitMQ** — cualquier dominio publica eventos sin conocer los detalles de colas/exchanges, solo llama a `EventPublisher.publish(evento)`.
- **`informes/` separado de `grupos/`** — el caso de uso de Administración (vistas agregadas, sin acceso al detalle de cada corrección) es una responsabilidad distinta de "gestionar quién está en qué grupo".

## Pendiente de decidir

- Si `alumnos/` e `instructores/` deberían fusionarse con `grupos/` en vez de mantenerse separados