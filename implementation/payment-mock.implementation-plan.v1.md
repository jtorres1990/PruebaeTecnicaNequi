---
artifact: payment-mock-implementation-plan
schema_version: 1.0
feature: ticketing-event-processing
component: CMP-019
version: 1
status: READY_FOR_HUMAN_PLAN_REVIEW
generated_at: 2026-10-04

agent:
  name: payment-mock-developer
  mode: planning

sources:
  human_review:
    - human-review/ticketing.architecture-review.yaml   # APPROVED; AV-004, FG-003, ADR-011 -> ADR-030
    - human-review/ticketing.functional-review.yaml     # HV-008
  feature_spec: feature-spec/ticketing.feature-spec.v5.md
  contract:
    artifact: architecture/payment-mock.openapi.v1.yaml
    version: 1.0.0
    sha256: 5360ffeed2334612b1b493ebef1224910218abd28b520f03bd97a2a7eec05124
    last_commit: 3e5ad64
  architecture:
    - architecture/ticketing.architecture.v2.md          # READY_FOR_DEVELOPMENT
    - architecture/adr/ticketing.adr-registry.v1.md
    - architecture/adr/ADR-030-payment-mock-independent-project-with-cancellation.md
    - architecture/adr/ADR-032-security-active-order-lock.md
    - architecture/adr/ADR-034-clean-architecture-structure-independent-mock.md
    - architecture/adr/ADR-035-error-model-reactive-retry-circuit-breaker.md
    - architecture/adr/ADR-036-local-topology-v2.md
    - architecture/adr/ADR-037-aws-target-topology-v2.md
    - architecture/adr/ADR-038-test-strategy-v2.md
    - architecture/ticketing.consolidation-addendum.v1.md  # no corrige ninguna fuente del mock
  environment: implementation/ticketing.local-environment.v1.md
  consumer_context_read_only:
    - implementation/increments/INC-007.report.md
    - CODE_REPO/ticketing/infrastructure/.../adapter/out/payment (lectura de diagnóstico)

repositories:
  spec_repo: { path: 'D:\Nequi\PruebaeTecnicaNequi', branch: main, commit: bd9121e }
  code_repo: { path: 'D:\Nequi\ticketing-platform', branch: main, commit: 6807639, working_tree: clean, payment_mock_dir: absent }

counts:
  increments: 6
  spikes: 6
  implementation_validations: 16
  blocking_items: 17

gate:
  human_review: human-review/payment-mock.implementation-plan-review.yaml
  implementation_can_start: false
---

# Payment Mock — Implementation Plan v1

## 0. Precondiciones verificadas

| Comprobación | Resultado |
|---|---|
| `ticketing.architecture.v2.md`: `status: READY_FOR_DEVELOPMENT`, `development_can_start: true`, `blocking_items: 0` | Cumple |
| `ticketing.architecture-review.yaml`: `review.status: APPROVED`, `gate.pending_blocking_items: []`, `gate.development_can_start: true` | Cumple |
| ADR-030 `ACCEPTED` en el registro de ADR (sucesor de ADR-011, `SUPERSEDED`) | Cumple |
| `payment-mock.openapi.v1.yaml` existe y declara `info.version: 1.0.0` | Cumple |
| `CODE_REPO/payment-mock` sin propietario activo | Cumple: el directorio no existe; el Backend Developer trabaja solo en `CODE_REPO/ticketing/**` (INC-008) |
| Artefactos de este plan inexistentes antes de escribirlos | Cumple |

Fuentes no usadas: ADR-011 y demás ADR `SUPERSEDED`, arquitectura v1, feature-spec anteriores a v5, OpenAPI de `ticketing` como sustituto. La invocación menciona ADR-011 como contrato; por el registro de ADR, ADR-011 está reemplazado por ADR-030, que es el único que se aplica. El código de `ticketing` se leyó solo para diagnosticar la integración (§11.3).

## 1. Scope

### 1.1 En alcance

`CODE_REPO/payment-mock/**`: proyecto independiente (ADR-030, ADR-034) que implementa `CMP-019` y las operaciones `API-101` a `API-111` de `payment-mock.openapi.v1.yaml`:

| ID | Operación | Responsabilidad |
|---|---|---|
| `API-101` | `POST /payments` | Autorizar de forma determinista e idempotente |
| `API-102` | `POST /payments/{paymentAttemptId}/cancellation` | Cancelación idempotente, incluida la anticipada |
| `API-103` | `GET /control/rules` | Listar reglas en orden de precedencia |
| `API-104` | `POST /control/rules` | Crear regla |
| `API-105` | `DELETE /control/rules` | Eliminar todas las reglas |
| `API-106` | `DELETE /control/rules/{ruleId}` | Eliminar una regla (idempotente) |
| `API-107` | `PUT /control/defaults` | Resultado por defecto y porcentaje de rechazo |
| `API-108` | `GET /control/authorizations/{paymentAttemptId}` | Inspeccionar autorizaciones |
| `API-109` | `GET /control/cancellations/{paymentAttemptId}` | Inspeccionar cancelaciones |
| `API-110` | `POST /control/reset` | Restablecer atómicamente el estado volátil |
| `API-111` | `GET /health` | Liveness sin autenticación |

Fuentes funcionales cubiertas: `TC-017`, `FR-015`, `FR-017`, `FR-023` (lado proveedor), `BR-003` (lado proveedor), `BR-020`, `BR-028` (lado proveedor), `BR-034`, `AC-019` a `AC-021`, `AC-023` a `AC-025` (inspección), `AC-034` (lado proveedor), `AC-035` (autoritativo), `ALT-004`, `ALT-005`, `ALT-009`, `ERR-008`, `AV-004`, `FG-003`, `TC-001`, `TC-002`, `TC-003`, `TC-008`, `TC-013`, `TC-014`, `TC-015`.

Decisiones vinculantes conservadas literalmente:

```text
AV-004  El resultado se selecciona con reglas configuradas en el mock;
        nunca con un campo nuevo en la solicitud de compra.
FG-003  La cancelación es obligatoria, idempotente y válida antes,
        durante o después de la autorización.
AC-035  Una cancelación anterior al cobro hace rechazar todo cobro posterior
        con el mismo paymentAttemptId.
TC-017  El mock es un servicio simulado independiente con respuestas
        exitosas y fallidas deterministas.
```

