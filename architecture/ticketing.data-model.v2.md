---
artifact: data-model
schema_version: 1.0
feature: ticketing-event-processing
version: 2
supersedes: architecture/ticketing.data-model.md
source:
  feature_spec: feature-spec/ticketing.feature-spec.v4.md
  human_review: human-review/ticketing.architecture-review.yaml
  architecture: architecture/ticketing.architecture.v2.md
status: READY_FOR_DEVELOPMENT
generated_at: 2026-10-04
---

# Data Model v2 — Ticketing Event Processing

Decisiones que gobiernan este documento: [ADR-022](adr/ADR-022-dynamodb-data-model-ticket-partitioning.md), [ADR-023](adr/ADR-023-atomic-multi-ticket-reservation-cross-partition.md), [ADR-024](adr/ADR-024-asynchronous-event-provisioning.md), [ADR-025](adr/ADR-025-state-transition-consistency-quarantine-reversal.md), [ADR-026](adr/ADR-026-persistence-plus-enqueue-with-republish-sweep.md), [ADR-027](adr/ADR-027-idempotency-purchase-event-creation.md), [ADR-028](adr/ADR-028-expiration-process-isolated-scheduling.md), [ADR-031](adr/ADR-031-audit-trail-extended-catalog.md), [ADR-032](adr/ADR-032-security-active-order-lock.md), [ADR-040](adr/ADR-040-availability-read-model-sharded-paginated.md), y [ADR-003](adr/ADR-003-tickets-per-order-limit.md) y [ADR-008](adr/ADR-008-payment-versus-expiration-race.md) aceptados.

Este documento reemplaza a `ticketing.data-model.md` (versión 1, que permanece intacto). Los valores numéricos son valores configurables desplegados en esta implementación, no garantías de capacidad. Los límites de servicio están marcados `TO_VERIFY`.

## 1. Access patterns

Se conservan los IDs de la versión 1 cuando el significado no cambia. `AP-007` y `AP-019` se retiran porque su significado cambió; sus sucesores son `AP-020` y `AP-027`. Ningún ID se reutiliza.

Frecuencia: `hot` = ruta de cada solicitud bajo carga; `warm` = frecuente pero no por compra; `cold` = esporádico u operativo.

