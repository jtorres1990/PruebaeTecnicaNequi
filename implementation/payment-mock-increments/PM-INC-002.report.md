---
artifact: payment-mock-increment-report
increment: PM-INC-002
result: DONE
code_revision: "CODE_REPO main 6807639 + untracked payment-mock/ (no commits by this agent)"
verified_at: 2026-10-04
plan: implementation/payment-mock.implementation-plan.v1.md
review: human-review/payment-mock.implementation-plan-review.yaml (APPROVED)
---

# PM-INC-002 — Estado en memoria, defaults, reglas y API de control

## 1. Implemented

- `domain` (sin Spring, Reactor ni Jackson, verificado por ArchUnit): `MatchField` (orden de declaración = precedencia `orderId`, `ticketId`, `customerRef`, `eventId`), `Outcome`, `BehaviourType`, `ReasonCode` (configurables por regla: `CARD_DECLINED`, `INSUFFICIENT_FUNDS`, `RULE_DECLINED`; por defecto `RULE_DECLINED`), `Behaviour` (validación estricta por tipo de PM-IV-007; los opcionales se conservan tal como se configuraron), `OutcomeRule` (`ruleId = "rule-" + secuencia`), `RuleSet` inmutable en orden efectivo (precedencia y, dentro de cada matcher, de la más reciente a la más antigua; la regla aplicable es la primera que coincide), `Defaults` (0..100, inicial `APPROVED`/0), `MockConfiguration`, `AttemptPayload` (`ticketIds` como conjunto; `toString` sin `customerRef`), `InvalidConfigurationException`.
- `application`: `Generation` (configuración en `AtomicReference`, actualización sin bloqueo por CAS con funciones puras), `MockState` (generación actual sustituible de una sola vez y contador global de reglas que nunca se reinicia, PM-IV-006/PM-IV-014), `ControlService` (listar, crear, borrar una/todas, defaults, reset).
- `web`: `ControlController` (`API-103` a `API-107`, `API-110`), `ControlJson` (DTO y parseo propios), `JsonRequests` (validación estricta sobre árbol JSON, ver PM-SPK-006), `JsonWarmUp` (ver §5).
- `ApiErrorHandler` ejercitado ahora con 400/404/405/406 reales.

## 2. Contract coverage

| Operación | Status cubiertos | Notas |
|---|---|---|
| `API-103` `GET /control/rules` | 200, 401 | Orden efectivo por categoría y recencia; lista vacía tras reset |
| `API-104` `POST /control/rules` | 201, 400, 401 | Cada matcher y cada comportamiento; eco literal de la regla con `ruleId`; 49 cuerpos inválidos → 400 `VALIDATION_ERROR` |
| `API-105` `DELETE /control/rules` | 204, 401 | Idempotente |
| `API-106` `DELETE /control/rules/{ruleId}` | 204, 401 | Existente, repetido e inexistente → 204; un `ruleId` anterior al reset no borra reglas nuevas |
| `API-107` `PUT /control/defaults` | 200, 400, 401 | Reemplazo completo; sin `declinePercentage` → 0; fronteras 0 y 100; 15 cuerpos inválidos → 400 |
| `API-110` `POST /control/reset` | 204, 401 | Limpia reglas y defaults (autorizaciones y cancelaciones en PM-INC-003/004) |

Todas las respuestas positivas se validan contra el contrato; en las negativas (solicitud inválida a propósito) se valida que la respuesta sea conforme. Las respuestas de `API-103`/`API-104` pasan por la tolerancia PM-IV-016 y por la revalidación con combinadores resueltos.

## 3. Spikes