### 1.2 Fuera de alcance

`CODE_REPO/ticketing/**`, `CMP-013` y su circuit breaker, Dockerfile (Platform, con los requisitos de §11.1), Docker Compose, `infra-init`, `local-idp`, `load-token-generator`, Terraform, CI/CD, README, colección `DEL-004`, E2E sobre Compose, carga `AC-029` a `AC-031` y escenarios de resiliencia completos (QA).

## 2. Toolchain

| Elemento | Propuesta | Estado |
|---|---|---|
| JDK | Zulu 25.0.4.1 en `D:\java\zulu25.36.205-ca-jdk25.0.4.1-win_x64`, seleccionado por invocación (`JAVA_HOME=... ./mvnw`) | Verificado en `ticketing.local-environment.v1.md` |
| Build | Maven Wrapper propio en `payment-mock/`, Maven 3.9.16, proyecto Maven de un solo módulo sin relación de herencia ni agregación con `ticketing` | `PM-IV-001` (ENV-001 aplica solo a `ticketing`) |
| Framework | Spring Boot 4.1.1 (`spring-boot-starter-parent`), `spring-boot-starter-webflux` (Reactor Netty), Jackson 3 del BOM | `PM-IV-001`, `PM-SPK-001` |
| Pruebas | JUnit Jupiter y Mockito gestionados por el BOM, `reactor-test`, AssertJ, soporte de pruebas de WebFlux (`WebTestClient`) | `PM-IV-002` |
| Detector de bloqueo | BlockHound 1.0.17.RELEASE con canaria | `PM-IV-002` |
| Validación de contrato | `swagger-request-validator-core` 2.46.1 sobre una copia literal del OpenAPI | `PM-IV-002`, `PM-IV-003`, `PM-SPK-003` |
| Reglas estructurales | ArchUnit 1.5.0 (`archunit-junit6`): sin dependencia de `com.nequi.ticketing..`, sin APIs bloqueantes en código de producción, núcleo sin Spring | `PM-IV-002` |
| Cobertura | JaCoCo 0.8.15, solo informe informativo, sin puerta (ADR-038) | `PM-IV-002` |
| Generación del wrapper | `maven-wrapper-plugin` 3.3.4 ejecutado con el Maven 3.8.1 instalado sobre JDK 25 y `distributionUrl` fijado a 3.9.16 (mismo procedimiento verificado por el backend en INC-001). No se copian archivos de `ticketing` | En PM-INC-001 |

Inspección del entorno (sin mutarlo):

- En `%USERPROFILE%\.m2` ya están `spring-boot-starter-parent` 4.1.1, `swagger-request-validator-core` 2.46.1, BlockHound 1.0.17.RELEASE, JaCoCo 0.8.15 y ArchUnit (junit6). Falta `spring-boot-starter-webflux` 4.1.1: se resolverá contra Maven Central en PM-INC-001 (acceso verificado).
- No existe ninguna herramienta de build previa en `payment-mock/` (el directorio no existe).
- El puerto 8090 está documentado como puerto del servicio por el `servers` del contrato (`http://payment-mock:8090`) y por el informe de INC-007. **En este host el puerto 8090 está ocupado** (`LISTENING`, PID 7728, `WsToastNotification.exe`). No afecta a las pruebas (puerto aleatorio) ni al puerto interno del contenedor; afecta a la publicación del puerto al host en Compose y a ejecutar el jar en 8090 localmente (§10, §11.1).
- Variables de entorno del mock: no hay ninguna documentada todavía; se proponen en `PM-IV-004`.

Comando de build del proyecto (desde `D:\Nequi\ticketing-platform\payment-mock`):

```bash
JAVA_HOME="D:\\java\\zulu25.36.205-ca-jdk25.0.4.1-win_x64" ./mvnw -B clean verify
```

## 3. Project layout

Proyecto Maven independiente. Paquete raíz `com.nequi.paymentmock` (inglés, `TC-013`). Los nombres internos son detalle local; las capas se separan por paquete (ADR-034 no impone módulos al mock).

```text
payment-mock/
  pom.xml                         # parent spring-boot-starter-parent 4.1.1; sin referencia a ticketing
  mvnw, mvnw.cmd, .mvn/wrapper/   # Maven 3.9.16
  src/main/java/com/nequi/paymentmock/
    PaymentMockApplication        # arranque
    domain/                       # reglas, comportamientos, selección determinista, máquina de estados
                                  # del intento, hash estable; sin Spring ni Reactor
    application/                  # servicios de autorización, cancelación, control e inspección;
                                  # almacén en memoria con generaciones; Reactor para latencia
    web/                          # rutas, DTOs propios, validación estricta, filtro X-Api-Key,
                                  # traducción de errores, health
    config/                       # propiedades (puerto, API key), reloj y planificador inyectables
  src/main/resources/application.yaml
  src/test/java/com/nequi/paymentmock/...
  src/test/resources/contracts/payment-mock.openapi.v1.yaml (+ .sha256)
```

Artefacto ejecutable: `payment-mock/target/payment-mock.jar` (nombre final fijo para Platform). Sin Actuator ni endpoints de gestión (`PM-IV-004`).

## 4. Contract interpretation

### 4.1 Definido de forma unívoca por las fuentes

