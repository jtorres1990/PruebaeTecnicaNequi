---
id: ADR-022
title: DynamoDB data model with per-Ticket partitioning and sharded indexes
status: ACCEPTED
priority: HIGH
supersedes: ADR-001
human_decision: CONFIRMED_WITH_CHANGE
source_ids: [TC-004, NFR-006, FR-003, FR-012, FR-002, FR-009, FR-011, BR-012, BR-018, BR-021, NFR-001, NFR-002, VAL-008, VAL-010]
related_adrs: [ADR-003, ADR-008, ADR-023, ADR-024, ADR-025, ADR-026, ADR-027, ADR-028, ADR-031, ADR-032, ADR-040]
---

# ADR-022 — DynamoDB data model with per-Ticket partitioning and sharded indexes

Reemplaza a ADR-001. El registro autoritativo de estados es `architecture/adr/ticketing.adr-registry.v1.md`.

## Human decision applied

Respuesta humana vinculante a ADR-001 (CONFIRMED_WITH_CHANGE), incorporada íntegramente:

> Se conserva una sola tabla DynamoDB, pero se rechaza que todos los Ticket de un Event compartan `PK = EVENT#<eventId>`. Cada Ticket debe utilizar una partition key de alta cardinalidad y direccionable de forma determinista, por ejemplo `PK = TICKET#<eventId>#<ticketId>` y `SK = #META`. Esto distribuye entre particiones las compras concurrentes de tickets distintos de un mismo Event.
>
> La disponibilidad por Event debe resolverse mediante un GSI con sharding calculado. El shard se deriva de forma determinista del ticket y el Event conserva la cantidad de shards configurada. Las consultas de disponibilidad deben ser paginadas y, cuando aplique, filtrables por sección; no se retornarán decenas de miles de tickets en una sola respuesta.
>
> La reserva all-or-nothing se mantiene mediante una única operación transaccional que puede actualizar Ticket ubicados en particiones diferentes, además de crear Order, Reservation, idempotencia y auditoría. Cada Ticket debe validar condicionalmente que pertenece al Event solicitado y que continúa en `AVAILABLE`.
>
> El diseño debe soportar como mínimo un Event con 50.000 Ticket y una prueba concentrada con más de 1.000 usuarios concurrentes. La creación de Events de gran capacidad debe realizar la escritura de tickets por lotes antes de habilitar el Event. El modelo, ADR-004, ADR-021 y OpenAPI deben propagarse de forma consistente con esta decisión.

Decisiones aprobadas relacionadas que este ADR incorpora: capacidad máxima de 50.000 Ticket (`FG-002`), creación asíncrona con ciclo `PROVISIONING` / `ENABLED` / `FAILED` (ADR-024, sucesor de ADR-004), Orders pendientes de encolado con índice disperso y sharding (ADR-026), item de bloqueo de Order activa por cliente y Event (ADR-032), idempotencia de creación de Events (ADR-027), atributo técnico de cuarentena (ADR-025), índice de reversos de pago (`FG-003`, ADR-008) y disponibilidad paginada con cantidad disponible (ADR-040, sucesor de ADR-021).

## Context

`TC-004` fija Amazon DynamoDB y `NFR-006` exige acceso rápido. El modelo se deriva de los access patterns de `ticketing.data-model.v2.md` §1, que salen de `MF-001` a `MF-004`, del consumidor de Orders, del aprovisionamiento asíncrono de Events, del barrido de republicación, del proceso de reversos y del proceso de expiración.

Fuerzas:

- La unidad de competencia es el Ticket individual (§17 de la especificación). Bajo una prueba concentrada de más de 1.000 usuarios sobre un único Event de 50.000 Ticket, concentrar todos los Ticket de un Event en una partición (diseño de ADR-001) crea una partición caliente en la ruta de compra.
- La disponibilidad se deriva de los Ticket (`FR-003`, `BR-012`) sin contador autoritativo (ADR-040).
- Las transiciones abarcan Order, N Ticket, auditoría y bloqueo, y deben ser atómicas (`FR-013`).
- Leer 50.000 Ticket en una respuesta es incompatible con `AC-030`.

## Options considered

### Option A — Tabla única, Ticket en la colección del Event (diseño de ADR-001)

`PK = EVENT#<eventId>`, `SK = TICKET#<ticketId>`; disponibilidad por Query de la colección.

- A favor: lectura de la disponibilidad desde la tabla base; posibilidad de lectura consistente.
- En contra: partición caliente por Event en la ruta de compra; coste de lectura proporcional a la capacidad. Rechazada por la decisión humana.