| ID | Description | Component | R/W | Frequency | Consistency | Latency target | Source IDs |
|---|---|---|---|---|---|---|---|
| AP-001 | Crear el Event en `PROVISIONING` junto con su registro de idempotencia de creación y su auditoría | CMP-003, CMP-010 | W | cold | Transacción | Dentro de la respuesta 202 | MF-001, FR-001, FR-017, AC-001 |
| AP-002 | Escribir un lote de Ticket de un Event (`AVAILABLE` / `COMPLIMENTARY`) | CMP-022, CMP-010 | W | cold | Escritura por lotes idempotente por clave | Asíncrono | FR-001, FR-003, BR-016, BR-021 |
| AP-003 | Habilitar el Event (`ENABLED`) con su auditoría tras verificar el inventario | CMP-022, CMP-010 | W | cold | Transacción | Asíncrono | FR-001, FR-003, AC-001, VAL-009 |
| AP-004 | Listar Events `ENABLED` futuros ordenados por fecha | CMP-004, CMP-010 | R | warm | Eventual | p95 < 500 ms (objetivo de prueba) | MF-002, FR-002, BR-022, AC-002 |
| AP-005 | Determinar si un Event tiene al menos un Ticket `AVAILABLE` (sondeo por shards) | CMP-004, CMP-010 | R | warm | Eventual | Incluido en AP-004 | FR-002, BR-012, AC-002, AV-001 |
| AP-006 | Obtener metadatos y definición de un Event por ID | CMP-004, CMP-005, CMP-010 | R | hot | Eventual, cacheable si `ENABLED` | Incluido en AP-008 / AP-020 | MF-002, MF-003, VAL-010 |
| AP-007 | **Retirado** (lectura de todos los Ticket de un Event). Sucesor: AP-020 | — | — | — | — | — | — |
| AP-008 | Reservar N Ticket, crear Order con su Reservation, idempotencia, auditoría y bloqueo de Order activa, todo atómicamente | CMP-005, CMP-010 | W | hot | Transacción | p95 < 1 s para la operación completa (AC-031) | MF-003, FR-004, FR-006, FR-010, ST-001, ST-006, VAL-002, VAL-010, AC-004, AC-007, AC-016 |
| AP-009 | Obtener el registro de idempotencia de compra | CMP-005, CMP-010 | R | hot | Fuerte | Incluido en AP-008 | FR-017, BR-019 |
| AP-010 | Obtener una Order por ID | CMP-006, CMP-007, CMP-008, CMP-023, CMP-024, CMP-010 | R | hot | Fuerte | p95 < 500 ms (objetivo de prueba) | MF-004, FR-009, VAL-011, AC-006, AC-027 |
| AP-011 | Marcar `enqueuedAt` y retirar la Order del índice de pendientes | CMP-005, CMP-023, CMP-010 | W | hot | Escritura condicional, best effort | Incluido en AP-008 | FR-005, AC-003 |
| AP-012 | Iniciar pago: PaymentAttempt activo y Ticket a `PENDING_CONFIRMATION` | CMP-007, CMP-010 | W | hot | Transacción | Asíncrono | FR-015, ST-003, BR-020, AC-005 |
| AP-013 | Reclamar el lease de un PaymentAttempt activo | CMP-007, CMP-010 | W | cold | Escritura condicional | Asíncrono | FR-017, BR-020, AC-024 |
| AP-014 | Confirmar: Order `CONFIRMED`, Ticket `SOLD`, bloqueo eliminado | CMP-007, CMP-010 | W | hot | Transacción | Asíncrono | FR-008, ST-004, ST-007, AC-019 |
| AP-015 | Cerrar con liberación: `REJECTED` / `FAILED` / `EXPIRED`, Ticket `AVAILABLE`, bloqueo eliminado, marca de reverso si aplica | CMP-005, CMP-007, CMP-008, CMP-010 | W | warm | Transacción | Asíncrono (síncrono en fallo de encolado) | FR-011, FR-016, ST-002, ST-005, ST-008, ST-009, ST-010, AC-008, AC-020, AC-021, AC-022, FG-003 |
| AP-016 | Encontrar Reservation activas vencidas | CMP-008, CMP-010 | R | warm | Eventual; la condición de AP-015 da la corrección | Por ciclo | FR-011, VAL-003, AC-008, AC-009 |
| AP-017 | Leer la auditoría de una Order o de un Event | Operación, CMP-010 | R | cold | Fuerte opcional | Sin objetivo | FR-014, AC-015 |
| AP-018 | Encontrar Events en `PROVISIONING` sin progreso más allá del umbral | CMP-015, CMP-010 | R | cold | Eventual | Por ciclo | FR-001, ADR-024 |
| AP-019 | **Retirado** (eliminar un Event incompleto y sus Ticket). Sucesor: AP-027 | — | — | — | — | — | — |
| AP-020 | Obtener una página de Ticket `AVAILABLE` de un Event, por cursor y filtrable por sección | CMP-004, CMP-010 | R | hot | Eventual (informativa) | p95 < 500 ms (AC-030) | MF-002, FR-012, BR-012, BR-018, AC-010, FG-002 |
| AP-021 | Contar Ticket `AVAILABLE` de un Event (paralelo por shards, caché 1 s) | CMP-004, CMP-010 | R | hot | Eventual (informativa) | Incluido en AP-020 | FR-012, BR-012, AC-010, AC-030 |
| AP-022 | Obtener el registro de idempotencia de creación de Event | CMP-003, CMP-010 | R | cold | Fuerte | Dentro de la respuesta | FR-001, FR-017 |
| AP-023 | Obtener el estado de aprovisionamiento de un Event | CMP-003, CMP-010 | R | cold | Fuerte | Sin objetivo | FR-001, AV-002, AV-005 |
| AP-024 | Tomar o renovar el lease de aprovisionamiento y registrar progreso antes de cada lote | CMP-022, CMP-010 | W | cold | Escritura condicional | Asíncrono | FR-001, ADR-024 |
| AP-025 | Verificar la existencia y el estado inicial de todos los Ticket de un Event | CMP-022, CMP-010 | R | cold | Fuerte (lectura por lotes) | Asíncrono | FR-003, BR-021, VAL-009, FG-002 |
| AP-026 | Marcar el Event `FAILED` con su auditoría | CMP-022, CMP-015, CMP-010 | W | cold | Transacción | Asíncrono | FR-001, ADR-024 |
| AP-027 | Purgar los Ticket de un Event `FAILED` y marcar la purga | CMP-015, CMP-010 | W | cold | Escritura por lotes + condicional | Asíncrono | FR-001, FR-003 |
| AP-028 | Encontrar Orders en `CREATED` sin `enqueuedAt` con antigüedad > 30 s | CMP-023, CMP-010 | R | warm | Eventual | Por ciclo | FR-005, ALT-006, ADR-026 |
| AP-029 | Encontrar Orders con reverso de pago pendiente cuyo próximo intento venció | CMP-024, CMP-010 | R | cold | Eventual | Por ciclo | FR-015, FG-003 |
| AP-030 | Completar, reprogramar o agotar un reverso de pago | CMP-024, CMP-010 | W | cold | Transacción (completar, agotar) / condicional (reprogramar) | Asíncrono | FR-014, FG-003 |
| AP-031 | Poner una Order en cuarentena con su auditoría | CMP-007, CMP-008, CMP-010 | W | cold | Transacción | Asíncrono | FR-013, FR-014, ADR-025 |
| AP-032 | Registrar una aprobación tardía no aplicada y asegurar la marca de reverso | CMP-007, CMP-010 | W | cold | Transacción | Asíncrono | FR-014, FR-015, ADR-008, FG-003 |
| AP-033 | Registrar la republicación de un aprovisionamiento estancado | CMP-015, CMP-010 | W | cold | Escritura condicional | Por ciclo | FR-001, ADR-024 |

