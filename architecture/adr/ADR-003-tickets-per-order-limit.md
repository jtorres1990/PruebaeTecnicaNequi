---
id: ADR-003
title: Tickets per Order limit
status: PROPOSED
priority: HIGH
source_ids: [TC-011, FR-004, FR-013, BR-013, BR-014, ST-001, AC-004, AC-014, EVAL-008]
related_adrs: [ADR-002, ADR-005, ADR-013]
---

# ADR-003 — Tickets per Order limit

## Context

La especificación define que una Order solicita "uno o más" Ticket (`BR-014`) sin máximo. `ADR-002` y `ADR-005` resuelven cada transición con una única escritura transaccional que incluye todos los Ticket de la Order más hasta tres items adicionales. El servicio limita la cantidad de items por transacción (`TO_VERIFY`, se asume 100).

El mecanismo impone por tanto un máximo técnico, y un máximo es una regla funcional observable: una solicitud que lo supere debe recibir una respuesta definida. Eso es un vacío funcional (`FG-001`).

## Options considered

### Option A — Máximo explícito y pequeño, validado en la entrada

Fijar un máximo de negocio muy por debajo del límite técnico y rechazar en la validación de entrada cualquier solicitud que lo supere.

- A favor: transacciones pequeñas (menor costo y menor probabilidad de conflicto); el límite sobrevive a cambios del límite técnico; reduce el acaparamiento de inventario por solicitud (`EVAL-008`).
- En contra: introduce una regla funcional que la especificación no contiene; requiere aprobación humana.

### Option B — Máximo igual al límite técnico

Permitir hasta el máximo que quepa en una transacción (límite del servicio menos 3).

- A favor: mínima restricción al usuario.
- En contra: acopla una regla de negocio a un límite de servicio no verificado; transacciones grandes aumentan costo y conflictos; una sola solicitud puede retener decenas de Ticket durante diez minutos.

### Option C — Sin máximo, dividiendo la reserva en varias transacciones

Reservar por bloques y compensar si un bloque falla.

- A favor: sin límite funcional.
- En contra: reintroduce reserva parcial observable, contraria a `AC-014`, `AC-016` y `BR-013`. Se descarta por violar la especificación.

## Decision

Se adopta la **Option A**.

- Máximo recomendado: **10 Ticket por Order**, configurable (`FG-001`).
- Mínimo: 1 (`BR-014`).
- Una solicitud con más Ticket que el máximo, con cero Ticket o con identificadores repetidos se rechaza en la validación de entrada con error de validación, sin crear Reservation, Order ni Order ID y sin tocar el inventario.
- El valor se valida en el punto de entrada HTTP y de nuevo en el caso de uso (`CMP-005`), para que la regla no dependa del adaptador.

## Rationale

- Con 10 Ticket la transacción más grande tiene 13 items: cabe con amplio margen bajo el límite asumido y también bajo un límite de 25, de modo que la regla no depende del dato `TO_VERIFY`.
- Un máximo pequeño mantiene baja la probabilidad de conflicto entre transacciones que comparten Ticket (`RISK-002`) y acota el costo de cada escritura.
- Limita el inventario que una sola solicitud puede retener durante la vigencia de la Reservation (`RISK-010`).

## Consequences

- La API expone el máximo en su contrato (`API-004` en `ticketing.openapi.yaml`).
- La lista `ticketIds` de la Order y de los registros de auditoría tiene tamaño acotado, lo que acota el tamaño de los items.
- Si el humano fija un máximo distinto, solo cambia un parámetro mientras `máximo + 3` no supere el límite del servicio. Si lo supera, `ADR-002` y `ADR-005` deben revisarse.
- Compras de grupos grandes requieren varias Orders independientes, sin atomicidad entre ellas. Es una limitación declarada.

## Risks and mitigations

| Risk | Mitigation |
|---|---|
| El máximo aprobado supera el límite técnico | Verificar el límite antes de aceptar un valor superior a 22 |
| `RISK-010` acaparamiento mediante muchas Orders pequeñas | Rate limiting por identidad (`ADR-013`); un límite de Reservation activas por CUSTOMER sería una regla funcional nueva y no se introduce |

## Items to verify

| Item | Qué comprobar | Impacto si difiere |
|---|---|---|
| Límite de items por escritura transaccional | Valor vigente en el servicio y en DynamoDB Local | Máximo técnico = límite − 3; no afecta el valor recomendado |

## Depends on

- `FG-001`: valor del máximo y comportamiento ante una solicitud que lo supere.