| Tema | Comportamiento | Fuente |
|---|---|---|
| Seguridad | `X-Api-Key` en todas las operaciones salvo `GET /health`; 401 con `Error` si falta o es inválida; nunca se registra | OpenAPI `security`, ADR-032 |
| Idempotency-Key | Obligatoria en `API-101`, máx. 80, debe ser igual a `paymentAttemptId` | OpenAPI `API-101` |
| Cuerpo de autorización | `additionalProperties: false`; requeridos los cinco campos; `orderId`/`eventId` `uuid`; `customerRef` ≤ 128; `ticketIds` 1..10, cada uno ≤ 24 | OpenAPI `AuthorizationRequest` |
| Precedencia de reglas | `orderId`, `ticketId`, `customerRef`, `eventId`; después porcentaje determinista; después resultado por defecto (inicial `APPROVED`) | ADR-030, OpenAPI `API-104` |
| Comportamientos | `APPROVE`, `DECLINE`, `DEFINITIVE_ERROR`, `TRANSIENT_THEN_OUTCOME`, `LATENCY` | ADR-030, OpenAPI |
| Exactamente un matcher por regla | 0 o ≥ 2 matchers es entrada inválida (400) | OpenAPI `OutcomeRuleInput.match` |
| Idempotencia de autorización | Un único resultado por `paymentAttemptId`; repetición devuelve el resultado almacenado sin contar un pago nuevo e informa si fue cancelado después | ADR-030 regla 1, OpenAPI |
| Idempotencia de cancelación | Mismo `cancellationStatus` en repeticiones, sin contar una cancelación nueva | ADR-030 regla 2, OpenAPI |
| Estados de cancelación | Aprobado → `REVERSED`; rechazado → `VOIDED`; inexistente o en tránsito (sin resultado) → `REGISTERED_BEFORE_CHARGE` | OpenAPI `CancellationResult`, ADR-030 |
| Cancelación anticipada | Con cancelación registrada y sin resultado, toda autorización posterior y la autorización en tránsito resuelven `DECLINED` con `ATTEMPT_CANCELLED` | ADR-030 regla 3, `BR-034`, `ALT-009`, `AC-035` |
| Inspección | `API-108`: `invocations` y `result`; 404 si no se recibió autorización. `API-109`: `received`, `cancellationStatus`, `firstReceivedAt`; 404 si no se recibió cancelación. Inspeccionar no modifica estado | OpenAPI |
| Reset | Limpia reglas, defaults (vuelven a `APPROVED`), autorizaciones y cancelaciones; 204 | OpenAPI `API-110` |
| Estado | Solo en memoria; se pierde al reiniciar; reinicio entre cobro y cancelación = intento inexistente | ADR-030 |
| Sin importe | No hay importe, moneda ni datos de tarjeta | ADR-030 regla 4 |
| Fallos simulados | 5xx (`500`/`503` declarados) o sin respuesta dentro del timeout del llamador (por latencia); nunca se almacenan como `APPROVED`/`DECLINED` | OpenAPI `API-101`, ADR-030 |
| Clasificación en el consumidor | `APPROVED` confirma; `DECLINED` rechaza; 4xx definitivo sin reverso; 5xx/timeout transitorio | ADR-030, ADR-035 |

### 4.2 Modelo de estado por intento (interpretación local sin efecto observable propio)

Cada `paymentAttemptId` tiene un registro inmutable que se sustituye atómicamente: carga útil fijada, número de invocaciones, progreso transitorio, decisión en curso, resultado almacenado y cancelación (estado, recibidas, primera recepción). Todas las transiciones de un intento (inicio de autorización, decisión, cancelación) se linealizan en una única frontera atómica por clave; la estrategia concreta (`ConcurrentHashMap.compute` o CAS sobre referencia inmutable) la decide `PM-SPK-005`. El estado completo (reglas, defaults, intentos) vive en una generación sustituible de una sola vez para `reset` (`PM-IV-014`).

Orden de evaluación de una autorización autenticada y válida (pasos 3 a 7 se derivan de §4.1; el lugar del paso 2 lo fija `PM-IV-010`):

1. Validación de cabeceras y cuerpo → 400 (no cuenta como invocación, `PM-IV-010`).
2. Carga útil distinta de la fijada para el intento → 422 (`PM-IV-010`).
3. Resultado almacenado → repetición (`replayed`, `cancelled`, `PM-IV-011`).
4. Decisión en curso (latencia) → se une a la misma decisión (`PM-IV-010`, `PM-IV-013`).
5. Cancelación registrada → `DECLINED`/`ATTEMPT_CANCELLED`, almacenado.
6. Regla aplicable (instantánea de reglas al llegar) → comportamiento.
7. Sin regla → porcentaje determinista (`PM-IV-008`) → resultado por defecto.

Invariantes que verifican las pruebas de carrera: como máximo un resultado por intento; nunca coexisten un `APPROVED` almacenado y `REGISTERED_BEFORE_CHARGE`; nunca coexisten `ATTEMPT_CANCELLED` y `REVERSED`/`VOIDED`; `providerReference` constante; `invocations` y `received` iguales al número de llamadas contabilizables enviadas.

### 4.3 Vacíos observables (todos elevados a `PM-IV`)

| Vacío | PM-IV |
|---|---|
| Herramienta de build, librerías de prueba y acceso al contrato | `PM-IV-001`, `PM-IV-002`, `PM-IV-003` |
| Variables de entorno, arranque sin API key, Actuator, cuerpo de `/health` | `PM-IV-004` |
| Reparto 400/422, códigos `Error.code`, situaciones no declaradas (longitud de parámetros de ruta en `API-102`/`108`/`109`, 404/405/406/415, cuerpo excesivo), 404 de inspección sin cuerpo | `PM-IV-005` |
| Identificador de regla, orden del listado, desempate dentro de una categoría (incluido `ticketId` con varios Ticket), duplicados | `PM-IV-006` |
| Validaciones cruzadas por tipo de comportamiento; `reasonCode` admitidos y por defecto (regla `DECLINE`, `defaultOutcome = DECLINED`) | `PM-IV-007` |
| Algoritmo del porcentaje, fronteras 0/100, semántica de `PUT` sin `declinePercentage` | `PM-IV-008` |
| Status y cuerpo de `DEFINITIVE_ERROR` y de los fallos transitorios; progreso por regla/intento; cancelación durante la fase transitoria | `PM-IV-009` |
| Carga útil incompatible con el mismo `paymentAttemptId`; repeticiones concurrentes durante una decisión en curso; qué llamadas cuentan como invocación | `PM-IV-010` |
| Semántica de `replayed` y `cancelled`; omisión frente a `null` en campos opcionales | `PM-IV-011` |
| `providerReference` estable (también en `DECLINED` y `ATTEMPT_CANCELLED`) | `PM-IV-012` |
| Resultado y composición de `LATENCY`; momento de la decisión; desconexión del llamador | `PM-IV-013` |
| `reset` concurrente con operaciones en vuelo | `PM-IV-014` |
| Fallos simulados en la cancelación (`API-102` declara 500/503 pero ninguna regla puede seleccionarlos) | `PM-IV-015` |
| Esquema `OutcomeRule` (`allOf` con `additionalProperties: false`) posiblemente insatisfacible | `PM-IV-016` |

## 5. Quality gates