## 2. Tables and indexes

### 2.1 Tabla

| Elemento | Valor |
|---|---|
| Nombre lógico | `ticketing` (nombre físico parametrizado por entorno) |
| Claves | `PK` (String), `SK` (String) |
| Capacidad | On-demand |
| TTL | Atributo `ttl` (epoch seconds), solo en registros de idempotencia |
| Cifrado / PITR | Habilitados en AWS |

### 2.2 Índices secundarios globales

Todos dispersos: un item solo aparece mientras posee el atributo de clave del índice.

| Índice | Partition key | Sort key | Proyección | Items presentes | Justificado por |
|---|---|---|---|---|---|
| `GSI1` events by lifecycle | `GSI1PK` | `GSI1SK` | INCLUDE: `entityType`, `eventId`, `name`, `venue`, `startsAt`, `startsAtMs`, `capacity`, `availabilityShards`, `provisioningStatus`, `createdAt`, `lastProgressAtMs`, `provisioningRepublishCount` | Events `PROVISIONING`, `ENABLED`, y `FAILED` no purgados | AP-004, AP-018, AP-027 |
| `GSI2` available tickets (sharded) | `GSI2PK` | `GSI2SK` | INCLUDE: `ticketId`, `section`, `row`, `seat` | Solo Ticket en `AVAILABLE` | AP-005, AP-020, AP-021 |
| `GSI3` work index | `GSI3PK` | `GSI3SK` | KEYS_ONLY | Orders en `CREATED` no en cuarentena (rango `RESV#`); Orders terminales con reverso pendiente o agotado (rangos `REVERSAL#`); Orders en cuarentena (`REVIEW#QUARANTINE`) | AP-016, AP-029, AP-031 |
| `GSI4` pending enqueue | `GSI4PK` | `GSI4SK` | KEYS_ONLY | Orders en `CREATED`, sin `enqueuedAt` y sin cuarentena | AP-028 |

Valores de clave:

| Índice | Rango | `PK` del índice | `SK` del índice |
|---|---|---|---|
| `GSI1` | Aprovisionando | `EVENTS#PROVISIONING` | `<createdAt>#<eventId>` |
| `GSI1` | Habilitados | `EVENTS#ENABLED` | `<startsAt>#<eventId>` |
| `GSI1` | Fallidos pendientes de purga | `EVENTS#FAILED` | `<failedAt>#<eventId>` |
| `GSI2` | Disponibles | `AVAIL#<eventId>#<shard>` con `shard = hash(ticketId) mod availabilityShards` | `<section>#<row>#<seat con 4 dígitos>` |
| `GSI3` | Expiración | `RESV#<shard>` con `shard = hash(orderId) mod 8` | `<expiresAtMs, 13 dígitos>#<orderId>` |
| `GSI3` | Reversos activos | `REVERSAL#<shard>` con `shard = hash(orderId) mod 4` | `<paymentReversalNextAttemptAtMs, 13 dígitos>#<orderId>` |
| `GSI3` | Reversos agotados | `REVERSAL#EXHAUSTED` | `<paymentReversalExhaustedAt>#<orderId>` |
| `GSI3` | Revisión de cuarentena | `REVIEW#QUARANTINE` | `<quarantinedAt>#<orderId>` |
| `GSI4` | Pendientes de encolado | `PENDQ#<shard>` con `shard = hash(orderId) mod 8` | `<createdAtMs, 13 dígitos>#<orderId>` |

