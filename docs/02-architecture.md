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
| Persistencia | PostgreSQL (relacional) | NoSQL (ej. MongoDB) | El dominio depende de relaciones estables (Alumno↔GrupoAsignatura↔Instructor), transacciones atómicas (publicar corrección + disparar evento) e informes agregados con joins — todo terreno natural de SQL. NoSQL no aporta nada aquí y metería complejidad sin beneficio, el mismo criterio que descartó Kafka. |
| Claves primarias | UUID | Long autoincremental | No predecible/enumerable (un Long secuencial permite deducir volumen de datos o iterar IDs). Generable sin coordinación central, relevante si el sistema crece a varios servicios — coherente con ya usar RabbitMQ pensando en esa posibilidad. El coste de rendimiento frente a Long es marginal a la escala de este proyecto. |
| Estados (Entrega, Título) | Enum de Java + tipo `ENUM` nativo de PostgreSQL | Enum de Java + columna `VARCHAR` | Refuerza la integridad a nivel de base de datos, no solo en código: si algo escribe directamente en la tabla o hay un bug, la BD rechaza un valor que no sea uno de los estados válidos, en vez de aceptar cualquier texto. |
| Gestión del esquema de BD | Flyway (migraciones versionadas) | Hibernate `ddl-auto` automático | Code-first con Hibernate automático es cómodo en prototipado pero arriesgado en cuanto el proyecto crece: puede generar cambios destructivos sin avisar (ej. renombrar un campo se interpreta como borrar+crear), no deja historial versionado de qué cambió y por qué, y no garantiza que entornos distintos (local/test/producción) queden idénticos. Flyway aplica migraciones SQL numeradas con historial completo — mismo concepto que Git, aplicado al esquema. |

## Principio rector

El reto de diseño no es el CRUD en sí — es decidir **qué reacciona a qué, cuándo debe ser inmediato y cuándo puede ser agregado**, y modelar eso con una arquitectura de eventos coherente en lugar de lógica condicional dispersa por el código.

Esto se traduce en dos flujos de eventos con productores distintos (ver [`01-domain-design.md`](01-domain-design.md#principio-de-diseño-eventos-transaccionales-vs-eventos-de-scheduler)), reflejados también en la estructura de paquetes ([`03-project-structure.md`](03-project-structure.md)).

## Seguridad: accesos vs. acciones

El modelo de permisos por rol ([`01-domain-design.md`](01-domain-design.md#decisiones-cerradas-para-no-reabrir)) define *qué puede ver o hacer cada actor*, pero eso no se aplica solo: hace falta un mecanismo que lo haga cumplir en cada petición. Sin él, cualquier endpoint es accesible por cualquiera que conozca la URL, sin importar su rol.

Se distinguen dos preguntas distintas, resueltas en dos sitios distintos del código:

**Accesos — "¿puedes entrar aquí?"**
Pregunta de rol, resuelta *antes* de que la petición llegue a la lógica de negocio, con Spring Security (filtros + `@PreAuthorize`). Ejemplo: un Alumno que ataca `POST /api/entregas/{id}/corregir` recibe `403 Forbidden` directamente — el filtro de seguridad lo bloquea por no tener el rol `INSTRUCTOR`, sin que el método ni siquiera se ejecute.

**Acciones — "¿puedes hacer esto en concreto?"**
Pregunta de negocio, resuelta *dentro* del Service, una vez que el rol ya pasó el filtro de acceso. Un Instructor sí tiene rol para corregir entregas, pero solo las de sus propios `GrupoAsignatura` — no las de cualquier asignatura del sistema. Esto no lo resuelve la anotación de rol: requiere comprobar pertenencia ("¿esta Entrega pertenece a un grupo donde este Instructor es el asignado?"). Si no, `403` igualmente, pero por un motivo distinto: no es que no sea instructor, es que no es *su* grupo.

Caso concreto que ilustra por qué hace falta la segunda capa: con varios `GrupoAsignatura` por Instructor (ver dominio), el ataque real a vigilar no es solo "un alumno se hace pasar por instructor" (lo resuelve el rol) — es también "un instructor de la Asignatura A intenta corregir una entrega de la Asignatura B, que no es suya" (lo resuelve la comprobación de pertenencia al grupo).

## Pendiente de decidir (técnico)

- Qué librería de generación de PDF usar (ej. iText, Apache PDFBox, o plantilla HTML→PDF)
- Qué servicio de envío de email usar (SMTP propio vía Spring Mail, o un proveedor externo tipo SendGrid/Mailgun) y cómo manejar reintentos si el envío falla
- Dónde vive exactamente la comprobación de pertenencia a grupo: en el Service de cada dominio, o centralizada en algún componente común reutilizable

## Seguridad — sesión pendiente en profundidad

Identificado como laguna real a trabajar con calma, no solo como casilla técnica. Cuando se aborde, cubrir desde cero (sin asumir conocimiento previo):

- Mecanismo de autenticación: qué es JWT, cómo se genera, cómo se valida en cada petición
- Spring Security: filtros, `@PreAuthorize`, cómo se configuran los roles
- La distinción accesos/acciones ya cerrada arriba, llevada a implementación real
- Buenas prácticas básicas: hashing de contraseñas, expiración de tokens, CORS