### Option B — Tabla única, Ticket con partition key propia y disponibilidad por GSI disperso con sharding calculado

`PK = TICKET#<eventId>#<ticketId>`, `SK = #META`. Los Ticket en `AVAILABLE` se indexan en un GSI cuya partition key incluye un shard derivado del Ticket.

- A favor: las compras de Ticket distintos del mismo Event escriben en particiones distintas de la tabla base; las escrituras de índice se reparten entre shards; la disponibilidad se pagina sin leer Ticket en otros estados.
- En contra: la disponibilidad es eventualmente consistente (GSI); el recuento requiere una consulta por shard; cada cambio hacia o desde `AVAILABLE` escribe en el índice.

### Option C — Una tabla por entidad

Tablas `events`, `tickets`, `orders`, `idempotency`, `audit`, `locks`.

- A favor: esquemas evidentes; IAM, TTL y respaldo por entidad.
- En contra: más recursos que crear y parametrizar en ambos entornos; no resuelve ningún access pattern que la Option B no resuelva; la partición del Ticket sigue requiriendo las mismas claves.

## Decision

Se adopta la **Option B**. Detalle de items, atributos y condiciones: `ticketing.data-model.v2.md`.

1. **Tabla única** `ticketing`, `PK` y `SK` String, on-demand, TTL sobre `ttl` (solo registros de idempotencia), cifrado y PITR en AWS.
2. **Claves por tipo de item**:

   | Item | PK | SK |
   |---|---|---|
   | Event | `EVENT#<eventId>` | `#META` |
   | Ticket | `TICKET#<eventId>#<ticketId>` | `#META` |
   | Order (con Reservation y PaymentAttempt) | `ORDER#<orderId>` | `#META` |
   | Idempotencia de compra | `IDEM#<customerId>#<idempotencyKey>` | `#META` |
   | Idempotencia de creación de Event | `IDEMEVT#<adminSubject>#<idempotencyKey>` | `#META` |
   | Bloqueo de Order activa | `ACTIVE#<customerId>#<eventId>` | `#META` |
   | Auditoría de Order | `ORDER#<orderId>` | `AUDIT#<occurredAt>#<code>#<suffix>` |
   | Auditoría de Event | `EVENT#<eventId>` | `AUDIT#<occurredAt>#<code>#<suffix>` |

3. **Pertenencia al Event**: el `eventId` forma parte de la clave del Ticket y además se guarda como atributo; toda transición de reserva condiciona `eventId = <solicitado>` y `state = AVAILABLE` (`VAL-010`).
4. **Identidad determinista**: el `ticketId` lo genera el servidor a partir de la definición compacta del inventario (`<sección>-<fila>-<asiento>`, ADR-024). Cualquier componente puede reconstruir todas las claves de Ticket de un Event a partir de su definición, sin índice por Event.
5. **GSIs** (todos dispersos):

   | Índice | Partition key | Sort key | Proyección | Uso |
   |---|---|---|---|---|
   | `GSI1` events by lifecycle | `EVENTS#PROVISIONING` / `EVENTS#ENABLED` / `EVENTS#FAILED` | `createdAt#eventId` / `startsAt#eventId` / `failedAt#eventId` | INCLUDE de metadatos (sin la definición del inventario) | AP-004, AP-018, AP-027 |
   | `GSI2` available tickets | `AVAIL#<eventId>#<shard>` | `<section>#<row>#<seat4>` | INCLUDE `ticketId`, `section`, `row`, `seat` | AP-005, AP-020, AP-021 |
   | `GSI3` work index | `RESV#<shard>`, `REVERSAL#<shard>`, `REVERSAL#EXHAUSTED`, `REVIEW#QUARANTINE` | instante o próximo intento `#orderId` | KEYS_ONLY | AP-016, AP-029, AP-031 |
   | `GSI4` pending enqueue | `PENDQ#<shard>` | `createdAtMs#orderId` | KEYS_ONLY | AP-028 |

6. **Sharding calculado**:
   - `GSI2`: `shard = hash(ticketId) mod availabilityShards`; `availabilityShards = min(32, max(1, ceil(capacity / 2000)))`, calculado al crear el Event y **conservado en el Event** (25 shards para 50.000 Ticket). Los parámetros son configurables; el valor de un Event no cambia después de creado.
   - `GSI3` rango de expiración y `GSI4`: 8 shards, `hash(orderId) mod 8`. `GSI3` rango de reversos: 4 shards.
   - La función de hash es estable y está fijada en el dominio (ADR-034), no depende de la plataforma.
