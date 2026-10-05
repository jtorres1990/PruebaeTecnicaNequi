---
artifact: payment-mock-increment-report
increment: PM-INC-005
result: DONE
code_revision: "CODE_REPO main 6807639 + untracked payment-mock/ (no commits by this agent)"
verified_at: 2026-10-04
plan: implementation/payment-mock.implementation-plan.v1.md
review: human-review/payment-mock.implementation-plan-review.yaml (APPROVED)
---

# PM-INC-005 — Fallos transitorios y definitivos y contrato completo

## 1. Implemented

- `PaymentService` conecta los pasos de fallo de la máquina de estados (ya presentes en el dominio desde PM-INC-003) con las respuestas aprobadas en PM-IV-009:
  - `DEFINITIVE_ERROR` → 422 `{"code":"SIMULATED_DEFINITIVE_ERROR",...}` en cada invocación, sin almacenar resultado; cuenta en `invocations`; fija la carga útil.
  - `TRANSIENT_THEN_OUTCOME` → las invocaciones k ≤ N responden 503 `{"code":"SIMULATED_TRANSIENT_FAILURE",...}`; la N+1 decide y almacena `finalOutcome` (con su `reasonCode` o `RULE_DECLINED`); N = 0 decide en la primera; el progreso se guarda por (`paymentAttemptId`, `ruleId`) y sobrevive a cambios de regla (si se vuelve a la regla anterior, continúa su propio progreso); las repeticiones tras el resultado no consumen fallos.
  - Con una cancelación registrada y sin resultado, las autorizaciones dejan de fallar y resuelven `DECLINED/ATTEMPT_CANCELLED` (BR-034).
  - `reset` borra el progreso transitorio (está en la generación).
- Se eliminan los errores internos provisionales de PM-INC-003/004: ningún comportamiento configurable produce ya 500.

## 2. Contract coverage

Barrido completo (`ContractSweepWebTest`): lee del contrato todos los pares operación/status declarados y exige que el conjunto de escenarios ejecutados más los inalcanzables justificados sea **exactamente** el declarado (un status nuevo en el contrato rompe la prueba). 29 pares producidos y validados contra el contrato, 3 justificados:

| Operación | Producidos y validados | Inalcanzables justificados |
|---|---|---|
| `API-101` | 200, 400, 401, 422, 503 | 500: solo ante error interno inesperado (PM-IV-005); el cuerpo 500 se verifica en `ApiErrorHandlerTest` |
| `API-102` | 200, 401 | 500 y 503: no se simulan fallos de cancelación (PM-IV-015); la indisponibilidad se demuestra deteniendo el contenedor (ADR-038, escenario 2) |
| `API-103` | 200, 401 | — |
| `API-104` | 201, 400, 401 | — |
| `API-105`, `API-106`, `API-110` | 204, 401 | — |
| `API-107` | 200, 400, 401 | — |
| `API-108`, `API-109` | 200, 401, 404 | — |
| `API-111` | 200 | — |

En los escenarios 400 la respuesta es conforme y el validador también detecta la solicitud inválida. Situaciones no declaradas de PM-IV-005 cubiertas en incrementos anteriores y vigentes: 400 por identificador de ruta > 80 en `API-102`/`108`/`109`, 404/405/406 del framework con `Error`, 415 → 400, cuerpo > 256 KB → 400, 401 en rutas inexistentes.

## 3. Spikes

Ninguno nuevo (según el plan).

## 4. Verification

| Comando (desde `D:\Nequi\ticketing-platform\payment-mock`) | Resultado |
|---|---|
| `JAVA_HOME="D:\java\zulu25.36.205-ca-jdk25.0.4.1-win_x64" ./mvnw -B -ntp -o clean verify` | `BUILD SUCCESS` en 33,4 s; 256 pruebas, 0 fallos, 0 errores, 0 omitidas |

Pruebas nuevas:

| Clase | Qué demuestra |
|---|---|
| `AttemptFailuresTest` (7) | Error definitivo en cada invocación sin resultado y cancelación posterior → `REGISTERED_BEFORE_CHARGE` y `ATTEMPT_CANCELLED`; N = 0, 1, 3: exactamente N fallos, decisión y repeticiones sin consumo; progreso por intento y regla (regla nueva desde cero, vuelta a la anterior continúa, otro intento independiente); cancelación interrumpe los fallos restantes |
| `FailureServiceTest` (3) | Error definitivo contado; transitorio con reset que borra el progreso; **G7**: 300 iteraciones × 8 invocaciones simultáneas con N = 3 → exactamente 3 fallos, 1 decisión, 4 repeticiones, `invocations = 8` |
| `FailuresWebTest` (6) | 422 y 503 con cuerpo `Error` conforme; `invocations` creciente sin `result`; N = 0/1/3 por HTTP; cancelación en fase transitoria; 20 fallos sostenidos (para el circuit breaker de `ticketing`) y reset |
| `ContractSweepWebTest` (1) | Barrido de §2 |
| `ApiErrorHandlerTest` (3) | 500 `{"code":"INTERNAL_ERROR","message":"Internal error"}` sin detalles y log solo con el tipo de excepción (el mensaje con datos no aparece); mapeo de 400/404/405/otros 4xx/5xx; respuesta ya comprometida propaga el error |
| `LogSecrecyTest` (1) | Escenario completo con aplicación propia (regla por `customerRef`, autorización, conflicto 422, 400 con `customerRef` > 128 y con propiedad extra, cancelación, inspecciones, ruta inexistente, clave errónea): ni la API key ni el `customerRef` aparecen en la salida; la clave no aparece en ninguna respuesta y el `customerRef` solo en el eco de la regla de control que lo configuró (exigido por el contrato) |

Correcciones de construcción de pruebas en este incremento (no del producto ni del contrato): `containsOnly(503)` sobre lista vacía para N = 0 se cambió por `hasSize(N).allMatch(503)`; el escenario 400 de `API-104` del barrido usaba `match: {}`, que es 400 por PM-IV-007 pero no es expresable en el esquema (el validador no lo ve), y se sustituyó por una propiedad no declarada, que sí lo es.

Puertas G1–G7 verdes. G8 (informativa): instrucciones 99,1 % (3597/3628), ramas 97,3 % (252/259), líneas 98,7 % (631/639). No cubierto: `NoSuchAlgorithmException` de SHA-256 (imposible en la plataforma), invariante defensiva de `AuthorizationResult`, dos ramas defensivas de `JsonRequests` (cabecera Content-Type no parseable que el framework rechaza antes) y una de `PaymentService` (`cancelledAfterResult` sin entrada), `main`.

## 5. Deviations

Ninguna.

## 6. Blockers

Ninguno.

## 7. Handoff to Platform and QA

- QA: `AC-021` (fallo definitivo) = regla `DEFINITIVE_ERROR` → 422 en cada invocación (el adaptador de `ticketing` la trata como 4xx definitivo: `FAILED` sin reverso). Fallos transitorios sostenidos para abrir el circuito (ADR-035) = `TRANSIENT_THEN_OUTCOME` con `transientFailures` alto (p. ej. 1 000 000) → 503 continuos; "falla N veces y luego aprueba" = N concreto. El progreso es por intento y regla y se borra con reset.
