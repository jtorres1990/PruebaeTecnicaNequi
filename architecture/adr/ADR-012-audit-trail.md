---
id: ADR-012
title: Audit trail
status: PROPOSED
priority: MEDIUM
source_ids: [FR-014, BR-010, NFR-005, AC-015, BR-009, FR-013, EVAL-013]
related_adrs: [ADR-001, ADR-005, ADR-018]
---

# ADR-012 — Audit trail

## Context

El sistema debe conservar evidencia auditable de las transiciones de Ticket y Order (`FR-014`, `BR-010`, `NFR-005`). `AC-015` exige que, completada una transición, exista evidencia consultable. La especificación deja contenido y mecanismo a arquitectura y no define una operación de API para consultar la auditoría.

Hay que decidir qué se registra, dónde, y cómo se garantiza que el registro es consistente con la transición.

## Options considered

### Option A — Registro de auditoría escrito en la misma escritura transaccional que la transición

Cada transición de negocio incluye un item de auditoría de solo inserción, dentro de la misma transacción que modifica la Order y los Ticket.

- A favor: no puede existir transición sin evidencia ni evidencia sin transición; consultable de inmediato y también en local; un registro por transición describe la Order y todos sus Ticket.
- En contra: un item más por transacción; la auditoría vive en la misma tabla que los datos operativos y comparte sus permisos.

### Option B — Captura de cambios del almacén hacia un destino de auditoría

Activar el stream de cambios de la tabla y entregarlo a un almacenamiento externo.

- A favor: captura todo cambio sin código en la ruta de escritura; destino inmutable y separado.
- En contra: la evidencia es asíncrona; el stream registra cambios de items, no la intención de negocio (causa, actor); el soporte en el emulador local es `TO_VERIFY`; añade componentes solo para AWS, lo que rompe la paridad local.

### Option C — Registro de aplicación (logs estructurados)

- A favor: sin costo de almacenamiento en la tabla.
- En contra: no es consistente con la transición (el log puede perderse o emitirse sin commit); no es evidencia consultable por Order.

## Decision

Se adopta la **Option A**. La Option B queda como complemento productivo, no como mecanismo primario.

**Qué se registra por transición** (un registro por transición de negocio):

| Campo | Contenido |
|---|---|
| Transiciones | Identificadores `ST-*` aplicados (por ejemplo `ST-004` y `ST-007`) |
| Order | `orderId`, estado origen y destino |
| Ticket | `eventId`, lista de `ticketId`, estado origen y destino común |
| Causa | Código funcional (reserva creada, pago iniciado, pago aprobado, pago rechazado, fallo de encolado, fallo de procesamiento, expiración) |
| Actor | Tipo (`CUSTOMER`, consumidor de Orders, proceso de expiración) e identificador (sujeto del JWT o identificador de instancia) |
| Correlación | `paymentAttemptId` cuando aplica, identificador de correlación de la solicitud o del mensaje |
| Instante | UTC |

Para la creación de un Event se escribe un registro con las cantidades de Ticket creados en `AVAILABLE` y en `COMPLIMENTARY`, junto con la habilitación (`ADR-004`).

**Dónde**: en la tabla, dentro de la colección de la Order (`SK` con prefijo `AUDIT#`), o del Event para su creación (`ticketing.data-model.md` §3).

**Consistencia**: el registro forma parte de la misma escritura transaccional que la transición (`ADR-005`). Si la transición no se confirma, el registro no existe.

**Inmutabilidad**: los registros solo se insertan; ninguna operación de la aplicación los modifica ni elimina.

**Consulta** (`AP-017`): por Order o por Event, mediante acceso operativo a la tabla. No se expone una operación de API, porque la especificación no la sustenta.

**Datos sensibles**: el registro no contiene tokens ni datos personales más allá del identificador de sujeto.

## Rationale

- La consistencia entre transición y evidencia es exactamente lo que piden `BR-009` y `BR-010` leídas juntas; solo la opción A la da por construcción.
- Un registro por transición de negocio (y no por Ticket) mantiene el tamaño de la transacción en N + 3 items.
- Guardar la auditoría junto a la Order permite reconstruir su historia completa con una sola Query.

## Consequences

- La historia de un Ticket concreto se obtiene filtrando los registros de las Orders que lo incluyeron; no hay un índice por Ticket. Es una limitación declarada.
- La auditoría crece sin límite en la tabla operativa (`RISK-015`).
- Los eventos que no son transiciones (duplicados descartados, reintentos) no generan auditoría; quedan en logs y métricas (`ADR-018`).
- La inmutabilidad es una convención de la aplicación; en AWS se refuerza con permisos y respaldo.

## Risks and mitigations

| Risk | Mitigation |
|---|---|
| `RISK-015` crecimiento e inmutabilidad débil | Respaldo PITR; evolución: stream de cambios hacia almacenamiento inmutable con retención y archivado de registros antiguos |
| Fuga de información en la auditoría | Solo códigos funcionales e identificadores; sin cuerpos de solicitud ni tokens |

## Items to verify

| Item | Qué comprobar | Impacto si difiere |
|---|---|---|
| Streams de cambios en DynamoDB Local | Soporte | Solo relevante para el complemento productivo |

## Depends on

- `FG-003`: registros adicionales de aprobación tardía y de reverso.