`availabilityShards = min(32, max(1, ceil(capacity / 2000)))`, calculado al crear el Event y conservado en él (25 para 50.000 Ticket). La función `hash` es estable y la define el dominio (ADR-034).

### 2.3 Modo de consistencia por access pattern

| Modo | Access patterns |
|---|---|
| Transacción (escritura transaccional con condiciones) | AP-001, AP-003, AP-008, AP-012, AP-014, AP-015, AP-026, AP-030 (completar, agotar), AP-031, AP-032 |
| Escritura condicional simple | AP-011, AP-013, AP-024, AP-027 (marca final), AP-030 (reprogramar), AP-033 |
| Escritura por lotes sin condición | AP-002, AP-027 (borrado de Ticket) |
| Lectura fuertemente consistente | AP-009, AP-010, AP-017, AP-022, AP-023, AP-025 |
| Lectura eventual (incluye todos los GSI) | AP-004, AP-005, AP-006, AP-016, AP-018, AP-020, AP-021, AP-028, AP-029 |

## 3. Item types

Atributos `*At` en ISO-8601 UTC; atributos `*Ms` en epoch milliseconds (Number), usados en condiciones.

| Entity | PK | SK | Attributes | Notes |
|---|---|---|---|---|
| Event | `EVENT#<eventId>` | `#META` | `entityType`, `eventId`, `name`, `venue`, `startsAt`, `startsAtMs`, `capacity`, `availableAtCreation`, `complimentaryCount`, `inventoryDefinition` (secciones, filas, asientos por fila, rangos de cortesía), `availabilityShards`, `totalBatches`, `provisioningStatus` (`PROVISIONING` / `ENABLED` / `FAILED`), `provisioningLeaseOwner`, `provisioningLeaseUntilMs`, `provisionedBatches`, `lastProgressAtMs`, `provisioningRepublishCount`, `provisioningFailureReason`, `createdAt`, `createdBy`, `enabledAt`, `failedAt`, `ticketsPurgedAt`, `GSI1PK`, `GSI1SK` | `provisioningStatus` es el ciclo técnico aprobado en `AV-002`, no un estado de Ticket ni de Order. Inmutable salvo el ciclo y el lease mientras está en `PROVISIONING`; totalmente inmutable desde `ENABLED`. Tamaño acotado por los límites de la definición (ADR-024) |
| Ticket | `TICKET#<eventId>#<ticketId>` | `#META` | `entityType`, `eventId`, `ticketId`, `section`, `row`, `seat`, `state`, `orderId` (solo `RESERVED`, `PENDING_CONFIRMATION`, `SOLD`), `updatedAt`, `GSI2PK`, `GSI2SK` (solo `AVAILABLE`) | `ticketId = <section>-<row>-<seat>`, generado por el servidor. `state` toma exactamente uno de DS-001..DS-005 (`BR-001`, `VAL-005`) |
| Order (con Reservation y PaymentAttempt) | `ORDER#<orderId>` | `#META` | Order: `entityType`, `orderId`, `customerId`, `eventId`, `ticketIds`, `status`, `failureCause`, `createdAt`, `createdAtMs`, `updatedAt`, `terminalAt`. Reservation: `reservationId`, `reservedAt`, `expiresAt`, `expiresAtMs`. Encolado: `enqueuedAt`. PaymentAttempt: `paymentAttemptId`, `paymentAttemptNo`, `paymentStartedAt`, `paymentOutcome`, `paymentCompletedAt`, `paymentProviderRef`, `paymentLeaseOwner`, `paymentLeaseUntilMs`. Reverso: `paymentReversalPending`, `paymentReversalRequestedAt`, `paymentReversalAttempts`, `paymentReversalNextAttemptAtMs`, `paymentReversalCompletedAt`, `paymentReversalExhaustedAt`. Cuarentena: `quarantinedAt`, `quarantineReason`. Índices: `GSI3PK`, `GSI3SK`, `GSI4PK`, `GSI4SK` | `status` toma uno de DS-006..DS-010. La Reservation está activa si y solo si `status = CREATED` y no hay cuarentena. Encolado, PaymentAttempt técnico, reverso y cuarentena son atributos técnicos fuera de las máquinas de estado funcionales; `API-005` no los expone |
| Idempotencia de compra | `IDEM#<customerId>#<idempotencyKey>` | `#META` | `entityType`, `orderId`, `requestHash`, `createdAt`, `ttl` | Creado en AP-008; vigencia mínima 24 h |
| Idempotencia de creación de Event | `IDEMEVT#<adminSubject>#<idempotencyKey>` | `#META` | `entityType`, `eventId`, `requestHash`, `createdAt`, `ttl` | Creado en AP-001; vigencia mínima 24 h |
| Bloqueo de Order activa | `ACTIVE#<customerId>#<eventId>` | `#META` | `entityType`, `orderId`, `customerId`, `eventId`, `createdAt` | Creado en AP-008; eliminado en toda transición terminal; sin TTL (se conserva con una Order en cuarentena) |
| Auditoría de Order | `ORDER#<orderId>` | `AUDIT#<occurredAt>#<code>#<suffix>` | `entityType`, `transitionIds`, `orderFrom`, `orderTo`, `ticketFrom`, `ticketTo`, `ticketIds`, `eventId`, `cause`, `actorType`, `actorId`, `paymentAttemptId`, `correlationId`, `occurredAt` | Solo inserción; catálogo de causas en ADR-031 |
| Auditoría de Event | `EVENT#<eventId>` | `AUDIT#<occurredAt>#<code>#<suffix>` | `entityType`, `eventId`, `cause`, `capacity`, `availableCount`, `complimentaryCount`, `actorType`, `actorId`, `correlationId`, `occurredAt` | `EVENT_PROVISIONING_REQUESTED`, `EVENT_ENABLED`, `EVENT_PROVISIONING_FAILED` |

