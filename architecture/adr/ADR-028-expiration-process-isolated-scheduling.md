---
id: ADR-028
title: Expiration process with isolated scheduling of worker periodic jobs
status: ACCEPTED
priority: HIGH
supersedes: ADR-009
human_decision: CONFIRMED_WITH_CHANGE
source_ids: [FR-011, BR-002, BR-011, VAL-001, VAL-003, AC-008, AC-009, ALT-001, ERR-003, ST-002, ST-010, NFR-003]
related_adrs: [ADR-008, ADR-022, ADR-024, ADR-025, ADR-026, ADR-029, ADR-034, ADR-035, ADR-037]
---

# ADR-028 — Expiration process with isolated scheduling of worker periodic jobs

Reemplaza a ADR-009.

## Human decision applied

Respuesta humana vinculante a ADR-009 (CONFIRMED_WITH_CHANGE), incorporada íntegramente:

> Se confirma el proceso periódico en el rol worker que consulta el índice disperso por instante de expiración y ejecuta la transición condicional a `EXPIRED`, liberando conjuntamente todos los Ticket de la Order. La periodicidad y la demora máxima son las aprobadas en AV-003. Todas las instancias del worker ejecutan el proceso sin líder ni lock distribuido; las guardas de Order en `CREATED` y de `expiresAt` vencido garantizan un único efecto y que no se libere una Reservation vigente. Se descartan la expiración por TTL y el mensaje diferido por Reservation. Se añaden dos cambios.
>
> Aislamiento del ciclo de expiración. El worker ejecuta además el barrido de republicación de ADR-006, los reversos de pago de FG-003 y la limpieza de aprovisionamiento de ADR-004. Cada proceso periódico debe tener su propia planificación y su propio límite de concurrencia, de modo que la duración o el fallo de uno no retrase a los demás. La expiración es el único con un plazo estricto y no debe compartir ciclo con ningún otro.
>
> Cuarentena. Una Order marcada en cuarentena conforme a ADR-005 debe retirarse del índice de expiración, para que el proceso no la reintente en cada ciclo.
>
> Se acepta como trade-off declarado que, sin coordinación entre instancias, dos de ellas pueden intentar expirar la misma Order, y que la transacción cancelada por condición consume capacidad de escritura. El recorrido aleatorio de shards con jitter lo reduce y no se justifica un lock distribuido para eliminarlo.

Decisiones aprobadas incorporadas: `AV-003` (cada 5 s, liberación como máximo 15 s después de `expiresAt`, guarda estricta, margen de corte de 15 s); barrido de ADR-026; reversos de `FG-003` (ADR-025); limpieza de ADR-024; cuarentena de ADR-025; la expiración continúa con el circuito del Payment Mock abierto (ADR-035, ADR-008).

## Context

Un proceso debe identificar las Reservation vencidas, llevar su Order a `EXPIRED` y liberar todos sus Ticket (`FR-011`, `AC-008`) sin tocar las vigentes (`AC-009`, `VAL-003`). Es además la última garantía de que no hay reservas indefinidas. El rol `worker` aloja ahora cuatro procesos periódicos.

## Options considered

### Option A — Proceso periódico sobre índice disperso por expiración, con planificación aislada por proceso

- A favor: precisión de segundos; coste proporcional a las vencidas; ningún otro proceso puede retrasarlo.
- En contra: sondeo continuo; un planificador por proceso.

### Option B — Expiración nativa del almacenamiento (TTL)

- En contra: demora de borrado no acotada (`TO_VERIFY`), incompatible con `BR-002`; destruiría la Order consultable. Descartada.

### Option C — Mensaje diferido por Reservation

- En contra: segunda publicación con su propia ventana de fallo; seguiría necesitando un proceso de respaldo. Descartada.

### Option D — Un único planificador que ejecuta todos los procesos periódicos en secuencia

- A favor: un solo componente.
- En contra: un barrido o una purga lentos retrasan la expiración; contradice la decisión humana. Descartada.

## Decision

Se adopta la **Option A**.

### Expiración (`CMP-008`, disparada por `CMP-014`)

