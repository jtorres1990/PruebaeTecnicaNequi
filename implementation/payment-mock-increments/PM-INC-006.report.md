---
artifact: payment-mock-increment-report
increment: PM-INC-006
result: DONE
code_revision: "CODE_REPO main f39b4eb (HEAD advanced by the Backend Developer's INC-008 commit during this run) + untracked payment-mock/; payment-mock/ has 0 commits; no commits by this agent"
verified_at: 2026-10-04
plan: implementation/payment-mock.implementation-plan.v1.md
review: human-review/payment-mock.implementation-plan-review.yaml (APPROVED)
overall_status: PAYMENT_MOCK_COMPLETE
---

# PM-INC-006 — Cierre, trazabilidad y handoff

## 1. Implemented

Sin código de producción nuevo. Actividades de cierre: tres builds completos consecutivos, matriz de trazabilidad con la prueba concreta de cada fila, cobertura, búsqueda de secretos, comprobación de independencia respecto de `ticketing`, medición de arranque y memoria del jar, y handoff a Platform y QA.

## 2. Contract coverage

### 2.1 Operaciones (ubicación primaria y pruebas)

| Operación | Incremento | Pruebas principales | Status producidos y validados contra el contrato |
|---|---|---|---|
| `API-101` `POST /payments` | PM-INC-003 (+004, 005) | `AuthorizationWebTest`, `LatencyWebTest`, `FailuresWebTest`, `CancellationWebTest`, `PaymentServiceTest`, `AttemptTest` | 200, 400, 401, 422, 503 (500 inalcanzable, justificado) |
| `API-102` `POST /payments/{id}/cancellation` | PM-INC-004 | `CancellationWebTest`, `LatencyAndCancellationServiceTest`, `AttemptCancellationTest` | 200, 401 (500/503 no simulados, PM-IV-015) |
| `API-103` `GET /control/rules` | PM-INC-002 | `ControlApiWebTest` | 200, 401 |
| `API-104` `POST /control/rules` | PM-INC-002 | `ControlApiWebTest`, `BehaviourTest` | 201, 400, 401 |
| `API-105` `DELETE /control/rules` | PM-INC-002 | `ControlApiWebTest` | 204, 401 |
| `API-106` `DELETE /control/rules/{ruleId}` | PM-INC-002 | `ControlApiWebTest` | 204, 401 |
| `API-107` `PUT /control/defaults` | PM-INC-002 | `ControlApiWebTest`, `RuleSetTest` | 200, 400, 401 |
| `API-108` `GET /control/authorizations/{id}` | PM-INC-003 | `AuthorizationWebTest` | 200, 401, 404 |
| `API-109` `GET /control/cancellations/{id}` | PM-INC-004 | `CancellationWebTest` | 200, 401, 404 |
| `API-110` `POST /control/reset` | PM-INC-002 (+003, 004, 005) | `ControlApiWebTest`, `RaceConditionTest`, `FailureServiceTest`, `VolatileStateTest` | 204, 401 |
| `API-111` `GET /health` | PM-INC-001 | `HealthAndSecurityWebTest` | 200 |
| Todas | PM-INC-005 | `ContractSweepWebTest` (conjunto declarado = producido ∪ justificado) | 29 producidos + 3 justificados |

### 2.2 Comportamientos de ADR-030 y familias de prueba