| Puerta | Criterio | Desde |
|---|---|---|
| G1 Build | `./mvnw -B clean verify` en verde con Java 25; Maven Enforcer exige Java `[25,26)` y Maven `[3.9.16,3.9.17)` | PM-INC-001 |
| G2 Pruebas | Unitarias del núcleo, servicios con tiempo virtual, capa web con `WebTestClient` sobre la aplicación arrancada en puerto aleatorio; 0 fallos, 0 omitidas | PM-INC-001 |
| G3 Contrato | Toda solicitud y respuesta de las pruebas web se valida contra la copia del OpenAPI; la prueba falla ante cualquier violación no aprobada; la copia se compara por SHA-256 con el original | PM-INC-001 |
| G4 No bloqueo | BlockHound instalado en todas las pruebas reactivas, con canaria que demuestra la detección; ninguna excepción añadida sin registrarla en el informe | PM-INC-001 |
| G5 Estructura | ArchUnit: ningún tipo depende de `com.nequi.ticketing..`; ninguna llamada a `Thread.sleep`, `Object.wait`, `block*()` en producción; `domain` sin Spring ni Reactor | PM-INC-001 |
| G6 Seguridad | 401 en todas las operaciones protegidas sin clave o con clave inválida; `/health` sin clave; logs capturados en pruebas sin API key ni `customerRef`; ningún secreto versionado (búsqueda en el árbol en cada informe) | PM-INC-001 |
| G7 Concurrencia | Cada prueba de carrera repite el escenario cientos de iteraciones en una ejecución y comprueba invariantes, no tiempos; PM-INC-006 ejecuta el build completo tres veces seguidas | PM-INC-003 |
| G8 Cobertura | Informe JaCoCo informativo en cada informe de incremento; sin puerta; se enumeran ramas relevantes no probadas | PM-INC-001 |

No hay perfil de integración ni contenedores: el mock no tiene dependencias externas.

## 6. Spikes

| ID | Fuente | Pregunta | Método | Criterio de éxito | Alternativa permitida | Incremento |
|---|---|---|---|---|---|---|
| `PM-SPK-001` | `TC-001`..`TC-003`, `TC-008`, RISK-012 | ¿Spring Boot 4.1.1 + WebFlux (Reactor Netty) arranca sobre Java 25 con Jackson 3, sin Actuator, en un puerto configurable? | Aplicación mínima con `/health`; prueba con `WebTestClient` y arranque del jar | Prueba verde; jar arranca y responde `/health` | Otra versión 4.x del BOM (cambio menor registrado en el informe); si ninguna funciona, `PM-IV` bloqueante | PM-INC-001 |
| `PM-SPK-002` | ADR-030, ADR-034, ADR-036 | ¿El build independiente con Maven Wrapper 3.9.16 produce un jar ejecutable sin ninguna referencia a `ticketing`, repetible en modo offline tras la primera resolución? | `./mvnw -B clean verify` dos veces (la segunda con `-o`); `java -jar` con variables de entorno; medir tiempo de arranque y memoria residente | Ambos builds verdes; jar arranca; `dependency:tree` sin artefactos de `ticketing` | Ajustar el plugin o la versión del wrapper dentro de 3.9.x | PM-INC-001 |
| `PM-SPK-003` | OpenAPI 3.0.3, ADR-038 | ¿`swagger-request-validator-core` valida solicitudes y respuestas de `WebTestClient` incluidos `additionalProperties`, `uuid`, `date-time`, enumerados, `nullable` y `allOf` de `OutcomeRule`? ¿Qué no valida (esquema `apiKey`)? | Pruebas de control positivas y negativas por cada característica; respuesta de `OutcomeRule` con y sin resolución de combinadores | Matriz de capacidades documentada; violaciones negativas detectadas; confirmación o descarte del defecto de `PM-IV-016` | Comprobaciones explícitas en las pruebas para lo que el validador no cubra (como hizo INC-007 con `apiKey`) | PM-INC-001 (validación del defecto antes de PM-INC-002) |
| `PM-SPK-004` | `TC-003`, `NFR-003`, ADR-030 regla 3 | ¿La latencia con `Mono.delay` sobre un planificador inyectable es determinista con tiempo virtual, no bloquea con BlockHound activo, y una decisión en curso sobrevive a la cancelación de la suscripción del llamador? | Pruebas con `StepVerifier.withVirtualTime`; prueba web con latencia corta real y cliente que corta a mitad; cancelación durante la espera | Tiempo virtual controla la decisión; sin detecciones de BlockHound; la decisión se compromete tras la desconexión (si `PM-IV-013` lo aprueba); la cancelación gana durante la espera | Sumidero compartido (`Sinks.One`) en lugar de `cache()`; si nada cumple, `PM-IV` bloqueante | PM-INC-004 |
| `PM-SPK-005` | ADR-030 reglas 1 a 3, `FG-003` | ¿Qué estrategia atómica por `paymentAttemptId` ofrece una única frontera linealizable para autorizar, decidir y cancelar, y un `reset` sin mezcla visible? | Dos prototipos (`compute` por clave y CAS sobre registro inmutable, ambos dentro de una generación con `AtomicReference`); prueba de estrés con barreras y miles de iteraciones; BlockHound activo | Cero violaciones de invariantes (§4.2) en todas las iteraciones; sin detecciones de bloqueo | La otra estrategia; si ninguna cumple, `PM-IV` bloqueante | PM-INC-003 (base) y PM-INC-004 (cancelación y reset) |
| `PM-SPK-006` | OpenAPI `additionalProperties: false`, tipos y enumerados | ¿Cómo se configura Jackson 3 en Spring Boot 4.1.1 para rechazar propiedades desconocidas, coerciones de tipo (`"3"` por `3`), `null` en campos no anulables y claves duplicadas, sin filtrar detalles internos en el error? | Pruebas de capa web con cuerpos negativos para `API-101`, `API-104`, `API-107` | Todos los casos negativos responden 400 con `Error` sin trazas ni valores de entrada | Validación explícita de un árbol JSON en la capa web | PM-INC-002 |

## 7. Increments

La partición ajusta la recomendada en un punto: la latencia y la decisión en curso pasan a PM-INC-004, porque la cancelación "durante una autorización en tránsito" no puede probarse sin ellas. PM-INC-005 conserva los fallos transitorios y definitivos y el barrido completo del contrato.

