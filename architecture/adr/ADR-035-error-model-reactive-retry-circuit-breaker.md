---
id: ADR-035
title: Error model, reactive retry and circuit breakers
status: ACCEPTED
priority: MEDIUM
supersedes: ADR-016
human_decision: CONFIRMED_WITH_CHANGE
source_ids: [TC-008, TC-009, NFR-003, ALT-002, ALT-004, ALT-007, ERR-002, ERR-005, ERR-006, ERR-009, BR-023, FR-009, AC-006, AC-016, AC-017, AC-027, EVAL-005, EVAL-008]
related_adrs: [ADR-008, ADR-023, ADR-024, ADR-026, ADR-027, ADR-028, ADR-029, ADR-030, ADR-032, ADR-034, ADR-037, ADR-038, ADR-039, ADR-040]
---

# ADR-035 — Error model, reactive retry and circuit breakers

Reemplaza a ADR-016.

## Human decision applied

Respuesta humana vinculante a ADR-016 (CONFIRMED_WITH_CHANGE), incorporada íntegramente:

> Se confirma el modelo de error con Problem Details, un `code` estable y `traceId`, sin trazas ni detalles técnicos; errores de dominio como jerarquía cerrada traducida a HTTP en un único punto con mapeo exhaustivo; y retry selectivo localizado en los adaptadores de salida, sin apilar reintentos donde reintenta el SDK y sin reintentar nunca un fallo de condición, una validación ni un rechazo de pago. Se descartan el cuerpo de error propio con retry genérico y las anotaciones HTTP en el dominio. Se añaden los siguientes cambios.
>
> Alineación con decisiones aprobadas. La publicación en SQS usa el presupuesto de ADR-006: 3 intentos, 500 ms por intento y 2 s en total. Se añade `ACTIVE_ORDER_EXISTS` con 409 para una segunda compra con una Order activa en el mismo Event conforme a ADR-013. `EVENT_NOT_ON_SALE` deja de ser condicional conforme a FG-005. La creación de un Event responde 202 con la ubicación de su estado de aprovisionamiento conforme a ADR-004, y su repetición con la misma `Idempotency-Key` responde 200 con cabecera de repetición y el estado actual conforme a ADR-007; `IDEMPOTENCY_KEY_REUSED` aplica también a esta operación. Un cursor de paginación inválido produce `VALIDATION_ERROR` conforme a ADR-021. Los presupuestos de la llamada de pago se mantienen.
>
> Circuit breaker. Se incorpora un circuit breaker para el Payment Mock y otro para la publicación en SQS. DynamoDB no tiene circuit breaker: es la fuente de verdad y se protege con el retry del SDK y backpressure. La composición, de adentro hacia afuera, es timeout, circuit breaker y retry: cada intento cuenta para la estadística del circuito y el retry no reintenta cuando el circuito está abierto. Solo cuentan como fallo los errores transitorios (timeout, conexión y 5xx); un rechazo de pago o un error de contrato no abren el circuito.
>
> Con el circuito del Payment Mock abierto, el consumidor de Orders y el proceso de reversos dejan de recibir mensajes, de modo que no consumen recepciones de `maxReceiveCount`. En semiabierto se procesan unos pocos mensajes de prueba y, si tienen éxito, se reanuda el consumo. La expiración continúa en su ciclo independiente conforme a ADR-009 y ADR-008.
>
> Con el circuito de publicación en SQS abierto, la API rechaza la compra síncronamente con `SERVICE_UNAVAILABLE` (503) y `Retry-After`, antes de reservar, sin crear Reservation, Order ni Order ID y sin modificar el inventario. Con el circuito cerrado o semiabierto se mantiene la ruta de Order `FAILED` por fallo definitivo de encolado conforme a ADR-006.
>
> Valores iniciales configurables. Payment Mock: ventana de las últimas 20 llamadas con un mínimo de 10, apertura con 50 % de fallos o de llamadas de más de 2 s, 15 s abierto y 3 llamadas de prueba. Publicación en SQS: ventana de las últimas 20 llamadas con un mínimo de 10, apertura con 50 % de fallos, 10 s abierto y 2 llamadas de prueba.
>
> Implementación. El retry usa operadores nativos de Reactor con backoff y jitter. El circuit breaker usa Resilience4j con su módulo de Reactor, sujeto a verificar compatibilidad con Spring Boot 4.x y Java 25; si no es compatible, se implementa un circuito mínimo propio en el adaptador. Ambos residen en los adaptadores de `infrastructure`; el caso de uso solo recibe un resultado tipado de dependencia no disponible, conforme a ADR-015. El estado del circuito es por instancia. Se publica una métrica del estado de cada circuito con alarma al abrirse. Las transiciones se prueban con tiempo virtual y se demuestran con los fallos transitorios del Payment Mock conforme a ADR-011.

