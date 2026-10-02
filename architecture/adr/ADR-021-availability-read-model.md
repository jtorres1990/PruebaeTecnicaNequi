---
id: ADR-021
title: Availability and inventory read model
status: PROPOSED
priority: MEDIUM
source_ids: [FR-002, FR-003, FR-012, BR-012, BR-018, BR-022, AC-002, AC-010, AC-030, NFR-002, NFR-004, MF-002]
related_adrs: [ADR-001, ADR-002, ADR-004]
---

# ADR-021 — Availability and inventory read model

## Context

La disponibilidad comercial se deriva de los Ticket en `AVAILABLE` y el inventario no es un contador independiente (`FR-003`, `BR-012`, modelo de dominio §5). La consulta de disponibilidad es bajo demanda, informativa y debe cumplir `p95 < 500 ms` en la prueba objetivo (`FR-012`, `BR-018`, `AC-030`). El listado de Events debe mostrar los agotados "con disponibilidad cero" (`AC-002`, `BR-022`).

Hay que decidir de dónde se lee la disponibilidad y cómo se marca un Event agotado sin comprometer la ruta de escritura.

## Options considered

### Option A — Derivar siempre de los Ticket: lectura de la colección del Event, más un sondeo de agotado por índice disperso

Disponibilidad de un Event: Query de todos sus Ticket y cálculo de totales por estado en la aplicación. Listado: por cada Event de la página, un sondeo de coste constante sobre `GSI2` que indica si existe al menos un Ticket `AVAILABLE`.

- A favor: la disponibilidad es coherente con los Ticket por construcción; la ruta de escritura no gana ningún item compartido; el listado tiene coste constante por Event.
- En contra: el coste de la consulta de disponibilidad crece con la capacidad del Event; el listado informa agotado o no agotado, no la cantidad exacta.

### Option B — Contador de disponibles en el item Event, actualizado en cada transacción

- A favor: lectura de coste constante y cantidad exacta en el listado.
- En contra: todas las transacciones de un mismo Event escribirían el mismo item; bajo la carga objetivo producirían conflictos transaccionales en cadena sobre el Event más demandado, justo donde importa `AC-031`.

### Option C — Contador mantenido fuera de la transacción o por captura de cambios

- A favor: sin contención en la ruta de escritura.
- En contra: el contador puede divergir de los Ticket (actualización perdida o duplicada), lo que contradice que el inventario sea coherente con los Ticket; la captura de cambios añade componentes y su soporte local es `TO_VERIFY`.

### Option D — Instantánea en caché de corta vida

- A favor: reduce drásticamente las lecturas bajo carga.
- En contra: introduce una antigüedad deliberada en lo que la especificación llama disponibilidad vigente; complica las pruebas de `AC-002` y `AC-010`. Se deja como evolución, no como diseño base.

## Decision

Se adopta la **Option A**.

1. **Consulta de disponibilidad** (`API-003`, `AP-006`, `AP-007`):
   - Metadatos del Event desde caché en memoria de Events habilitados (inmutables).
   - Query de la colección del Event restringida a Ticket, con lectura eventual y paginación interna.
   - La respuesta contiene cada Ticket con su estado y un resumen de totales por estado. Solo `AVAILABLE` se informa como disponibilidad comercial (`BR-012`, `AC-010`).
   - La respuesta no se pagina hacia el cliente; el tamaño queda acotado por la capacidad máxima (`FG-002`).
   - El contrato declara que la respuesta es informativa y no garantiza adquisición (`BR-018`).
2. **Listado de Events** (`API-002`, `AP-004`, `AP-005`):
   - Query de `GSI1` por Events habilitados con fecha posterior al instante actual, paginada por cursor (`BR-022`).
   - Por cada Event de la página, sondeo sobre `GSI2` con límite 1, en paralelo y con concurrencia acotada.
   - Cada Event se informa con un indicador `soldOut` (`AV-001`).
3. **Consistencia**: lectura eventual en ambas. La compra siempre revalida en el commit (`ADR-002`, `VAL-002`).
4. **Sin contadores persistidos** de disponibilidad.

## Rationale

- Derivar de los Ticket es la única opción que mantiene la coherencia de inventario sin añadir contención ni riesgo de divergencia (`NFR-004`).
- El sondeo por índice disperso resuelve el caso costoso del listado, que es precisamente el Event agotado: sin índice habría que recorrer todos sus Ticket para concluir que ninguno está disponible.
- La lectura eventual es coherente con `BR-018` y reduce el coste de la lectura más frecuente.

## Consequences

- El coste y la latencia de `API-003` son proporcionales a la capacidad del Event (`RISK-009`); por eso se necesita `FG-002`.
- El listado no informa la cantidad exacta de Ticket disponibles; esa cifra está en `API-003` (`AV-001`).
- `GSI2` añade una escritura de índice por cada Ticket que entra o sale de `AVAILABLE`.
- La respuesta de disponibilidad expone el estado de cada Ticket, incluido `COMPLIMENTARY`.

## Risks and mitigations

| Risk | Mitigation |
|---|---|
| `RISK-009` coste y latencia de la lectura de disponibilidad en Events grandes | Capacidad máxima (`FG-002`), items de Ticket pequeños, lectura eventual, compresión de la respuesta; evolución: Option D o partición del Event por secciones |
| Listado retrasado respecto del inventario real | Aceptado por `BR-018`; la compra revalida |

## Items to verify

| Item | Qué comprobar | Impacto si difiere |
|---|---|---|
| Latencia de la Query de un Event de capacidad máxima en DynamoDB Local y en AWS | Medición bajo la carga objetivo | Si no cumple `AC-030`, reducir el máximo de `FG-002` o adoptar la Option D |
| Tamaño máximo de página de una Query | Valor vigente | Solo cambia el número de páginas internas |

## Depends on

- `AV-001`: indicador `soldOut` en el listado en lugar de cantidad exacta.
- `FG-002`: capacidad máxima por Event.
- `FG-005`: comportamiento de la consulta sobre un Event pasado.