| Comportamiento | Prueba que lo demuestra |
|---|---|
| Proyecto independiente, build propio, sin código compartido | `ArchitectureTest.productionCodeHasNoDependencyOnTicketing` (+ control negativo); `dependency:tree` sin `ticketing`; POM sin parent/módulo de `ticketing` |
| `APPROVED` / `DECLINED` | `AuthorizationWebTest.approvesByDefaultWithTheContractShape`, `declinesByRuleWithItsReasonCode` |
| Idempotencia secuencial de autorización | `AuthorizationWebTest.repetitionReplaysWithStableProviderReferenceAndCountsInvocations`; `AttemptTest.repetitionReplays...` |
| Idempotencia concurrente de autorización | `PaymentServiceTest.concurrentAuthorizationsOfTheSameAttemptDecideOnce` (500 × 8); `AuthorizationWebTest.concurrentRepetitionsOverHttpDecideOnce` (24 HTTP) |
| `providerReference` estable | `StableHashTest` (vectores externos); `AuthorizationWebTest` (repeticiones e inspección) |
| Carga útil incompatible | `AttemptTest.anotherPayloadIsAConflict...`; `AuthorizationWebTest.payloadConflictAnswers422AndKeepsTheAttempt` |
| Cancelación de aprobado (`REVERSED`), exactamente una vez | `CancellationWebTest.approvedAttemptIsReversedExactlyOnce` |
| Cancelación de rechazado (`VOIDED`) | `CancellationWebTest.declinedAttemptIsVoided`; `AttemptCancellationTest.declinedIsVoided` |
| Cancelación de inexistente (`REGISTERED_BEFORE_CHARGE`) | `CancellationWebTest.earlyCancellationDeclinesEveryLaterAuthorization` |
| Cancelación en tránsito (durante latencia) | `LatencyAndCancellationServiceTest.cancellationDuringTheWaitWins...`; `LatencyWebTest.cancellationDuringTheWaitWinsOverHttp` |
| Cancelación anticipada → `ATTEMPT_CANCELLED` (`AC-035`), también bajo carrera | `LatencyAndCancellationServiceTest.earlyCancellationRejectsEveryLaterAuthorization...`; `RaceConditionTest.authorizeAndCancelRace...`, `cancellationRacesTheCommitOfALatencyDecision` |
| Idempotencia de cancelación secuencial y concurrente | `LatencyAndCancellationServiceTest.cancellationStatusesAndIdempotency`; `RaceConditionTest.concurrentCancellationsFixOneStatusAndCountEveryCall` |
| Repetición informa cancelación posterior (`cancelled`) | `CancellationWebTest.approvedAttemptIsReversedExactlyOnce`, `declinedAttemptIsVoided` |
| Reglas por `orderId`, `ticketId`, `customerRef`, `eventId` | `OutcomeSelectorTest.eachMatcherSelectsItsRule`; `ControlApiWebTest.createsEveryMatcherAndBehaviour...` |
| Precedencia entre categorías y desempate | `OutcomeSelectorTest.higherCategoryWinsWhateverTheCreationOrder` (12 pares), `mostRecentRuleWinsWithinACategoryIncludingDifferentTickets` |
| Porcentaje determinista por hash estable; defaults | `StableHashTest`; `OutcomeSelectorTest.ruleWinsOverPercentageAndPercentageOverDefault`; `AuthorizationWebTest.defaultsAndPercentageAreDeterministic` |
| Error definitivo | `AttemptFailuresTest`, `FailuresWebTest.definitiveErrorAnswers422OnEveryInvocation` |
| N transitorios y resultado final (también concurrentes) | `AttemptFailuresTest`, `FailuresWebTest.exactlyNTransientFailuresThenTheStoredFinalResult`, `FailureServiceTest.concurrentInvocationsConsumeExactlyTheConfiguredFailures` |
| Latencia sin bloqueo; desconexión del llamador; unión de repeticiones | `LatencyAndCancellationServiceTest`, `LatencyWebTest`, `RaceConditionTest.concurrentRepetitionsJoinAPendingDecision` con BlockHound activo |
| Inspección de autorizaciones / cancelaciones sin efectos | `AuthorizationWebTest.inspectionAnswers404WithoutBodyAndHasNoSideEffects`; `CancellationWebTest.inspectionAnswers404WithoutBodyAndPathLimitsApply` |
| Reset atómico y en carrera | `ControlApiWebTest.resetAnswers204WithoutBody`; `RaceConditionTest.resetRacing...`, `resetDuringPendingDecisionsLeavesNothingVisible`; `LatencyAndCancellationServiceTest.resetDuringTheWait...` |
| Estado inicial en un contexto nuevo | `VolatileStateTest` |
| API key y health sin autenticación | `HealthAndSecurityWebTest` (10 operaciones × 8 credenciales inválidas, rutas inexistentes) |
| Secretos y `customerRef` ausentes de logs y respuestas | `StartupConfigurationTest.startsWithTheKeyAndNeverLogsIt`, `LogSecrecyTest`, `ApiErrorHandlerTest` |
| Validación completa contra el OpenAPI | `ContractValidator` en todas las pruebas web; `OpenApiValidatorCapabilityTest`; `ContractSweepWebTest`; `ContractCopyTest` |