Correspondencia con sucesores: ADR-004 → ADR-024, ADR-006 → ADR-026, ADR-007 → ADR-027, ADR-009 → ADR-028, ADR-011 → ADR-030, ADR-013 → ADR-032, ADR-015 → ADR-034, ADR-021 → ADR-040.

## Context

`TC-008` exige API reactiva y `TC-009` manejo reactivo de errores con retry apropiado. La consulta de Order expone una causa funcional sin detalles técnicos (`FR-009`, `AC-006`). `EVAL-005` valora resiliencia ante dependencias degradadas.

## Options considered

### Option A — Problem Details con código estable, errores tipados, retry selectivo y circuit breakers en adaptadores

- A favor: formato estándar; mapeo exhaustivo; retry y circuito localizados; dependencias degradadas no consumen reintentos ni recepciones.
- En contra: disciplina de clasificación de errores en cada adaptador; estado del circuito por instancia.

### Option B — Cuerpo propio y retry genérico en los casos de uso

- En contra: reintenta resultados de negocio y multiplica reintentos. Descartada.

### Option C — Anotaciones HTTP en el dominio

- En contra: acopla el dominio a HTTP. Descartada.

### Option D — Circuit breaker también sobre DynamoDB

- En contra: DynamoDB es la fuente de verdad; abrir un circuito no ofrece alternativa útil. Descartada por la decisión humana.

## Decision

Se adopta la **Option A**.

### Modelo de error

Problem Details con `code`, `traceId` y, en validación, `errors`. Sin trazas, clases, mensajes del SDK ni identificadores internos.

### Mapeo de resultados de dominio

| Resultado | `code` | HTTP | Operaciones |
|---|---|---|---|
| Entrada inválida (incluye `VAL-006` a `VAL-009`, capacidad > 50.000, definición inválida, 0 o más de 10 Ticket, repetidos, `Idempotency-Key` ausente o inválida, cursor inválido, sección inválida) | `VALIDATION_ERROR` | 400 | todas |
| Sin token o token inválido | `UNAUTHENTICATED` | 401 | todas |
| Autoridad insuficiente | `FORBIDDEN` | 403 | todas |
| Event inexistente o no `ENABLED` | `EVENT_NOT_FOUND` | 404 | `API-003`, `API-004`; `API-006` solo si no existe |
| Order inexistente o ajena | `ORDER_NOT_FOUND` | 404 | `API-005` |
| Algún Ticket no disponible | `TICKETS_UNAVAILABLE` | 409 | `API-004` |
| Event pasado (`startsAt <= ahora`) | `EVENT_NOT_ON_SALE` | 409 | `API-004` |
| Order activa del cliente en el mismo Event | `ACTIVE_ORDER_EXISTS` | 409 | `API-004` |
| Algún Ticket inexistente en el Event | `UNKNOWN_TICKETS` | 422 | `API-004` |
| Misma clave con contenido distinto | `IDEMPOTENCY_KEY_REUSED` | 422 | `API-001`, `API-004` |
| Cuerpo demasiado grande | `PAYLOAD_TOO_LARGE` | 413 | `API-001`, `API-004` |
| Límite de tasa | `RATE_LIMITED` | 429 con `Retry-After` | todas |
| Circuito de publicación abierto antes de reservar, o indisponibilidad temporal tras agotar reintentos | `SERVICE_UNAVAILABLE` | 503 con `Retry-After` | todas (circuito: `API-004`) |
| No clasificado | `INTERNAL_ERROR` | 500 | todas |

### Precedencia en `API-004`

1. Validación de entrada (400). 2. Idempotencia: repetición (200) o clave reutilizada (422). 3. `EVENT_NOT_FOUND` (404). 4. `EVENT_NOT_ON_SALE` (409). 5. `UNKNOWN_TICKETS` por comprobación contra la definición (422). 6. Circuito de publicación abierto (503). 7. Transacción: tras relectura de idempotencia, `ACTIVE_ORDER_EXISTS`, `UNKNOWN_TICKETS`, `TICKETS_UNAVAILABLE`; conflicto persistente (503).

### Respuestas correctas

| Operación | Situación | HTTP | Cuerpo |
|---|---|---|---|
| `API-001` | Event aceptado | 202 con `Location` de `API-006` | `eventId`, `provisioningStatus = PROVISIONING` |
| `API-001` | Repetición | 200 con `Idempotency-Replayed: true` | `eventId`, `provisioningStatus` actual |
| `API-004` | Order creada y encolada | 201 con `Location` | Order `CREATED` |
| `API-004` | Order creada con fallo definitivo de encolado | 201 con `Location` | Order `FAILED`, causa `PROCESSING_UNAVAILABLE` |
| `API-004` | Repetición | 200 con `Idempotency-Replayed: true` | Order en su estado actual |

### Causa funcional de una Order no confirmada

