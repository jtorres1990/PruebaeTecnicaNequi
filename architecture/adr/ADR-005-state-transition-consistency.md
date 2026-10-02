---
id: ADR-005
title: Order, Reservation and Ticket consistency per state transition
status: PROPOSED
priority: HIGH
source_ids: [FR-008, FR-013, FR-016, BR-001, BR-006, BR-007, BR-009, BR-013, BR-015, ST-001, ST-002, ST-003, ST-004, ST-005, ST-006, ST-007, ST-008, ST-009, ST-010, AC-011, AC-012, AC-013, AC-014, VAL-005]
related_adrs: [ADR-001, ADR-002, ADR-006, ADR-008, ADR-009, ADR-012]
---

# ADR-005 — Order, Reservation and Ticket consistency per state transition

## Context

Order y Ticket tienen máquinas de estado separadas (`ST-001` a `ST-010`) cuyo resultado conjunto debe ser coherente (invariante 10 de §5.1) y nunca parcial (`FR-013`, `BR-013`, `AC-014`). La especificación prohíbe introducir estados intermedios por conveniencia técnica (§7.1).

Hay que definir, para cada transición, qué escrituras ocurren juntas y qué condiciones las protegen.

## Options considered

### Option A — Una escritura transaccional por transición de negocio, protegida por la Order

Cada transición de negocio es una única escritura transaccional que incluye el item Order, todos sus Ticket y el registro de auditoría. El item Order actúa como guardián: toda transición terminal exige `status = CREATED`.

- A favor: atomicidad real; ningún estado intermedio; un solo ganador entre transiciones terminales concurrentes; la auditoría es consistente por construcción.
- En contra: tamaño de transacción ligado a la cantidad de Ticket (`ADR-003`); costo transaccional.

### Option B — Saga con pasos compensables y estados técnicos intermedios

Actualizar primero la Order a un estado "en transición" y después los Ticket, con compensación.

- A favor: sin límite de tamaño; escrituras simples.
- En contra: requiere estados intermedios de Order, prohibidos por la especificación; expone subconjuntos de Ticket en estados distintos (`AC-014`).

### Option C — Order como fuente única y estado de Ticket derivado

Persistir solo el estado de la Order y derivar el de los Ticket al leer.

- A favor: una sola escritura por transición.
- En contra: la exclusión por Ticket (`FR-010`) necesita un item por Ticket con condición propia; la lectura de disponibilidad (`AP-007`) requeriría cruzar Orders. No es viable con el modelo de `ADR-001`.

## Decision

Se adopta la **Option A**. Fases internas se distinguen con atributos técnicos del item Order (`enqueuedAt`, `paymentAttemptId`, `paymentLease*`), que no forman parte de ninguna máquina de estados funcional.

| Transición de negocio | ST | Escrituras conjuntas | Condiciones de guarda | Ejecutor |
|---|---|---|---|---|
| Reservar y crear Order | `ST-001` + `ST-006` | N Ticket → `RESERVED`; Order creada en `CREATED` con Reservation; Idempotency record; Audit record | Cada Ticket existe y está `AVAILABLE`; Order e Idempotency record no existen | `CMP-005` |
| Iniciar pago | `ST-003` | Order registra PaymentAttempt activo y lease; N Ticket → `PENDING_CONFIRMATION`; Audit record | Order `CREATED`, sin PaymentAttempt, `expiresAt` posterior a ahora más el margen de corte; cada Ticket `RESERVED` y de esta Order | `CMP-007` |
| Confirmar | `ST-004` + `ST-007` | Order → `CONFIRMED`; N Ticket → `SOLD`; Audit record | Order `CREATED`, PaymentAttempt coincide, `expiresAt` posterior a ahora; cada Ticket `PENDING_CONFIRMATION` y de esta Order | `CMP-007` |
| Rechazar | `ST-005` + `ST-008` | Order → `REJECTED` con causa; N Ticket → `AVAILABLE`; Audit record | Order `CREATED`, PaymentAttempt coincide; cada Ticket `RESERVED` o `PENDING_CONFIRMATION` y de esta Order | `CMP-007` |
| Fallar (procesamiento) | `ST-005` + `ST-009` | Order → `FAILED` con causa; N Ticket → `AVAILABLE`; Audit record | Order `CREATED`, identidad del PaymentAttempt igual a la leída; Ticket igual que en Rechazar | `CMP-007` |
| Fallar (encolado) | `ST-005` + `ST-009` | Igual que el anterior | Order `CREATED` y sin PaymentAttempt; Ticket igual que en Rechazar | `CMP-005` |
| Expirar | `ST-002` + `ST-010` | Order → `EXPIRED` con causa; N Ticket → `AVAILABLE`; Audit record | Order `CREATED` y `expiresAt` anterior o igual a ahora; Ticket igual que en Rechazar | `CMP-008` |

