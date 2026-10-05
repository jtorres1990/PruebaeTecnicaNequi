---
artifact: payment-mock-increment-report
increment: PM-INC-004
result: DONE
code_revision: "CODE_REPO main 6807639 + untracked payment-mock/ (no commits by this agent)"
verified_at: 2026-10-04
plan: implementation/payment-mock.implementation-plan.v1.md
review: human-review/payment-mock.implementation-plan-review.yaml (APPROVED)
---

# PM-INC-004 — Cancelación, cancelación anticipada, latencia y carreras

## 1. Implemented

- `domain`: `Attempt.cancel(Instant)` → `CancellationStep`: la primera cancelación fija el estado (`REVERSED` tras `APPROVED`, `VOIDED` tras `DECLINED`, `REGISTERED_BEFORE_CHARGE` sin resultado: intento inexistente, con fallos sin resultado o con decisión en curso); las repeticiones devuelven el mismo estado con `replayed = true` y solo incrementan `received`; `firstReceivedAt` no cambia. `Attempt.completePending` compromete la decisión con latencia: si entretanto hay cancelación, `ATTEMPT_CANCELLED`. `cancelled` = la cancelación llegó después del resultado (PM-IV-011).
- `application`:
  - `PaymentService(MockState, Scheduler, Clock)`: `cancel` aplica la transición con `compute` en la generación capturada; `authorize` maneja `DecisionStarted` y `JoinPending` (PM-IV-010, PM-IV-013): la primera invocación crea, dentro de la transición atómica, una única decisión compartida `Mono.delay(addedLatencyMs, scheduler).map(commit).cache()` (perezosa, sin efectos) y la arranca después con una suscripción propia, desacoplada del llamador; las repeticiones durante la espera se unen a ella (`replayed = true`) sin reevaluar reglas ni añadir latencia; el `commit` aplica `completePending` con `computeIfPresent` en la misma generación. Ningún hilo espera: la latencia es un temporizador del planificador.
  - `ControlService.cancellation` (API-109); `CancellationReply`, `CancellationView`.
  - `PaymentMockConfiguration`: `Clock.systemUTC()` y `Schedulers.parallel()` inyectables.
- `web`: `POST /payments/{id}/cancellation` (`API-102`, sin cuerpo ni Content-Type requeridos; un cuerpo enviado se ignora), `GET /control/cancellations/{id}` (`API-109`, 404 sin cuerpo), identificador de ruta > 80 → 400 (PM-IV-005); `firstReceivedAt` en UTC con precisión de milisegundos.
- `reset` (API-110) olvida también cancelaciones y decisiones en curso: lo iniciado antes del reset termina en la generación antigua (PM-IV-014).

## 2. Contract coverage

| Operación | Status cubiertos | Notas |
|---|---|---|
| `API-102` `POST /payments/{id}/cancellation` | 200 (`REVERSED`, `VOIDED`, `REGISTERED_BEFORE_CHARGE`, con `replayed` false/true), 401; 400 no declarado (PM-IV-005) | 500/503 no se simulan (PM-IV-015); ver PM-INC-005 |
| `API-109` `GET /control/cancellations/{id}` | 200, 401, 404 sin cuerpo; 400 no declarado | |
| `API-101` | 200 tras latencia, `ATTEMPT_CANCELLED`, `cancelled = true` en repeticiones | |
| `API-108`, `API-110` | Ampliados | `result.cancelled` refleja el estado actual |

## 3. Spikes

