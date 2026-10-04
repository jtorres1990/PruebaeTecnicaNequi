---
id: ADR-040
title: Availability read model over a sharded index with paginated pages and cached count
status: ACCEPTED
priority: MEDIUM
supersedes: ADR-021
human_decision: CONFIRMED_WITH_CHANGE
source_ids: [FR-002, FR-003, FR-012, BR-012, BR-018, BR-022, AC-002, AC-010, AC-030, NFR-002, NFR-004, MF-002]
related_adrs: [ADR-022, ADR-023, ADR-024, ADR-032, ADR-035, ADR-038, ADR-039]
---

# ADR-040 — Availability read model over a sharded index with paginated pages and cached count

Reemplaza a ADR-021.

## Human decision applied

Respuesta humana vinculante a ADR-021 (CONFIRMED_WITH_CHANGE), incorporada íntegramente:

> Se confirma que la disponibilidad se deriva siempre de los Ticket, sin contadores persistidos. Se descartan el contador en el item Event, el contador mantenido fuera de la transacción o por captura de cambios y la instantánea en caché como modelo base. Se rechaza la lectura completa de la colección del Event en una sola respuesta, incompatible con ADR-001 y con la capacidad de 50.000 Ticket aprobada en FG-002.
>
> La fuente de lectura es el GSI disperso de Ticket en `AVAILABLE`, con sharding calculado conforme a ADR-001. La cantidad de shards de cada Event se deriva de su capacidad y se conserva en el Event.
>
> La consulta de disponibilidad de un Event retorna la cantidad de Ticket disponibles y una página de Ticket disponibles. La página es paginada por cursor, filtrable por sección y con tamaño máximo acotado, y solo contiene Ticket en `AVAILABLE`. La respuesta deja de listar los Ticket en otros estados.
>
> La cantidad disponible se calcula mediante un conteo en paralelo sobre los shards del índice, cacheado durante 1 s por instancia y compartido entre solicitudes simultáneas del mismo Event. La respuesta incluye `generatedAt` y se declara informativa conforme a BR-018; la compra revalida en la transacción conforme a ADR-002.
>
> El listado marca los Events agotados mediante un sondeo con límite 1 sobre los shards del índice, que se detiene en el primer resultado, conforme a AV-001. La consulta de un Event pasado sigue respondiendo conforme a FG-005, y un Event no habilitado se trata como inexistente para el `CUSTOMER` conforme a ADR-004. OpenAPI y el modelo de datos deben propagarse de forma consistente con esta decisión.

Correspondencia: ADR-001 → ADR-022, ADR-002 → ADR-023, ADR-004 → ADR-024. Además `AV-001`, `AV-002`, `AV-005`, `FG-002`, `FG-005`; cursor inválido → `VALIDATION_ERROR` (ADR-035); conteo paralelo en el adaptador (ADR-039).

## Context

La disponibilidad comercial se deriva de los Ticket en `AVAILABLE` (`FR-003`, `BR-012`); la consulta es bajo demanda e informativa (`FR-012`, `BR-018`) con `p95 < 500 ms` en la prueba objetivo (`AC-030`); el listado marca los agotados (`AC-002`, `BR-022`). Con 50.000 Ticket por Event y Ticket en partition keys propias (ADR-022), la lectura completa de un Event no es viable.

## Options considered

### Option A — Derivar de los Ticket mediante el GSI disperso de disponibles con sharding: página por cursor y recuento paralelo cacheado

- A favor: coherente con los Ticket por construcción; sin item compartido en la ruta de escritura; respuesta acotada.
- En contra: lectura eventual; recuento con una consulta por shard; caché de 1 s.

### Option B — Contador de disponibles en el item Event actualizado en cada transacción

- En contra: todas las compras de un Event escribirían el mismo item; conflictos en cadena. Descartada.

### Option C — Contador fuera de la transacción o por captura de cambios

- En contra: puede divergir de los Ticket. Descartada.

### Option D — Instantánea en caché como modelo base

- En contra: antigüedad deliberada. Descartada como modelo base; solo se cachea el recuento 1 s.

### Option E — Lectura completa de la colección del Event (diseño de ADR-021)

- En contra: incompatible con ADR-022 y con 50.000 Ticket. Rechazada por la decisión humana.