Reglas generales:

1. **Guardián único**: toda transición terminal exige `status = CREATED` en la Order. Dos transiciones terminales concurrentes no pueden confirmarse ambas.
2. **Propiedad del Ticket**: toda transición de un Ticket posterior a la reserva exige que su `orderId` sea el de la Order que la ejecuta. Un Ticket liberado y vuelto a reservar por otra Order no puede ser afectado por un mensaje tardío de la primera.
3. **Estados finales** (`BR-006`, `BR-007`): ninguna condición admite `SOLD` ni `COMPLIMENTARY` como origen; ninguna operación posterior a la creación del Event escribe `COMPLIMENTARY`.
4. **Reservation**: no tiene estado persistido propio. Está activa si y solo si la Order está en `CREATED`; queda confirmada, cancelada o expirada según el estado terminal de la Order.
5. **Fallo de una condición**: no es un error técnico. El ejecutor relee la Order y actúa según su estado actual (normalmente, no hacer nada porque otra transición ya ganó).
6. **Condiciones de Ticket defensivas**: si la guarda de la Order se cumple, las de sus Ticket deben cumplirse por invariante. Si una transacción se cancela por un Ticket teniendo la Order en `CREATED`, se registra como inconsistencia y se alerta; no se fuerza ninguna escritura.

## Rationale

- La opción A es la única que cumple a la vez `FR-013`, `AC-014` y la prohibición de estados intermedios.
- Usar la Order como guardián hace que la exclusión entre pago, rechazo, fallo y expiración sea una propiedad del almacén y no de la coordinación entre procesos (`ADR-008`).
- La condición de propiedad sobre el Ticket protege `FR-017` frente a mensajes tardíos o duplicados.

## Consequences

- Todas las transiciones tienen como máximo N + 3 items; el máximo de Ticket por Order es una restricción de diseño (`ADR-003`).
- `DS-006` admite Ticket en `RESERVED` o `PENDING_CONFIRMATION` mientras la Order está en `CREATED`; la distinción se refleja en el atributo técnico `paymentAttemptId`, no en un estado nuevo de Order.
- El puerto de persistencia expone operaciones con intención de negocio (reservar, iniciar pago, confirmar, cerrar con liberación) en lugar de operaciones CRUD por entidad (`ADR-015`).
- `AC-011`: cada Ticket tiene un único atributo `state`, modificado solo por estas operaciones.

## Risks and mitigations

| Risk | Mitigation |
|---|---|
| `RISK-002` conflictos transaccionales | Reintento acotado con jitter en el ejecutor; en el consumidor, el reintento natural de SQS |
| Divergencia de invariantes por defecto de implementación | Condiciones defensivas en Ticket, alerta de inconsistencia y verificación de invariantes al final de la prueba de carga (`ADR-019`) |

## Items to verify

| Item | Qué comprobar | Impacto si difiere |
|---|---|---|
| Aislamiento de la escritura transaccional | Que la escritura transaccional sea serializable respecto de otras escrituras transaccionales y simples | Es la base de la regla de guardián único; si no se cumple, el diseño debe revisarse |
| Interacción entre escritura simple y transacción en curso sobre el mismo item | Comportamiento de la actualización de `enqueuedAt` y del lease | Solo afecta reintentos de escrituras no críticas |

## Depends on

- `FG-001` (tamaño máximo de las transacciones).
- `FG-003` (atributos adicionales al cerrar una Order con un pago de resultado desconocido o aprobado tardíamente).
- `AV-003` (margen de corte y guarda temporal de la confirmación).
