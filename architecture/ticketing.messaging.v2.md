---
artifact: messaging
schema_version: 1.0
feature: ticketing-event-processing
version: 2
supersedes: architecture/ticketing.messaging.md
source:
  feature_spec: feature-spec/ticketing.feature-spec.v4.md
  human_review: human-review/ticketing.architecture-review.yaml
  architecture: architecture/ticketing.architecture.v2.md
status: READY_FOR_DEVELOPMENT
generated_at: 2026-10-04
---

# Messaging v2 — Ticketing Event Processing

Decisiones que gobiernan este documento: [ADR-024](adr/ADR-024-asynchronous-event-provisioning.md), [ADR-026](adr/ADR-026-persistence-plus-enqueue-with-republish-sweep.md), [ADR-027](adr/ADR-027-idempotency-purchase-event-creation.md), [ADR-029](adr/ADR-029-sqs-operational-policy-orders-and-provisioning.md), [ADR-035](adr/ADR-035-error-model-reactive-retry-circuit-breaker.md), [ADR-039](adr/ADR-039-aws-integration-technology-v2.md) y [ADR-008](adr/ADR-008-payment-versus-expiration-race.md) aceptado. Reemplaza a `ticketing.messaging.md` (versión 1, intacto).

Valores configurables desplegados en esta implementación. Límites del servicio `TO_VERIFY` (ADR-029).

## 1. Queues

| Cola | Tipo | Propósito | Productores | Consumidor | Source IDs |
|---|---|---|---|---|---|
| `ticketing-orders` | SQS Standard | Procesamiento asíncrono de una Order | `CMP-011` desde `CMP-005` (rol `api`) y desde `CMP-023` barrido (rol `worker`) | `CMP-012` → `CMP-007` (rol `worker`) | TC-005, FR-005, FR-007 |
| `ticketing-orders-dlq` | SQS Standard | Retención de mensajes de Orders que agotaron recepciones | SQS por redrive | Ninguno automático; redrive manual | TC-010, ERR-005, ERR-008 |
| `ticketing-event-provisioning` | SQS Standard | Aprovisionamiento asíncrono del inventario de un Event | `CMP-011` desde `CMP-003` (rol `api`) y desde `CMP-015` detección de estancados (rol `worker`) | `CMP-025` → `CMP-022` (rol `worker`) | TC-005, FR-001, FG-002 |
| `ticketing-event-provisioning-dlq` | SQS Standard | Retención de mensajes de aprovisionamiento que agotaron recepciones | SQS por redrive | Ninguno automático; redrive manual | TC-010 |

Nombres físicos parametrizados por entorno; cifrado en reposo en AWS.

| Atributo | `ticketing-orders` | `ticketing-orders-dlq` | `ticketing-event-provisioning` | `ticketing-event-provisioning-dlq` |
|---|---|---|---|---|
| Visibility timeout | 60 s | 60 s | 120 s, extendido por heartbeat | 120 s |
| Long polling | 20 s, hasta 10 mensajes | — | 20 s, 1 mensaje | — |
| Retención | 1 hora | 14 días | 1 día | 14 días |
| Redrive | `maxReceiveCount = 5` hacia su DLQ | — | `maxReceiveCount = 5` hacia su DLQ | — |
| Retardo de entrega | 0 | 0 | 0 | 0 |

## 2. Message contracts

### MSG-001

`OrderProcessingRequested`, versión de esquema 1 (sin cambios de esquema respecto de la versión 1 del documento).

- Producer: `CMP-005` tras el commit de la reserva, y en la ruta de repetición de una Order en `CREATED` sin `enqueuedAt` (ADR-027); `CMP-023` barrido de republicación para Orders en `CREATED` sin `enqueuedAt` de más de 30 s y con tiempo restante ≥ 15 s (ADR-026). Ambos mediante `CMP-011`, con timeout 500 ms, 3 intentos, 2 s de presupuesto y circuit breaker (ADR-035).
- Consumer: `CMP-012`, que invoca a `CMP-007`.
- Schema:

  | Campo | Tipo | Obligatorio | Descripción |
  |---|---|---|---|
  | `schemaVersion` | integer | Sí | `1` |
  | `messageType` | string | Sí | `OrderProcessingRequested` |
  | `orderId` | string (UUID) | Sí | Order a procesar |
  | `eventId` | string (UUID) | Sí | Informativo, diagnóstico |
  | `occurredAt` | string (instante UTC) | Sí | Instante de creación de la Order |
  | `correlationId` | string | Sí | Correlación de la solicitud original o del ciclo de barrido |

  Atributos: `messageType`, `schemaVersion`, contexto de traza, `publisher` (`api` o `sweep`, diagnóstico).

  Reglas: referencia pura (el consumidor lee la Order, `AP-010`); sin datos personales ni tokens; campos nuevos opcionales; versión desconocida = venenoso.