### 2.3 IDs funcionales

`TC-017`, `AV-004` (PM-INC-002/003); `FR-015`, `AC-019`, `AC-020`, `ALT-005` (PM-INC-003); `AC-021`, `ALT-004`, `ERR-008` (PM-INC-005); `FR-017`, `BR-020`, `AC-023`, `AC-024`, `AC-025` lado proveedor (PM-INC-003); `FG-003`, `BR-034`, `ALT-009`, `AC-035`, `FR-023`, `BR-028`, `BR-003`, `AC-034` lado proveedor (PM-INC-004); `TC-001`, `TC-002`, `TC-003`, `TC-008`, `TC-013`, `TC-014`, `TC-015` (todos).

## 3. Spikes

Resumen final (detalle en los informes de cada incremento):

| Spike | Incremento | Resultado |
|---|---|---|
| `PM-SPK-001` Spring Boot 4.1.1 + WebFlux + Java 25 | PM-INC-001 | `CONFIRMED` |
| `PM-SPK-002` Build independiente y jar | PM-INC-001 (+ medición aquí) | `CONFIRMED` |
| `PM-SPK-003` Validación OpenAPI 3.0.3 | PM-INC-001 | `CONFIRMED` con dos limitaciones: `apiKey` no evaluado (aserciones explícitas) y defecto de `OutcomeRule` confirmado → tolerancia PM-IV-016 aplicada; nota para el Architect en PM-INC-001 §3 |
| `PM-SPK-004` Temporización reactiva | PM-INC-004 | `CONFIRMED` |
| `PM-SPK-005` Estrategia atómica | PM-INC-003 / PM-INC-004 | `CONFIRMED` (`ConcurrentHashMap.compute` por clave + generación en `AtomicReference`) |
| `PM-SPK-006` Jackson 3 estricto | PM-INC-002 | `CONFIRMED` con la alternativa permitida (validación sobre árbol JSON) |

## 4. Verification

| Comando (desde `D:\Nequi\ticketing-platform\payment-mock`) | Resultado |
|---|---|
| `JAVA_HOME="D:\java\zulu25.36.205-ca-jdk25.0.4.1-win_x64" ./mvnw -B -ntp -o clean verify` (3 veces seguidas) | 3 × `BUILD SUCCESS` (33,4 s, 33,5 s, 34,8 s); cada uno 256 pruebas, 0 fallos, 0 errores, 0 omitidas |
| `... ./mvnw -B -ntp -o dependency:tree` | Directas: `spring-boot-starter-webflux` 4.1.1 (compile); `spring-boot-starter-test`, `reactor-test` 3.8.7, `blockhound-junit-platform` 1.0.17.RELEASE, `swagger-request-validator-core` 2.46.1, `archunit-junit6` 1.5.0 (test). 0 coincidencias de `ticketing` |
| Búsqueda en `src/main`, `pom.xml`, `.mvn` de asignaciones de clave/secreto/token | Ninguna. Las únicas claves son literales ficticios de prueba en `src/test` (`test-api-key-5b1e7c2a`, `log-secret-key-3c8a1f`, …) |
| Búsqueda de `com.nequi.ticketing` / rutas a `ticketing/` | Solo el literal de la regla ArchUnit que lo prohíbe |
| `git status` en `CODE_REPO` | Solo `?? payment-mock/` de este agente; los cambios de `ticketing/` son del Backend Developer y no se tocaron; `payment-mock/` sin commits |