### PM-INC-001 — Build independiente, bootstrap, health, seguridad y puertas de prueba

- Objetivo: proyecto compilable y ejecutable con todas las puertas activas desde el inicio.
- Operaciones: `API-111`; filtro `X-Api-Key` para todas las demás rutas (respuesta 401 verificable aunque la operación aún no exista, según `PM-IV-005`).
- Fuentes: `TC-001`, `TC-002`, `TC-003`, `TC-008`, `TC-013`, `TC-014`, `TC-017`, ADR-030 (proyecto), ADR-032 (secretos, CORS deshabilitado, solo salud externa), ADR-034 (regla 7), ADR-036, ADR-038.
- Spikes: `PM-SPK-001`, `PM-SPK-002`, `PM-SPK-003`.
- Pruebas: arranque del contexto; `/health` 200 sin clave y con clave; 401 sin clave, con clave vacía y con clave incorrecta, cuerpo `Error` válido contra el contrato; comparación constante de la clave; arranque fallido sin API key (`PM-IV-004`); copia del contrato con SHA-256; canaria de BlockHound; reglas ArchUnit con control negativo; logs sin la API key.
- Terminado cuando: G1 a G6 y G8 verdes; jar ejecutable arranca con las variables propuestas; informe con la matriz de capacidades del validador.
- Depende de: aprobación del plan, `PM-IV-001` a `PM-IV-005`.

### PM-INC-002 — Estado en memoria, defaults, reglas y API de control

- Objetivo: API de control completa sobre el estado volátil, sin autorización todavía.
- Operaciones: `API-103`, `API-104`, `API-105`, `API-106`, `API-107`, `API-110` (reglas y defaults; la limpieza de autorizaciones y cancelaciones se amplía en PM-INC-003/004).
- Fuentes: `AV-004`, `TC-017`, ADR-030 (selección y API de control).
- Spikes: `PM-SPK-006`.
- Pruebas: creación por cada tipo de matcher y comportamiento; rechazo de 0 o 2+ matchers, propiedades adicionales, tipos y enumerados inválidos y combinaciones no permitidas (`PM-IV-007`); orden del listado y `ruleId` (`PM-IV-006`); borrado individual idempotente (existente e inexistente: 204); borrado total; `PUT /control/defaults` con y sin `declinePercentage`, fronteras 0 y 100 y fuera de rango; reset devuelve reglas vacías y defaults `APPROVED`/0; creación concurrente de reglas sin pérdidas; respuestas validadas contra el contrato (con la decisión de `PM-IV-016`).
- Terminado cuando: operaciones conformes al contrato y a las respuestas de `PM-IV-005` a `PM-IV-008` y `PM-IV-016`.
- Depende de: PM-INC-001, `PM-IV-006`, `PM-IV-007`, `PM-IV-008`, `PM-IV-016`.

### PM-INC-003 — Autorización determinista e idempotente

- Objetivo: `API-101` con `APPROVE`/`DECLINE`, porcentaje y defaults; inspección de autorizaciones.
- Operaciones: `API-101`, `API-108`; amplía `API-110`.
- Fuentes: `FR-015`, `FR-017`, `BR-020`, `AC-019`, `AC-020`, `AC-023`, `AC-024`, `ALT-005`, ADR-027 (idempotencia ante el proveedor), ADR-030 regla 1 y selección 1 a 3.
- Spikes: `PM-SPK-005` (base sin cancelación).
- Pruebas: cada matcher; `ticketId` contenido en `ticketIds`; precedencia entre categorías (las 4×3 combinaciones relevantes); desempate dentro de la categoría; porcentaje con vectores fijos y fronteras 0/100; default `APPROVED` inicial y `DECLINED` configurado con su `reasonCode`; validación de cabeceras y cuerpo (Idempotency-Key ausente, > 80, distinta de `paymentAttemptId`; `uuid`; longitudes; cardinalidad; propiedades adicionales); repetición secuencial (`replayed`, `providerReference` estable, `invocations`); carga útil incompatible (`PM-IV-010`); N autorizaciones concurrentes con un único resultado; la regla modificada después de decidir no altera el resultado almacenado; `API-108` 200/404 sin efectos secundarios; reset limpia autorizaciones.
- Terminado cuando: G7 incluido; invariantes de §4.2 sin violaciones.
- Depende de: PM-INC-002, `PM-IV-008`, `PM-IV-010`, `PM-IV-011`, `PM-IV-012`.

### PM-INC-004 — Cancelación, cancelación anticipada, latencia y carreras

- Objetivo: `API-102` y `API-109`; comportamiento `LATENCY` con decisión en curso; todas las carreras authorize/cancel/reset.
- Operaciones: `API-102`, `API-109`; `API-101` (`LATENCY`); amplía `API-108` y `API-110`.
- Fuentes: `FG-003`, `BR-034`, `ALT-009`, `AC-035`, `AC-034` (lado proveedor), `FR-023`, `BR-028` (lado proveedor), ADR-030 reglas 2 y 3, ADR-038 (cancelación anticipada; reversado exactamente una vez).
- Spikes: `PM-SPK-004`, `PM-SPK-005` (completo).
- Pruebas:
  - `REVERSED` tras `APPROVED`; `VOIDED` tras `DECLINED`; `REGISTERED_BEFORE_CHARGE` para intento inexistente y para intento sin resultado; repetición con el mismo estado y `replayed`; `received` y `firstReceivedAt` (reloj inyectado); `API-109` 404 sin cancelación.
  - Cancelación antes de autorizar → `DECLINED`/`ATTEMPT_CANCELLED` en la primera y en todas las autorizaciones posteriores, también con reglas `APPROVE` y default `APPROVED` (`AC-035`).
  - Cancelación durante `LATENCY` (tiempo virtual y una prueba web con latencia real corta) → la autorización en tránsito resuelve `ATTEMPT_CANCELLED`; cancelación `REGISTERED_BEFORE_CHARGE`.
  - Autorización y cancelación simultáneas repetidas cientos de veces: solo los pares `(APPROVED, REVERSED)` o `(ATTEMPT_CANCELLED, REGISTERED_BEFORE_CHARGE)` (con regla `DECLINE`: `(RULE_DECLINED, VOIDED)` o `(ATTEMPT_CANCELLED, REGISTERED_BEFORE_CHARGE)`).
  - N cancelaciones concurrentes: un estado, `received = N`, una sola no repetida.
  - Repeticiones concurrentes durante una decisión en curso se unen a ella (`PM-IV-010`); desconexión del llamador durante la latencia (`PM-IV-013`); latencia sin bloqueo (BlockHound).
  - `reset` concurrente con autorizaciones en curso y con cancelaciones: tras el reset no hay mezcla de estado (`PM-IV-014`).
  - Repetición de autorización tras `REVERSED` → resultado almacenado con `cancelled` según `PM-IV-011`.
