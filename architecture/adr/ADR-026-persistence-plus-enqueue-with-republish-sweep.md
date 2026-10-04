---
id: ADR-026
title: Persistence plus enqueue with bounded publish budget and republish sweep
status: ACCEPTED
priority: HIGH
supersedes: ADR-006
human_decision: CONFIRMED_WITH_CHANGE
source_ids: [FR-005, FR-006, ALT-006, ERR-007, AC-003, AC-022, AC-031, ST-009, MF-003, DS-006, DS-009, TC-005, TC-009]
related_adrs: [ADR-022, ADR-023, ADR-025, ADR-027, ADR-028, ADR-029, ADR-035, ADR-037]
---

# ADR-026 — Persistence plus enqueue with bounded publish budget and republish sweep

Reemplaza a ADR-006.

## Human decision applied

Respuesta humana vinculante a ADR-006 (CONFIRMED_WITH_CHANGE), incorporada íntegramente:

> Se confirma la publicación directa en SQS después del commit de la reserva, con compensación síncrona ante un fallo definitivo de encolado: el caso de uso de compra lleva la Order a `FAILED`, libera sus Ticket en la misma transacción y responde con el Order ID. Se descartan el outbox con relay como mecanismo principal y el encolado previo a la persistencia. Se añaden dos cambios.
>
> Presupuesto de tiempo. La ruta de publicación debe tener una duración acotada: como máximo 3 intentos, con un timeout por intento de 500 ms y backoff con jitter, dentro de un presupuesto total de 2 s. Agotado el presupuesto o ante un error no reintentable, el fallo es definitivo y se ejecuta la compensación. Los valores serán configurables y los indicados son los desplegados para esta implementación. Los reintentos solo ocurren en la ruta de fallo y no forman parte del objetivo de latencia de la ruta normal.
>
> Barrido de republicación. Un proceso periódico del rol worker debe localizar las Orders en `CREATED` que no tienen `enqueuedAt` y cuya creación supera un umbral de 30 s, republicar su mensaje y marcar `enqueuedAt`. El umbral debe ser superior al presupuesto de la ruta síncrona para no competir con ella. El barrido no republica una Order cuyo tiempo restante sea inferior al margen de corte que se apruebe en AV-003; esa Order se cierra por expiración. Los mensajes duplicados que produzca el barrido son tolerados por el consumidor conforme a ADR-007.
>
> Las Orders pendientes de encolado se localizan mediante un índice disperso. La Order entra en el índice en la transacción de reserva y sale al marcarse `enqueuedAt` o al alcanzar un estado terminal. Como toda Order pasa por este índice, su partition key debe usar sharding calculado, de forma consistente con ADR-001, para no concentrar las escrituras en una única partición. Se acepta como trade-off declarado que esto añade escrituras de índice a la ruta de compra.
>
> La marca `enqueuedAt` sigue siendo una escritura condicional cuyo fallo no hace fallar la compra; si no llega a aplicarse, el barrido republica y el duplicado se descarta. El barrido cubre la caída del proceso entre el commit y la publicación y el doble fallo de publicación y compensación. La expiración se mantiene como última garantía de que no existe una reserva indefinida.

Decisiones aprobadas incorporadas: margen de corte de 15 s (`AV-003`); índice `GSI4` con 8 shards (ADR-022); tolerancia a duplicados por `orderId` y estado de la Order (ADR-027); circuit breaker de publicación con rechazo 503 antes de reservar cuando está abierto (ADR-035); planificación independiente del barrido (ADR-028); apagado ordenado del `api` que completa publicaciones en curso (ADR-037).

## Context

Tras crear la Reservation y la Order se intenta encolar en SQS Standard (`FR-005`) y se retorna el Order ID sin esperar el pago, tanto si la solicitud quedó aceptada como si existe un resultado persistido por fallo definitivo de encolado (`FR-006`, `AC-003`). El fallo definitivo de encolado lleva la Order a `FAILED` y libera sus Ticket (`ALT-006`, `ERR-007`, `AC-022`). DynamoDB y SQS no comparten transacción.

## Options considered

### Option A — Publicación directa tras el commit, compensación síncrona, presupuesto acotado y barrido de republicación

- A favor: determina dentro de la solicitud los dos resultados de `AC-003`; latencia mínima; el barrido cierra la ventana de caída sin relay de streams.
- En contra: un índice disperso más y sus escrituras en la ruta de compra; duplicados ocasionales.

### Option B — Transactional outbox con relay como mecanismo principal

- A favor: publicación garantizada al menos una vez.
- En contra: la respuesta síncrona no puede informar `FAILED`; latencia de sondeo; relay adicional. Descartada por la decisión humana.

### Option C — Encolar antes de persistir

- En contra: contradice `HV-002` y `HV-018`. Descartada.

## Decision

Se adopta la **Option A**.

### Ruta síncrona (`CMP-005`)