## 4. Access pattern resolution

| AP ID | Operation | Table / index | Key condition |
|---|---|---|---|
| AP-001 | Escritura transaccional de 3 items | Tabla base | Event `EVENT#<id>`/`#META`; `IDEMEVT#<sub>#<key>`/`#META`; auditoría `EVENT#<id>`/`AUDIT#...` |
| AP-002 | Escritura por lotes (hasta 4 solicitudes en paralelo de 25 items, `TO_VERIFY`), reintentando no procesados | Tabla base | `TICKET#<eventId>#<ticketId>`/`#META` generados desde la definición |
| AP-003 | Escritura transaccional de 2 items | Tabla base | Event `#META`; auditoría del Event |
| AP-004 | Query paginada ascendente | `GSI1` | `GSI1PK = EVENTS#ENABLED AND GSI1SK > <now ISO>` |
| AP-005 | Query con límite 1 por shard, en oleadas de 4, parada en el primer resultado | `GSI2` | `GSI2PK = AVAIL#<eventId>#<shard>` |
| AP-006 | Get por clave, caché en memoria si `ENABLED` | Tabla base | `EVENT#<eventId>`/`#META` |
| AP-008 | Escritura transaccional de N + 4 items (máx. 14) | Tabla base | N `TICKET#<eventId>#<ticketId>`, Order, `IDEM#...`, auditoría, `ACTIVE#<customerId>#<eventId>` |
| AP-009 | Get fuertemente consistente | Tabla base | `IDEM#<customerId>#<key>`/`#META` |
| AP-010 | Get fuertemente consistente | Tabla base | `ORDER#<orderId>`/`#META` |
| AP-011 | Update condicional | Tabla base | `ORDER#<orderId>`/`#META` |
| AP-012 | Escritura transaccional de N + 2 items (máx. 12) | Tabla base | Order, N Ticket, auditoría |
| AP-013 | Update condicional | Tabla base | `ORDER#<orderId>`/`#META` |
| AP-014 | Escritura transaccional de N + 3 items (máx. 13) | Tabla base | Order, N Ticket, auditoría, bloqueo (borrado) |
| AP-015 | Escritura transaccional de N + 3 items (máx. 13) | Tabla base | Order, N Ticket, auditoría, bloqueo (borrado) |
| AP-016 | Query por shard, paginada | `GSI3` | `GSI3PK = RESV#<shard> AND GSI3SK <= <nowMs>~` |
| AP-017 | Query | Tabla base | `PK = ORDER#<id> AND begins_with(SK, AUDIT#)` o `PK = EVENT#<id> AND begins_with(SK, AUDIT#)` |
| AP-018 | Query con filtro sobre `lastProgressAtMs` | `GSI1` | `GSI1PK = EVENTS#PROVISIONING` |
| AP-020 | Query por shard en orden 0..S−1, límite = tamaño de página restante, reanudación por cursor | `GSI2` | `GSI2PK = AVAIL#<eventId>#<shard>` [`AND begins_with(GSI2SK, <section>#)`] |
| AP-021 | Query de solo cantidad por shard, en paralelo, paginada | `GSI2` | `GSI2PK = AVAIL#<eventId>#<shard>` |
| AP-022 | Get fuertemente consistente | Tabla base | `IDEMEVT#<sub>#<key>`/`#META` |
| AP-023 | Get fuertemente consistente | Tabla base | `EVENT#<eventId>`/`#META` |
| AP-024 | Update condicional | Tabla base | `EVENT#<eventId>`/`#META` |
| AP-025 | Lectura por lotes consistente (100 claves por solicitud, `TO_VERIFY`) | Tabla base | Todas las claves `TICKET#<eventId>#<ticketId>` generadas |
| AP-026 | Escritura transaccional de 2 items | Tabla base | Event `#META`; auditoría del Event |
| AP-027 | Query de `GSI1` + borrado por lotes de las claves generadas + update condicional del Event | `GSI1`, tabla base | `GSI1PK = EVENTS#FAILED`; claves generadas; `EVENT#<id>`/`#META` |
| AP-028 | Query por shard | `GSI4` | `GSI4PK = PENDQ#<shard> AND GSI4SK < <nowMs − 30000>` |
| AP-029 | Query por shard | `GSI3` | `GSI3PK = REVERSAL#<shard> AND GSI3SK <= <nowMs>~` |
| AP-030 | Escritura transaccional de 2 items (completar, agotar) o update condicional (reprogramar) | Tabla base | Order; auditoría |
| AP-031 | Escritura transaccional de 2 items | Tabla base | Order; auditoría |
| AP-032 | Escritura transaccional de 2 items | Tabla base | Order; auditoría |
| AP-033 | Update condicional | Tabla base | `EVENT#<eventId>`/`#META` |