- Terminado cuando: cancelación anticipada demostrada bajo carrera; invariantes sin violaciones; G7.
- Depende de: PM-INC-003, `PM-IV-011`, `PM-IV-013`, `PM-IV-014`, `PM-IV-015`.

### PM-INC-005 — Fallos transitorios y definitivos y contrato completo

- Objetivo: `DEFINITIVE_ERROR` y `TRANSIENT_THEN_OUTCOME`; barrido de contrato de las once operaciones.
- Operaciones: `API-101` (fallos), las once en el barrido.
- Fuentes: `AC-021`, `ALT-004`, `ERR-008`, ADR-030 (comportamientos), ADR-035 (fallos transitorios sostenidos para el circuito), ADR-038.
- Spikes: ninguno nuevo.
- Pruebas: `DEFINITIVE_ERROR` en cada invocación con status y cuerpo de `PM-IV-009`, sin resultado almacenado y con `invocations` creciente; `TRANSIENT_THEN_OUTCOME` con N = 0, 1 y 3: exactamente N respuestas 5xx y después el resultado final, almacenado y repetido sin consumir más fallos; N concurrentes; progreso por regla/intento al sustituir la regla; cancelación durante la fase transitoria y después de `DEFINITIVE_ERROR` → `REGISTERED_BEFORE_CHARGE` y autorizaciones posteriores `ATTEMPT_CANCELLED`; reset limpia el progreso; barrido: cada operación con cada status declarado alcanzable, seguridad, cabeceras y esquemas validados; situaciones no declaradas según `PM-IV-005`.
- Terminado cuando: todas las respuestas declaradas alcanzables están cubiertas o justificadas como inalcanzables (p. ej. 500 de `API-102` según `PM-IV-015`).
- Depende de: PM-INC-004, `PM-IV-009`.

### PM-INC-006 — Cierre, trazabilidad y handoff

- Objetivo: verificación final y entrega a Platform y QA.
- Actividades: tres builds completos consecutivos; matriz §8 con la prueba concreta de cada fila; informe de cobertura; búsqueda de secretos; `dependency:tree` sin `ticketing`; medición de arranque y memoria del jar; handoff de §11.1 y §11.2 con valores medidos; vectores del porcentaje para QA.
- Terminado cuando: todos los criterios de `PAYMENT_MOCK_COMPLETE` (§12 del contrato del agente) verificados.
- Depende de: PM-INC-001 a PM-INC-005.

## 8. API and behaviour traceability

### 8.1 Operaciones

| Operación | Ubicación primaria | Verificación compartida |
|---|---|---|
| `API-101` | PM-INC-003 | PM-INC-004 (latencia, cancelación anticipada), PM-INC-005 (fallos, barrido) |
| `API-102` | PM-INC-004 | PM-INC-005 (barrido) |
| `API-103` | PM-INC-002 | PM-INC-005 |
| `API-104` | PM-INC-002 | PM-INC-003/004/005 (reglas usadas por los comportamientos) |
| `API-105` | PM-INC-002 | PM-INC-005 |
| `API-106` | PM-INC-002 | PM-INC-005 |
| `API-107` | PM-INC-002 | PM-INC-003 (porcentaje y default en la autorización) |
| `API-108` | PM-INC-003 | PM-INC-004, PM-INC-005 |
| `API-109` | PM-INC-004 | PM-INC-005 |
| `API-110` | PM-INC-002 | PM-INC-003 (autorizaciones), PM-INC-004 (cancelaciones, carrera), PM-INC-005 (progreso transitorio) |
| `API-111` | PM-INC-001 | PM-INC-005 |

### 8.2 Comportamientos de ADR-030 y familias de prueba

| Comportamiento | Ubicación primaria | Familia |
|---|---|---|
| Proyecto independiente, build propio, sin código compartido | PM-INC-001 | Estructura (ArchUnit, `dependency:tree`) |
| Autorización `APPROVED` / `DECLINED` | PM-INC-003 | Autorización |
| Idempotencia de autorización secuencial | PM-INC-003 | Autorización |
| Idempotencia de autorización concurrente | PM-INC-003 | Carreras |
| `providerReference` estable | PM-INC-003 | Autorización |
| Carga útil incompatible con el mismo intento | PM-INC-003 | Contrato HTTP |
| Cancelación de aprobado (`REVERSED`) | PM-INC-004 | Cancelación |
| Cancelación de rechazado (`VOIDED`) | PM-INC-004 | Cancelación |
| Cancelación de inexistente (`REGISTERED_BEFORE_CHARGE`) | PM-INC-004 | Cancelación |
| Cancelación en tránsito (durante latencia) | PM-INC-004 | Reactividad, Carreras |
| Cancelación anticipada → `ATTEMPT_CANCELLED` (`AC-035`) | PM-INC-004 | Cancelación, Carreras |
| Idempotencia de cancelación secuencial y concurrente | PM-INC-004 | Cancelación, Carreras |
| Repetición informa cancelación posterior (`cancelled`) | PM-INC-004 | Autorización |
| Reglas por `orderId`, `ticketId`, `customerRef`, `eventId` | PM-INC-003 (selección), PM-INC-002 (CRUD) | Reglas |
| Precedencia entre categorías y desempate | PM-INC-003 | Reglas |
| Porcentaje determinista por hash estable | PM-INC-003 | Reglas |
| Resultado por defecto (inicial `APPROVED`) | PM-INC-003 | Reglas |
| Error definitivo | PM-INC-005 | Autorización |
| N transitorios seguidos de resultado final | PM-INC-005 | Autorización, Carreras |
| Latencia añadida sin bloqueo | PM-INC-004 | Reactividad |
| Inspección de autorizaciones | PM-INC-003 | Inspección |
| Inspección de cancelaciones (reversado exactamente una vez) | PM-INC-004 | Inspección |
| Reset atómico | PM-INC-002 (base), PM-INC-004 (carrera) | Estado volátil, Carreras |
| Estado inicial en un contexto nuevo | PM-INC-002 | Estado volátil |
| API key y health sin autenticación | PM-INC-001 | Seguridad |
| Secretos y `customerRef` ausentes de logs y respuestas | PM-INC-001 | Seguridad |
| Validación completa contra el OpenAPI | PM-INC-001 (arnés), PM-INC-005 (barrido) | Contrato HTTP |

