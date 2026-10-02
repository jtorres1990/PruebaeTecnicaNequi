---
id: ADR-001
title: DynamoDB data model
status: PROPOSED
priority: HIGH
source_ids: [TC-004, NFR-006, FR-003, FR-012, FR-002, FR-009, FR-011, BR-012, BR-018, NFR-002]
related_adrs: [ADR-002, ADR-004, ADR-005, ADR-007, ADR-009, ADR-012, ADR-021]
---

# ADR-001 — DynamoDB data model

## Context

`TC-004` fija Amazon DynamoDB como persistencia y `NFR-006` exige acceso rápido. La especificación no define claves, índices ni consistencia (§31). El modelo debe derivarse de los access patterns `AP-001` a `AP-019` (ver `ticketing.data-model.md` §1), que salen de `MF-001` a `MF-004`, del consumidor asíncrono y del proceso de expiración.

Fuerzas principales:

- La unidad de competencia es el Ticket individual (§17 de la especificación): cada Ticket debe ser un item direccionable por clave para admitir una condición propia.
- La disponibilidad se deriva de los estados de los Ticket (`FR-003`, `BR-012`); no existe un contador independiente autoritativo.
- Las transiciones abarcan varios items (Order y N Ticket) y deben ser atómicas (`FR-013`).
- El proceso de expiración necesita encontrar Reservation vencidas sin recorrer la tabla (`FR-011`).

## Options considered

### Option A — Tabla única con claves genéricas y GSIs dispersos

Una tabla `ticketing` con `PK` / `SK` genéricos. Colecciones de items: `EVENT#<eventId>` (metadatos del Event, sus Ticket y su auditoría de creación) y `ORDER#<orderId>` (Order con su Reservation y PaymentAttempt co-localizados, y su auditoría). Tres GSIs dispersos, cada uno justificado por un access pattern.

- A favor: un solo recurso que crear en local y en AWS; las transacciones operan sobre una tabla; `AP-007` y `AP-017` se resuelven con una Query sobre una colección; el diseño demuestra derivación desde access patterns.
- En contra: claves sobrecargadas menos legibles; IAM por tabla no separa entidades; todos los tipos de item comparten configuración de TTL, streams y respaldo.

### Option B — Una tabla por entidad

Tablas `events`, `tickets`, `orders`, `idempotency`, `audit`.

- A favor: esquemas más evidentes; IAM, TTL y respaldo por entidad; menor curva para quien no conoce single-table.
- En contra: cinco recursos que crear y parametrizar en ambos entornos; las transacciones abarcan varias tablas (permitido, pero más superficie de configuración); no aporta ningún access pattern que la opción A no resuelva.

### Option C — Ticket con partition key propia e índice por Event

Cada Ticket con `PK = EVENT#<eventId>#TICKET#<ticketId>` y un GSI por Event para leer la disponibilidad.

- A favor: distribución de escritura ideal en la tabla base; elimina la partición caliente por Event.
- En contra: `AP-007` pasaría a leerse desde un GSI (más costo de escritura por cada cambio de estado y la partición caliente se traslada al índice); se pierde la posibilidad de lectura consistente de la disponibilidad.

## Decision

Se adopta la **Option A**.

1. **Tabla única** `ticketing`, `PK` y `SK` de tipo String. Tipos de item, claves y atributos: `ticketing.data-model.md` §3.
2. **Partition y sort keys**:
   - Event: `EVENT#<eventId>` / `#META`.
   - Ticket: `EVENT#<eventId>` / `TICKET#<ticketId>` (`AP-007`, `AP-008`).
   - Order: `ORDER#<orderId>` / `#META` (`AP-010`); Reservation y PaymentAttempt son atributos del mismo item.
   - Idempotency record: `IDEM#<customerId>#<idempotencyKey>` / `#META` (`AP-009`).
   - Audit record: en la colección de su Order o Event, `SK = AUDIT#...` (`AP-017`).
3. **GSIs** (todos dispersos):
   - `GSI1` events by start — `AP-004`, `AP-018`.
   - `GSI2` available tickets — `AP-005`.
   - `GSI3` active reservations by expiry, con 4 shards de escritura — `AP-016`.