- Idempotency identity: `orderId` y estado de la Order (ADR-027). El identificador de mensaje de SQS no es identidad.
- Source IDs: FR-005, FR-007, FR-017, TC-005, TC-010, MF-003, AC-003, AC-005, AC-023.

### MSG-002

`EventProvisioningRequested`, versión de esquema 1.

- Producer: `CMP-003` tras el commit de `AP-001` (respuesta 202 aunque la publicación falle); `CMP-015` al republicar un Event estancado (`AP-033`). Ambos mediante `CMP-011` con la misma política de publicación.
- Consumer: `CMP-025`, que invoca a `CMP-022`.
- Schema:

  | Campo | Tipo | Obligatorio | Descripción |
  |---|---|---|---|
  | `schemaVersion` | integer | Sí | `1` |
  | `messageType` | string | Sí | `EventProvisioningRequested` |
  | `eventId` | string (UUID) | Sí | Único dato de negocio: el Event a aprovisionar (ADR-024) |

  Atributos: `messageType`, `schemaVersion`, contexto de traza, `correlationId`.

  Reglas: el mensaje solo transporta el `eventId`; la definición del inventario se lee del Event; sin datos del `ADMIN`.
- Idempotency identity: `eventId` con `provisioningStatus`, lease y `provisionedBatches` del Event (ADR-027).
- Source IDs: FR-001, FR-003, BR-021, VAL-009, AC-001, FG-002.

## 3. Delivery and retry policy

| Aspecto | `ticketing-orders` | `ticketing-event-provisioning` | Source IDs |
|---|---|---|---|
| Semántica | Al menos una vez, sin orden | Al menos una vez, sin orden | TC-010 |
| Publicación | Timeout 500 ms, 3 intentos, backoff con jitter, 2 s; circuit breaker de publicación | Igual | FR-005, ADR-026, ADR-035 |
| Fallo definitivo de publicación | Order `FAILED` (`PROCESSING_UNAVAILABLE`) en la ruta síncrona; omitido en barrido | Event sigue en `PROVISIONING`; detección de estancados a los 3 min | ALT-006, AC-022 |
| Circuito de publicación abierto | Compra rechazada con 503 antes de reservar; barrido omite el ciclo | Creación aceptada; el Event espera a la detección de estancados | ADR-035 |
| Concurrencia por instancia | Configurable, 16 | 1 | NFR-003 |
| Confirmación | Eliminar solo con resultado estable | Eliminar solo con Event `ENABLED` o ya terminal | TC-010 |
| Reintento | Por reentrega, visibilidad 5 s, 15 s, 30 s, 60 s con jitter | Por reentrega, visibilidad 30 s, 60 s, 120 s, 240 s con jitter | ALT-004, ERR-005 |
| Máximo de recepciones | 5 | 5 | ERR-005 |
| Tope / heartbeat | Tope 30 s < lease 45 s < visibilidad 60 s | Heartbeat cada 30 s a 120 s mientras haya progreso; sin progreso en 60 s, se detiene | BR-020, ADR-029 |
| Pausa | Con el circuito del Payment Mock abierto, el bucle no recibe; en semiabierto solo mensajes de prueba | No aplica | ADR-035 |
| Capacidad | Alarma de antigüedad del mensaje más antiguo a los 2 min; autoescalado del worker por pendientes y antigüedad | — | ADR-029, ADR-037 |
| Presupuesto total | Reintentos en pocos minutos, por debajo de los diez de la Reservation | Minutos; el Event no es visible mientras tanto | BR-002 |

## 4. Dead-letter handling

| Aspecto | `ticketing-orders-dlq` | `ticketing-event-provisioning-dlq` |
|---|---|---|
| Llegada | Tras 5 recepciones sin eliminar | Tras 5 recepciones sin eliminar |
| Estado de la entidad | Normal: Order `FAILED` cerrada en la quinta recepción, Ticket liberados, reverso marcado si el resultado del pago era desconocido. Caída en la quinta recepción: Order en `CREATED`, la expiración la cierra como `EXPIRED` (con reverso si tenía PaymentAttempt) | Normal: Event `FAILED` marcado en la quinta recepción; la limpieza purga sus Ticket. Caída: Event en `PROVISIONING`; la detección de estancados lo republica o lo marca `FAILED` |
| Venenosos | Sin Order procesable; llegan sin efectos | Sin Event; llegan sin efectos |
| Consumo | Ninguno automático | Ninguno automático |
| Señal | Alarma por profundidad > 0; motivo registrado | Alarma por profundidad > 0; motivo registrado |
| Redrive | Manual; inocuo (Order terminal sin efectos, `AC-025`) | Manual; inocuo (Event `ENABLED`/`FAILED` sin efectos) |
| Retención | 14 días | 14 días |

## 5. Consumer processing rules

### 5.1 Orders (`CMP-007`)

El orden es significativo.