## Decision

Se adopta la **Option A**.

### Consulta de disponibilidad (`API-003`)

| Aspecto | Decisión |
|---|---|
| Acceso | `ADMIN` o `CUSTOMER` (`AV-005`) |
| Event no `ENABLED` | `EVENT_NOT_FOUND` para cualquier rol; el `ADMIN` consulta el aprovisionamiento por `API-006` (ADR-024, `AV-002`) |
| Event pasado | Responde de forma informativa (`FG-005`) |
| Metadatos | Event desde caché en memoria de Events `ENABLED` (inmutables): nombre, lugar, `startsAt`, capacidad, secciones de la definición |
| Cantidad disponible | `availableCount`: recuento en paralelo sobre los `availabilityShards` del Event en `GSI2` (`AP-021`), cacheado 1 s por instancia y por Event, con una única consulta en curso compartida por las solicitudes simultáneas; `generatedAt` es el instante del recuento |
| Página | Solo Ticket en `AVAILABLE` (`AP-020`): `ticketId`, sección, fila, asiento; tamaño por defecto 50, máximo 100 |
| Filtro | Parámetro `section` opcional; debe ser un código de sección de la definición, si no `VALIDATION_ERROR` |
| Orden y cursor | Recorre los shards en orden 0..S−1 y, dentro de cada shard, por sección, fila y asiento. El cursor opaco codifica versión, filtro, shard y última clave; se valida estrictamente y uno inválido o de otro Event o filtro produce `VALIDATION_ERROR`. El orden es estable por cursor pero no global por sección |
| Ticket en otros estados | No se listan (`AC-010`: solo `AVAILABLE` es disponibilidad comercial) |
| Carácter | `informative: true`; la compra revalida en la transacción (ADR-023, `BR-018`) |
| Consistencia | Eventual (GSI) |

### Listado (`API-002`)

- Query de `GSI1` bajo `EVENTS#ENABLED` con `startsAt` posterior al instante actual, paginada por cursor (`BR-022`).
- Por cada Event de la página, sondeo con límite 1 sobre sus shards de `GSI2` (`AP-005`), en oleadas de 4 shards en orden aleatorio, que se detiene en el primer resultado; en paralelo entre Events con concurrencia acotada.
- Indicador `soldOut` por Event (`AV-001`); sin cantidad exacta.
- Un Event no `ENABLED` no aparece.

### Sin contadores persistidos

Ningún contador de disponibilidad se persiste. El caché de 1 s del recuento es memoria por instancia, no estado.

## Rationale

- Derivar de los Ticket es la única opción coherente sin contención ni divergencia (`NFR-004`).
- El recuento paralelo con caché de 1 s y consulta compartida acota el coste bajo carga concentrada sobre un mismo Event.
- La paginación por cursor acota la respuesta independientemente de la capacidad.

## Consequences

- La respuesta ya no informa estados distintos de `AVAILABLE` (cambio de contrato, `ticketing.functional-clarifications.v1.md`).
- `availableCount` puede tener hasta 1 s más la latencia del índice de antigüedad.
- El sondeo de un Event agotado consulta todos sus shards (`RISK-023`).

## Risks and mitigations

| Risk | Mitigation |
|---|---|
| `RISK-009` coste y latencia de la disponibilidad | Página acotada, recuento cacheado y compartido, proyección mínima |
| `RISK-019` recuento costoso con muchos shards o caché ineficaz con muchas instancias | Límite de 32 shards; recuento solo de items KEYS/INCLUDE pequeños; métrica de aciertos de caché |
| `RISK-023` sondeo de agotados | Parada en el primer resultado; listado paginado |

## Items to verify

| Item | Qué comprobar | Impacto si difiere |
|---|---|---|
| Recuento con selección de solo cantidad y paginación por tamaño leído | Comportamiento y coste en el servicio y en DynamoDB Local | Ajustar la concurrencia del recuento |
| Latencia de página y recuento para un Event de 50.000 Ticket bajo la carga objetivo | Medición local y en AWS | Ajustar shards, tamaño de página o duración del caché |

## Depends on

- `AV-001`, `AV-002`, `FG-002`, `FG-005` (resueltos).
