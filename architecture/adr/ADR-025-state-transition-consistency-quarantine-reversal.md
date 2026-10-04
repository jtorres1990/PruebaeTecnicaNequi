---
id: ADR-025
title: Order, Reservation and Ticket consistency per state transition, with quarantine and payment reversal marking
status: ACCEPTED
priority: HIGH
supersedes: ADR-005
human_decision: CONFIRMED_WITH_CHANGE
source_ids: [FR-008, FR-013, FR-015, FR-016, FR-017, BR-001, BR-003, BR-006, BR-007, BR-009, BR-013, BR-015, ST-001, ST-002, ST-003, ST-004, ST-005, ST-006, ST-007, ST-008, ST-009, ST-010, AC-011, AC-012, AC-013, AC-014, AC-019, VAL-005]
related_adrs: [ADR-003, ADR-008, ADR-022, ADR-023, ADR-026, ADR-028, ADR-029, ADR-030, ADR-031, ADR-032, ADR-034]
---

# ADR-025 — Order, Reservation and Ticket consistency per state transition, with quarantine and payment reversal marking

Reemplaza a ADR-005.

## Human decision applied

Respuesta humana vinculante a ADR-005 (CONFIRMED_WITH_CHANGE), incorporada íntegramente:

> Se confirma que cada transición de negocio se ejecuta como una única operación `TransactWriteItems` que incluye la Order, todos sus Ticket y el registro de auditoría, sin estados de negocio nuevos ni estados técnicos intermedios observables. Se descartan la saga y el estado de Ticket derivado.
>
> Toda transición terminal exige condicionalmente que la Order esté en `CREATED`. Toda transición de un Ticket posterior a la reserva exige que el Ticket pertenezca a la Order que la ejecuta y se encuentre en el estado de origen esperado. Ninguna condición admite `SOLD` ni `COMPLIMENTARY` como origen. El fallo de una condición no es un error técnico: el ejecutor relee la Order y actúa según su estado actual.
>
> Las transacciones deben usar las claves de Ticket aprobadas en ADR-001: los Ticket de una Order residen en partition keys diferentes y la transacción los actualiza junto con la Order. Con el máximo de 10 Ticket por Order de ADR-003, ninguna transición supera 13 items.
>
> La atomicidad se garantiza para la escritura. Una lectura no transaccional de varios items puede observar unos antes y otros después del commit, por lo que la consulta de una Order debe resolverse leyendo únicamente el item Order con lectura fuertemente consistente y no debe componer el estado individual de sus Ticket. La consulta de disponibilidad se mantiene como informativa.
>
> Si la transacción se cancela por la condición de un Ticket mientras la Order continúa en `CREATED`, se trata como una inconsistencia: se audita, se alerta y no se fuerza ninguna escritura sobre los Ticket. Para evitar que el proceso de expiración la reintente indefinidamente, la Order debe marcarse con un atributo técnico de cuarentena que la retire del índice de expiración y la deje disponible para revisión manual. Este atributo no es un estado de negocio y no se expone.
>
> Se acepta como trade-off declarado que las escrituras transaccionales consumen el doble de capacidad y que una compra confirmada requiere tres transacciones. Los márgenes temporales de las guardas se resuelven en AV-003 y ADR-008, y el tratamiento de un pago aprobado sin confirmar en FG-003.

Decisiones aprobadas incorporadas: claves por Ticket (ADR-022); bloqueo de Order activa creado en la reserva y eliminado en toda transición terminal "conforme a ADR-005" (ADR-032, sucesor de ADR-013); guardas temporales (`AV-003`, ADR-008); marca de reverso de pago en la misma transacción de cierre (`FG-003`); salida del índice de pendientes de encolado (ADR-026); salida del índice de expiración en cuarentena (ADR-028); catálogo de auditoría (ADR-031).

### Observación de consolidación sobre el número de items

La respuesta humana a ADR-005 calcula "ninguna transición supera 13 items" con el conjunto de items vigente en la versión 1. La respuesta humana a ADR-013, aprobada en la misma revisión y que remite expresamente a ADR-005, añade a la transacción de reserva el item de bloqueo de Order activa. Con ello la reserva (`ST-001` + `ST-006`) tiene 14 items con el máximo de 10 Ticket; las transiciones terminales tienen 13 y el inicio de pago 12.

Se interpreta la cifra de 13 como una cota derivada, no como una regla independiente: su propósito es asegurar que toda transición cabe holgadamente en el límite de `TransactWriteItems` (`TO_VERIFY`, se asume 100 items), propósito que se mantiene con 14. No se modifica ninguna regla normativa de ADR-005 ni de ADR-013 y no se reduce el máximo de ADR-003. Esta interpretación se declara en `ticketing.architecture.v2.md` §14 y en el registro de ADR.

## Context

Order y Ticket tienen máquinas de estado separadas (`ST-001` a `ST-010`) cuyo resultado conjunto debe ser coherente y nunca parcial (`FR-013`, `BR-013`, `AC-014`). No se admiten estados intermedios por conveniencia técnica (§7.1). Con ADR-022 los Ticket de una Order están en partition keys distintas.