| Spike | Resultado | Evidencia |
|---|---|---|
| `PM-SPK-004` temporización reactiva sin bloqueo | `CONFIRMED` (opción principal `cache()`; no hizo falta `Sinks.One`) | `LatencyAndCancellationServiceTest` con `VirtualTimeScheduler`: sin resultado a 499 ms y decidido a 500 ms; la cancelación a 400 ms gana (`ATTEMPT_CANCELLED`); repeticiones a 100 y 200 ms se unen y responden a la vez con `replayed = true`; la desconexión del llamador a 3 000 ms de una latencia de 3 500 ms no impide el commit (y la cancelación posterior da `REVERSED`); reset durante la espera: el iniciador recibe su resultado pero nada es visible en la nueva generación. Con `Schedulers.parallel()` real y BlockHound activo: commit sin detecciones (`realParallelSchedulerLatencyCommitsWithoutBlockingReactorThreads`, `LatencyWebTest`, `RaceConditionTest`). Por HTTP con espera real: respuesta ≥ 290 ms con latencia 300 ms y repetición inmediata; un `WebClient` que corta a 150 ms una latencia de 600 ms recibe `TimeoutException` y el servidor compromete `APPROVED` con `invocations = 1`; cancelación durante una espera de 1 500 ms → la autorización en tránsito responde `DECLINED/ATTEMPT_CANCELLED`; repetición durante la espera se une a la decisión |
| `PM-SPK-005` (completo) | `CONFIRMED` dentro del proyecto con BlockHound | `RaceConditionTest`, 400 iteraciones por escenario, alternando el orden de envío: authorize×2/cancel×2 sin regla → solo `APPROVED/REVERSED` (≈200) y `ATTEMPT_CANCELLED/REGISTERED_BEFORE_CHARGE` (≈200); con regla `DECLINE` → solo `CARD_DECLINED/VOIDED` y `ATTEMPT_CANCELLED/REGISTERED_BEFORE_CHARGE`; **carrera exacta en la frontera de commit** (un hilo avanza el tiempo virtual y ejecuta el commit mientras otro cancela): solo `APPROVED/REVERSED` (52–58) y `ATTEMPT_CANCELLED/REGISTERED_BEFORE_CHARGE` (342–348), nunca un aprobado con cancelación previa; 6 cancelaciones simultáneas → un estado, `received = 6`, una no repetida; 6 repeticiones simultáneas durante una decisión con latencia → una decisión, `invocations = 6`; reset simultáneo con autorizar y cancelar → solo estados coherentes de una única generación; reset con 50 decisiones en curso → nada visible después. Distribuciones de tres ejecuciones consecutivas registradas en la salida (`[race]`), sin aserciones probabilísticas |

## 4. Verification

| Comando (desde `D:\Nequi\ticketing-platform\payment-mock`) | Resultado |
|---|---|
| `JAVA_HOME="D:\java\zulu25.36.205-ca-jdk25.0.4.1-win_x64" ./mvnw -B -ntp -o clean verify` | `BUILD SUCCESS` en 30,9 s; 235 pruebas, 0 fallos, 0 errores, 0 omitidas |
| `... ./mvnw -B -ntp -o test -Dtest=RaceConditionTest -Djacoco.skip=true` (×3 antes del cambio de la prueba de frontera y ×2 después) | 7/7 verdes en cada ejecución |

Pruebas nuevas: `AttemptCancellationTest` (5), `LatencyAndCancellationServiceTest` (11), `RaceConditionTest` (7), `CancellationWebTest` (7: `REVERSED` exactamente una vez con `received`, `firstReceivedAt` y `cancelled` posterior; `VOIDED`; cancelación anticipada con regla `APPROVE` → `ATTEMPT_CANCELLED` en la primera y siguientes; cuerpo ignorado; 404 sin cuerpo; límites de ruta 80/81; reset; 25 carreras HTTP autorizar/cancelar), `LatencyWebTest` (4).

Puertas G1–G7 verdes (G4: sin excepciones de BlockHound). G8 (informativa): instrucciones 94,3 % (3423/3630), ramas 92,3 % (239/259), líneas 93,9 % (602/641). No cubierto aún: ramas de `DEFINITIVE_ERROR` y `TRANSIENT_THEN_OUTCOME` (PM-INC-005) y 500 de `ApiErrorHandler`.

## 5. Deviations

- Corrección de una prueba (no del producto): la primera versión de `zeroLatencyCommitsThroughTheSameDecisionPath` suponía que `VirtualTimeScheduler` difiere las tareas de retardo 0; las ejecuta de inmediato. Se corrigió la suposición; la aserción sobre el resultado comprometido se mantiene.
- Interpretación local de PM-IV-013 (sin efecto fuera de un caso límite): "la regla aplicable se toma como instantánea al llegar" se aplica a toda la selección; con `LATENCY` sin `finalOutcome`, el porcentaje y el default también se toman al llegar (prueba `latencyWithoutFinalOutcomeUsesThePercentageAndDefaultSnapshotTakenOnArrival`). Solo difiere si alguien cambia los defaults durante la espera de esa misma autorización. Se informa por transparencia; no se considera un vacío bloqueante.

## 6. Blockers

Ninguno.

## 7. Handoff to Platform and QA

- QA: "sin respuesta dentro del timeout" de `ticketing` (3 s) = regla `LATENCY` con `addedLatencyMs` > 3000; el cobro queda "en tránsito" y se compromete aunque `ticketing` corte; una cancelación antes del fin de la espera produce `REGISTERED_BEFORE_CHARGE` y la autorización en tránsito termina `DECLINED/ATTEMPT_CANCELLED`; después del fin, `REVERSED`. "Reversado exactamente una vez": `GET /control/cancellations/{id}` (`received`, `cancellationStatus`, `firstReceivedAt`). `AC-035`: `GET /control/authorizations/{id}` muestra `ATTEMPT_CANCELLED` y `invocations`.
- Platform: sin cambios.