`REJECTED` → `PAYMENT_DECLINED`; `FAILED` por encolado → `PROCESSING_UNAVAILABLE`; `FAILED` por procesamiento → `PROCESSING_FAILED`; `EXPIRED` → `RESERVATION_EXPIRED`.

### Retry reactivo

| Punto | Estrategia |
|---|---|
| DynamoDB, transitorios | Retry del SDK, acotado; sin retry adicional; backpressure por concurrencia acotada |
| DynamoDB, conflicto transaccional | Retry de la transacción completa, máximo 2, con jitter |
| Publicación en SQS (`MSG-001`, `MSG-002`) | Timeout 500 ms por intento, 3 intentos, backoff con jitter, presupuesto total 2 s |
| Autorización de pago | Timeout 3 s por llamada, hasta 2 reintentos con backoff y jitter solo para transitorios, mismo `paymentAttemptId`, plazo limitado por `expiresAt` (ADR-008) |
| Cancelación de pago | Una llamada por ciclo del proceso de reversos con timeout 3 s; el reintento lo gobierna el backoff persistido (ADR-025) |
| Procesamiento de mensajes | Reentrega de SQS con backoff de visibilidad (ADR-029) |
| Bucles de consumo y procesos periódicos | Continúan tras un error con espera creciente acotada |

### Dónde no se reintenta

Fallo de condición; validación, autenticación, autorización; rechazo de pago; error de contrato del proveedor; con el circuito abierto; los casos de uso no se envuelven en retry; no se apilan reintentos sobre los del SDK.

### Circuit breakers

| Aspecto | Payment Mock (autorizar y cancelar) | Publicación en SQS (ambas colas) |
|---|---|---|
| Ventana | Últimas 20 llamadas, mínimo 10 | Últimas 20 llamadas, mínimo 10 |
| Apertura | 50 % de fallos o 50 % de llamadas de más de 2 s | 50 % de fallos |
| Duración abierto | 15 s | 10 s |
| Llamadas de prueba en semiabierto | 3 | 2 |
| Cuenta como fallo | Timeout, conexión, 5xx | Timeout, conexión, 5xx o error de servicio transitorio |
| No cuenta como fallo | `DECLINED`, 4xx de contrato | Error no reintentable de configuración (se clasifica como definitivo) |
| Efecto abierto | Consumidor de Orders y proceso de reversos dejan de recibir; la expiración continúa | `API-004` responde 503 con `Retry-After` antes de reservar; el barrido omite el ciclo; `API-001` no se rechaza: el Event queda en `PROVISIONING` y lo recupera la detección de estancados (ADR-024) |
| Efecto semiabierto | Se procesan solo los mensajes de prueba; éxito → cierre y reanudación | Se permiten las llamadas de prueba; la compra sigue la ruta normal y, ante fallo definitivo, `FAILED` (ADR-026) |

Composición, de adentro hacia afuera: timeout → circuit breaker → retry. Cada intento cuenta en la estadística; el retry no reintenta con el circuito abierto. Estado por instancia. Métrica del estado de cada circuito con alarma al abrirse.

### Implementación

Retry con operadores nativos de Reactor. Circuit breaker con Resilience4j y su módulo de Reactor (`TO_VERIFY` con Spring Boot 4.x y Java 25); si no es compatible, circuito mínimo propio en el adaptador con las mismas propiedades. Ambos en `infrastructure`; el caso de uso recibe un resultado tipado de dependencia no disponible (ADR-034). Pausa y reanudación de los bucles gobernadas por eventos de estado del circuito (ADR-039).

## Rationale

- El circuito evita que una dependencia caída consuma reintentos, recepciones y latencia.
- Rechazar la compra antes de reservar con SQS caído evita crear Orders que terminarían en `FAILED` en masa.
- No abrir el circuito por rechazos o errores de contrato evita confundir resultados de negocio con indisponibilidad.

## Consequences

- Dos códigos nuevos (`ACTIVE_ORDER_EXISTS`) y uno incondicional (`EVENT_NOT_ON_SALE`); 202 en `API-001`.
- El estado del circuito puede diferir entre instancias (`RISK-017`).

## Risks and mitigations

| Risk | Mitigation |
|---|---|
| `RISK-017` estado por instancia y rechazos 503 en masa | Ventanas cortas, apertura breve, métricas y alarmas; el cliente repite con la misma clave |
| Fuga de detalle técnico | Punto único de traducción |

## Items to verify

| Item | Qué comprobar | Impacto si difiere |
|---|---|---|
| Resilience4j y su módulo de Reactor con Spring Boot 4.x y Java 25 | Compatibilidad | Circuito mínimo propio |
| Problem Details en el framework reactivo de Spring Boot 4.x | Tipos y propiedades | Solo implementación |
| Política de retry por defecto del SDK de AWS | Modo e intentos | Ajuste para no apilar |

## Depends on

- `FG-004`, `FG-005` (resueltos).
