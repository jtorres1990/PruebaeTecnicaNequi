---
id: ADR-016
title: Error model and reactive retry
status: PROPOSED
priority: MEDIUM
source_ids: [TC-008, TC-009, NFR-003, ALT-002, ALT-004, ALT-007, ERR-002, ERR-005, ERR-006, ERR-009, BR-023, FR-009, AC-006, AC-016, AC-017, AC-027, EVAL-008]
related_adrs: [ADR-002, ADR-006, ADR-007, ADR-010, ADR-011, ADR-013]
---

# ADR-016 — Error model and reactive retry

## Context

`TC-008` exige una API reactiva y `TC-009` manejo de errores reactivo con retry cuando sea apropiado. `HV-021` dejó a arquitectura la representación HTTP y el modelo de error. La consulta de Order debe exponer una causa funcional sin detalles técnicos (`FR-009`, `AC-006`).

Hay que decidir el modelo de errores, el mapeo de errores de dominio a respuestas y dónde se reintenta y dónde no.

## Options considered

### Option A — Problem Details estándar con código estable, errores de dominio tipados y retry selectivo por adaptador

Respuestas de error en el formato estándar de Problem Details para HTTP, con un campo `code` estable. Los errores de dominio son una jerarquía cerrada traducida en un único punto. El retry se aplica solo en los adaptadores de salida, por clase de error.

- A favor: formato estándar y uniforme; el cliente decide por `code`, no por texto; el mapeo exhaustivo lo verifica el compilador; el retry queda localizado y acotado.
- En contra: requiere disciplina para que ningún adaptador lance excepciones técnicas hacia el caso de uso sin clasificar.

### Option B — Cuerpo de error propio y retry genérico en los casos de uso

- A favor: libertad de formato.
- En contra: formato no estándar; un retry genérico en el caso de uso reintenta también resultados de negocio y multiplica los reintentos del SDK.

### Option C — Excepciones con anotaciones de estado HTTP en el dominio

- A favor: menos código de mapeo.
- En contra: acopla el dominio a HTTP, contrario a `TC-007`.

## Decision

Se adopta la **Option A**.

### Modelo de error de la API

Tipo de contenido de Problem Details, con los campos estándar más: `code` (código estable), `traceId` y, en validación, `errors` (lista de campo y motivo). Nunca incluye trazas, nombres de clases, mensajes del SDK ni identificadores internos.

### Mapeo de resultados de dominio a respuestas

| Resultado de dominio | `code` | Estado HTTP | Source IDs |
|---|---|---|---|
| Solicitud malformada o que incumple validaciones de entrada (incluye `VAL-006` a `VAL-009`, máximo de Ticket, `Idempotency-Key` ausente) | `VALIDATION_ERROR` | 400 | `AC-017`, `VAL-006`..`VAL-009` |
| Sin token o token inválido | `UNAUTHENTICATED` | 401 | `FR-018` |
| Autoridad insuficiente | `FORBIDDEN` | 403 | `FR-019`, `AC-028` |
| Event inexistente o no habilitado | `EVENT_NOT_FOUND` | 404 | `FR-012` |
| Order inexistente o ajena | `ORDER_NOT_FOUND` | 404 | `BR-023`, `ERR-006`, `ERR-009`, `AC-027` |
| Algún Ticket no disponible | `TICKETS_UNAVAILABLE` | 409 | `ALT-002`, `ERR-002`, `AC-016` |
| Event no en venta (depende de `FG-005`) | `EVENT_NOT_ON_SALE` | 409 | `FR-002` |
| Algún Ticket no existe en el Event | `UNKNOWN_TICKETS` | 422 | `VAL-010` |
| Misma idempotency key con contenido distinto | `IDEMPOTENCY_KEY_REUSED` | 422 | `BR-019` |
| Cuerpo demasiado grande | `PAYLOAD_TOO_LARGE` | 413 | `EVAL-008` |
| Límite de tasa superado | `RATE_LIMITED` | 429 | `EVAL-008` |
| Indisponibilidad temporal tras agotar reintentos | `SERVICE_UNAVAILABLE` | 503 con indicación de reintento | `ERR-005` |
| Error no clasificado | `INTERNAL_ERROR` | 500 | — |

Si una solicitud tiene a la vez Ticket inexistentes y no disponibles, prevalece `UNKNOWN_TICKETS`.

### Respuestas correctas de la compra

