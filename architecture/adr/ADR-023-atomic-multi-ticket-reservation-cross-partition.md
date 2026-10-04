---
id: ADR-023
title: Atomic multi-ticket reservation across partitions
status: ACCEPTED
priority: HIGH
supersedes: ADR-002
human_decision: CONFIRMED_WITH_CHANGE
source_ids: [TC-011, ST-001, ST-006, FR-004, FR-006, FR-010, FR-016, VAL-002, VAL-004, VAL-010, BR-008, BR-013, BR-014, BR-022, AC-004, AC-007, AC-016, ERR-001, ERR-002, ALT-002]
related_adrs: [ADR-003, ADR-022, ADR-024, ADR-025, ADR-026, ADR-027, ADR-032, ADR-035, ADR-039]
---

# ADR-023 — Atomic multi-ticket reservation across partitions

Reemplaza a ADR-002.

## Human decision applied

Respuesta humana vinculante a ADR-002 (CONFIRMED_WITH_CHANGE), incorporada íntegramente:

> La reserva de múltiples Ticket debe ejecutarse mediante una única operación `TransactWriteItems` all-or-nothing. La transacción puede actualizar Ticket ubicados en partition keys y particiones físicas diferentes, conforme al modelo aprobado en ADR-001, y debe crear en el mismo commit la Order con su Reservation, el registro de idempotencia y el registro de auditoría.
>
> Cada actualización de Ticket debe comprobar condicionalmente que el item existe, pertenece al Event solicitado y continúa en estado `AVAILABLE`. Si falla la condición de cualquier Ticket, se cancela la transacción completa: ningún Ticket cambia, no se crean Reservation ni Order y no se retorna Order ID.
>
> El Event se valida antes de ejecutar la transacción y queda fuera de ella para evitar que todas las compras del mismo Event compitan por un único item. Esto requiere que el Event sea inmutable después de habilitarse y que la validación compruebe que puede venderse.
>
> Las cancelaciones producidas por conflictos técnicos transaccionales pueden reintentarse de forma acotada con jitter. Una condición fallida por un Ticket inexistente o no disponible es un rechazo funcional completo y no debe reintentarse como conflicto técnico. El máximo de Ticket por Order se resolverá en FG-001 respetando los límites vigentes de la transacción.

Decisiones aprobadas incorporadas: claves por Ticket (ADR-022); máximo 1..10 sin repetidos (`FG-001`, ADR-003); bloqueo de Order activa por cliente y Event dentro de la misma transacción (ADR-032); prioridad del registro de idempotencia en la carrera con la misma clave (ADR-027); Event pasado rechazado como `EVENT_NOT_ON_SALE` antes de la transacción (`FG-005`); Event no `ENABLED` tratado como inexistente (ADR-024, `AV-002`); rechazo `SERVICE_UNAVAILABLE` antes de reservar con el circuito de SQS abierto (ADR-035); motivos de cancelación por item (ADR-039).

## Context

Al iniciar una compra deben reservarse todos los Ticket o ninguno (`FR-004`, `ST-001`, `AC-016`); solo una solicitud concurrente gana un Ticket (`FR-010`, `AC-007`); Reservation y Order nacen consistentes con los Ticket (`ST-006`, `AC-004`). Un Ticket no disponible produce un rechazo sin Reservation, Order ni Order ID (`HC-001`, `ALT-002`, `ERR-002`). `TC-011` admite optimistic locking o conditional writes. Con ADR-022 los Ticket de una Order están en partition keys distintas.

## Options considered

### Option A — Conditional writes sobre el estado dentro de una única escritura transaccional multi-partición

- A favor: atomicidad garantizada por el almacén aunque los items estén en particiones distintas; el predicado de negocio es exactamente la condición; sin lectura previa de Ticket.
- En contra: tamaño de transacción acotado; coste de escritura transaccional; cancelaciones por conflicto bajo contención.

### Option B — Optimistic locking con atributo de versión

- A favor: patrón conocido.
- En contra: N lecturas adicionales en la ruta caliente; la versión es un predicado más débil que el estado y falla por cambios irrelevantes.

### Option C — Conditional writes independientes por Ticket con compensación

- A favor: sin límite de items.
- En contra: expone reservas parciales observables durante la compensación (`AC-014`, `AC-016`).

## Decision

Se adopta la **Option A** (`AP-008`).

### Contenido de la transacción (N + 4 items)

| Item | Operación | Condición |
|---|---|---|
| Cada Ticket solicitado (`TICKET#<eventId>#<ticketId>`) | `state` → `RESERVED`, registra `orderId`, sale de `GSI2` | El item existe, `eventId` = Event solicitado y `state = AVAILABLE` |
| Order | Creación en `CREATED` con Reservation (`expiresAt = reservedAt + 10 min`), entrada en `GSI3` rango `RESV#` y en `GSI4` | El item no existe |
| Idempotencia de compra | Creación con `orderId` y hash del contenido | El item no existe |
| Auditoría (`ST-001`, `ST-006`) | Inserción | El item no existe |
| Bloqueo de Order activa (`ACTIVE#<customerId>#<eventId>`) | Creación con `orderId` | El item no existe (ADR-032) |

