---
id: ADR-036
title: Local topology with provisioning queue, independent mock build and pinned LocalStack
status: ACCEPTED
priority: LOW
supersedes: ADR-017
human_decision: CONFIRMED_WITH_CHANGE
source_ids: [TC-006, TC-012, DEL-005, TC-004, TC-005, TC-016, TC-017, DEL-002, DEL-004]
related_adrs: [ADR-022, ADR-024, ADR-026, ADR-029, ADR-030, ADR-032, ADR-033, ADR-034, ADR-037, ADR-038, ADR-040]
---

# ADR-036 — Local topology with provisioning queue, independent mock build and pinned LocalStack

Reemplaza a ADR-017. No define el archivo de Docker Compose.

## Human decision applied

Respuesta humana vinculante a ADR-017 (CONFIRMED_WITH_CHANGE), incorporada íntegramente:

> Se confirma la topología local con la misma forma que el objetivo en AWS: `ticketing-api` y `ticketing-worker` como dos contenedores de la misma imagen, recursos creados por una tarea `infra-init` de un solo uso e idempotente, comprobaciones de salud con dependencias explícitas, configuración por variables de entorno con un archivo de ejemplo versionado, imágenes multi-etapa con usuario sin privilegios y perfil opcional de prueba de carga. Se descarta el contenedor único que crea sus propios recursos. Se añaden cuatro cambios.
>
> Recursos de `infra-init`. Además de la tabla y la cola de Orders con su DLQ, `infra-init` crea el GSI de Ticket disponibles con sharding (ADR-001, ADR-021), el índice de Orders pendientes de encolado (ADR-006), el índice de reversos de pago (FG-003) y la cola de aprovisionamiento con su DLQ y política de redrive (ADR-004, ADR-010).
>
> Payment Mock. `payment-mock` se construye desde su propia carpeta y build, independiente del build de `ticketing`, conforme a ADR-015.
>
> Identidades para la prueba de carga. `local-idp` emite tokens para sujetos arbitrarios conforme a ADR-014, y el perfil de prueba de carga incluye un paso previo que genera más de 1.000 tokens `CUSTOMER` distintos.
>
> LocalStack. Desde el 23 de marzo de 2026 la imagen `localstack/localstack:latest` exige `LOCALSTACK_AUTH_TOKEN` para arrancar. Para que el entorno se levante sin cuentas externas, LocalStack se fija a un tag publicado antes de esa exigencia, que se verificará como primera tarea del desarrollo, incluyendo el soporte de redrive a DLQ, el contador aproximado de recepciones y el cambio de visibilidad. El README documenta como alternativa el uso de la versión actual con un token gratuito suministrado por un archivo no versionado, nunca incluido en el repositorio conforme a ADR-013. Si el tag fijado no cumple, el respaldo es ElasticMQ, previa revisión de HV-010 en la especificación funcional.

Correspondencia: ADR-001 → ADR-022, ADR-004 → ADR-024, ADR-006 → ADR-026, ADR-010 → ADR-029, ADR-013 → ADR-032, ADR-014 → ADR-033, ADR-015 → ADR-034, ADR-021 → ADR-040. El índice de reversos de pago se materializa como el rango `REVERSAL#` del índice de trabajo `GSI3`, conforme a ADR-008 (aceptado) y ADR-022; `infra-init` lo crea al crear `GSI3`.

El dato sobre la exigencia de token de LocalStack lo aporta la revisión humana; el tag concreto queda `TO_VERIFY`.

## Context

`TC-012` y `DEL-005` exigen un Docker Compose con la aplicación, DynamoDB Local y SQS mediante LocalStack, más las dependencias necesarias; `TC-006` exige contenerización.

## Options considered

### Option A — Misma forma que el objetivo, recursos creados por un inicializador de un solo uso

- A favor: reproduce las unidades de despliegue de AWS; la aplicación nunca crea infraestructura.
- En contra: más contenedores.

### Option B — Un único contenedor con todos los roles que crea sus recursos al arrancar

- En contra: código de creación de infraestructura que no existe en AWS. Descartada.

## Decision

Se adopta la **Option A**.

### Servicios