### 8.3 IDs funcionales

| ID | Ubicación |
|---|---|
| `TC-017`, `AV-004` | PM-INC-002, PM-INC-003 |
| `FR-015`, `AC-019`, `AC-020`, `ALT-005` | PM-INC-003 |
| `AC-021`, `ALT-004`, `ERR-008` | PM-INC-005 |
| `FR-017`, `BR-020`, `AC-023`, `AC-024`, `AC-025` (lado proveedor e inspección) | PM-INC-003 |
| `FG-003`, `BR-034`, `ALT-009`, `AC-035`, `FR-023`, `BR-028`, `BR-003`, `AC-034` (lado proveedor) | PM-INC-004 |
| `TC-001`, `TC-002`, `TC-003`, `TC-008`, `TC-013`, `TC-014`, `TC-015` | PM-INC-001 (y todos) |

## 9. Implementation validations

Todas bloquean el inicio de la implementación (el humano puede aceptar las recomendaciones en bloque). Detalle completo, opciones e impacto en `human-review/payment-mock.implementation-plan-review.yaml`.

| ID | Pregunta | Recomendación | Antes de |
|---|---|---|---|
| `PM-IV-001` | Herramienta de build del mock | Maven Wrapper 3.9.16 propio, un módulo, parent Spring Boot 4.1.1, `release 25`, Enforcer | PM-INC-001 |
| `PM-IV-002` | Librerías de prueba y calidad | BOM (JUnit, Mockito), reactor-test, AssertJ, BlockHound 1.0.17, swagger-request-validator-core 2.46.1, ArchUnit 1.5.0, JaCoCo 0.8.15 solo informe | PM-INC-001 |
| `PM-IV-003` | Acceso de las pruebas al OpenAPI | Copia literal propia en `src/test/resources/contracts` con SHA-256 y comparación con el original si `SPEC_REPO` existe | PM-INC-001 |
| `PM-IV-004` | Configuración de ejecución | Puerto 8090 (`SERVER_PORT`), `PAYMENT_MOCK_API_KEY` obligatoria con fallo de arranque si falta, sin Actuator, `/health` 200 sin cuerpo, sin CORS, sin logs por solicitud | PM-INC-001 |
| `PM-IV-005` | Catálogo de errores | 400 `VALIDATION_ERROR` para todo lo detectable en la solicitud (incluida Idempotency-Key distinta, parámetros de ruta > 80, 415, cuerpo excesivo); 401 `UNAUTHENTICATED` antes del enrutado; 422 para conflictos de estado y error definitivo; 404 de inspección sin cuerpo; 404/405/406 del framework con `Error` | PM-INC-001 |
| `PM-IV-006` | Reglas: identidad, orden y desempate | `rule-<n>` con contador que no se reinicia; gana la más reciente dentro de la categoría; listado por categoría y luego de más reciente a más antigua; duplicados permitidos | PM-INC-002 |
| `PM-IV-007` | Validación por comportamiento y `reasonCode` | Campos estrictos por tipo; `reasonCode` ∈ {`CARD_DECLINED`, `INSUFFICIENT_FUNDS`, `RULE_DECLINED`}; por defecto `RULE_DECLINED` (regla y `defaultOutcome = DECLINED`); eco literal de la regla | PM-INC-002 |
| `PM-IV-008` | Porcentaje y defaults | SHA-256 UTF-8, 8 primeros bytes sin signo módulo 100, rechaza si < porcentaje; `PUT` reemplaza y ausencia = 0; respuesta siempre con `declinePercentage` | PM-INC-002 |
| `PM-IV-009` | Fallos simulados | `DEFINITIVE_ERROR` = 422 `SIMULATED_DEFINITIVE_ERROR` en cada invocación; transitorio = 503 `SIMULATED_TRANSIENT_FAILURE`; progreso por (intento, regla); la cancelación interrumpe los fallos restantes | PM-INC-005 |
| `PM-IV-010` | Idempotencia de autorización | Carga útil fijada en la primera invocación válida; diferencia → 422 `IDEMPOTENCY_KEY_REUSED` sin cambios (`ticketIds` como conjunto); repeticiones concurrentes se unen a la decisión en curso; cuentan todas las llamadas autenticadas y válidas | PM-INC-003 |
| `PM-IV-011` | `replayed`, `cancelled` y forma de respuestas | `replayed` = false solo para la llamada que decide; `cancelled` literal (cancelación posterior al resultado); booleanos siempre presentes; opcionales sin valor se omiten, nunca `null` | PM-INC-003 |
| `PM-IV-012` | `providerReference` | `pm-` + 24 hex del SHA-256 de `paymentAttemptId`, en todo resultado incluido `ATTEMPT_CANCELLED` | PM-INC-003 |
| `PM-IV-013` | `LATENCY` | Instantánea de la regla al llegar, decisión al terminar la espera, la cancelación durante la espera gana, decisión desacoplada de la desconexión del llamador, sin `finalOutcome` el resultado sale del porcentaje y el default, repeticiones con resultado inmediatas | PM-INC-004 |
| `PM-IV-014` | `reset` con operaciones en vuelo | Sustitución atómica de generación; lo iniciado antes termina en la generación antigua y no es visible después | PM-INC-004 |
| `PM-IV-015` | Fallos simulados en la cancelación | No se simulan (las reglas no aplican a `API-102`); 500 solo ante error interno inesperado | PM-INC-004 |
| `PM-IV-016` | `OutcomeRule` posiblemente insatisfacible | Responder con `ruleId` (intención evidente), validar con combinadores resueltos o tolerancia acotada a esa violación, y notificar al Architect para una corrección 1.0.1 | PM-INC-002 |

## 10. Risks