4. **Consistencia por access pattern**: transaccional para `AP-008`, `AP-012`, `AP-014`, `AP-015`; fuerte para `AP-009`, `AP-010`, `AP-017`; eventual para `AP-004`, `AP-005`, `AP-006`, `AP-007`, `AP-016`, `AP-018` (detalle en `ticketing.data-model.md` §2.3).
5. **Modelo de capacidad**: on-demand.
6. **Reservation y PaymentAttempt co-localizados en el item Order**. Siguen siendo conceptos de dominio diferenciados (`CMP-009`); la co-localización es física.

## Rationale

- `AP-007` (lectura caliente, `p95 < 500 ms`) se resuelve con una Query sobre una sola partición de la tabla base, sin índice y con items pequeños.
- `AP-008` necesita una condición por Ticket: cada Ticket es un item con clave determinista `(eventId, ticketId)`, lo que además hace inexpresable una Order con Ticket de Events distintos (`BR-014`, `VAL-010`).
- Co-localizar Reservation y PaymentAttempt en la Order reduce el tamaño de cada transacción en hasta dos items y elimina una clase completa de inconsistencias: la Reservation está activa si y solo si `status = CREATED` (invariantes 6 y 7 de §5.1 de la especificación; `BR-015`, `BR-020`).
- Separar la colección `ORDER#` de la colección `EVENT#` evita que Orders, idempotencia y auditoría escriban en la partición del Event, que ya concentra la contención.
- `GSI3` disperso hace que el costo del proceso de expiración dependa de las Reservation vencidas, no del total de Orders (`AP-016`).
- On-demand responde a la carga impulsiva descrita en §2 de la especificación sin dimensionamiento previo. Capacidad aprovisionada con autoescalado es más barata con carga estable, pero reacciona tarde ante picos.
- La opción B no resuelve ningún access pattern adicional y duplica recursos en ambos entornos. La opción C empeora el costo de escritura de la ruta más caliente.

## Consequences

- Los Ticket de un Event comparten partition key: existe una partición caliente por Event popular (`RISK-001`).
- La lectura de disponibilidad cuesta en proporción a la capacidad del Event (`ADR-021`, `RISK-009`), por lo que el diseño requiere un máximo de capacidad (`FG-002`).
- Los GSIs son eventualmente consistentes: `AP-004`, `AP-005` y `AP-016` pueden ir retrasados respecto de la tabla base. Es aceptable porque la información es informativa (`BR-018`) o porque la escritura posterior está protegida por condición (`ADR-009`).
- Cada cambio de un Ticket hacia o desde `AVAILABLE` genera una escritura en `GSI2`; cada creación o cierre de Order genera una escritura en `GSI3`.
- La auditoría crece en la misma tabla (`RISK-015`).
- No se usa un atributo de versión: el control de concurrencia se basa en condiciones sobre estado y propietario (`ADR-002`).

## Risks and mitigations

| Risk | Mitigation |
|---|---|
| `RISK-001` partición caliente por Event | Items pequeños, on-demand, reintento acotado; ruta de evolución: sharding de la partición del Event por sección o bucket y lectura en paralelo |
| `RISK-009` costo de lectura de disponibilidad | Máximo de capacidad (`FG-002`), lectura eventual; evolución descrita en `ADR-021` |
| `RISK-015` crecimiento de auditoría | Solo inserción, respaldo PITR; evolución: exportación por streams a almacenamiento inmutable (`ADR-012`) |

## Items to verify

| Item | Qué comprobar | Impacto si difiere |
|---|---|---|
| Throughput por partición y división automática de particiones calientes | Límites vigentes y condiciones en las que DynamoDB divide una colección de items | Si el límite es inferior a la escritura esperada sobre un Event, adelantar el sharding de la partición del Event |
| Costo de escritura en GSI bajo transacción | Si las escrituras propagadas a GSIs se facturan al factor transaccional | Solo afecta la estimación de costo |
| Soporte de GSIs dispersos y TTL en DynamoDB Local | Paridad con el servicio | Si TTL no existe en local no hay impacto funcional; si los GSIs difieren, las pruebas de `AP-004`, `AP-005` y `AP-016` deben ejecutarse contra AWS |

## Depends on

- `FG-001` (máximo de Ticket por Order): fija el tamaño máximo de `ticketIds` y de las transacciones.
- `FG-002` (capacidad máxima por Event): acota el tamaño de la colección `EVENT#`.
- `AV-001` (marca de agotado en el listado): si se rechaza, `GSI2` cambia de uso (ver `ADR-021`).
