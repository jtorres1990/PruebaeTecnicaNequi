---
artifact: payment-mock-increment-report
increment: PM-INC-001
result: DONE
code_revision: "CODE_REPO main 6807639 + untracked payment-mock/ (no commits by this agent); concurrent uncommitted ticketing/ changes belong to the Backend Developer (INC-008) and were not touched"
verified_at: 2026-10-04
plan: implementation/payment-mock.implementation-plan.v1.md
review: human-review/payment-mock.implementation-plan-review.yaml (APPROVED)
---

# PM-INC-001 — Build independiente, bootstrap, health, seguridad y puertas de prueba

## 1. Implemented

- Proyecto Maven independiente `CODE_REPO/payment-mock/` (PM-IV-001): `pom.xml` con parent `spring-boot-starter-parent` 4.1.1, `groupId com.nequi.paymentmock`, sin herencia, agregación ni dependencia con `ticketing`; `release 25`; Maven Enforcer (Java `[25,26)`, Maven `[3.9.16,3.9.17)`); `finalName payment-mock` → `target/payment-mock.jar`.
- Maven Wrapper propio generado con `maven-wrapper-plugin` 3.3.4 (ejecutado con Maven 3.8.1 instalado sobre JDK 25, `-Dmaven=3.9.16 -Dtype=bin`); `distributionUrl` fijado a 3.9.16. No se copió ningún archivo de `ticketing`.
- `.gitattributes` propio: la copia del contrato se marca `-text` para que `core.autocrlf=true` (activo en este equipo) no altere su SHA-256 al hacer checkout; `.gitignore` con `target/`.
- Dependencias (PM-IV-002): `spring-boot-starter-webflux`; en pruebas `spring-boot-starter-test` (JUnit Jupiter 6.0.3, AssertJ, Mockito del BOM), `reactor-test`, `blockhound-junit-platform` 1.0.17.RELEASE, `swagger-request-validator-core` 2.46.1, `archunit-junit6` 1.5.0; JaCoCo 0.8.15 solo `prepare-agent` + `report` (sin puerta).
- Surefire: `-XX:+AllowRedefinitionToAddDeleteMethods -XX:+EnableDynamicAgentLoading -Xshare:off` (BlockHound en Java 25) y `PAYMENT_MOCK_API_KEY` vaciada en el entorno de las pruebas para que nunca hereden una clave real.
- Configuración (PM-IV-004): `payment-mock.api-key: ${PAYMENT_MOCK_API_KEY:}`; `PaymentMockProperties` rechaza clave nula o en blanco (el proceso no arranca) y su `toString` la redacta; `server.port: 8090` (sobrescribible con `SERVER_PORT` por el binding estándar de Spring Boot); sin Actuator; sin CORS; banner desactivado; sin logs por solicitud.
- `API-111` `GET /health`: 200 sin cuerpo, sin autenticación.
- `ApiKeyWebFilter` (orden máximo, antes del enrutado): toda ruta salvo `/health` exige exactamente un `X-Api-Key` igual a la clave configurada (comparación de tiempo constante con `MessageDigest.isEqual`); en otro caso 401 `UNAUTHENTICATED`, también en rutas inexistentes (PM-IV-005).
- `ApiErrorHandler` (orden -2): catálogo PM-IV-005 para errores fuera de las operaciones: 400 `VALIDATION_ERROR` (validación, entrada ilegible, cuerpo excesivo, media type), 404 `NOT_FOUND`, 405 `METHOD_NOT_ALLOWED`, 406 `NOT_ACCEPTABLE`, 500 `INTERNAL_ERROR` sin traza; el log de error interno registra solo el tipo de excepción y se descarga a `boundedElastic` (el appender de consola es E/S bloqueante).
- Copia literal del contrato en `src/test/resources/contracts/payment-mock.openapi.v1.yaml` con `.sha256` (PM-IV-003).
- Arnés de contrato `ContractValidator`: valida solicitud y respuesta de cada interacción de las pruebas web contra la copia; incorpora la tolerancia PM-IV-016 (ver §3).

## 2. Contract coverage

| Operación | Cobertura en este incremento |
|---|---|
| `API-111` | Completa: 200 sin cuerpo sin clave, con clave válida y con clave inválida; validada contra el contrato |
| `API-101` a `API-110` | Solo seguridad: 401 `UNAUTHENTICATED` con cuerpo `Error` validado contra el contrato para 8 variantes de credencial inválida (ausente, vacía, incorrecta, con sufijo, truncada, mayúsculas, cabecera repetida, `Authorization: Bearer`). Las operaciones se implementan en PM-INC-002 a PM-INC-005 |
| Rutas no declaradas | 401 sin clave; 404 `NOT_FOUND` con clave; `/actuator`, `/actuator/health`, `/actuator/env`, `/error` → 404 |