## 5. Write operations and conditions

`N` = Ticket de la Order (1..10). "Liberar Ticket" = `state = AVAILABLE`, restablecer `GSI2PK`/`GSI2SK` con su shard, eliminar `orderId`. "Salir de índices de Order activa" = eliminar `GSI3PK`/`GSI3SK` del rango `RESV#`, `GSI4PK`/`GSI4SK` y atributos de lease.

| Operation | Items written | Conditions | ST / FR IDs |
|---|---|---|---|
| Crear Event (AP-001) | Event (`PROVISIONING`, `GSI1` aprovisionando), idempotencia de creación, auditoría | Los tres items no existen | FR-001, FR-017 |
| Tomar lease de aprovisionamiento (AP-024) | Event | `provisioningStatus = PROVISIONING` y (sin lease, o `provisioningLeaseUntilMs < now`, o lease propio) | FR-001 |
| Comprobar y registrar progreso antes de un lote (AP-024) | Event: renueva lease, `provisionedBatches`, `lastProgressAtMs` | `provisioningStatus = PROVISIONING` y lease propio | FR-001 |
| Escribir lote (AP-002) | Ticket × hasta 100 | Ninguna (claves deterministas; solo tras la comprobación anterior) | FR-001, FR-003, BR-016, BR-021 |
| Habilitar (AP-003) | Event (`ENABLED`, `GSI1` habilitados, sin lease), auditoría | Event: `provisioningStatus = PROVISIONING` y lease propio; auditoría no existe | FR-001, FR-003, FR-014 |
| Marcar `FAILED` (AP-026) | Event (`FAILED`, `GSI1` fallidos, sin lease), auditoría | Event: `provisioningStatus = PROVISIONING` | FR-001, FR-014 |
| Registrar republicación (AP-033) | Event: `provisioningRepublishCount + 1`, `lastProgressAtMs = now` | `provisioningStatus = PROVISIONING` y `lastProgressAtMs` igual al leído | FR-001 |
| Purgar (AP-027) | Ticket × capacidad (borrado por lotes); Event: `ticketsPurgedAt`, sin `GSI1PK`/`GSI1SK` | Event: `provisioningStatus = FAILED` | FR-001, FR-003 |
| Reservar y crear Order (AP-008) | Ticket × N, Order (en `GSI3` `RESV#` y `GSI4`), idempotencia, auditoría, bloqueo | Cada Ticket: existe, `eventId` coincide, `state = AVAILABLE`. Order, idempotencia, auditoría, bloqueo: no existen | ST-001, ST-006, FR-004, FR-006, FR-010, FR-017, VAL-002, VAL-010 |
| Marcar encolada (AP-011) | Order: `enqueuedAt`, sin `GSI4` | `status = CREATED` y sin `enqueuedAt` | FR-005 |
| Iniciar pago (AP-012) | Order (PaymentAttempt, lease), Ticket × N, auditoría | Order: `status = CREATED`, sin `paymentAttemptId`, sin `quarantinedAt`, `expiresAtMs > now + 15000`. Ticket: `state = RESERVED` y `orderId` coincide | ST-003, FR-015, BR-020 |
| Reclamar lease (AP-013) | Order | `status = CREATED`, `paymentAttemptId` coincide, `paymentLeaseUntilMs < now`, sin `quarantinedAt` | FR-017, BR-020 |
| Confirmar (AP-014) | Order (`CONFIRMED`, sale de índices de Order activa), Ticket × N (`SOLD`), auditoría, bloqueo (borrado) | Order: `status = CREATED`, sin `quarantinedAt`, `paymentAttemptId` coincide, `expiresAtMs > now`. Ticket: `state = PENDING_CONFIRMATION` y `orderId` coincide. Bloqueo: no existe o `orderId` coincide | ST-004, ST-007, FR-008, FR-013 |
| Rechazar (AP-015) | Order (`REJECTED`, `PAYMENT_DECLINED`), Ticket × N liberados, auditoría, bloqueo | Order: `status = CREATED`, sin `quarantinedAt`, `paymentAttemptId` coincide. Ticket: `state` en (`RESERVED`, `PENDING_CONFIRMATION`) y `orderId` coincide. Bloqueo: como en Confirmar | ST-005, ST-008, FR-016 |
| Fallar, procesamiento (AP-015) | Order (`FAILED`, `PROCESSING_FAILED`; marca de reverso y `GSI3` `REVERSAL#` si el resultado del pago es desconocido), Ticket × N liberados, auditoría, bloqueo | Order: `status = CREATED`, sin `quarantinedAt`, identidad del PaymentAttempt igual a la leída. Ticket y bloqueo: como en Rechazar | ST-005, ST-009, FR-016, FG-003 |
| Fallar, encolado (AP-015) | Order (`FAILED`, `PROCESSING_UNAVAILABLE`), Ticket × N liberados, auditoría, bloqueo | Order: `status = CREATED` y sin `paymentAttemptId`. Ticket y bloqueo: como en Rechazar | ST-005, ST-009, ALT-006, ERR-007 |
| Expirar (AP-015) | Order (`EXPIRED`, `RESERVATION_EXPIRED`; marca de reverso y `GSI3` `REVERSAL#` si tiene PaymentAttempt), Ticket × N liberados, auditoría, bloqueo | Order: `status = CREATED`, sin `quarantinedAt`, `expiresAtMs <= now`. Ticket y bloqueo: como en Rechazar | ST-002, ST-010, FR-011, VAL-003, FG-003 |
| Cuarentena (AP-031) | Order (`quarantinedAt`, `quarantineReason`, sale de `RESV#` y `GSI4`, entra en `REVIEW#QUARANTINE`), auditoría | Order: `status = CREATED` y sin `quarantinedAt` | FR-013, ADR-025 |
| Aprobación tardía (AP-032) | Order (marca de reverso si no existe ni está completado), auditoría `LATE_APPROVAL_NOT_APPLIED` | Order: `status` en (`EXPIRED`, `FAILED`, `REJECTED`) y `paymentAttemptId` coincide; si el reverso ya está completado, solo auditoría | ADR-008, FG-003 |
| Completar reverso (AP-030) | Order (sin marca, sin `GSI3`, `paymentReversalCompletedAt`), auditoría | `paymentReversalPending = true` y `paymentAttemptId` coincide | FG-003 |
| Reprogramar reverso (AP-030) | Order (`paymentReversalAttempts + 1`, `paymentReversalNextAttemptAtMs`, `GSI3SK`) | `paymentReversalPending = true` y `paymentReversalAttempts` igual al leído | FG-003 |
| Agotar reverso (AP-030) | Order (`paymentReversalExhaustedAt`, `GSI3PK = REVERSAL#EXHAUSTED`), auditoría | `paymentReversalPending = true` | FG-003 |