| Servicio | Propósito | Depende de | Expuesto al host |
|---|---|---|---|
| `dynamodb-local` | Persistencia (`TC-004`) | — | Sí, inspección |
| `localstack` (tag fijado) | SQS Standard: colas de Orders y de aprovisionamiento con sus DLQ (`TC-005`) | — | Sí, inspección |
| `local-idp` | Emisor de JWT con sujetos arbitrarios (ADR-033) | — | Sí, obtención de tokens |
| `payment-mock` | Payment Mock, construido desde su propia carpeta y build (ADR-030, ADR-034) | — | Sí, API de control |
| `infra-init` | Un solo uso, idempotente: crea la tabla `ticketing` con `GSI1`, `GSI2` (disponibles con sharding), `GSI3` (índice de trabajo: expiración, reversos de pago y revisión), `GSI4` (pendientes de encolado) y TTL; las colas `ticketing-orders`, `ticketing-orders-dlq`, `ticketing-event-provisioning`, `ticketing-event-provisioning-dlq` con sus atributos y políticas de redrive | `dynamodb-local` y `localstack` saludables | No |
| `ticketing-api` | Rol `api` | `infra-init` completado; `local-idp` saludable | Sí (puerto de aplicación; puerto de gestión solo en local) |
| `ticketing-worker` | Rol `worker` | `infra-init` completado; `payment-mock` saludable | No |
| `load-token-generator` (perfil de carga) | Genera de antemano más de 1.000 tokens `CUSTOMER` con sujetos distintos | `local-idp` saludable | No |
| `load-test` (perfil de carga) | Prueba de carga (ADR-038) | `ticketing-api` saludable; `load-token-generator` completado | No |

### Orden de arranque

1. `dynamodb-local`, `localstack`, `local-idp`, `payment-mock`.
2. `infra-init`, con ambos emuladores saludables.
3. `ticketing-api` y `ticketing-worker`.
4. Bajo el perfil de carga: `load-token-generator` y después `load-test`.

### Reglas

- Comprobaciones de salud en todos los servicios; dependencias por "saludable" o "completado con éxito".
- `ticketing-api` y `ticketing-worker` son la misma imagen.
- Configuración por variables de entorno; valores sensibles (API key del mock, token opcional de LocalStack) en un archivo no versionado; se versiona un ejemplo con valores ficticios.
- Imágenes multi-etapa y usuario sin privilegios.
- `infra-init` idempotente; su definición de recursos es equivalente a la que materializa la infraestructura como código (`ticketing.aws-target.v2.md` §11).
- Datos de los emuladores efímeros por defecto.

### LocalStack

1. Por defecto: imagen fijada a un tag publicado antes del 23 de marzo de 2026, sin token. Primera tarea del desarrollo: verificar SQS Standard, redrive a DLQ, contador aproximado de recepciones y cambio de visibilidad (incluido el heartbeat de ADR-029).
2. Alternativa documentada en el README: versión actual con token gratuito en un archivo no versionado.
3. Respaldo si el tag no cumple: ElasticMQ, previa revisión de `HV-010` / `TC-005` / `TC-012` en la especificación (registrado en `ticketing.functional-clarifications.v1.md`).

## Rationale

- Misma forma que AWS y misma responsabilidad de creación de recursos.
- Fijar el tag mantiene `TC-012` sin cuentas externas.

## Consequences

- Siete contenedores más dos del perfil de carga.
- El README documenta arranque, obtención de tokens, configuración del mock, alternativa de LocalStack y escenarios de resiliencia (ADR-038, `DEL-002`).

## Risks and mitigations

| Risk | Mitigation |
|---|---|
| `RISK-007` fidelidad de emuladores | Verificación temprana; pruebas repetibles contra AWS |
| `RISK-021` el tag fijado no ofrece las capacidades requeridas | Token gratuito o ElasticMQ con revisión funcional |

## Items to verify

| Item | Qué comprobar | Impacto si difiere |
|---|---|---|
| Tag de LocalStack anterior al 23 de marzo de 2026 | Existencia, arranque sin token, SQS con redrive, contador de recepciones, cambio de visibilidad | Alternativas del punto anterior |
| Imagen para `infra-init` | Herramienta de línea de comandos de AWS contra los emuladores | Mecanismos de inicialización propios de cada emulador |
| Imagen base de Java 25 | Disponibilidad | Cambia la imagen |

## Depends on

- `FG-003` (resuelto).