## 3. Spikes

| Spike | Resultado | Evidencia |
|---|---|---|
| `PM-SPK-001` Spring Boot 4.1.1 + WebFlux + Java 25 | `CONFIRMED` | Contexto WebFlux (Reactor Netty, Jackson 3.1.5) arranca en las pruebas con puerto aleatorio; `java -jar target/payment-mock.jar` con `PAYMENT_MOCK_API_KEY` y `SERVER_PORT=18091` arrancó en 1,771 s y respondió `GET /health` 200 (`content-length: 0`), 401 sin clave y 404 `NOT_FOUND` con clave; sin la variable el proceso terminó con `APPLICATION FAILED TO START` y el motivo `PAYMENT_MOCK_API_KEY must be set to a non-blank value` (sin valor alguno impreso). Sin Actuator |
| `PM-SPK-002` Build independiente y jar ejecutable | `CONFIRMED` | `./mvnw -B -ntp clean verify` verde (primera resolución online de `spring-boot-starter-webflux` 4.1.1 y dependencias, sin purgar `~/.m2`); segundo build `./mvnw -B -ntp -o clean verify` verde en 17,9 s; `dependency:tree -Dscope=runtime`: único directo `spring-boot-starter-webflux:4.1.1`, 0 coincidencias de `ticketing`; jar 35,5 MB; memoria residente del proceso tras arrancar sin límites de JVM ≈ 178 MB (Windows, `tasklist`); se vuelve a medir con límites en PM-INC-006 |
| `PM-SPK-003` Validación automatizada contra OpenAPI 3.0.3 | `CONFIRMED` con dos limitaciones | Matriz fijada como prueba permanente `OpenApiValidatorCapabilityTest` (control positivo y negativo por característica) |

Matriz de capacidades de `swagger-request-validator-core` 2.46.1 (PM-SPK-003):

| Característica | ¿Detecta? |
|---|---|
| `additionalProperties: false` en solicitud y respuesta | Sí |
| `format: uuid` | Sí |
| `format: date-time` | Sí (acepta `yyyy-MM-dd'T'HH:mm:ss[.f{1,12}]Z`) |
| Enumerados, tipos (`"true"` por booleano), `minItems`/`maxItems`, `maximum`, `maxLength` en cuerpo, cabecera y ruta | Sí |
| Cabecera requerida ausente (`Idempotency-Key`) | Sí |
| Content-Type no declarado, ruta inexistente, método no declarado | Sí |
| Status no declarado (`validation.response.status.unknown`) y cuerpo inesperado (`validation.response.body.unexpected`) | Sí |
| Propiedad requerida ausente en `Error` | Sí |
| `nullable: true` junto a `enum` (`reasonCode: null`) | Aceptado |
| `nullable: true` junto a `allOf` (`AuthorizationRecord.result: null`) | Rechazado → confirma la omisión de PM-IV-011 |
| Esquema de seguridad `apiKey` | **No se evalúa** (una solicitud sin `X-Api-Key` no produce violación). Mitigación permitida: aserciones explícitas de 401 por operación (`HealthAndSecurityWebTest`) |
| `OutcomeRule` = `allOf(OutcomeRuleInput, {ruleId})` | **Defecto confirmado (PM-IV-016)**: toda respuesta con `ruleId`, `match` y `behaviour` falla con `allOf` "matched only 0 out of 2": la rama 1 rechaza `["ruleId"]` por su `additionalProperties: false` y el validador inyecta además `additionalProperties: false` en la rama 2, que rechaza `["behaviour","match"]`. Con `withResolveCombinators(true)` el mismo objeto valida y siguen detectándose `ruleId` ausente, propiedades extra en la raíz y en `match`, y enumerados inválidos |

### PM-IV-016 — tolerancia aplicada

Respuesta humana aplicada: responder con `ruleId` y tolerar exactamente esa violación en `API-103`/`API-104`. Implementación en `ContractValidator`:

- Solo para `GET`/`POST /control/rules` se descarta un mensaje `validation.response.body.schema.allOf` cuyos únicos mensajes anidados son los dos `additionalProperties` sobre `["ruleId"]` y `["behaviour","match"]` (con o sin prefijo de índice de lista).
- Toda respuesta tolerada se valida además con combinadores resueltos y debe dar cero errores.
- `OpenApiValidatorCapabilityTest.outcomeRuleDefectIsConfirmedAndOnlyThatViolationIsTolerated` demuestra: el defecto sin tolerancia; aceptación del objeto correcto (respuesta individual y lista); y que siguen fallando `ruleId` ausente, propiedad extra en la raíz, propiedad extra en `match` y enumerado inválido.