## Options considered

### Option A — Una escritura transaccional por transición de negocio, protegida por la Order

- A favor: atomicidad real entre particiones; un solo ganador entre transiciones terminales; auditoría consistente por construcción.
- En contra: coste transaccional; tamaño ligado a N.

### Option B — Saga con pasos compensables y estados técnicos intermedios

- A favor: escrituras simples.
- En contra: estados intermedios prohibidos; subconjuntos observables. Rechazada.

### Option C — Order como fuente única con estado de Ticket derivado

- A favor: una escritura por transición.
- En contra: la exclusión por Ticket necesita un item por Ticket con condición propia. Rechazada.

## Decision

Se adopta la **Option A**.

### Transiciones

`N` = Ticket de la Order (1..10). "Libera" significa: Ticket → `AVAILABLE`, reentra en `GSI2` con su shard, se elimina `orderId`.

| Transición | ST | Escrituras conjuntas | Guardas | Items máx. | Ejecutor |
|---|---|---|---|---|---|
| Reservar y crear Order | `ST-001` + `ST-006` | N Ticket → `RESERVED`; Order `CREATED` con Reservation, en `GSI3` `RESV#` y `GSI4`; idempotencia; auditoría; bloqueo creado | Ticket existe, `eventId` coincide, `AVAILABLE`; Order, idempotencia, auditoría y bloqueo no existen | 14 | `CMP-005` |
| Iniciar pago | `ST-003` | Order registra PaymentAttempt y lease; N Ticket → `PENDING_CONFIRMATION`; auditoría | Order `CREATED`, sin PaymentAttempt, sin cuarentena, `expiresAt > ahora + margen de corte (15 s)`; cada Ticket `RESERVED` y de esta Order | 12 | `CMP-007` |
| Confirmar | `ST-004` + `ST-007` | Order → `CONFIRMED`, sale de `GSI3`/`GSI4`; N Ticket → `SOLD`; auditoría; bloqueo eliminado | Order `CREATED`, sin cuarentena, PaymentAttempt coincide, `expiresAt > ahora`; cada Ticket `PENDING_CONFIRMATION` y de esta Order | 13 | `CMP-007` |
| Rechazar | `ST-005` + `ST-008` | Order → `REJECTED` (`PAYMENT_DECLINED`); libera N Ticket; auditoría; bloqueo eliminado | Order `CREATED`, sin cuarentena, PaymentAttempt coincide; cada Ticket `RESERVED` o `PENDING_CONFIRMATION` y de esta Order | 13 | `CMP-007` |
| Fallar (procesamiento) | `ST-005` + `ST-009` | Order → `FAILED` (`PROCESSING_FAILED`); marca de reverso si aplica; libera N Ticket; auditoría; bloqueo eliminado | Order `CREATED`, sin cuarentena, identidad del PaymentAttempt igual a la leída; Ticket como en Rechazar | 13 | `CMP-007` |
| Fallar (encolado) | `ST-005` + `ST-009` | Order → `FAILED` (`PROCESSING_UNAVAILABLE`); libera N Ticket; auditoría; bloqueo eliminado | Order `CREATED` y sin PaymentAttempt; Ticket como en Rechazar | 13 | `CMP-005` |
| Expirar | `ST-002` + `ST-010` | Order → `EXPIRED` (`RESERVATION_EXPIRED`); marca de reverso si aplica; libera N Ticket; auditoría; bloqueo eliminado | Order `CREATED`, sin cuarentena, `expiresAt <= ahora`; Ticket como en Rechazar | 13 | `CMP-008`, `CMP-007` |

Toda transición terminal elimina las entradas de `GSI3` rango `RESV#`, de `GSI4` y el lease. La eliminación del bloqueo lleva la condición "no existe o pertenece a esta Order", que nunca bloquea una transición válida.

### Reglas generales

1. **Guardián único**: toda transición terminal exige `status = CREATED`.
2. **Propiedad del Ticket**: toda transición posterior a la reserva exige `orderId` igual a la Order ejecutora y el estado de origen esperado.
3. **Estados finales** (`BR-006`, `BR-007`): ninguna condición admite `SOLD` ni `COMPLIMENTARY` como origen; `COMPLIMENTARY` solo se escribe en el aprovisionamiento (ADR-024).
4. **Reservation**: activa si y solo si la Order está en `CREATED` (y no en cuarentena).
5. **Fallo de condición**: no es un error técnico; el ejecutor relee la Order y actúa según su estado.
6. **Lectura de la Order** (`API-005`, `AP-010`): solo el item Order con lectura fuertemente consistente; no compone el estado de sus Ticket.
7. **Disponibilidad**: informativa (ADR-040).

### Cuarentena

