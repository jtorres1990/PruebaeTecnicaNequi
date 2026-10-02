---
id: ADR-006
title: Persistence plus enqueue
status: PROPOSED
priority: HIGH
source_ids: [FR-005, FR-006, ALT-006, ERR-007, AC-003, AC-022, ST-009, MF-003, DS-006, DS-009, TC-005, TC-009]
related_adrs: [ADR-002, ADR-005, ADR-007, ADR-009, ADR-010, ADR-016]
---

# ADR-006 — Persistence plus enqueue

## Context

Después de crear consistentemente la Reservation y la Order, el sistema debe intentar encolar la solicitud en SQS Standard (`FR-005`) y retornar el Order ID sin esperar el pago, cuando la solicitud quedó aceptada o cuando existe un resultado persistido consultable por fallo definitivo de encolado (`FR-006`, `AC-003`). Ante un fallo definitivo de encolado la Order termina en `FAILED` y sus Ticket vuelven a `AVAILABLE` (`ALT-006`, `ERR-007`, `AC-022`).

DynamoDB y SQS no comparten transacción: entre el commit de la Order y la publicación del mensaje existe una ventana de fallo que debe tratarse explícitamente.

## Options considered

### Option A — Publicación directa tras el commit, con compensación síncrona

La operación síncrona confirma la transacción de reserva, publica el mensaje y, si la publicación falla definitivamente, ejecuta la transición a `FAILED`. La respuesta refleja el estado persistido.

- A favor: coincide literalmente con el orden de `MF-003` (pasos 10 a 12) y con `AC-003`; el fallo definitivo de encolado se determina dentro de la solicitud; sin componentes adicionales; latencia mínima hasta el procesamiento.
- En contra: si el proceso muere entre el commit y la publicación, la Order queda en `CREATED` sin mensaje hasta que expire.

### Option B — Transactional outbox con relay

La transacción de reserva incluye un registro outbox; un relay (por sondeo de un índice o por stream de cambios) publica en SQS y marca el registro.

- A favor: elimina la ventana de fallo; publicación garantizada al menos una vez.
- En contra: añade un relay, un índice o stream y su operación; el fallo definitivo de encolado pasa a decidirlo el relay de forma asíncrona, con lo que la respuesta síncrona nunca puede informar `FAILED`; añade latencia de sondeo antes del procesamiento; el soporte de streams en el emulador local es `TO_VERIFY`.

### Option C — Encolar antes de persistir

Publicar primero y crear la Order en el consumidor.

- A favor: sin ventana entre commit y publicación.
- En contra: contradice `HV-002` y `HV-018` (la reserva precede al encolado y la Order se crea tras reservar). Se descarta por violar la especificación.

## Decision

Se adopta la **Option A, reforzada con un marcador de encolado en el propio item Order**.

Secuencia de la operación síncrona (`CMP-005`):

1. Commit de la transacción de reserva (`ADR-002`). La Order existe en `CREATED`.
2. Publicar el mensaje `MSG-001` en la cola principal.
3. Si la publicación tiene éxito: marcar `enqueuedAt` en la Order (`AP-011`, best effort) y responder con el Order ID y estado `CREATED`.
4. Si la publicación falla definitivamente: ejecutar la transición "Fallar (encolado)" de `ADR-005` y responder con el Order ID y estado `FAILED`.

Definiciones precisas:

| Pregunta | Decisión |
|---|---|
| Cuándo es definitivo un fallo de encolado | Cuando se agotan 3 intentos de publicación (1 inicial más 2 reintentos con backoff y jitter) ante errores transitorios, o de inmediato ante un error no reintentable (acceso denegado, cola inexistente, mensaje inválido) |
| Quién lleva la Order a `FAILED` | El caso de uso de inicio de compra (`CMP-005`), en la misma solicitud, mediante la transición "Fallar (encolado)" |
| Cómo se revierte la Reservation | En la misma escritura transaccional: Order → `FAILED` con causa funcional, todos los Ticket → `AVAILABLE`, registro de auditoría. Guardas: Order en `CREATED` y sin PaymentAttempt |
| Qué recibe el cliente | Encolado correcto: respuesta de creación con Order ID y `status = CREATED`. Fallo definitivo de encolado: respuesta de creación con Order ID, `status = FAILED` y causa funcional `PROCESSING_UNAVAILABLE`. En ambos casos la Order es consultable por `API-005` |