| Riesgo | Impacto | Mitigación |
|---|---|---|
| RISK-012: Spring Boot 4.1.1 / librerías sobre Java 25 en un build distinto del de `ticketing` | Bloqueo de PM-INC-001 | `PM-SPK-001`, `PM-SPK-002`; mismas versiones ya verificadas por el backend |
| Pruebas de concurrencia o latencia inestables en Windows | Falsos rojos o, peor, falsos verdes | Invariantes en lugar de tiempos; tiempo virtual; latencias reales cortas con tolerancia amplia; tres builds en el cierre |
| Validador sin soporte de `apiKey` y posible rechazo de `allOf` + `additionalProperties` | Contrato no verificable automáticamente en dos puntos | `PM-SPK-003`; comprobaciones explícitas; `PM-IV-016` |
| Jackson tolerante por defecto (propiedades desconocidas, coerciones) | Aceptar entradas que el contrato prohíbe | `PM-SPK-006` y pruebas negativas |
| Crecimiento de memoria con muchos intentos (prueba de carga en modo porcentaje) | Presión de memoria del contenedor | Registros pequeños e inmutables; reset entre escenarios; memoria medida en PM-INC-006 y entregada a Platform |
| Decisión desacoplada que sigue tras la desconexión | Suscripciones huérfanas si no se acotan | La espera máxima es 60.000 ms (`addedLatencyMs.maximum`); una única decisión por intento; reset las vuelve invisibles |
| Puerto 8090 ocupado en el host local (`WsToastNotification.exe`, PID 7728) | Conflicto al publicar 8090 en Compose o al ejecutar el jar localmente | Pruebas en puerto aleatorio; puerto configurable; aviso a Platform (§11.1) |
| Cambios concurrentes del Backend Developer en `CODE_REPO` | Solapamiento de escrituras | Este agente solo escribe en `payment-mock/`; comprobación de `git status` antes de cada incremento |

## 11. Dependencies and handoffs

### 11.1 Platform (Dockerfile de ADR-030 y servicio de Compose de ADR-036)

Requisitos que este agente entregará con valores medidos en PM-INC-006 (los marcados como propuesta dependen de `PM-IV-004`):

| Requisito | Valor |
|---|---|
| Build | Desde `payment-mock/`: `./mvnw -B clean verify` (JDK 25); independiente del build de `ticketing` |
| Artefacto | `payment-mock/target/payment-mock.jar` (jar ejecutable de Spring Boot) |
| Arranque | `java -jar payment-mock.jar` sobre un runtime Java 25; sin argumentos obligatorios |
| Puerto | 8090 dentro del contenedor (propuesta: sobrescribible con `SERVER_PORT`). En este host el 8090 está ocupado: Platform puede necesitar otro puerto publicado al host |
| Healthcheck | `GET /health` → 200, sin API key (la imagen necesita una herramienta para invocarlo, decisión de Platform) |
| Variables | `PAYMENT_MOCK_API_KEY` (obligatoria, desde archivo no versionado en local y gestor de secretos en AWS no productivo, ADR-032); el arranque falla si falta (propuesta) |
| Sistema de archivos | No escribe en disco; compatible con sistema de archivos de solo lectura salvo el temporal de la JVM |
| Usuario | Sin privilegios (ADR-036); no requiere puertos < 1024 |
| Memoria | A medir en `PM-SPK-002` y PM-INC-006 |
| Red | Solo alcanzable por `ticketing-worker` y la red de pruebas; nunca en producción (ADR-037) |

### 11.2 QA

- Configurar el mock antes de cada escenario con `POST /control/reset` y después reglas o defaults (`AV-004`).
- "Sin respuesta dentro del timeout" de `ticketing` (3 s): regla `LATENCY` con `addedLatencyMs` > 3000. Llamadas lentas para el circuito: `addedLatencyMs` > 2000. Fallos sostenidos: `TRANSIENT_THEN_OUTCOME` con N alto.
- `AC-035` y "reversado exactamente una vez": `API-108` (`invocations`, `result`) y `API-109` (`received`, `cancellationStatus`).
- Vectores del porcentaje (pares `paymentAttemptId` → cubo) en el informe de PM-INC-006 para predecir resultados en la carga.

### 11.3 Observaciones sobre el consumidor `ticketing` (INC-007, solo lectura)

No se detectó ninguna contradicción entre el adaptador y el contrato. Restricciones y puntos de atención:

1. El adaptador exige `paymentAttemptId` igual al enviado y `providerReference` no vacío en toda respuesta 200 de autorización, también en `DECLINED`; si falta, trata la respuesta como transitoria. Ambos son requeridos por el contrato; el mock debe emitirlos siempre, incluido `ATTEMPT_CANCELLED` (`PM-IV-012`).
2. El adaptador trata todo 4xx como definitivo (Order `FAILED` sin reverso) y todo 5xx como transitorio (reintento y, en la quinta recepción, `FAILED` con reverso). El status elegido para `DEFINITIVE_ERROR` (`PM-IV-009`) decide cuál de los dos flujos demuestra `AC-021`; ambos son compatibles con el contrato.
3. La cancelación se envía sin cuerpo y sin `Content-Type`; el mock debe aceptarla (el contrato no declara `requestBody` en `API-102`).
4. El adaptador corta llamadas en curso (timeout de 3 s y plazo `expiresAt − 2 s`). Que exista o no un "cobro en tránsito" tras esa desconexión depende de `PM-IV-013`.
5. `customerRef` es el sujeto del JWT sin límite de longitud en `ticketing`; el contrato limita a 128. Un sujeto mayor produciría 400 y `FAILED` sin reverso. Riesgo bajo (el `sub` de Cognito es un UUID); queda para el Backend Developer o el humano, no para el mock.
6. El doble de pruebas de INC-007 ofrece modos que el mock real no tiene vía reglas (sin respuesta, cierre de conexión, cuerpo inválido, status configurable en `DEFINITIVE_ERROR`, fallos en `API-102`). No es una incoherencia del contrato; QA debe reproducirlos con latencia o deteniendo el contenedor, y los fallos de cancelación dependen de `PM-IV-015`.

### 11.4 Architect

`PM-IV-016` (si `PM-SPK-003` confirma el defecto): corregir `OutcomeRule` en una versión 1.0.1 del contrato. Este agente no modifica `architecture/**`.

### 11.5 Humano

Responder la revisión (`PLAN` y `PM-IV-001` a `PM-IV-016`). Hasta entonces `implementation_can_start: false`.