- **Cuándo**: una transacción cuyo guardián de Order se cumpliría (Order en `CREATED`) se cancela por la condición de un Ticket. Lo detecta el ejecutor al releer la Order tras la cancelación y analizar el motivo por item.
- **Escritura** (`AP-031`, 2 items): Order recibe `quarantinedAt` y `quarantineReason`, sale del rango `RESV#` de `GSI3` y de `GSI4`, y entra en `GSI3` rango `REVIEW#QUARANTINE`; auditoría `ORDER_QUARANTINED`. Condición: Order `CREATED` y sin cuarentena. No se escribe sobre los Ticket.
- **Efectos**: el proceso de expiración, el barrido de republicación y el consumidor no la tocan (guardas "sin cuarentena"); se emite alarma; la Order conserva su bloqueo hasta la revisión manual (ADR-032). El atributo no se expone en `API-005`: el `CUSTOMER` sigue viendo `CREATED`.
- La cuarentena es un atributo técnico, no un estado de negocio (§7.1 de la especificación).

### Marca de reverso de pago (`FG-003`, ADR-008)

- **Cuándo se marca**: la transición terminal que cierra sin confirmar una Order con PaymentAttempt cuyo resultado fue aprobado o es desconocido (Expirar con PaymentAttempt; Fallar por agotamiento de reintentos con resultado desconocido). No se marca en Rechazar (resultado conocido no aprobado), en Fallar por error definitivo del proveedor (resultado conocido no aprobado) ni en Fallar (encolado) (sin PaymentAttempt).
- **Escritura**: en la misma transacción terminal, la Order recibe `paymentReversalPending`, `paymentReversalRequestedAt` y entra en `GSI3` rango `REVERSAL#<shard>`.
- **Aprobación tardía sobre una Order ya terminal** (`AP-032`, 2 items): auditoría `LATE_APPROVAL_NOT_APPLIED` y, si la Order aún no tiene reverso pendiente ni completado, se marca. Condición: Order terminal distinta de `CONFIRMED` y PaymentAttempt coincidente.
- **Proceso de reversos** (`CMP-024`, rol `worker`, planificación propia cada 10 s, ADR-028; pausado con el circuito del Payment Mock abierto, ADR-035):
  1. Consulta `GSI3` rango `REVERSAL#<shard>` con próximo intento vencido (`AP-029`).
  2. Solicita al Payment Mock la cancelación del `paymentAttemptId` (idempotente, válida en cualquier estado del intento, ADR-030).
  3. Confirmada: `AP-030` (2 items) retira la marca y la entrada del índice, registra `paymentReversalCompletedAt`; auditoría `PAYMENT_REVERSAL_CONFIRMED`.
  4. Fallo transitorio: incrementa `paymentReversalAttempts` y fija el próximo intento con backoff (10 s, 30 s, 1 min, 2 min, 5 min y luego cada 10 min), condición "reverso pendiente".
  5. Agotamiento (10 intentos, configurable): `AP-030` variante, la Order pasa al rango `REVERSAL#EXHAUSTED`, auditoría `PAYMENT_REVERSAL_EXHAUSTED`, alarma; queda pendiente para revisión manual.
- La marca y su resultado son atributos técnicos; la Order conserva su estado terminal y una aprobación tardía nunca la reabre ni la confirma.

## Rationale

- La opción A es la única que cumple `FR-013`, `AC-014` y la prohibición de estados intermedios.
- La Order como guardián hace la exclusión entre pago, rechazo, fallo y expiración una propiedad del almacén.
- Marcar el reverso en la misma transacción de cierre garantiza que ninguna Order cerrada con un pago posiblemente aprobado quede sin reverso pendiente.
- La cuarentena convierte una inconsistencia en un caso acotado y visible, sin reintentos infinitos.

## Consequences

- Una compra confirmada requiere tres transacciones (reserva, inicio de pago, confirmación) y una escritura simple (`enqueuedAt`); las transaccionales consumen el doble de capacidad (trade-off aceptado).
- Una Order en cuarentena retiene sus Ticket y su bloqueo hasta la revisión manual (`RISK-018`).
- Un reverso agotado deja un cobro sin compra pendiente de revisión manual (`RISK-020`).

## Risks and mitigations

| Risk | Mitigation |
|---|---|
| `RISK-002` conflictos transaccionales | Reintento acotado con jitter; en el consumidor, reentrega de SQS |
| `RISK-018` Orders en cuarentena | Alarma, índice de revisión, invariantes posteriores a la carga (ADR-038) |
| `RISK-020` reversos agotados | Backoff largo, alarma, índice `REVERSAL#EXHAUSTED`, revisión manual |
| Divergencia de invariantes | Condiciones defensivas, cuarentena, verificación de invariantes tras la carga |

## Items to verify

| Item | Qué comprobar | Impacto si difiere |
|---|---|---|
| Aislamiento serializable de `TransactWriteItems` frente a escrituras simples y transaccionales, también entre particiones | Comportamiento documentado y en DynamoDB Local | Base de la regla de guardián único; si no se cumple, revisar el diseño |
| Límite de items de `TransactWriteItems` | Se asume 100 | 14 cabe con margen |

## Depends on

- `FG-001`, `FG-003` (resueltos); `AV-003` (resuelto).