Casos límite:

- **Publicación ambigua** (el envío expira pero el mensaje sí llegó): los reintentos pueden producir duplicados, que el consumidor tolera (`ADR-007`). Si tras agotar intentos se intenta la compensación y la guarda "sin PaymentAttempt" falla, significa que el consumidor ya empezó a procesar: no se compensa y se responde con el estado actual de la Order.
- **Doble fallo** (falla la publicación y también la compensación): se responde indisponibilidad temporal sin Order ID. La Order queda en `CREATED` sin mensaje y se cierra por expiración. Una repetición con la misma idempotency key encuentra la Order y reintenta la publicación (siguiente punto).
- **Caída del proceso entre el commit y la publicación**: la Order queda en `CREATED` sin `enqueuedAt`. Dos mecanismos la cubren: (a) si el cliente repite la solicitud con la misma idempotency key, la ruta de repetición detecta `CREATED` sin `enqueuedAt` y vuelve a publicar; (b) en cualquier caso el proceso de expiración la cierra como `EXPIRED` al cumplirse los diez minutos (`ADR-009`). No queda una reserva indefinida.

## Rationale

- `AC-003` describe dos resultados síncronos distinguibles (aceptada o fallo definitivo persistido); solo la publicación directa permite determinarlos dentro de la solicitud.
- La ventana de fallo residual no puede producir sobreventa, venta parcial ni reserva indefinida: su peor efecto es una Order que expira sin haber sido procesada. Ese costo no justifica un relay y un índice adicionales para los objetivos de esta implementación.
- El marcador `enqueuedAt` convierte al item Order en un registro de publicación pendiente sin añadir infraestructura; deja preparada la evolución a outbox completo (añadir un índice disperso y un relay) sin cambiar el modelo.

## Consequences

- El consumidor puede recibir mensajes de Orders ya terminales (publicación ambigua seguida de compensación): los descarta (`ADR-007`).
- La ruta caliente de compra añade una escritura simple (`AP-011`). Su fallo se ignora.
- En el caso de caída sin repetición del cliente, el resultado funcional es `EXPIRED` y no `FAILED`. Es una limitación declarada (`RISK-003`).
- La operación síncrona incluye una llamada a SQS dentro del presupuesto de `p95 < 1 s` (`AC-031`); los reintentos solo ocurren en la ruta de fallo.

## Risks and mitigations

| Risk | Mitigation |
|---|---|
| `RISK-003` Order `CREATED` sin mensaje tras una caída | Marcador `enqueuedAt` y republicación en la repetición; cierre por expiración; métrica de Orders expiradas sin `enqueuedAt`; evolución a outbox con relay |
| Tickets retenidos hasta diez minutos por una Order que nunca se procesó | Acotado por `BR-002`; visible en la métrica anterior |

## Items to verify

| Item | Qué comprobar | Impacto si difiere |
|---|---|---|
| Clasificación de errores del cliente SQS | Qué errores son reintentables y cuáles no en el SDK usado | Ajusta la tabla de clasificación de `ADR-016`; no cambia la decisión |
| Escritura simple concurrente con una transacción sobre el mismo item | Comportamiento de `AP-011` si el consumidor ya está transicionando la Order | Ninguno funcional: `AP-011` es best effort |
| Streams en DynamoDB Local | Soporte real | Solo relevante si se elige la Option B |

## Depends on

- `FG-004` (la ruta de repetición es la que reintenta la publicación).
- Sin dependencias de `AV-*`.