**Nota para el Architect (corrección 1.0.1 de `payment-mock.openapi.v1.yaml`)**: `OutcomeRule` no es satisfacible por un validador estricto. Recomendación: definir `OutcomeRule` como objeto propio (`type: object`, `additionalProperties: false`, `required: [ruleId, match, behaviour]`, propiedades `ruleId`, `match` y `behaviour` con las mismas definiciones que `OutcomeRuleInput`) en lugar de `allOf`; quitar solo `additionalProperties` de la rama 1 no basta con este validador, que inyecta `additionalProperties: false` en las ramas de `allOf`. Mientras tanto, la tolerancia anterior queda documentada. Este agente no modifica `architecture/**`.

## 4. Verification

| Comando (desde `D:\Nequi\ticketing-platform\payment-mock`) | Resultado |
|---|---|
| `JAVA_HOME="D:\java\zulu25.36.205-ca-jdk25.0.4.1-win_x64" ./mvnw -B -ntp clean verify` | `BUILD SUCCESS`; 34 pruebas, 0 fallos, 0 errores, 0 omitidas |
| `... ./mvnw -B -ntp -o clean verify` | `BUILD SUCCESS` offline; 34/0/0/0 |
| `... ./mvnw -B -ntp -o dependency:tree -Dscope=runtime` | Sin artefactos de `ticketing` |
| `java -jar target/payment-mock.jar` sin y con `PAYMENT_MOCK_API_KEY` | Ver PM-SPK-001 |

Puertas:

| Puerta | Estado | Pruebas |
|---|---|---|
| G1 Build | Verde | Enforcer + `clean verify` |
| G2 Pruebas | Verde, 0 omitidas | 34 |
| G3 Contrato | Activa | `ContractValidator` en `WebTestSupport.send(...)`; `ContractCopyTest` (SHA-256 `5360ffee…05124` igual al registrado y al original de `SPEC_REPO`) |
| G4 No bloqueo | Activa | BlockHound instalado por `blockhound-junit-platform`; canaria `BlockHoundCanaryTest` detecta `Thread.sleep` en hilo `parallel`. Sin excepciones añadidas a BlockHound |
| G5 Estructura | Activa | `ArchitectureTest`: sin dependencia de `com.nequi.ticketing..`; sin `Thread.sleep`, `Object.wait`, `Mono/Flux.block*`, `toIterable`, `toStream`, `Future.get/join` en producción; `domain` sin Spring/Reactor/Jackson/Netty (vacío aún, `allowEmptyShould` hasta PM-INC-002); sin Actuator. Cada regla con control negativo (`ForbiddenFixture`) |
| G6 Seguridad | Verde | `HealthAndSecurityWebTest` (10 operaciones × 8 credenciales inválidas, rutas inexistentes, sin gestión, sin CORS), `ApiKeyVerifierTest`, `StartupConfigurationTest` (no arranca sin clave ni con clave en blanco; la salida capturada del arranque y de solicitudes con clave válida e inválida no contiene la clave). Búsqueda en el árbol: las únicas claves son valores ficticios de prueba en `src/test` |
| G8 Cobertura (informativa) | Instrucciones 63,9 % (202/316), ramas 60,7 % (17/28), líneas 61,8 % (42/68) | Ramas no cubiertas: casos 400/405/406/415/500 de `ApiErrorHandler` (sin operaciones aún; se cubren en PM-INC-002 a PM-INC-005) y `main` |

## 5. Deviations

- Ninguna respecto del plan aprobado.
- Detalle local: el arnés valida con `swagger-request-validator-core` sin resolver combinadores (contrato publicado) y aplica la tolerancia PM-IV-016 solo a `API-103`/`API-104`, con revalidación resuelta.

## 6. Blockers

Ninguno.

## 7. Handoff to Platform and QA

- Artefacto: `payment-mock/target/payment-mock.jar`; arranque `java -jar payment-mock.jar` (Java 25); variables: `PAYMENT_MOCK_API_KEY` (obligatoria; sin ella el proceso termina al arrancar) y `SERVER_PORT` (opcional, por defecto 8090). `GET /health` → 200 sin cuerpo y sin clave. En este equipo el puerto 8090 del host está ocupado (PM-IV-004): publicar otro puerto del host si se ejecuta localmente.
- QA: toda operación salvo `/health` exige `X-Api-Key` y responde 401 `{"code":"UNAUTHENTICATED",...}` también en rutas inexistentes.
