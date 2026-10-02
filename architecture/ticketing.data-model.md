---
artifact: data-model
schema_version: 1.0
feature: ticketing-event-processing
version: 1
source:
  feature_spec: feature-spec/ticketing.feature-spec.v4.md
  architecture: architecture/ticketing.architecture.md
status: READY_FOR_HUMAN_ARCHITECTURE_REVIEW
generated_at: 2026-10-01
---

# Data Model — Ticketing Event Processing

Decisiones que gobiernan este documento: [ADR-001](adr/ADR-001-dynamodb-data-model.md), [ADR-002](adr/ADR-002-atomic-multi-ticket-reservation.md), [ADR-004](adr/ADR-004-event-creation-at-capacity-scale.md), [ADR-005](adr/ADR-005-state-transition-consistency.md), [ADR-007](adr/ADR-007-idempotency.md), [ADR-009](adr/ADR-009-expiration-process.md), [ADR-012](adr/ADR-012-audit-trail.md), [ADR-021](adr/ADR-021-availability-read-model.md).

Los valores numéricos son valores iniciales configurables, no garantías de capacidad (ver §6). Los límites de servicio citados están marcados `TO_VERIFY`.

## 1. Access patterns

Los access patterns se enumeraron antes de diseñar claves e índices. Frecuencia: `hot` = ruta de cada solicitud bajo carga; `warm` = frecuente pero no por solicitud de compra; `cold` = esporádico u operativo.

| ID | Description | Component | R/W | Frequency | Consistency | Latency target | Source IDs |
|---|---|---|---|---|---|---|---|
| AP-001 | Crear el item Event en estado técnico no habilitado | CMP-003, CMP-010 | W | cold | Escritura condicional (no existe) | Sin objetivo específico | MF-001, FR-001, AC-001 |
| AP-002 | Escribir por lotes los N Ticket de un Event (`AVAILABLE` / `COMPLIMENTARY`) | CMP-003, CMP-010 | W | cold | Escritura por lotes idempotente por clave | Sin objetivo específico | MF-001, FR-001, FR-003, BR-021, VAL-009 |
| AP-003 | Habilitar el Event una vez escrito todo su inventario | CMP-003, CMP-010 | W | cold | Escritura condicional | Sin objetivo específico | FR-001, FR-003, AC-001, AC-017 |
| AP-004 | Listar Events futuros habilitados ordenados por fecha | CMP-004, CMP-010 | R | warm | Eventual | p95 < 500 ms (objetivo de prueba, por analogía con NFR-002) | MF-002, FR-002, BR-022, AC-002 |
| AP-005 | Determinar si un Event tiene al menos un Ticket `AVAILABLE` (marca de agotado) | CMP-004, CMP-010 | R | warm | Eventual | Incluido en AP-004 | FR-002, BR-012, BR-022, AC-002 |
| AP-006 | Obtener metadatos de un Event por ID | CMP-004, CMP-005, CMP-010 | R | hot | Eventual, cacheable (datos inmutables tras habilitar) | Incluido en AP-007 / AP-008 | MF-002, MF-003, FR-012, VAL-010 |
| AP-007 | Obtener todos los Ticket de un Event con su estado | CMP-004, CMP-010 | R | hot | Eventual (informativa, BR-018) | p95 < 500 ms (NFR-002, AC-030) | MF-002, FR-012, BR-012, BR-018, AC-010 |
| AP-008 | Reservar N Ticket, crear Order con su Reservation, registrar idempotencia y auditoría, todo atómicamente | CMP-005, CMP-010 | W | hot | Transacción serializable | p95 < 1 s para la operación síncrona completa (NFR-002, AC-031) | MF-003, FR-004, FR-006, FR-010, ST-001, ST-006, VAL-002, VAL-010, AC-004, AC-007, AC-016 |
| AP-009 | Obtener el registro de idempotencia por (customer, idempotency key) | CMP-005, CMP-010 | R | hot | Fuerte | Incluido en AP-008 | FR-017, BR-019, AC-023 |
| AP-010 | Obtener una Order por ID | CMP-006, CMP-007, CMP-008, CMP-010 | R | hot | Fuerte | p95 < 500 ms (objetivo de prueba por analogía) | MF-004, FR-009, VAL-011, AC-006, AC-027 |
| AP-011 | Marcar la Order como encolada (`enqueuedAt`) | CMP-005, CMP-010 | W | hot | Escritura condicional, best effort | Incluido en AP-008 | FR-005, AC-003 |
| AP-012 | Iniciar pago: registrar PaymentAttempt activo y pasar todos los Ticket a `PENDING_CONFIRMATION` | CMP-007, CMP-010 | W | hot | Transacción serializable | Asíncrono, sin objetivo | FR-015, ST-003, BR-020, AC-005, AC-024 |
| AP-013 | Reclamar el lease de procesamiento de un PaymentAttempt activo | CMP-007, CMP-010 | W | cold | Escritura condicional | Asíncrono | FR-017, BR-020, AC-023, AC-024 |
| AP-014 | Confirmar: Order `CONFIRMED` y todos los Ticket `SOLD` | CMP-007, CMP-010 | W | hot | Transacción serializable | Asíncrono | FR-008, FR-013, ST-004, ST-007, AC-019 |
| AP-015 | Cerrar con liberación: Order `REJECTED` / `FAILED` / `EXPIRED` y todos los Ticket `AVAILABLE` | CMP-005, CMP-007, CMP-008, CMP-010 | W | warm | Transacción serializable | Asíncrono (síncrono solo en fallo de encolado) | FR-011, FR-016, ST-002, ST-005, ST-008, ST-009, ST-010, AC-008, AC-020, AC-021, AC-022 |
| AP-016 | Encontrar Reservation activas cuyo instante de expiración ya pasó | CMP-008, CMP-010 | R | warm (periódico) | Eventual; la corrección la da la condición de AP-015 | Por ciclo del proceso | FR-011, VAL-003, AC-008, AC-009 |
| AP-017 | Leer la evidencia de auditoría de una Order o de la creación de un Event | Operación / soporte, CMP-010 | R | cold | Fuerte opcional | Sin objetivo | FR-014, BR-010, NFR-005, AC-015 |
| AP-018 | Encontrar Events no habilitados cuya creación quedó incompleta | CMP-015, CMP-010 | R | cold | Eventual | Sin objetivo | FR-001, AC-017 |
| AP-019 | Eliminar un Event incompleto y sus Ticket | CMP-015, CMP-010 | W | cold | Escritura condicional sobre el Event | Sin objetivo | FR-001, FR-003 |