Con el máximo de 10 Ticket (ADR-003) la transacción tiene 14 items.

### Validaciones previas a la transacción (en este orden)

1. Validación de entrada en el punto HTTP y de nuevo en el caso de uso: 1 a 10 `ticketIds`, sin repetidos, formato válido, un único `eventId`, `Idempotency-Key` válida (ADR-003, ADR-027).
2. Lectura fuertemente consistente del registro de idempotencia; si existe, se resuelve como repetición conforme a ADR-027 y no se continúa.
3. Lectura del Event (`AP-006`, cacheable solo si está `ENABLED`, porque entonces es inmutable). Si no existe o no está `ENABLED`: `EVENT_NOT_FOUND`. Si `startsAt` es menor o igual al instante actual del servidor en UTC: `EVENT_NOT_ON_SALE` (`FG-005`). No hay guardas sobre el Event dentro de la transacción.
4. Comprobación de los `ticketIds` contra la definición del inventario del Event: un identificador que no corresponde a ninguna ubicación produce `UNKNOWN_TICKETS` sin ejecutar la transacción. Es una optimización; la condición de existencia dentro de la transacción sigue siendo autoritativa.
5. Si el circuito de publicación en SQS está abierto, rechazo `SERVICE_UNAVAILABLE` con `Retry-After`, sin reservar (ADR-035).

### Interpretación del resultado

| Resultado | Tratamiento |
|---|---|
| Commit | Order en `CREATED`, todos sus Ticket en `RESERVED`; se continúa con el encolado (ADR-026) |
| Cualquier cancelación (por condición o por conflicto) | Primero, relectura fuertemente consistente del registro de idempotencia. Si existe: repetición (mismo contenido) o `IDEMPOTENCY_KEY_REUSED` (contenido distinto), conforme a ADR-027. Esta comprobación prevalece sobre cualquier otro motivo |
| Cancelación por condición del bloqueo | `ACTIVE_ORDER_EXISTS` (ADR-032) |
| Cancelación por condición de un Ticket sin item previo | `UNKNOWN_TICKETS` |
| Cancelación por condición de un Ticket con estado distinto de `AVAILABLE` | `TICKETS_UNAVAILABLE`, rechazo funcional completo; nunca se reintenta |
| Cancelación por conflicto transaccional | Reintento de la transacción completa, máximo 2, con backoff y jitter; si persiste, `SERVICE_UNAVAILABLE` sin haber persistido nada (ADR-035) |

Precedencia cuando varios items fallan a la vez: idempotencia, bloqueo, Ticket inexistente, Ticket no disponible. Los motivos por item se obtienen solicitando el item que no cumplió la condición (ADR-039); como respaldo, lectura posterior de los items señalados.

Ningún rechazo persiste nada: ni Reservation, ni Order, ni registro de idempotencia, ni bloqueo (`HC-001`, `FG-004`).

### Event fuera de la transacción

El Event no forma parte de la transacción. Es seguro porque es inmutable desde que pasa a `ENABLED` (ADR-024, `AV-002`) y sus Ticket no son direccionables por compra mientras no está `ENABLED` (la validación previa lo rechaza).

## Rationale

- La transacción multi-partición preserva el todo-o-nada con la distribución de escritura de ADR-022.
- La condición `state = AVAILABLE` garantiza un único ganador por Ticket (`AC-007`, `VAL-004`); la condición de `eventId` hace inexpresable mezclar Events (`BR-014`).
- Releer la idempotencia ante cualquier cancelación hace correcta la carrera de dos solicitudes con la misma clave aunque una se cancele también por estado de Ticket.
- Las validaciones baratas antes de la transacción devuelven errores funcionales deterministas sin consumir capacidad transaccional.

## Consequences

- Transacción de 14 items como máximo.
- Bajo contención real sobre un mismo Ticket parte de las solicitudes recibe conflicto y reintenta (`RISK-002`).
- El rechazo no deja rastro persistente; su repetición se reevalúa (`FG-004`).
- El caché de Events cubre solo Events `ENABLED`; un Event recién habilitado se lee de la tabla hasta que entra en caché.

## Risks and mitigations

| Risk | Mitigation |
|---|---|
| `RISK-002` conflictos transaccionales | Transacciones de 14 items como máximo, reintento acotado con jitter, métrica de conflictos, 503 reintentable con la misma clave |
| Clasificación errónea entre inexistente y no disponible | Comprobación previa contra la definición; motivo por item; lectura de respaldo; ambos casos rechazan sin efectos |

## Items to verify

| Item | Qué comprobar | Impacto si difiere |
|---|---|---|
| Límite de items y tamaño de `TransactWriteItems` | Se asume 100 items | 14 cabe con amplio margen; un aumento del máximo de ADR-003 exige reverificar |
| Motivos de cancelación por item y devolución del item que falló la condición | Soporte en el SDK, el servicio y DynamoDB Local | Si no existe, usar siempre la lectura de respaldo |
| Exclusión entre transacciones concurrentes multi-partición en DynamoDB Local | Fidelidad bajo concurrencia | Si no, las pruebas de concurrencia de integración se ejecutan contra AWS (ADR-038) |

## Depends on

- `FG-001`, `FG-004`, `FG-005` (resueltos).