| Spike | Resultado | Evidencia |
|---|---|---|
| `PM-SPK-006` Jackson 3 estricto | `CONFIRMED` con la alternativa permitida | Prueba exploratoria con Jackson 3.1.5 (eliminada tras el spike): con el `JsonMapper` por defecto y también con `FAIL_ON_UNKNOWN_PROPERTIES`, `FAIL_ON_NULL_FOR_PRIMITIVES`, `FAIL_ON_TRAILING_TOKENS` y `STRICT_DUPLICATE_DETECTION`, el binding a records **acepta** `"3"`→3, `3.7`→3, `1`→`"1"` y `null` en campos no anulables; los mensajes de error incluyen nombres de clases internas. Se adopta la alternativa aprobada en el plan: validación explícita de un árbol JSON (`JsonRequests`) con un `JsonMapper` propio que rechaza claves duplicadas y tokens finales al parsear; comprobación de tipo JSON exacto, enteros sin parte fraccionaria y en rango de `long`, enumerados sensibles a mayúsculas, longitudes en code points, propiedades no declaradas. Los 64 casos negativos de `ControlApiWebTest` responden 400 `Error` sin valores de entrada, sin `Exception` ni trazas |

## 4. Verification

| Comando (desde `D:\Nequi\ticketing-platform\payment-mock`) | Resultado |
|---|---|
| `JAVA_HOME="D:\java\zulu25.36.205-ca-jdk25.0.4.1-win_x64" ./mvnw -B -ntp -o clean verify` | `BUILD SUCCESS` en 27,7 s; 133 pruebas, 0 fallos, 0 errores, 0 omitidas |

Pruebas nuevas:

| Clase | Qué demuestra |
|---|---|
| `BehaviourTest` | Combinaciones válidas e inválidas por tipo, rangos, `reasonCode` solo con `DECLINED` y solo configurables |
| `RuleSetTest` | Orden efectivo, duplicados permitidos, borrado idempotente, `ruleId`, validación de defaults, `ticketIds` como conjunto, `customerRef` ausente de `toString` |
| `ControlServiceTest` | Estado inicial; identificadores monotónicos no reutilizados tras reset; borrado idempotente; reemplazo de defaults; reset; 8 hilos × 250 creaciones concurrentes sin pérdidas ni duplicados |
| `ControlApiWebTest` | 83 casos HTTP: 9 creaciones válidas con eco literal, orden del listado, identificadores tras reset, borrados, defaults, reset, 49 reglas inválidas, 15 defaults inválidos, Content-Type no JSON / ausente / malformado → 400, cuerpo > 256 KB → 400, 405/406/404 del framework con `Error`, 40 creaciones HTTP concurrentes |
| `VolatileStateTest` | Un contexto nuevo empieza sin reglas y con defaults `APPROVED`/0 |

Puertas: G1–G6 verdes; G5 ahora verifica `domain` sin `allowEmptyShould`. G8 (informativa): instrucciones 88,0 % (1709/1942), ramas 77,8 % (119/153), líneas 87,7 % (341/389). No cubierto aún: `RuleSet.select`/`OutcomeRule.matches` y los validadores de arrays de `JsonRequests` (se usan desde PM-INC-003); ramas 500 de `ApiErrorHandler`.

## 5. Deviations

- `JsonWarmUp` (no previsto en el plan; detalle local sin efecto observable): la prueba de 40 creaciones concurrentes produjo 500 porque BlockHound detectó `Unsafe#park` en hilos Reactor dentro de `DeserializerCache._createAndCacheValueDeserializer` de Jackson (lock interno con la caché fría). Se corrigió en producción, no con una excepción de BlockHound: al terminar de crear los singletons, en el hilo de arranque, se resuelven y cachean el deserializador de árbol, el serializador de `Error` y los serializadores de las respuestas de control (incluido `List<RuleResponse>`). Desde entonces la prueba pasa en todos los builds. Sin excepciones añadidas a BlockHound.

## 6. Blockers

Ninguno.

## 7. Handoff to Platform and QA

- QA: `POST /control/reset` antes de cada escenario; reglas con exactamente un matcher; `reasonCode` de regla solo `CARD_DECLINED`, `INSUFFICIENT_FUNDS` o `RULE_DECLINED`; `GET /control/rules` devuelve el orden de evaluación; los `ruleId` (`rule-<n>`) no se reutilizan tras reset; `PUT /control/defaults` sin `declinePercentage` lo pone a 0.
- Platform: sin cambios respecto de PM-INC-001.
