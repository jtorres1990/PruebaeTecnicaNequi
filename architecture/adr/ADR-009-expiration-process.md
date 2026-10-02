---
id: ADR-009
title: Expiration process
status: PROPOSED
priority: HIGH
source_ids: [FR-011, BR-002, BR-011, VAL-001, VAL-003, AC-008, AC-009, ALT-001, ERR-003, ST-002, ST-010, NFR-003]
related_adrs: [ADR-001, ADR-005, ADR-006, ADR-008, ADR-015]
---

# ADR-009 — Expiration process

## Context

Un proceso debe identificar las Reservation que cumplieron diez minutos sin confirmación, llevar su Order a `EXPIRED` y devolver conjuntamente todos sus Ticket a `AVAILABLE` (`FR-011`, `AC-008`). No debe tocar Reservation vigentes (`AC-009`, `VAL-003`). La especificación lo describe como proceso periódico (`AC-009`) y como actor autónomo (§4).

Además, este proceso es la red de seguridad para cualquier Order que quede en `CREATED` sin ser procesada (`ADR-006`, `ADR-010`).

## Options considered

### Option A — Proceso periódico que consulta un índice disperso por instante de expiración

El worker consulta cada pocos segundos `GSI3`, obtiene las Orders en `CREATED` con `expiresAt` vencido y ejecuta la transición "Expirar" de `ADR-005` para cada una.

- A favor: coincide con la redacción de la especificación; precisión de segundos; costo proporcional a las Reservation vencidas; funciona igual en local y en AWS; recupera el retraso acumulado tras una caída.
- En contra: sondeo continuo aunque no haya trabajo; demora de liberación igual a la periodicidad más el procesamiento.

### Option B — Expiración nativa del almacenamiento (TTL) con reacción al borrado

Poner un TTL en la Reservation y reaccionar al evento de borrado.

- A favor: sin sondeo ni índice.
- En contra: el borrado por TTL no ocurre en el instante de expiración sino con una demora no acotada en la práctica (`TO_VERIFY`, del orden de horas o más), incompatible con el límite de diez minutos de `BR-002` y con `VAL-001`; borrar el item destruiría además una Order que debe seguir siendo consultable (`MF-004`). Se descarta.

### Option C — Mensaje diferido por Reservation

Publicar al reservar un mensaje con retardo de diez minutos que dispare la expiración.

- A favor: dirigido por eventos, sin sondeo, precisión alta.
- En contra: depende de una segunda publicación con su propia ventana de fallo; si ese mensaje se pierde, la reserva queda indefinida, que es justo lo que `ALT-006` prohíbe; necesitaría igualmente un proceso periódico de respaldo. El retardo máximo permitido por la cola es `TO_VERIFY`.

## Decision

Se adopta la **Option A**.

| Aspecto | Decisión |
|---|---|
| Mecanismo de detección | Query sobre `GSI3` por cada shard: Orders en `CREATED` cuyo `GSI3SK` es menor o igual al instante actual (`AP-016`). El límite superior de la Query es el instante actual seguido de un carácter máximo, para incluir todas las Orders de ese instante |
| Acción por cada candidata | Lectura consistente de la Order (`AP-010`) y, si sigue en `CREATED`, transición "Expirar" (`AP-015`) con guardas `status = CREATED` y `expiresAt` menor o igual al instante actual |
| Periodicidad | Cada 5 segundos (valor inicial configurable), con ejecución no solapada dentro de una instancia |
| Demora máxima tolerada | Objetivo de 15 segundos entre `expiresAt` y la liberación en operación normal (`AV-003`). Se mide como métrica (`expiration lag`) y se alarma |
| Paginación y volumen | Cada ciclo procesa todas las candidatas disponibles, con concurrencia acotada |
| Múltiples instancias | Todas las instancias del worker ejecutan el proceso, sin elección de líder ni lock distribuido. La guarda de la transición garantiza un único efecto; el trabajo duplicado se limita a una transacción cancelada. Cada instancia recorre los shards en orden aleatorio y con jitter para reducir colisiones |
| Idempotencia de la liberación | La transición es condicional sobre `status = CREATED`; una segunda ejecución falla por condición y no produce efectos ni auditoría adicional (`BR-011`, `AC-025`) |
| Recuperación | Tras una indisponibilidad, el primer ciclo encuentra todas las Reservation vencidas acumuladas, porque el índice conserva las Orders en `CREATED` |
| Ubicación | Rol worker, mediante un bucle reactivo no bloqueante (`NFR-003`); el caso de uso (`CMP-008`) no conoce el mecanismo de planificación (`CMP-014`) |

El consumidor de Orders puede ejecutar la misma transición "Expirar" cuando detecta una Order en `CREATED` con `expiresAt` vencido (`ADR-008`); es el mismo caso de uso y la misma guarda.

## Rationale

- La especificación pide un proceso periódico; esta opción lo implementa sin dependencia de características con precisión insuficiente.
- La lectura eventual del índice es segura: un candidato obsoleto se descarta al fallar la guarda; un candidato que aún no aparece se recoge en el ciclo siguiente.
- La guarda `expiresAt <= ahora` dentro de la transacción hace imposible liberar una Reservation vigente (`AC-009`), incluso si el índice devolviera un candidato erróneo.
- No coordinar instancias elimina un punto único de fallo; el costo es trabajo duplicado ocasional y barato.

## Consequences

- Los Ticket de una Reservation vencida permanecen retenidos durante la demora del proceso, por encima de los diez minutos nominales. Ninguna confirmación puede ocurrir en ese intervalo (`ADR-008`). Requiere validación humana (`AV-003`).
- `GSI3` añade una escritura al crear y otra al cerrar cada Order.
- El proceso consume lecturas constantes aunque no haya trabajo (una Query vacía por shard y ciclo).
- Si no hay ninguna instancia del worker, las Reservation no expiran hasta que vuelva a haber una (`RISK-005`).

## Risks and mitigations

| Risk | Mitigation |
|---|---|
| `RISK-005` demora de expiración superior a la tolerancia | Al menos dos instancias del worker en AWS, métrica y alarma de `expiration lag`, autoescalado del worker |
| `RISK-006` desalineación de relojes | La guarda usa el reloj de la instancia que ejecuta; márgenes de segundos |
| Partición caliente del índice | 4 shards de escritura configurables |

## Items to verify

| Item | Qué comprobar | Impacto si difiere |
|---|---|---|
| Demora real del borrado por TTL en DynamoDB | Documentación vigente | Confirma el descarte de la Option B |
| Retardo máximo de entrega de un mensaje en SQS | Valor vigente | Solo relevante si se reconsidera la Option C |
| Consistencia de los GSIs dispersos en DynamoDB Local | Que la eliminación del atributo de clave retire el item del índice | Si no, las candidatas obsoletas aumentan; no afecta la corrección |

## Depends on

- `AV-003`: periodicidad y demora máxima tolerada.
- `FG-003`: marca de reverso al expirar una Order con PaymentAttempt activo.