Distribución observada en las carreras de los tres builds (informativa, no asertada): authorize/cancel ≈ 200/200 entre `APPROVED/REVERSED` y `ATTEMPT_CANCELLED/REGISTERED_BEFORE_CHARGE`; frontera de commit 26–44 `APPROVED/REVERSED` frente a 356–374 `ATTEMPT_CANCELLED/REGISTERED_BEFORE_CHARGE`; nunca un par inválido.

Medición del jar (Windows, Zulu 25.0.4.1, `java -jar target/payment-mock.jar`, `PAYMENT_MOCK_API_KEY` y `SERVER_PORT` por entorno; carga = 5 000 autorizaciones con identificadores únicos + 1 000 cancelaciones, 32 en vuelo):

| Flags JVM | Arranque (Spring) | Memoria residente en reposo | Tras la carga | Carga | Clave en el log |
|---|---|---|---|---|---|
| por defecto | 1,50 s (proceso 1,92 s) | ≈ 173 MB | ≈ 276 MB | 1,52 s, todos 200 | 0 apariciones |
| `-Xmx128m` | 1,45 s (proceso 1,81 s) | ≈ 187 MB | ≈ 241 MB | 1,70 s, todos 200 | 0 apariciones |

G8 cobertura (informativa, sin puerta, ADR-038): instrucciones 99,1 % (3597/3628), ramas 97,3 % (252/259), líneas 98,7 % (631/639). Comportamientos no probados: ninguno observable; ramas sin cubrir: `NoSuchAlgorithmException` de SHA-256, invariante defensiva de `AuthorizationResult`, dos ramas defensivas de `JsonRequests`, una de `PaymentService`, `main`.

Criterios de `PAYMENT_MOCK_COMPLETE` (§12 del contrato del agente): API-101 a API-111 cubiertas ✔; cancelación anticipada bajo carrera ✔; idempotencia secuencial y concurrente ✔; fallos y latencia deterministas ✔; API key y ausencia de secretos ✔; handoff a Platform ✔ (§7.1); handoff a QA ✔ (§7.2).

## 5. Deviations

Acumuladas en el proyecto (ninguna cambia el contrato ni una decisión aprobada):

1. `JsonWarmUp` (PM-INC-002): calentamiento de serializadores Jackson en el hilo de arranque, para que BlockHound no detecte el lock de la caché fría en hilos Reactor. Sin excepciones de BlockHound.
2. Interpretación local de PM-IV-013 (PM-INC-004): con `LATENCY` sin `finalOutcome`, porcentaje y default se toman al llegar (como la regla). Solo difiere si se cambian los defaults durante la espera de esa autorización.
3. Estado intermedio PM-INC-003/004 (500 provisional para comportamientos aún no entregados) eliminado en PM-INC-005.

## 6. Blockers

Ninguno. No se crearon revisiones `human-review/payment-mock.implementation-review.<n>.yaml`.

Pendiente fuera del alcance de este agente: corrección 1.0.1 de `OutcomeRule` en `architecture/payment-mock.openapi.v1.yaml` por el Architect (PM-IV-016; recomendación en PM-INC-001 §3). Cuando exista, actualizar la copia del contrato y su `.sha256` y retirar la tolerancia de `ContractValidator`.

## 7. Handoff to Platform and QA

### 7.1 Platform (Dockerfile de ADR-030 y servicio de Compose de ADR-036)