| Situación | Estado HTTP | Cuerpo |
|---|---|---|
| Order creada y encolada | 201 con ubicación del recurso | Order con `status = CREATED` |
| Order creada con fallo definitivo de encolado (`ADR-006`) | 201 con ubicación del recurso | Order con `status = FAILED` y causa |
| Repetición de una operación ya registrada (`ADR-007`) | 200 con cabecera que indica repetición | Order en su estado actual |

### Causa funcional de una Order no confirmada

Campo `failureCause` de la Order, con código y texto comprensible:

| Estado | Código de causa |
|---|---|
| `REJECTED` | `PAYMENT_DECLINED` |
| `FAILED` por fallo de encolado | `PROCESSING_UNAVAILABLE` |
| `FAILED` por fallo de procesamiento | `PROCESSING_FAILED` |
| `EXPIRED` | `RESERVATION_EXPIRED` |

### Retry reactivo

| Punto | Estrategia |
|---|---|
| DynamoDB, errores transitorios (throttling, indisponibilidad, timeout) | Reintento del propio SDK, acotado. No se añade otro reintento encima |
| DynamoDB, conflicto transaccional | Reintento reactivo de la transacción completa, máximo 2, con jitter (`ADR-002`) |
| Publicación en SQS | 3 intentos en total con backoff exponencial y jitter; después, fallo definitivo de encolado (`ADR-006`) |
| Llamada al Payment Mock | Timeout por llamada de 3 s; hasta 2 reintentos con backoff y jitter solo para errores transitorios; siempre con el mismo `paymentAttemptId` (`ADR-007`) |
| Procesamiento de un mensaje | Reintento por reentrega de SQS con backoff de visibilidad (`ADR-010`) |
| Recepción de mensajes y ciclo de expiración | El bucle continúa tras un error, con espera creciente; nunca termina |

### Dónde no se reintenta

- Un fallo de condición nunca se reintenta: es un resultado de negocio (Ticket no disponible, repetición, carrera ya resuelta).
- Errores de validación, autenticación y autorización.
- Rechazo del pago y errores de contrato del proveedor.
- Los casos de uso no envuelven su ejecución completa en un reintento; el reintento vive en el adaptador que conoce la clase del error.
- No se apilan reintentos: donde reintenta el SDK no reintenta la aplicación.
- Ninguna solicitud HTTP se reintenta dentro del servidor más allá de los presupuestos anteriores; el cliente puede repetirla con la misma idempotency key.

### No bloqueo

Todo timeout, espera y reintento usa operadores reactivos; no hay esperas bloqueantes (`NFR-003`). La concurrencia hacia DynamoDB, SQS y el proveedor está acotada.

## Rationale

- Un formato estándar con código estable permite al cliente distinguir "reintenta con la misma clave" de "elige otros Ticket" sin interpretar textos.
- Localizar el retry en los adaptadores evita los dos fallos típicos: reintentar resultados de negocio y multiplicar reintentos entre capas.
- Todas las escrituras son condicionales o idempotentes por clave, por lo que reintentarlas es seguro (`ERR-005`).
- Responder 201 en los dos resultados que crean una Order refleja que el recurso existe y es consultable (`AC-003`); el cliente distingue por `status`.

## Consequences

- Cada adaptador de salida debe clasificar sus errores en transitorio, definitivo o resultado de negocio.
- Los presupuestos de reintento se suman a la latencia solo en rutas de fallo.
- Un circuit breaker para el proveedor de pago queda como mejora productiva; no se incluye.

## Risks and mitigations

| Risk | Mitigation |
|---|---|
| Fuga de detalle técnico en errores | Punto único de traducción; error por defecto genérico |
| Tormenta de reintentos ante degradación | Presupuestos acotados, jitter, backpressure |

## Items to verify

| Item | Qué comprobar | Impacto si difiere |
|---|---|---|
| Soporte de Problem Details en el framework reactivo de Spring Boot 4.x | Nombres de tipos y propiedades | Solo afecta la implementación del traductor |
| Política de reintento por defecto del SDK de AWS | Modo, número de intentos y errores cubiertos | Ajustar para no duplicar reintentos |

## Depends on

- `FG-005`: el código `EVENT_NOT_ON_SALE` solo existe si se aprueba el rechazo de compras sobre Events pasados.
- `FG-004`: determina si una repetición tras `TICKETS_UNAVAILABLE` se reevalúa o devuelve el mismo rechazo.
- `FG-001`, `FG-002`: límites que producen `VALIDATION_ERROR`.
