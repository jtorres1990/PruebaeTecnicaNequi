---
id: ADR-031
title: Audit trail with extended cause catalog and declared immutability limits
status: ACCEPTED
priority: MEDIUM
supersedes: ADR-012
human_decision: CONFIRMED_WITH_CHANGE
source_ids: [FR-014, BR-010, NFR-005, AC-015, BR-009, FR-013, EVAL-013]
related_adrs: [ADR-008, ADR-022, ADR-024, ADR-025, ADR-026, ADR-037]
---

# ADR-031 — Audit trail with extended cause catalog and declared immutability limits

Reemplaza a ADR-012.

## Human decision applied

Respuesta humana vinculante a ADR-012 (CONFIRMED_WITH_CHANGE), incorporada íntegramente:

> Se confirma un registro de auditoría de solo inserción por transición de negocio, escrito en la misma escritura transaccional que la transición y guardado en la colección de la Order, o del Event para su creación. El registro contiene las transiciones aplicadas, los estados de origen y destino de la Order y de sus Ticket, la causa funcional, el actor, la correlación y el instante en UTC, sin tokens ni datos personales más allá del identificador de sujeto. No se expone una operación de API de consulta. Se descartan la captura de cambios como mecanismo primario y los logs estructurados como evidencia. Se añaden dos cambios.
>
> Catálogo de causas. Además de las causas originales, se auditan: la solicitud de reverso de un pago y su confirmación o agotamiento (FG-003), la aprobación tardía que no pudo aplicarse (ADR-008), la puesta en cuarentena de una Order (ADR-005) y el aprovisionamiento fallido de un Event (ADR-004). La republicación del barrido de ADR-006 no es una transición de negocio y queda en logs y métricas.
>
> Inmutabilidad. Dentro de la tabla, la inmutabilidad es una convención de la aplicación respaldada por PITR. Como los registros comparten partition key con su Order, los permisos IAM no pueden impedir su modificación sin impedir también la actualización de la Order. La garantía fuerte de inmutabilidad en AWS se declara como mejora de producción mediante la captura de cambios hacia almacenamiento con bloqueo de objetos y retención.

Decisiones aprobadas incorporadas: transiciones y escrituras de ADR-025; ciclo de aprovisionamiento de ADR-024; barrido de ADR-026; mejora productiva en ADR-037.

## Context

`FR-014`, `BR-010` y `NFR-005` exigen evidencia auditable de las transiciones; `AC-015` exige que exista tras cada transición. La especificación no define una operación de consulta.

## Options considered

### Option A — Registro de solo inserción en la misma escritura transaccional

- A favor: no hay transición sin evidencia ni evidencia sin transición; paridad local.
- En contra: un item más por transacción; comparte tabla y permisos con los datos operativos.

### Option B — Captura de cambios como mecanismo primario

- En contra: evidencia asíncrona; registra cambios de items, no intención; soporte local `TO_VERIFY`. Queda como mejora productiva.

### Option C — Logs estructurados como evidencia

- En contra: no consistentes con la transición. Descartada.

## Decision

Se adopta la **Option A**.

### Contenido del registro

Transiciones `ST-*` aplicadas; `orderId` y estados de origen y destino de la Order; `eventId`, `ticketIds` y estados de origen y destino de los Ticket; causa funcional; actor (tipo e identificador: sujeto del JWT, instancia del worker o proceso); correlación (`correlationId`, `paymentAttemptId` cuando aplica); instante UTC. Sin tokens ni datos personales más allá del identificador de sujeto.

### Catálogo de causas

| Causa | Item | Escrito con | Origen |
|---|---|---|---|
| `RESERVATION_CREATED` | Order | `AP-008` | `ST-001`, `ST-006` |
| `PAYMENT_STARTED` | Order | `AP-012` | `ST-003` |
| `PAYMENT_APPROVED` | Order | `AP-014` | `ST-004`, `ST-007` |
| `PAYMENT_DECLINED` | Order | `AP-015` | `ST-005`, `ST-008` |
| `ENQUEUE_FAILED` | Order | `AP-015` | `ST-005`, `ST-009` |
| `PROCESSING_FAILED` | Order | `AP-015` | `ST-005`, `ST-009` |
| `RESERVATION_EXPIRED` | Order | `AP-015` | `ST-002`, `ST-010` |
| `PAYMENT_REVERSAL_REQUESTED` | Order | En la misma transacción de cierre que marca el reverso (incluido en el registro de esa transición) | `FG-003` |
| `PAYMENT_REVERSAL_CONFIRMED` | Order | `AP-030` | `FG-003` |
| `PAYMENT_REVERSAL_EXHAUSTED` | Order | `AP-030` variante | `FG-003` |
| `LATE_APPROVAL_NOT_APPLIED` | Order | `AP-032`, o dentro de la transición Expirar cuando el consumidor la ejecuta tras una aprobación tardía | ADR-008 |
| `ORDER_QUARANTINED` | Order | `AP-031` | ADR-025 |
| `EVENT_PROVISIONING_REQUESTED` | Event | `AP-001` | ADR-024 |
| `EVENT_ENABLED` (con cantidades `AVAILABLE` y `COMPLIMENTARY`) | Event | `AP-003` | ADR-024, `FR-001` |
| `EVENT_PROVISIONING_FAILED` | Event | `AP-026` | ADR-024 |

No se auditan como transición: la republicación del barrido, los duplicados descartados, los reintentos, los cambios de lease ni el progreso de lotes; quedan en logs y métricas.

### Dónde y cómo

- Colección de la Order (`ORDER#<orderId>`, `SK = AUDIT#...`) o del Event (`EVENT#<eventId>`).
- Inserción condicionada a que el item no exista.
- Consulta operativa por Order o Event (`AP-017`); sin operación de API.

### Inmutabilidad

- En la tabla: convención de la aplicación (solo inserción) respaldada por PITR. IAM no puede separar la escritura de la auditoría de la escritura de la Order porque comparten partition key; se declara.
- En AWS, mejora productiva: captura de cambios de la tabla hacia almacenamiento con bloqueo de objetos y retención (ADR-037).

## Rationale

- Solo la opción A garantiza la consistencia entre transición y evidencia (`BR-009` con `BR-010`).
- Ampliar el catálogo da evidencia de los mecanismos añadidos en esta revisión sin auditar actividad técnica.

## Consequences

- Historia de un Ticket concreto: filtrando las auditorías de las Orders que lo incluyeron; sin índice por Ticket.
- La auditoría crece en la tabla operativa (`RISK-015`).

## Risks and mitigations

| Risk | Mitigation |
|---|---|
| `RISK-015` crecimiento e inmutabilidad por convención | PITR; mejora productiva con almacenamiento inmutable |
| Fuga de información | Solo códigos e identificadores |

## Items to verify

| Item | Qué comprobar | Impacto si difiere |
|---|---|---|
| Captura de cambios de DynamoDB hacia almacenamiento con bloqueo de objetos | Mecanismo y configuración vigentes | Solo la mejora productiva |

## Depends on

- `FG-003` (resuelto).