Reglas transversales:

- Ninguna condición acepta `SOLD` ni `COMPLIMENTARY` como origen (`BR-006`, `BR-007`, `AC-012`, `AC-013`); solo "Escribir lote" escribe `COMPLIMENTARY` (`BR-016`, `AC-026`).
- Toda transición terminal exige `status = CREATED`.
- Toda inserción de auditoría exige que el item no exista.
- Una Order está en un solo rango de `GSI3` a la vez.

## 6. Capacity and limits

| Tema | Decisión | Estado |
|---|---|---|
| Modelo de capacidad | On-demand; precalentamiento como mejora productiva (ADR-037) | Decidido |
| Tamaño máximo de transacción | 14 items (reserva con 10 Ticket); 13 en terminales; 12 en inicio de pago; 3 en creación de Event | Decidido (ADR-003, ADR-025) |
| Límite de items de `TransactWriteItems` | Se asumen 100 | `TO_VERIFY` |
| Escritura por lotes | Se asumen 25 items por solicitud | `TO_VERIFY` |
| Lectura por lotes | Se asumen 100 claves por solicitud, con lectura consistente | `TO_VERIFY` |
| Tamaño de item | Se asume 400 KB; acota la definición del Event | `TO_VERIFY` |
| Capacidad máxima por Event | 50.000 Ticket (`FG-002`) | Decidido |
| Shards de `GSI2` | `min(32, max(1, ceil(capacity / 2000)))` por Event | Decidido; divisor ajustable tras verificar throughput por partición |
| Shards de `GSI3` / `GSI4` | 8 (`RESV#`), 4 (`REVERSAL#`), 8 (`PENDQ#`) | Decidido |
| Throughput por partición | Se asume del orden de 1.000 escrituras y 3.000 lecturas por segundo | `TO_VERIFY` |
| Escrituras de índice por compra confirmada | `GSI2`: 2N (sale al reservar; no reentra al vender); `GSI3`: 2 (entra y sale); `GSI4`: 2 (entra y sale) | Trade-off aceptado (ADR-026) |
| Coste transaccional | Doble que una escritura simple | `TO_VERIFY` (factor) |
| Borrado por TTL | Diferido; solo afecta a idempotencia | `TO_VERIFY` (demora) |