## 2. Tables and indexes

### 2.1 Tabla

Una única tabla (single-table design, ADR-001).

| Elemento | Valor |
|---|---|
| Nombre lógico | `ticketing` (nombre físico parametrizado por entorno) |
| Partition key | `PK` (String) |
| Sort key | `SK` (String) |
| Modelo de capacidad | On-demand (ADR-001) |
| TTL | Atributo `ttl` (epoch seconds), usado solo por registros de idempotencia |
| Cifrado en reposo | Habilitado en AWS (ver `ticketing.aws-target.md` §5) |
| Recuperación | Point-in-time recovery habilitado en AWS |

### 2.2 Índices secundarios globales

Todos son dispersos (sparse): un item solo aparece en el índice mientras posee el atributo de clave del índice.

| Índice | Partition key | Sort key | Proyección | Items presentes | Justificado por |
|---|---|---|---|---|---|
| `GSI1` — events by start | `GSI1PK` | `GSI1SK` | ALL | Solo items Event | AP-004, AP-018 |
| `GSI2` — available tickets | `GSI2PK` | `SK` (de la tabla base) | KEYS_ONLY | Solo Ticket en `AVAILABLE` | AP-005 |
| `GSI3` — active reservations by expiry | `GSI3PK` | `GSI3SK` | KEYS_ONLY | Solo Order en `CREATED` | AP-016 |

`GSI3` queda reservado además para el índice de trabajo de reversos de pago, únicamente si se aprueba FG-003 (ver ADR-008).

### 2.3 Modo de consistencia por access pattern

| Modo | Access patterns | Motivo |
|---|---|---|
| Transacción serializable (escritura transaccional con condiciones) | AP-008, AP-012, AP-014, AP-015 | FR-013, BR-009, BR-013: no pueden observarse subconjuntos parciales |
| Escritura condicional simple | AP-001, AP-003, AP-011, AP-013, AP-019 | Un solo item; la condición garantiza idempotencia |
| Lectura fuertemente consistente | AP-009, AP-010, AP-017 | Decisiones de control (idempotencia, propiedad, estado de la Order) y lectura posterior a la propia escritura |
| Lectura eventual | AP-004, AP-005, AP-006, AP-007, AP-016, AP-018 | Información declarada informativa (BR-018) o protegida por una condición posterior (AP-016); los GSI solo admiten lectura eventual |

## 3. Item types

Formato de instantes: los atributos con sufijo `At` son ISO-8601 en UTC; los atributos con sufijo `Ms` son epoch milliseconds (Number) y se usan en expresiones de condición.

