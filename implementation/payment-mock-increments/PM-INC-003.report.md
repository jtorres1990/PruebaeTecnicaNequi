---
artifact: payment-mock-increment-report
increment: PM-INC-003
result: DONE
code_revision: "CODE_REPO main 6807639 + untracked payment-mock/ (no commits by this agent)"
verified_at: 2026-10-04
plan: implementation/payment-mock.implementation-plan.v1.md
review: human-review/payment-mock.implementation-plan-review.yaml (APPROVED)
---

# PM-INC-003 — Autorización determinista e idempotente

## 1. Implemented

- `domain`:
  - `StableHash`: cubo = 8 primeros bytes de SHA-256(UTF-8) como entero sin signo big-endian módulo 100 (PM-IV-008); `providerReference = "pm-" + 24 hex` del mismo SHA-256 (PM-IV-012). Nunca `hashCode()`.
  - `OutcomeSelector`: primera regla aplicable en orden efectivo → porcentaje (`PERCENTAGE_DECLINED` si cubo < porcentaje) → default (`DECLINED` usa `RULE_DECLINED`). Produce planes `Decide`, `DefinitiveError`, `TransientThenOutcome`, `Delay` (los tres últimos se conectan en PM-INC-004/005).
  - `Attempt` (estado inmutable por intento) y `AuthorizationStep`: orden de §4.2 del plan (conflicto de carga útil → repetición → unión a decisión en curso → cancelación registrada → regla → porcentaje → default); toda invocación autenticada y válida se cuenta, incluida la 422 por carga útil distinta (PM-IV-010); `ticketIds` se compara como conjunto. `AuthorizationResult`, `Cancellation` y `CancellationStatus` (los dos últimos se usan desde PM-INC-004).
- `application`: `Generation` incorpora el mapa de intentos (`ConcurrentHashMap<String, AttemptEntry>`); `PaymentService.authorize` captura la generación una vez y aplica la transición con `compute` por `paymentAttemptId` (PM-SPK-005); `ControlService.authorization` (API-108, solo lectura). En este incremento los pasos `DefinitiveError`, `TransientFailure`, `DecisionStarted` y `JoinPending` lanzan un error interno explícito (500) porque se entregan en PM-INC-004 y PM-INC-005; ninguna prueba de este incremento los alcanza.
- `web`: `PaymentController` (`API-101`), `PaymentJson` (validación de cabeceras y cuerpo, respuestas), `ControlController` amplía `API-108`. Respuesta 200: `paymentAttemptId`, `status`, `providerReference`, `reasonCode` solo en `DECLINED`, `replayed` y `cancelled` siempre presentes; `AuthorizationRecord.result` omitido sin resultado y sin `replayed` (PM-IV-011). Errores: 400 `VALIDATION_ERROR` (`Idempotency-Key` ausente, repetida, > 80 o distinta de `paymentAttemptId`; esquema; Content-Type), 422 `IDEMPOTENCY_KEY_REUSED`, 404 sin cuerpo en API-108, 400 para identificador de ruta > 80 (PM-IV-005).
- `JsonWarmUp` incluye ahora las respuestas de pago (y con ellas la primera inicialización de SHA-256) en el hilo de arranque.

## 2. Contract coverage

| Operación | Status cubiertos | Notas |
|---|---|---|
| `API-101` `POST /payments` | 200 (`APPROVED`, `DECLINED`), 400, 401, 422 (`IDEMPOTENCY_KEY_REUSED`) | 422 `SIMULATED_DEFINITIVE_ERROR`, 503 y 500 en PM-INC-005 |
| `API-108` `GET /control/authorizations/{id}` | 200, 401, 404 sin cuerpo; 400 no declarado (PM-IV-005) | El validador marca solo `validation.response.status.unknown` para el 400 aprobado, lo que se comprueba explícitamente |
| `API-110` | 204 | Ahora también olvida autorizaciones |

## 3. Spikes

| Spike | Resultado | Evidencia |
|---|---|---|
| `PM-SPK-005` (base) estrategia atómica | `CONFIRMED`: se adopta `ConcurrentHashMap.compute` por clave dentro de una generación en `AtomicReference` | Prototipo independiente (Java 25, programa de un archivo en el scratchpad, fuera de los repositorios) con la máquina de estados mínima autorizar/cancelar: `compute` por clave y CAS sobre registro inmutable. Por estrategia: 20 000 iteraciones de 2 autorizaciones + 2 cancelaciones simultáneas con barrera y 5 000 iteraciones autorizar + cancelar + reset simultáneos. Resultado: **0 violaciones** de invariantes en ambas (un único resultado y un único estado de cancelación, una sola respuesta no repetida de cada tipo, contadores exactos, solo los pares `APPROVED/REVERSED` o `ATTEMPT_CANCELLED/REGISTERED_BEFORE_CHARGE`, sin mezcla tras reset); tiempos 973 ms y 881 ms. Motivo de la elección: la función de `compute` se ejecuta exactamente una vez por llamada, lo que permite crear la decisión en curso de `LATENCY` una sola vez sin reintentos (con CAS se reevaluaría en cada reintento). La sección crítica es una función pura en memoria sin E/S. La parte de cancelación y reset dentro del proyecto con BlockHound se verifica en PM-INC-004 |

