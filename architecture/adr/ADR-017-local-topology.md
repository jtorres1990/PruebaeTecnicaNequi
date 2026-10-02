---
id: ADR-017
title: Local topology
status: PROPOSED
priority: LOW
source_ids: [TC-006, TC-012, DEL-005, TC-004, TC-005, TC-016, TC-017, DEL-002]
related_adrs: [ADR-011, ADR-014, ADR-015, ADR-018, ADR-019]
---

# ADR-017 — Local topology

## Context

`TC-012` y `DEL-005` exigen un Docker Compose que levante la aplicación, DynamoDB Local y SQS mediante LocalStack, además de las dependencias necesarias. `TC-006` exige contenerización. Este ADR enumera servicios, dependencias y orden de arranque; no define el archivo.

## Options considered

### Option A — Misma forma que el objetivo: API y worker como procesos separados, recursos creados por un inicializador de un solo uso

- A favor: reproduce las unidades de despliegue de AWS; el escalado independiente y las entregas duplicadas se pueden demostrar en local; la aplicación nunca crea infraestructura, igual que en AWS.
- En contra: más contenedores.

### Option B — Un único contenedor de aplicación con todos los roles, que crea tabla y colas al arrancar

- A favor: mínimo número de contenedores.
- En contra: introduce en la aplicación código de creación de infraestructura que no existe en AWS; oculta la separación entre API y worker.

## Decision

Se adopta la **Option A**.

### Servicios

| Servicio | Propósito | Depende de | Expuesto al host |
|---|---|---|---|
| `dynamodb-local` | Persistencia (`TC-004`) | — | Sí, para inspección |
| `localstack` | SQS Standard: cola principal y DLQ (`TC-005`) | — | Sí, para inspección |
| `local-idp` | Emisor de JWT local (`ADR-014`) | — | Sí, para obtener tokens |
| `infra-init` | Tarea de un solo uso: crea la tabla con sus GSIs y TTL, las colas y la política de redrive | `dynamodb-local` y `localstack` saludables | No |
| `payment-mock` | Payment Mock (`ADR-011`) | — | Sí, API de control para pruebas |
| `ticketing-api` | Aplicación con rol `api` | `infra-init` completado; `local-idp` saludable | Sí |
| `ticketing-worker` | Aplicación con rol `worker` | `infra-init` completado; `payment-mock` saludable | No |
| `load-test` (perfil opcional) | Ejecuta la prueba de carga (`ADR-019`) | `ticketing-api` saludable | No |

### Orden de arranque

1. `dynamodb-local`, `localstack`, `local-idp`, `payment-mock`.
2. `infra-init`, cuando los dos emuladores están saludables; termina con éxito.
3. `ticketing-api` y `ticketing-worker`, cuando `infra-init` terminó y sus dependencias están saludables.
4. `load-test`, solo bajo su perfil.

### Reglas

- Todos los servicios declaran comprobación de salud; las dependencias esperan a "saludable" o a "completado con éxito", no solo al arranque.
- `ticketing-api` y `ticketing-worker` son la misma imagen con distinto rol (`ADR-015`).
- Toda la configuración llega por variables de entorno; los valores sensibles vienen de un archivo no versionado, del que se versiona un ejemplo con valores ficticios (`ADR-013`).
- Las imágenes de la aplicación se construyen en varias etapas y se ejecutan con un usuario sin privilegios.
- `infra-init` es idempotente: puede ejecutarse de nuevo sin fallar si los recursos existen.
- Los datos de los emuladores son efímeros por defecto.
- La definición de recursos que usa `infra-init` debe ser equivalente a la que materializa la infraestructura como código en AWS (`ticketing.aws-target.md` §11).

## Rationale

- Mantener la misma separación de procesos y la misma responsabilidad de creación de recursos que en AWS es lo que exige el principio de coherencia entre local y objetivo.
- Un inicializador de un solo uso evita condiciones de carrera entre API y worker al crear recursos.

## Consequences

- El entorno local tiene siete contenedores, más el perfil opcional.
- Las diferencias inevitables con AWS quedan registradas en `ticketing.aws-target.md` §10.
- El README debe documentar el arranque, la obtención de tokens y la configuración del Payment Mock (`DEL-002`).

## Risks and mitigations

| Risk | Mitigation |
|---|---|
| `RISK-007` diferencias de fidelidad de los emuladores | Lista de verificación en "Items to verify"; pruebas de integración repetibles contra AWS |
| Arranque no determinista por dependencias no saludables | Comprobaciones de salud y condiciones de dependencia explícitas |

## Items to verify

| Item | Qué comprobar | Impacto si difiere |
|---|---|---|
| Imágenes de DynamoDB Local y LocalStack | Disponibilidad, comprobación de salud y soporte de SQS con redrive en la edición sin licencia | Si SQS no está disponible, sustituir por otro emulador compatible con la API de SQS |
| Imagen para `infra-init` | Herramienta de línea de comandos de AWS apuntando a los emuladores | Alternativa: usar los mecanismos de inicialización propios de cada emulador |
| Imagen base de Java 25 | Disponibilidad | Cambia la imagen base |

## Depends on

- Sin dependencias de `FG-*` ni `AV-*`.