| Entity | PK | SK | Attributes | Notes |
|---|---|---|---|---|
| Event | `EVENT#<eventId>` | `#META` | `entityType`, `eventId`, `name`, `venue`, `startsAt`, `startsAtMs`, `capacity`, `complimentaryCount`, `enabled`, `createdAt`, `createdBy`, `GSI1PK`, `GSI1SK` | `enabled` es un atributo técnico, no un estado de negocio (ADR-004, AV-002). Mientras `enabled = false`: `GSI1PK = EVENTS#PROVISIONING`, `GSI1SK = <createdAt>#<eventId>`. Al habilitar: `GSI1PK = EVENTS#ENABLED`, `GSI1SK = <startsAt>#<eventId>`. Inmutable después de habilitado. |
| Ticket | `EVENT#<eventId>` | `TICKET#<ticketId>` | `entityType`, `ticketId`, `state`, `orderId` (solo mientras está retenido o vendido), `updatedAt`, `GSI2PK` (solo en `AVAILABLE`) | `state` toma exactamente uno de DS-001..DS-005 (BR-001, VAL-005). `GSI2PK = EVENT#<eventId>#AVAILABLE`. Item deliberadamente pequeño (AP-007). `ticketId` es único dentro del Event. |
| Order (incluye Reservation y PaymentAttempt) | `ORDER#<orderId>` | `#META` | Order: `entityType`, `orderId`, `customerId`, `eventId`, `ticketIds`, `status`, `failureCause`, `createdAt`, `updatedAt`, `terminalAt`. Reservation: `reservationId`, `reservedAt`, `expiresAt`, `expiresAtMs`. Encolado: `enqueuedAt`. PaymentAttempt: `paymentAttemptId`, `paymentAttemptNo`, `paymentStartedAt`, `paymentOutcome`, `paymentCompletedAt`, `paymentProviderRef`, `paymentLeaseOwner`, `paymentLeaseUntilMs`. Índice: `GSI3PK`, `GSI3SK` (solo en `CREATED`) | `status` toma exactamente uno de DS-006..DS-010. Reservation y PaymentAttempt son conceptos de dominio propios, co-localizados físicamente en el item Order porque su cardinalidad es 0..1 y siempre cambian junto con la Order (ADR-001, ADR-005). La Reservation está activa si y solo si `status = CREATED`. `enqueuedAt`, `paymentLease*` y `paymentOutcome` son atributos técnicos fuera de las máquinas de estado funcionales. `GSI3PK = RESV#<shard>`, `GSI3SK = <expiresAt>#<orderId>`. |
| Idempotency record | `IDEM#<customerId>#<idempotencyKey>` | `#META` | `entityType`, `orderId`, `requestHash`, `createdAt`, `ttl` | Creado en la misma transacción que la Order (AP-008). Vigencia mínima 24 h (ADR-007). |
| Audit record (Order) | `ORDER#<orderId>` | `AUDIT#<occurredAt>#<transitionCode>` | `entityType`, `transitionIds` (por ejemplo `ST-001`, `ST-006`), `orderFrom`, `orderTo`, `ticketFrom`, `ticketTo`, `ticketIds`, `eventId`, `cause`, `actorType`, `actorId`, `paymentAttemptId`, `correlationId`, `occurredAt` | Solo inserción; escrito en la misma transacción que la transición (ADR-012). |
| Audit record (Event) | `EVENT#<eventId>` | `AUDIT#<occurredAt>#EVENT_CREATED` | `entityType`, `eventId`, `availableCount`, `complimentaryCount`, `capacity`, `actorId`, `occurredAt` | Evidencia de creación del inventario inicial; escrito junto con la habilitación (AP-003). |

Atributos condicionados a un FG:

| Atributo | Entity | Depende de |
|---|---|---|
| `paymentReversalPending`, `paymentReversalRequestedAt` | Order | FG-003 |
| Rango `GSI3PK = REVERSAL#<shard>` | Order | FG-003 |

## 4. Access pattern resolution