| Requisito | Valor verificado |
|---|---|
| Build | Desde `payment-mock/`: `./mvnw -B clean verify` con JDK 25 (Enforcer exige Java `[25,26)` y Maven 3.9.16 vía wrapper propio); independiente del build de `ticketing`. Para una imagen sin pruebas: `./mvnw -B -DskipTests package` produce el mismo jar |
| Artefacto | `payment-mock/target/payment-mock.jar` (jar ejecutable de Spring Boot, ≈ 35,5 MB) |
| Arranque | `java -jar payment-mock.jar` sobre un runtime Java 25; sin argumentos |
| Variables | `PAYMENT_MOCK_API_KEY` **obligatoria** (sin ella o en blanco el proceso termina con `APPLICATION FAILED TO START` y el motivo, sin imprimir valores); `SERVER_PORT` opcional (por defecto 8090). La clave viene de un archivo de entorno no versionado en local y del gestor de secretos en AWS no productivo (ADR-032); nunca se registra |
| Puerto | 8090 dentro del contenedor. En este equipo el 8090 del host está ocupado (`WsToastNotification.exe`): publicar otro puerto del host si se necesita acceso local |
| Healthcheck | `GET /health` → 200 sin cuerpo y sin API key (la imagen necesita una herramienta para invocarlo: decisión de Platform) |
| Arranque medido | ≈ 1,5 s (Spring) / ≈ 1,9 s (proceso) |
| Memoria | Residente ≈ 175–190 MB en reposo; ≈ 240–280 MB tras 5 000 intentos. Con `-Xmx128m` funciona sin errores; el estado crece con los intentos y se libera con `POST /control/reset` (o reinicio). Sugerencia: límite de contenedor ≥ 384 MB |
| Sistema de archivos | No escribe en disco (solo el temporal de la JVM); compatible con raíz de solo lectura con `/tmp` escribible |
| Usuario | Sin privilegios; no requiere puertos < 1024 |
| Gestión | Sin Actuator ni puerto de gestión; sin CORS |
| Red | Solo alcanzable por `ticketing-worker` y la red de pruebas; nunca en producción (ADR-037) |

### 7.2 QA

- Antes de cada escenario: `POST /control/reset` y después reglas o defaults (`AV-004`), siempre con `X-Api-Key`.
- Reglas: exactamente un matcher (`orderId`, `ticketId`, `customerRef`, `eventId`; precedencia en ese orden; dentro de un matcher gana la regla más reciente); `GET /control/rules` lista en orden de evaluación; `ruleId` `rule-<n>` nunca reutilizado.
- Comportamientos: `APPROVE`; `DECLINE` (`reasonCode` opcional: `CARD_DECLINED`, `INSUFFICIENT_FUNDS`, `RULE_DECLINED` por defecto); `DEFINITIVE_ERROR` → 422 `SIMULATED_DEFINITIVE_ERROR` siempre; `TRANSIENT_THEN_OUTCOME` (N × 503 `SIMULATED_TRANSIENT_FAILURE` y luego `finalOutcome`; N alto = fallos sostenidos para el circuit breaker); `LATENCY` (`addedLatencyMs` ≤ 60 000; > 3000 = "sin respuesta" para el timeout de 3 s de `ticketing`; el cobro se compromete aunque el llamador corte).
- Defaults: `PUT /control/defaults` reemplaza el objeto; porcentaje determinista por cubo SHA-256 (vectores: `a-1`→41, `pm-vector-1`→62, `pm-vector-2`→39, `pm-vector-3`→77, `pm-vector-4`→87, `pm-vector-5`→96, `8a7d2c8e-5f0e-4c39-9d2e-1f4b2a6b7c11-1`→31; rechaza si cubo < porcentaje, `PERCENTAGE_DECLINED`).
- Inspección: `GET /control/authorizations/{id}` (`invocations` cuenta toda llamada autenticada y válida; `result` con `cancelled`); `GET /control/cancellations/{id}` (`received`, `cancellationStatus`, `firstReceivedAt`) para "reversado exactamente una vez"; 404 sin cuerpo si no hubo llamadas.
- Cancelación: `REVERSED` / `VOIDED` / `REGISTERED_BEFORE_CHARGE`; tras `REGISTERED_BEFORE_CHARGE` toda autorización del mismo `paymentAttemptId` responde `DECLINED/ATTEMPT_CANCELLED` (`AC-035`). Los fallos de cancelación no se simulan (PM-IV-015): detener el contenedor.
- Errores: 400 `VALIDATION_ERROR` (incluida `Idempotency-Key` ≠ `paymentAttemptId`), 401 `UNAUTHENTICATED`, 422 `IDEMPOTENCY_KEY_REUSED` / `SIMULATED_DEFINITIVE_ERROR`, 503 `SIMULATED_TRANSIENT_FAILURE`.
- El estado se pierde al reiniciar el contenedor (un reinicio entre cobro y cancelación da `REGISTERED_BEFORE_CHARGE`).