| Aspecto | Decisión |
|---|---|
| Detección | Query de `GSI3` rango `RESV#<shard>` (8 shards) con `GSI3SK <= ahora` (`AP-016`) |
| Acción | Lectura consistente de la Order (`AP-010`); si sigue en `CREATED`, sin cuarentena y vencida, transición "Expirar" (ADR-025) con guardas `status = CREATED`, sin cuarentena, `expiresAt <= ahora`, eliminación del bloqueo y marca de reverso si tenía PaymentAttempt |
| Periodicidad | Cada 5 s, sin solape dentro de una instancia (`AV-003`) |
| Demora máxima | 15 s entre `expiresAt` y la liberación; métrica `expiration lag` con alarma |
| Múltiples instancias | Todas ejecutan el proceso, sin líder ni lock; recorrido aleatorio de shards con jitter |
| Idempotencia | Transición condicional; una segunda ejecución falla por condición sin efectos |
| Cuarentena | Una Order en cuarentena no está en el rango `RESV#`; además la guarda "sin cuarentena" la excluye |
| Cancelación por condición de un Ticket con la Order en `CREATED` | Se aplica la cuarentena (ADR-025) |
| Circuito del Payment Mock abierto | No afecta: la expiración no llama al proveedor |

### Procesos periódicos del rol `worker`

| Proceso | Componente | Periodicidad inicial | Concurrencia propia | Plazo estricto | ADR |
|---|---|---|---|---|---|
| Expiración | `CMP-008` | 5 s | 16 transiciones en paralelo | Sí (15 s) | este ADR |
| Barrido de republicación | `CMP-023` | 10 s | 8 | No | ADR-026 |
| Reversos de pago | `CMP-024` | 10 s | 4 | No | ADR-025 |
| Limpieza y detección de estancados de aprovisionamiento | `CMP-015` | 60 s | 2 | No | ADR-024 |

Reglas de aislamiento:

1. Cada proceso tiene su propia planificación (`CMP-014` mantiene un disparador independiente por proceso) y su propio límite de concurrencia.
2. Ningún proceso espera a otro; un ciclo que no termina antes del siguiente disparo de ese mismo proceso se omite (sin solape), sin afectar a los demás.
3. El fallo de un ciclo se registra y el proceso continúa en el siguiente disparo con espera creciente acotada.
4. Los consumidores de SQS (`CMP-012`, `CMP-025`) tienen bucles propios, independientes de estos procesos (ADR-039).
5. Todos los valores son configurables; los indicados son los desplegados.

## Rationale

- Aislar planificación y concurrencia garantiza que el único proceso con plazo estricto no herede la latencia de los demás.
- La guarda `expiresAt <= ahora` hace imposible liberar una Reservation vigente aun con un índice obsoleto (`AC-009`).
- Sin coordinación entre instancias no hay punto único de fallo; el coste es trabajo duplicado ocasional (trade-off aceptado).

## Consequences

- Los Ticket de una Reservation vencida pueden seguir retenidos hasta 15 s después de `expiresAt`; ninguna confirmación es posible en ese intervalo (ADR-008).
- Cuatro disparadores independientes en cada instancia del worker.
- Si no hay ninguna instancia del worker, nada expira hasta que vuelva una (`RISK-005`).

## Risks and mitigations

| Risk | Mitigation |
|---|---|
| `RISK-005` demora de expiración | Dos instancias mínimas del worker en producción, alarma de `expiration lag`, aislamiento de ciclos, autoescalado |
| `RISK-006` desalineación de relojes | Guardas con el reloj de la instancia ejecutora; márgenes de segundos |
| Partición caliente del índice | 8 shards en el rango `RESV#` |

## Items to verify

| Item | Qué comprobar | Impacto si difiere |
|---|---|---|
| Retirada de un item de un GSI disperso al eliminar el atributo de clave, en DynamoDB Local | Paridad | Si no, más candidatas obsoletas; no afecta la corrección |
| Sincronización horaria de las tareas en AWS | Desalineación máxima | Ampliar márgenes |

## Depends on

- `AV-003`, `FG-003` (resueltos).