| AP ID | Operation | Table / index | Key condition |
|---|---|---|---|
| AP-001 | Put condicional | Tabla base | `PK = EVENT#<eventId>`, `SK = #META`; condición: el item no existe |
| AP-002 | Escritura por lotes en bloques, con reintento de items no procesados | Tabla base | `PK = EVENT#<eventId>`, `SK = TICKET#<ticketId>` |
| AP-003 | Escritura transaccional de 2 items (Event + Audit record) | Tabla base | `PK = EVENT#<eventId>`, `SK = #META`; condición: `enabled = false` |
| AP-004 | Query paginada ascendente | `GSI1` | `GSI1PK = EVENTS#ENABLED AND GSI1SK > <now ISO>` |
| AP-005 | Query con límite 1 | `GSI2` | `GSI2PK = EVENT#<eventId>#AVAILABLE` |
| AP-006 | Get por clave, con caché en memoria de Events habilitados | Tabla base | `PK = EVENT#<eventId>`, `SK = #META` |
| AP-007 | Query con paginación interna | Tabla base | `PK = EVENT#<eventId> AND begins_with(SK, TICKET#)` |
| AP-008 | Escritura transaccional de N + 3 items | Tabla base | N Ticket por (`EVENT#<eventId>`, `TICKET#<ticketId>`), Order, Idempotency record, Audit record |
| AP-009 | Get fuertemente consistente | Tabla base | `PK = IDEM#<customerId>#<idempotencyKey>`, `SK = #META` |
| AP-010 | Get fuertemente consistente | Tabla base | `PK = ORDER#<orderId>`, `SK = #META` |
| AP-011 | Update condicional | Tabla base | `PK = ORDER#<orderId>`, `SK = #META` |
| AP-012 | Escritura transaccional de N + 2 items | Tabla base | Order, N Ticket, Audit record |
| AP-013 | Update condicional | Tabla base | `PK = ORDER#<orderId>`, `SK = #META` |
| AP-014 | Escritura transaccional de N + 2 items | Tabla base | Order, N Ticket, Audit record |
| AP-015 | Escritura transaccional de N + 2 items | Tabla base | Order, N Ticket, Audit record |
| AP-016 | Query por shard, paginada | `GSI3` | `GSI3PK = RESV#<shard> AND GSI3SK <= <now ISO>~` |
| AP-017 | Query | Tabla base | `PK = ORDER#<orderId> AND begins_with(SK, AUDIT#)`; o `PK = EVENT#<eventId> AND begins_with(SK, AUDIT#)` |
| AP-018 | Query | `GSI1` | `GSI1PK = EVENTS#PROVISIONING AND GSI1SK < <now - umbral>` |
| AP-019 | Query de Ticket + borrado por lotes + borrado condicional del Event | Tabla base | `PK = EVENT#<eventId>`; condición del borrado del Event: `enabled = false` |

## 5. Write operations and conditions

`N` = cantidad de Ticket de la Order. El detalle de qué transición ejecuta qué operación está en ADR-005 y en `ticketing.architecture.md` §8.

| Operation | Items written | Conditions | ST / FR IDs |
|---|---|---|---|
| Create Event (provisioning) | Event | El item no existe | FR-001 |
| Write tickets | Ticket × capacidad, en bloques | Ninguna (claves deterministas; reescritura idempotente mientras el Event no está habilitado) | FR-001, FR-003, BR-016, BR-021 |
| Enable Event | Event, Audit record (Event) | Event: `enabled = false` | FR-001, FR-003, FR-014 |
| Reserve and create Order | Ticket × N, Order, Idempotency record, Audit record | Cada Ticket: existe y `state = AVAILABLE`. Order: no existe. Idempotency record: no existe | ST-001, ST-006, FR-004, FR-006, FR-010, FR-017, VAL-002, VAL-010 |
| Mark enqueued | Order | El item existe | FR-005 |
| Start payment | Order, Ticket × N, Audit record | Order: `status = CREATED`, sin `paymentAttemptId`, `expiresAtMs > now + cutoff`. Cada Ticket: `state = RESERVED` y `orderId` coincide | ST-003, FR-015, BR-020 |
| Claim payment lease | Order | `status = CREATED`, `paymentAttemptId` coincide, `paymentLeaseUntilMs < now` | FR-017, BR-020 |
| Confirm | Order, Ticket × N, Audit record | Order: `status = CREATED`, `paymentAttemptId` coincide, `expiresAtMs > now`. Cada Ticket: `state = PENDING_CONFIRMATION` y `orderId` coincide | ST-004, ST-007, FR-008, FR-013 |
| Reject (payment declined) | Order, Ticket × N, Audit record | Order: `status = CREATED`, `paymentAttemptId` coincide. Cada Ticket: `state` en (`RESERVED`, `PENDING_CONFIRMATION`) y `orderId` coincide | ST-005, ST-008, FR-016 |
| Fail (technical, consumer) | Order, Ticket × N, Audit record | Order: `status = CREATED` y la identidad del PaymentAttempt coincide con la leída (presente o ausente). Tickets: igual que Reject | ST-005, ST-009, FR-016 |
| Fail (enqueue, API) | Order, Ticket × N, Audit record | Order: `status = CREATED` y sin `paymentAttemptId`. Tickets: igual que Reject | ST-005, ST-009, ALT-006, ERR-007 |
| Expire | Order, Ticket × N, Audit record | Order: `status = CREATED` y `expiresAtMs <= now`. Tickets: igual que Reject | ST-002, ST-010, FR-011, VAL-003 |
| Delete orphan Event | Ticket × n (por lotes), Event | Event: `enabled = false` | FR-001 |