7. **GSI3 como índice de trabajo con rangos lógicos disjuntos**: el rango `RESV#` es el índice de expiración (Orders en `CREATED` no en cuarentena), el rango `REVERSAL#` es el índice de reversos de pago (Orders terminales con reverso pendiente, conforme a ADR-008 aceptado y `FG-003`), y `REVIEW#QUARANTINE` lista las Orders en cuarentena para revisión manual. Una Order está como máximo en un rango a la vez. Salir del rango `RESV#` equivale a salir del índice de expiración (ADR-028).
8. **Consistencia por access pattern**: transaccional para AP-001, AP-003, AP-008, AP-012, AP-014, AP-015, AP-026, AP-030, AP-031, AP-032; fuerte para AP-009, AP-010, AP-017, AP-022, AP-023, AP-025; eventual para AP-004, AP-005, AP-006, AP-016, AP-018, AP-020, AP-021, AP-028, AP-029 (los GSI solo admiten lectura eventual).
9. **Capacidad**: on-demand. Preparación de throughput antes de una venta masiva como mejora productiva (ADR-037).
10. **Reservation y PaymentAttempt** siguen co-localizados en el item Order (`BR-015`, `BR-020`).

## Rationale

- La partition key por Ticket elimina la contención por partición entre compras de Ticket distintos del mismo Event; la contención que queda es la funcionalmente inevitable: dos compras del mismo Ticket.
- La transacción multi-partición conserva la atomicidad de ADR-023 sin depender de la co-localización.
- El sharding de `GSI2` reparte las escrituras de índice que produce cada cambio hacia o desde `AVAILABLE`; conservar la cantidad de shards en el Event hace que la lectura conozca exactamente qué particiones consultar.
- La identidad determinista evita un índice "Ticket por Event" para verificar, aprovisionar o purgar el inventario: las claves se recalculan desde la definición.
- La proyección INCLUDE de `GSI1` evita copiar la definición del inventario al índice de listado.

## Consequences

- La consulta de disponibilidad es eventual y paginada (ADR-040); ya no puede leerse con consistencia fuerte.
- El recuento de disponibles cuesta una consulta por shard (25 para 50.000 Ticket); se cachea 1 s (ADR-040).
- El sondeo de agotado de un Event agotado recorre todos sus shards (RISK-023).
- La transacción de reserva tiene N + 4 items (Ticket, Order, idempotencia, auditoría, bloqueo): 14 con el máximo de 10 de ADR-003. Las transiciones terminales tienen N + 3 = 13. Ver la observación de consolidación en ADR-025.
- Cinco tipos de escritura de índice en la ruta de compra (`GSI2` por Ticket, `GSI3`, `GSI4`) aumentan el coste por compra (trade-off aceptado en ADR-026).

## Risks and mitigations

| Risk | Mitigation |
|---|---|
| `RISK-001` partición caliente | Ticket con partition key propia; shards en `GSI2`, `GSI3` y `GSI4`; on-demand; preparación de throughput en AWS |
| `RISK-009` coste de disponibilidad | Paginación, recuento por shards con caché de 1 s, sin lectura de Ticket no disponibles (ADR-040) |
| `RISK-023` sondeo de agotado de coste proporcional a los shards | Parada en el primer resultado; límite de 32 shards; listado paginado |
| `RISK-015` crecimiento de la auditoría | Solo inserción, PITR; captura de cambios a almacenamiento inmutable como mejora productiva (ADR-031) |

## Items to verify

| Item | Qué comprobar | Impacto si difiere |
|---|---|---|
| Throughput por partición de tabla y de GSI | Límites vigentes (se asume del orden de 1.000 escrituras y 3.000 lecturas por segundo por partición) | Ajustar el divisor de `availabilityShards` y los shards de `GSI3`/`GSI4` |
| Escritura en GSI ante actualizaciones de atributos no proyectados | Si se factura escritura de índice cuando no cambian claves ni atributos proyectados | Solo coste |
| Número máximo de GSIs por tabla | Se asume 20 | Se usan 4; sin impacto salvo límite inferior |
| Tamaño máximo de item | Se asume 400 KB | Acota la definición compacta del inventario guardada en el Event (ADR-024) |
| Soporte de GSIs dispersos y TTL en DynamoDB Local | Paridad con el servicio | Si difiere, las pruebas de AP-005, AP-016, AP-020, AP-028 se ejecutan contra AWS |

## Depends on

- `FG-002` (resuelto: 50.000 Ticket por Event).
- `AV-001` (resuelto: indicador `soldOut`).