| # | Condición | Acción | Mensaje | Source IDs |
|---|---|---|---|---|
| 1 | Mensaje ilegible, tipo o versión desconocidos, sin `orderId` | Registrar venenoso | No eliminar; visibilidad corta | ERR-005 |
| 2 | Order inexistente (lectura consistente) | Registrar venenoso | No eliminar; visibilidad corta | ERR-005 |
| 3 | Order terminal | Ninguna | Eliminar | AC-025, FR-017 |
| 4 | Order en cuarentena | Ninguna (revisión manual) | Eliminar | ADR-025 |
| 5 | Order `CREATED`, sin PaymentAttempt, tiempo restante < 15 s | Si `expiresAt` venció, Expirar; si no, nada | Eliminar | BR-002, AV-003, ADR-008 |
| 6 | Order `CREATED`, sin PaymentAttempt | Iniciar pago (`ST-003`); si la condición falla, releer y reevaluar desde 3 | Continuar | AC-005, BR-020 |
| 7 | PaymentAttempt con lease vigente de otro consumidor | Ninguna | Posponer visibilidad hasta el fin del lease | AC-023, AC-024 |
| 8 | PaymentAttempt con lease vencido | Reclamar lease; si no se obtiene, regla 7 | Continuar | AC-023, ALT-004 |
| 9 | Se posee el lease | Autorizar en el Payment Mock con `paymentAttemptId`, plazo limitado por `expiresAt` y presupuesto de pago (ADR-035) | Continuar | FR-015, AC-024 |
| 10 | `APPROVED` | Confirmar (`ST-004`, `ST-007`). Si falla por condición: releer; si sigue `CREATED` y venció, Expirar con marca de reverso y auditoría de aprobación tardía; si ya es terminal, `AP-032` | Eliminar | AC-019, ADR-008, FG-003 |
| 11 | `DECLINED` | Rechazar (`ST-005`, `ST-008`) | Eliminar | AC-020, ALT-005 |
| 12 | Error definitivo del proveedor (4xx) | Fallar sin reverso (`ST-005`, `ST-009`) | Eliminar | AC-021, ERR-008 |
| 13 | Transitorio (timeout, conexión, 5xx, circuito abierto) con recepciones restantes | Ninguna | No eliminar; backoff | ALT-004, ERR-005 |
| 14 | Transitorio en la última recepción | Fallar con marca de reverso si hubo PaymentAttempt de resultado desconocido | No eliminar; pasa a la DLQ | AC-021, ERR-008, FG-003 |
| 15 | Cancelación por condición de un Ticket con la Order en `CREATED` en cualquier transición | Cuarentena (`AP-031`) | Eliminar | ADR-025 |

Reglas transversales: un fallo de condición no es un error (releer y reevaluar); solo se decide por `orderId`; la regla 14 no se aplica si el lease pertenece a otro consumidor; al apagar, el bucle deja de recibir y completa lo que tiene en vuelo.

### 5.2 Aprovisionamiento (`CMP-022`)

| # | Condición | Acción | Mensaje | Source IDs |
|---|---|---|---|---|
| 1 | Mensaje ilegible, tipo o versión desconocidos, sin `eventId` | Registrar venenoso | No eliminar; visibilidad corta | ERR-005 |
| 2 | Event inexistente | Registrar venenoso | No eliminar; visibilidad corta | ERR-005 |
| 3 | Event `ENABLED` o `FAILED` | Ninguna (reentrega o duplicado) | Eliminar | FR-017, ADR-024 |
| 4 | Lease vigente de otro worker | Ninguna | Posponer visibilidad hasta el fin del lease | ADR-024 |
| 5 | Lease obtenido | Reanudar desde `provisionedBatches`; antes de cada lote, comprobación condicional (`AP-024`); escribir el lote (`AP-002`); heartbeat de visibilidad | Continuar | FR-001, BR-016 |
| 6 | Comprobación antes de un lote falla | Detenerse sin escribir; releer y reevaluar desde 3 | Según reevaluación | ADR-024 |
| 7 | Todos los lotes escritos | Verificar (`AP-025`); faltantes: reescribir y verificar hasta 3 veces | Continuar | VAL-009, FG-002 |
| 8 | Verificación correcta | Habilitar (`AP-003`) | Eliminar | AC-001 |
| 9 | Transitorio con recepciones restantes | Ninguna | No eliminar; backoff | ALT-004 |
| 10 | Transitorio en la última recepción | Marcar `FAILED` (`AP-026`) | No eliminar; pasa a la DLQ | ADR-024 |

### 5.3 Procesos periódicos relacionados (sin mensaje propio)

| Proceso | Mensaje que produce | Regla |
|---|---|---|
| Barrido de republicación (`CMP-023`) | `MSG-001` | ADR-026 |
| Detección de estancados (`CMP-015`) | `MSG-002` | ADR-024 |
| Reversos de pago (`CMP-024`) | Ninguno; trabaja sobre `GSI3` rango `REVERSAL#` | ADR-025, ADR-008 |
| Expiración (`CMP-008`) | Ninguno | ADR-028 |