Reglas transversales:

- Ninguna condición acepta `SOLD` ni `COMPLIMENTARY` como estado de origen (BR-006, BR-007, AC-012, AC-013), y ninguna operación escribe `COMPLIMENTARY` fuera de "Write tickets" (BR-016, AC-026).
- Toda transición terminal de la Order exige `status = CREATED`; por ello solo una transición terminal puede confirmarse (ADR-005, ADR-008).
- Al salir de `CREATED` se eliminan `GSI3PK`, `GSI3SK` y los atributos de lease.
- Al pasar un Ticket a `AVAILABLE` se restablece `GSI2PK` y se elimina `orderId`; al salir de `AVAILABLE` se elimina `GSI2PK`.

## 6. Capacity and limits

| Tema | Decisión | Estado |
|---|---|---|
| Modelo de capacidad | On-demand: la carga es impulsiva (apertura de ventas) y no hay línea base conocida. Alternativa evaluada en ADR-001 | Decidido (PROPOSED) |
| Tamaño de transacción | Máximo N + 3 items. Con el máximo recomendado de 10 Ticket por Order (FG-001): 13 items | Depende de FG-001 |
| Límite de items por transacción de DynamoDB | Se asume 100 items por transacción y un tamaño agregado máximo | `TO_VERIFY` |
| Límite de escritura por lotes | Se asume 25 items por solicitud | `TO_VERIFY` |
| Capacidad máxima por Event | 5.000 Ticket recomendados (FG-002); acota AP-002 y AP-007 | Depende de FG-002 |
| Partición caliente | Todos los Ticket de un Event comparten partition key. Riesgo RISK-001; ruta de evolución: sharding de la partición por sección o bucket | Aceptado para los objetivos de prueba |
| Throughput por partición | Se asume un límite por partición física | `TO_VERIFY` |
| Shards de `GSI3` | 4 (configurable), `shard = hash(orderId) mod 4` | Decidido (PROPOSED) |
| Costo de escritura transaccional | Las escrituras transaccionales consumen más capacidad que las simples | `TO_VERIFY` (factor exacto) |
| Borrado por TTL | No es inmediato; el diseño no depende de su precisión (ADR-007, ADR-009) | `TO_VERIFY` (demora típica) |

Los objetivos de NFR-001 y NFR-002 son objetivos de prueba de esta implementación; este modelo no garantiza capacidad productiva.

## 7. Local versus AWS differences

| Aspecto | Local (DynamoDB Local) | AWS objetivo (Amazon DynamoDB) | Impacto en el diseño |
|---|---|---|---|
| Modelo lógico | Misma tabla, claves, GSIs y operaciones | Igual | Ninguno; es el mismo modelo |
| Creación de la tabla | Contenedor de inicialización de un solo uso (ADR-017) | Infraestructura como código (handoff en `ticketing.aws-target.md` §11) | La aplicación nunca crea tablas |
| Particionado y throttling | No hay particiones físicas ni throttling real | Límites por partición y throttling | RISK-001 y RISK-007: no observable en local |
| Modelo de capacidad | No aplica | On-demand | Solo configuración |
| Fidelidad de transacciones y motivos de cancelación | `TO_VERIFY` | Comportamiento documentado del servicio | Si difiere, la clasificación de fallos de AP-008 usa la lectura posterior de respaldo (ADR-002) |
| TTL | Soporte `TO_VERIFY` | Borrado diferido | Sin impacto funcional: un registro presente se respeta |
| Credenciales | Credenciales ficticias por variable de entorno | Rol de tarea IAM | Solo configuración (ADR-013) |
| Cifrado, PITR, IAM | No aplica | Habilitados | Solo AWS |
| Consistencia eventual | Las lecturas se comportan de hecho como consistentes | Lecturas eventuales reales en tabla e índices | Las pruebas no deben asumir lectura inmediata en AP-004, AP-005, AP-007, AP-016 |