1. Commit de la reserva (ADR-023). La Order está en `CREATED` y en `GSI4`.
2. Publicación de `MSG-001` con la política: timeout por intento de 500 ms, máximo 3 intentos, backoff con jitter, presupuesto total de 2 s; composición timeout → circuit breaker → retry (ADR-035).
3. Éxito: marca `enqueuedAt` (`AP-011`), escritura condicional (`status = CREATED` y sin `enqueuedAt`) que retira la Order de `GSI4`; su fallo se ignora. Respuesta 201 con `status = CREATED`.
4. Fallo definitivo (presupuesto agotado, error no reintentable o circuito que se abre durante los reintentos): transición "Fallar (encolado)" (ADR-025). Respuesta 201 con `status = FAILED` y causa `PROCESSING_UNAVAILABLE`.
5. Circuito abierto antes de reservar: la compra se rechaza con 503 sin reservar (ADR-035); no se llega al paso 1.

| Pregunta | Decisión |
|---|---|
| Cuándo es definitivo un fallo de encolado | Al agotar 3 intentos o el presupuesto de 2 s ante errores transitorios, o de inmediato ante un error no reintentable (acceso denegado, cola inexistente, mensaje inválido) |
| Quién lleva la Order a `FAILED` | El caso de uso de compra, en la misma solicitud |
| Cómo se revierte la Reservation | Una transacción: Order → `FAILED`, Ticket → `AVAILABLE`, auditoría, bloqueo eliminado; guardas: Order `CREATED` y sin PaymentAttempt |
| Qué recibe el cliente | 201 con Order ID y `status` `CREATED` o `FAILED`; consultable por `API-005` |

### Casos límite

- **Publicación ambigua** (el envío expira pero llegó): duplicados tolerados. Si la compensación falla porque ya hay PaymentAttempt, no se compensa y se responde con el estado actual.
- **Doble fallo** (publicación y compensación): 503 sin Order ID; la Order queda en `CREATED` sin `enqueuedAt` y en `GSI4`; el barrido la republica tras 30 s; una repetición con la misma clave la republica antes (ADR-027); en último término expira.
- **Caída del proceso entre commit y publicación**: igual que el anterior, sin respuesta al cliente.

### Barrido de republicación (`CMP-023`, rol `worker`)

| Aspecto | Decisión |
|---|---|
| Planificación | Propia, cada 10 s (configurable), con límite de concurrencia propio; no comparte ciclo con la expiración (ADR-028) |
| Detección | Query de `GSI4` por cada uno de los 8 shards, `createdAtMs < ahora − 30 s` (`AP-028`) |
| Acción | Lectura consistente de la Order; si sigue en `CREATED`, sin `enqueuedAt`, sin cuarentena y con `expiresAt − ahora ≥ 15 s`: publica `MSG-001` y marca `enqueuedAt` (`AP-011`) |
| Exclusión | Tiempo restante inferior al margen de corte: no republica; la Order se cierra por expiración |
| Múltiples instancias | Sin coordinación; duplicados tolerados por el consumidor |
| Circuito de publicación abierto | El barrido omite el ciclo |
| Registro | No es una transición de negocio: logs y métrica de Orders republicadas, sin auditoría (ADR-031) |

## Rationale

- El presupuesto de 2 s acota la ruta de fallo; la ruta normal publica en un intento.
- El umbral de 30 s es muy superior al presupuesto síncrono de 2 s, por lo que el barrido no compite con la solicitud en curso.
- El índice disperso con sharding hace que el coste del barrido dependa de las Orders pendientes y no del total.
- La expiración sigue siendo la última garantía de que no hay reservas indefinidas.

## Consequences

- Cada Order añade una escritura en `GSI4` al reservar y otra al marcar `enqueuedAt` (trade-off aceptado).
- El caso de caída sin repetición del cliente termina procesándose tras 30 s en lugar de expirar; solo expira si el tiempo restante es inferior al margen de corte (`RISK-003` reducido).
- El rol `worker` necesita permiso de envío a la cola de Orders (ADR-037).

## Risks and mitigations

| Risk | Mitigation |
|---|---|
| `RISK-003` Order en `CREATED` sin mensaje | Barrido a los 30 s, republicación en la repetición, expiración; métrica de republicadas y de expiradas sin `enqueuedAt` |
| `RISK-025` duplicados y escrituras de índice adicionales | Consumidor idempotente; índice KEYS_ONLY con sharding |

## Items to verify

| Item | Qué comprobar | Impacto si difiere |
|---|---|---|
| Clasificación de errores del cliente SQS | Reintentables y no reintentables | Ajusta la clasificación de ADR-035 |
| Timeout por intento de 500 ms frente a la latencia real de SQS y LocalStack | Medición | Ajustar el valor configurable |

## Depends on

- `FG-004`, `AV-003` (resueltos).
