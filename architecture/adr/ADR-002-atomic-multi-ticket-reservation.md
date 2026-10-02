---
id: ADR-002
title: Atomic multi-ticket reservation
status: PROPOSED
priority: HIGH
source_ids: [TC-011, ST-001, ST-006, FR-004, FR-006, FR-010, FR-016, VAL-002, VAL-004, VAL-010, BR-008, BR-013, AC-004, AC-007, AC-016, ERR-001, ERR-002, ALT-002]
related_adrs: [ADR-001, ADR-003, ADR-005, ADR-007, ADR-016]
---

# ADR-002 — Atomic multi-ticket reservation

## Context

Al iniciar formalmente una compra deben reservarse todos los Ticket solicitados o ninguno (`FR-004`, `ST-001`, `AC-016`), solo una solicitud concurrente puede ganar un Ticket (`FR-010`, `AC-007`) y la Reservation y la Order deben nacer de forma consistente con los Ticket (`ST-006`, `AC-004`). Si algún Ticket no está disponible no se crea Reservation, Order ni Order ID (`HC-001`, `ALT-002`, `ERR-002`).

`TC-011` preserva la alternativa entre optimistic locking y conditional writes; esta decisión elige y detalla.

## Options considered

### Option A — Conditional writes dentro de una única escritura transaccional

Una sola escritura transaccional de DynamoDB que contiene: una actualización condicional por Ticket (`state = AVAILABLE`), la creación de la Order (con su Reservation), la creación del registro de idempotencia y la creación del registro de auditoría.

- A favor: atomicidad todo-o-nada garantizada por el almacén; la condición expresa exactamente el predicado de negocio; no requiere lectura previa de los Ticket; nunca existe un estado intermedio observable.
- En contra: tamaño de transacción acotado por el servicio (impone `FG-001`); costo de escritura transaccional superior al de escrituras simples; posibles cancelaciones por conflicto entre transacciones concurrentes.

### Option B — Optimistic locking con atributo de versión

Leer cada Ticket, comprobar su estado en la aplicación y escribir condicionando a que la versión no haya cambiado, dentro de una transacción.

- A favor: patrón genérico y conocido; detecta cualquier modificación concurrente.
- En contra: exige una lectura adicional de N items en la ruta caliente (`AC-031`); la condición sobre versión es más débil que la condición sobre estado (falla ante cambios irrelevantes y obliga a releer y reintentar); el predicado real del negocio es el estado, no la versión.

### Option C — Conditional writes independientes por Ticket con compensación

Una escritura condicional por Ticket, sin transacción; si alguna falla, se revierten las exitosas.

- A favor: sin límite de items; menor costo por escritura; sin conflictos transaccionales.
- En contra: durante la ventana de compensación existen Ticket reservados parcialmente, observables por otras solicitudes y por la consulta de disponibilidad, lo que contradice `AC-014` y `AC-016`; un fallo durante la compensación deja reservas huérfanas.

## Decision

Se adopta la **Option A: conditional writes sobre el estado, agrupados en una única escritura transaccional** (`AP-008`).

Contenido de la transacción (N + 3 items):

| Item | Operación | Condición |
|---|---|---|
| Cada Ticket solicitado | Pasa a `RESERVED`, registra `orderId`, sale de `GSI2` | El item existe y `state = AVAILABLE` |
| Order | Creación en `CREATED` con `reservationId`, `reservedAt`, `expiresAt = reservedAt + 10 min`, entrada en `GSI3` | El item no existe |
| Idempotency record | Creación | El item no existe |
| Audit record | Creación (`ST-001`, `ST-006`) | Ninguna |

Reglas:

1. **Revalidación** (`VAL-002`): la condición se evalúa en el momento del commit; no hay lectura previa de Ticket cuyo resultado se confíe.
2. **Mismo Event** (`VAL-010`, `BR-014`): la solicitud identifica un único `eventId` y una lista de `ticketId`; las claves de los Ticket se construyen con ese `eventId`, por lo que mezclar Events es inexpresable.
3. **Ticket duplicados en la solicitud**: se rechazan en la validación de entrada antes de la transacción.
4. **Interpretación del resultado**:
   - Commit: la Order existe en `CREATED` con todos sus Ticket en `RESERVED`.
   - Cancelación por condición del Idempotency record: es una repetición; se resuelve según `ADR-007`.
   - Cancelación por condición de algún Ticket: rechazo síncrono completo, sin Reservation, Order ni Order ID. Se distingue Ticket inexistente de Ticket no disponible usando el valor previo devuelto por la condición fallida; como respaldo, una lectura posterior de los Ticket señalados.
   - Cancelación por conflicto transaccional: se reintenta la transacción completa un máximo de 2 veces con jitter. Si persiste, se responde indisponibilidad temporal sin haber persistido nada (`ADR-016`).
5. **Comprobación previa del Event**: antes de la transacción se lee el Event (`AP-006`, cacheable porque es inmutable tras habilitarse). El Event **no** forma parte de la transacción.
6. No se añade atributo de versión. Toda transición posterior de un Ticket se condiciona a su estado esperado y a su `orderId` (`ADR-005`).

## Rationale

- Cumple `TC-011` eligiendo conditional writes, y cumple `FR-013` y `BR-009` delegando la atomicidad al almacén en lugar de a una compensación.
- La condición `state = AVAILABLE` hace que, de varias transacciones concurrentes sobre un mismo Ticket, a lo sumo una confirme (`AC-007`, `VAL-004`). Las transacciones que comparten solo parte de los Ticket tampoco pueden dejar un subconjunto reservado (`AC-016`).
- Crear la Order, la Reservation y el registro de idempotencia en la misma transacción hace imposible una Order sin Reservation o una Reservation sin Order (`ST-006`, invariantes 6 y 7).
- Mantener el Event fuera de la transacción evita que todas las compras de un mismo Event compitan por un único item, lo que convertiría cada pico en una cascada de conflictos.
- La opción B añade una lectura y no mejora la garantía. La opción C expone parcialidad.

## Consequences

- Existe un máximo técnico de Ticket por Order, que se convierte en regla funcional observable (`ADR-003`, `FG-001`).
- Bajo contención real sobre los mismos Ticket, una parte de las solicitudes recibe conflicto transaccional en lugar de fallo de condición; el reintento acotado añade latencia (`RISK-002`).
- El rechazo por indisponibilidad no deja rastro persistente (coherente con `HC-001`); su repetición se trata en `FG-004`.
- El adaptador de persistencia debe exponer los motivos de cancelación por item; por eso se usa el cliente de bajo nivel (`ADR-020`).
- La comprobación del Event fuera de la transacción es segura porque el Event es inmutable después de habilitado y sus Ticket no son direccionables antes (`ADR-004`).

## Risks and mitigations

| Risk | Mitigation |
|---|---|
| `RISK-002` conflictos transaccionales bajo contención | Transacciones pequeñas (`FG-001`), reintento acotado con jitter, métrica de conflictos, respuesta de indisponibilidad temporal reintentable con la misma idempotency key |
| Clasificación incorrecta entre "no disponible" e "inexistente" | Respaldo por lectura posterior; ambos casos rechazan sin efectos |

## Items to verify

| Item | Qué comprobar | Impacto si difiere |
|---|---|---|
| Límite de items y tamaño por escritura transaccional | Valor vigente (se asume 100 items) | Recalcular el máximo técnico de `ADR-003`; el máximo recomendado de `FG-001` cabe incluso con un límite de 25 |
| Motivos de cancelación por item y devolución del valor previo en condición fallida | Disponibilidad en el SDK, en el servicio y en DynamoDB Local | Si no existe, usar siempre la lectura posterior de respaldo para clasificar |
| Conflictos entre transacciones que comparten un item | Si una comprobación de condición sin escritura también genera conflicto | Confirma la decisión de dejar el Event fuera de la transacción |
| Fidelidad transaccional de DynamoDB Local bajo concurrencia | Que respete la exclusión entre transacciones concurrentes | Si no, las pruebas de concurrencia de integración deben ejecutarse contra AWS (`ADR-019`) |

## Depends on

- `FG-001` (máximo de Ticket por Order).
- `FG-004` (repetición de una solicitud rechazada).
- `FG-005` (compra sobre un Event pasado): define el resultado de la comprobación previa del Event.