## 4. Verification

| Comando (desde `D:\Nequi\ticketing-platform\payment-mock`) | Resultado |
|---|---|
| `JAVA_HOME="D:\java\zulu25.36.205-ca-jdk25.0.4.1-win_x64" ./mvnw -B -ntp -o clean verify` | `BUILD SUCCESS` en 30,5 s; 201 pruebas, 0 fallos, 0 errores, 0 omitidas |

Pruebas nuevas:

| Clase | Qué demuestra |
|---|---|
| `StableHashTest` | 8 vectores calculados fuera de Java (.NET `SHA256` + `BigInteger`, comprobados con `sha256sum`), incluido un identificador no ASCII; equivalencia con `BigInteger` para 5 000 identificadores; independencia de `hashCode()` (`"Aa"`/`"BB"`); fronteras 0, 100, cubo = porcentaje (no rechaza) y cubo + 1 (rechaza); reparto 25 % sobre 8 000 identificadores fijos |
| `OutcomeSelectorTest` | Cada matcher (incluido `ticketId` contenido en `ticketIds`), valores distintos o con distinta capitalización no aplican, las 12 combinaciones de precedencia entre categorías, desempate por la regla más reciente (también entre Tickets distintos), regla > porcentaje > default, planes de fallo y latencia (latencia sin `finalOutcome` compone con porcentaje y default) |
| `AttemptTest` | Primera invocación decide y fija la carga útil; repetición devuelve el resultado almacenado aunque cambie la configuración; carga útil distinta (orden, evento, cliente, Tickets) → conflicto sin cambios salvo el contador; mismos Tickets en otro orden o repetidos = misma carga útil; `toString` sin `customerRef` |
| `PaymentServiceTest` | Aprobación por defecto y repetición; rechazo por regla persistente tras cambiar reglas; conflicto contado; inspección de solo lectura; reset; **G7**: 500 iteraciones × 8 hilos con barrera autorizando el mismo intento (alternando `APPROVED`/`DECLINED`): exactamente una respuesta no repetida, un único resultado, `invocations = 8` |
| `AuthorizationWebTest` | 29 casos HTTP: forma exacta de `APPROVED` (sin `reasonCode` ni `null`, sin `customerRef`), `DECLINED` con su `reasonCode`, `RULE_DECLINED` por defecto, precedencia, defaults y porcentaje con vectores, repetición con `providerReference` estable e `invocations`, conflicto 422 sin eco, 17 cuerpos inválidos → 400 no contados, `Idempotency-Key` ausente/distinta/> 80/repetida, Content-Type no JSON, longitudes en caracteres Unicode, inspección 404 sin cuerpo y sin efectos, identificador de ruta > 80, reset, 24 repeticiones HTTP concurrentes con una sola decisión |

Puertas G1–G7 verdes. G8 (informativa): instrucciones 88,4 % (2827/3199), ramas 85,8 % (205/239), líneas 89,4 % (525/587). No cubierto aún (por diseño del reparto): ramas de latencia, fallos y cancelación de `Attempt`/`PaymentService`/`PaymentController`, `Cancellation`, ramas 500 de `ApiErrorHandler`.

## 5. Deviations

- Ninguna sobre el contrato ni las decisiones aprobadas. Detalle local: los comportamientos `DEFINITIVE_ERROR`, `TRANSIENT_THEN_OUTCOME` y `LATENCY`, aunque ya pueden configurarse por `API-104`, responden 500 `INTERNAL_ERROR` hasta PM-INC-004/005 (estado intermedio entre incrementos; no se entrega a QA en este estado).

## 6. Blockers

Ninguno.

## 7. Handoff to Platform and QA

- QA: vectores del porcentaje para predecir resultados (`paymentAttemptId` → cubo; rechaza si cubo < `declinePercentage`): `a-1`→41, `pm-vector-1`→62, `pm-vector-2`→39, `pm-vector-3`→77, `pm-vector-4`→87, `pm-vector-5`→96, `8a7d2c8e-5f0e-4c39-9d2e-1f4b2a6b7c11-1`→31. `providerReference` = `pm-` + 24 primeros hex de SHA-256 del identificador (p. ej. `a-1` → `pm-2f8fe63a6224321de5d0a24c`). `API-108` cuenta toda autorización autenticada y válida (incluidas repeticiones y 422 por carga útil distinta) y excluye 400/401.