Estimación orientativa del escenario concentrado (más de 1.000 usuarios sobre un Event de 50.000 Ticket): los Ticket están en particiones propias, por lo que la contención por partición en la tabla base se limita a compras del mismo Ticket; las escrituras de `GSI2` se reparten entre 25 shards. Los objetivos de `NFR-001` y `NFR-002` son objetivos de prueba, no capacidad productiva.

## 7. Local versus AWS differences

| Aspecto | Local (DynamoDB Local) | AWS (Amazon DynamoDB) | Impacto |
|---|---|---|---|
| Modelo lógico | Misma tabla, claves, GSIs y operaciones | Igual | Ninguno |
| Creación | `infra-init` (ADR-036) | IaC (`ticketing.aws-target.v2.md` §11) | La aplicación nunca crea tablas |
| Particionado y throttling | No existen | Existen | `RISK-001`, `RISK-007`: el beneficio del particionado por Ticket solo es observable en AWS |
| Transacciones, motivos por item, lectura por lotes consistente | `TO_VERIFY` | Documentado | Lecturas de respaldo si difieren |
| Consistencia eventual de GSIs | De hecho casi inmediata | Real | Las pruebas no deben asumir lectura inmediata en AP-005, AP-016, AP-020, AP-021, AP-028, AP-029 |
| TTL | `TO_VERIFY` | Diferido | Sin impacto funcional |
| Credenciales, cifrado, PITR | Ficticias; no aplica | Rol de tarea; habilitados | Solo configuración |
