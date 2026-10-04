---
artifact: implementation-plan
schema_version: 1.0
feature: ticketing-event-processing
version: 1

agent:
  name: backend-developer
  version: 1.0

source:
  feature_spec:
    artifact: feature-spec/ticketing.feature-spec.v5.md
    version: 5
  architecture:
    artifact: architecture/ticketing.architecture.v2.md
    version: 2
  adr_registry: architecture/adr/ticketing.adr-registry.v1.md
  addendum: architecture/ticketing.consolidation-addendum.v1.md
  local_environment: implementation/ticketing.local-environment.v1.md

status: READY_FOR_HUMAN_PLAN_REVIEW
human_validation_required: true
increments: 11
spikes: 25
open_iv: 7
blocking_items: 3
implementation_can_start: false

generated_at: 2026-10-04
---

# Implementation Plan — Ticketing Event Processing

## 0. Precondición y fuentes

Precondición del contrato (§2) verificada el 2026-10-04:

| Condición | Valor observado | Resultado |
|---|---|---|
| `human-review/ticketing.architecture-review.yaml` `review.status` | `APPROVED` | PASS |
| `gate.pending_blocking_items` | `[]` | PASS |
| `gate.development_can_start` | `true` | PASS |
| `feature-spec/ticketing.feature-spec.v5.md` `blocking_questions` | `0` | PASS |
| `architecture/adr/ticketing.adr-registry.v1.md` | Existe | PASS |
| `implementation/ticketing.implementation-plan.v1.md` | No existía → MODO 1 | PASS |

Fuentes leídas completas: Feature Specification v5; `ticketing.architecture.v2.md`; registro de ADR; ADR-003, ADR-008, ADR-022 a ADR-040 (`ACCEPTED`); `ticketing.data-model.v2.md`; `ticketing.openapi.v2.yaml`; `ticketing.messaging.v2.md`; `payment-mock.openapi.v1.yaml`; `ticketing.aws-target.v2.md` (solo paridad de configuración); `ticketing.consolidation-addendum.v1.md`; respuestas `AV-*` y `FG-*` de la revisión de arquitectura; `implementation/ticketing.local-environment.v1.md`. No se usó ningún ADR `SUPERSEDED` ni artefacto de arquitectura v1.

Reglas de lectura aplicadas: el registro de ADR prevalece sobre el `status` del frontmatter de ADR-001..ADR-021; las erratas de `ticketing.architecture.v2.md` §14.2 prevalecen; el addendum fija los tamaños máximos de transacción (14 reserva, 13 terminales, 12 inicio de pago, 3 creación de Event).

## 1. Scope

### 1.1 En alcance (este agente)

| Elemento | Ubicación |
|---|---|
| Build multi-módulo con Maven Wrapper 3.9.16 (`ENV-001`) y JDK 25 (`ENV-002`) | `CODE_REPO/ticketing/` |
| `domain` — `CMP-009` | `CODE_REPO/ticketing/domain` |
| `application` — `CMP-003`, `CMP-004`, `CMP-005`, `CMP-006`, `CMP-007`, `CMP-008`, `CMP-015`, `CMP-022`, `CMP-023`, `CMP-024` y todos los puertos de ADR-034 | `CODE_REPO/ticketing/application` |
| `infrastructure` — `CMP-001`, `CMP-002`, `CMP-010`, `CMP-011`, `CMP-012`, `CMP-013`, `CMP-014`, `CMP-016`, `CMP-017`, `CMP-018`, `CMP-025`, `CMP-026` | `CODE_REPO/ticketing/infrastructure` |
| `bootstrap` — `CMP-021` (composición, roles `api` / `worker`, configuración, reloj, identificadores) | `CODE_REPO/ticketing/bootstrap` |
| Pruebas unitarias (dominio, casos de uso, adaptadores), capa web reactiva, integración con contenedores de prueba (DynamoDB Local 3.3.1, LocalStack 4.14.0) | Módulos anteriores |
| Dobles de prueba del Payment Mock basados en `payment-mock.openapi.v1.yaml` | Pruebas de `infrastructure` y `bootstrap` |
| Puertas de calidad: cobertura ≥ 90 % (ADR-038, `AV-006`), pruebas de arquitectura (ADR-034), detector de bloqueo (`NFR-003`) | Build de `ticketing` |

### 1.2 Fuera de alcance

| Elemento | Agente responsable |
|---|---|
| `CODE_REPO/payment-mock/**` (`CMP-019`, `API-101` a `API-111` del lado servidor) | Payment Mock |
| `local-idp` (`CMP-020`), `infra-init`, `load-token-generator`, `docker-compose*.yml`, Dockerfile e imagen base de Java 25 (parte de imagen del item §13 #23) | Platform |
| Terraform, `infra/**`, pipelines CI/CD | Platform/IaC |
| README de ambos repositorios (`DEL-002`) y colección de solicitudes (`DEL-004`) | Documentation / Platform |
| Pruebas extremo a extremo sobre Docker Compose (`MF-001` a `MF-004`), escenarios de resiliencia de ADR-038, prueba de carga (`AC-029`, `AC-030`, `AC-031`), invariantes posteriores a la carga | QA / Resilience |
| Verificaciones contra AWS real (items §13 #6, #7, #27 a #30) | Platform/IaC |

Ningún archivo se escribe en la raíz de `CODE_REPO` (prohibido por el contrato); todo el build vive dentro de `CODE_REPO/ticketing/`.

## 2. Toolchain

Comprobado el 2026-10-04 sin instalar nada.

| Elemento | Requerido | Disponible en el entorno | Acción |
|---|---|---|---|
| JDK | Java 25 (`TC-001`), `ENV-002` | Zulu 25.36.205, `25.0.4.1+1-LTS` en `D:\java\zulu25.36.205-ca-jdk25.0.4.1-win_x64` (`java -version` correcto) | Ninguna. `JAVA_HOME` se fija por invocación; el global sigue en Zulu 17 y no se toca |
| Herramienta de build | Maven Wrapper multi-módulo (`ENV-001`), Maven 3.9.16 | Wrapper aún no creado (repositorio vacío). Maven 3.8.1 local en `D:\apache-maven-3.8.1` solo para generar el wrapper si hace falta | Crear el wrapper en `ticketing/` en INC-001; compatibilidad con Spring Boot 4.x en `SPK-001` |
| Spring Boot | 4.x (`TC-002`) | Se resuelve desde Maven Central | Versión estable concreta fijada y verificada en `SPK-001` |
| Docker | Motor para Testcontainers | Docker Desktop 29.8.0, backend Linux, 12 CPU, ≈ 8 GB, en ejecución | Ninguna. Hay un contenedor ajeno llamado `dynamodb-local` en ejecución; Testcontainers usa puertos aleatorios, sin conflicto esperado |
| Imagen DynamoDB Local | `amazon/dynamodb-local:3.3.1` | Presente, digest `sha256:ff89bd48…bc0dab` (coincide con el artefacto de entorno) | Ninguna |
| Imagen LocalStack | `localstack/localstack:4.14.0` | Presente, digest `sha256:3ebc3759…bc364a` (coincide) | Ninguna |
| `testcontainers/ryuk` | Bajo demanda | No descargada | La descarga Testcontainers en el primer uso (`SPK-006`) |
| Disco | Espacio para `.m2` y build | C: 88 GB libres (82 %), `.m2` 1,8 GB; D: 829 GB libres | Ninguna; el código vive en D: |
| `CODE_REPO` | Vacío, rama `main` | Solo `.git`, sin commits | Los informes indicarán "sin commits" hasta que el humano versione |

No se detectó ninguna declaración del entorno que haya dejado de cumplirse; no se genera ningún `IV-*` de toolchain. `ENV-001` y `ENV-002` no se replantean.

## 3. Repository layout

### 3.1 Repositorios (contrato §5.0)

| Repositorio | Contenido que produce este agente |
|---|---|
| `SPEC_REPO` = `D:\Nequi\PruebaeTecnicaNequi` | Este plan, su revisión, los informes `implementation/increments/INC-NNN.report.md` y, si surgen, `human-review/ticketing.implementation-review.<n>.yaml`. Nunca código |
| `CODE_REPO` = `D:\Nequi\ticketing-platform` | Solo `ticketing/**`: POM padre, Maven Wrapper (`mvnw`, `mvnw.cmd`, `.mvn/wrapper/`), `.gitignore` propio de `ticketing/` y los cuatro módulos |

### 3.2 Módulos de `CODE_REPO/ticketing/` (ADR-034)

```text
ticketing/
  pom.xml                (padre: versiones, plugins, puertas de calidad)
  domain/                Domain      → solo biblioteca estándar
  application/           Use Cases   → domain + reactor-core
  infrastructure/        Infrastructure → application, domain, Spring, SDK de AWS, Resilience4j
  bootstrap/             Infrastructure (composición) → todos los anteriores
```

Dirección de dependencias (impuesta por el compilador mediante dependencias de módulo, y verificada además por reglas de prohibición del build y por pruebas de arquitectura):

```text
bootstrap ──► infrastructure ──► application ──► domain
     └──────────────┴──────────────────┴────────────► domain
```

- `domain` sin dependencias de compilación (solo JDK); `application` solo `domain` y `reactor-core`.
- `infrastructure` contiene adaptadores de entrada (HTTP, consumidores SQS, planificador) y de salida (DynamoDB, publicación SQS, pago), seguridad, errores, resiliencia y observabilidad.
- `bootstrap` contiene la clase de arranque, la composición y la activación de roles; es además el módulo donde se agrega el informe de cobertura (depende de los tres módulos medidos) y donde residen las pruebas de arquitectura (ven todo el classpath) y las pruebas de integración de componentes por rol.
- Sin quinto módulo: no se altera la estructura aprobada en ADR-034.
- Los dobles en memoria de los puertos de salida viven en las fuentes de prueba de `application` y se publican como artefacto de pruebas del propio build para reutilizarlos en `bootstrap`.

## 4. Quality gates

### 4.1 Cobertura (ADR-038, `AV-006`, `NFR-013`)

| Aspecto | Decisión aplicada |
|---|---|
| Métrica | Cobertura de líneas agregada ≥ 90 % |
| Alcance | `domain`, `application`, `infrastructure` de `ticketing`; `payment-mock` fuera |
| Pruebas que cuentan | Solo pruebas sin contenedores (fase de pruebas unitarias). Las pruebas `*IT` con contenedores no alimentan la métrica |
| Exclusiones | Únicamente la clase de arranque y las clases de pura configuración. `bootstrap` (arranque y composición) queda fuera por alcance; dentro de `infrastructure` solo se excluyen clases de pura configuración sin lógica, enumeradas explícitamente en el POM y listadas en cada informe de incremento |
| Herramienta | Según `IV-001` (recomendado: JaCoCo, verificado en `SPK-002`) |
| Efecto | El build falla por debajo del umbral. Activa desde INC-001 y nunca relajada |

### 4.2 Pruebas de arquitectura (ADR-034)

Herramienta según `IV-001` (recomendado: ArchUnit, `SPK-004`; alternativa definida por ADR-034: reglas internas por revisión, con la frontera garantizada igualmente por el compilador). Reglas mínimas:

1. `domain` solo depende de `java.*`.
2. `application` solo depende de `java.*`, `reactor.*` y `domain`; ningún tipo de Spring, SDK de AWS, Netty, HTTP, Jackson, Resilience4j ni Micrometer.
3. `infrastructure` no depende de `bootstrap`.
4. Retry (operadores de reintento de Reactor) y circuit breaker solo en paquetes de adaptadores de `infrastructure` (ADR-035).
5. Sin API de Virtual Threads (ADR-034).
6. Sin llamadas bloqueantes en código de producción (`block*`, `toIterable`, `toStream`, `Thread.sleep`).
7. Tipos del SDK de AWS confinados a los adaptadores de DynamoDB y SQS; tipos del adaptador de pago confinados a su paquete (regla de frontera 7).
8. Los adaptadores de entrada solo acceden a puertos de entrada; los casos de uso solo a puertos de salida.

Complemento en el build: reglas de prohibición de dependencias para `domain` (ninguna) y `application` (solo `reactor-core`).

### 4.3 Detector de llamadas bloqueantes (`NFR-003`, ADR-038)

Herramienta según `IV-001` (recomendado: BlockHound con su integración de JUnit Platform, `SPK-005`). Se instala automáticamente en toda JVM de pruebas de `application`, `infrastructure` y `bootstrap`. Una prueba canaria demuestra en cada build que el detector está activo (detecta una llamada bloqueante deliberada en un hilo no bloqueante). Ninguna excepción para código propio; cualquier excepción para internos de terceros se documenta en el informe del incremento.

### 4.4 Separación y nombres de pruebas

- `*Test`: sin contenedores, ejecutadas en `mvnw verify`.
- `*IT`: con Testcontainers, ejecutadas en `mvnw verify -Pintegration` (perfil del propio build).
- Toda prueba que verifica un criterio o regla lleva el ID en su nombre visible (por ejemplo `AC-007 …`, `BR-031 …`, `ADR-038/mecanismo …`), de modo que la matriz de §7 se reconstruye buscando el ID en el código de pruebas.
- Tiempo controlado: reloj inyectado (puerto `Clock`) y tiempo virtual de reactor-test; ninguna prueba depende de esperas reales salvo las de integración con plazos acotados.

### 4.5 Comando de verificación (por incremento)

```text
cd D:\Nequi\ticketing-platform\ticketing
JAVA_HOME=D:\java\zulu25.36.205-ca-jdk25.0.4.1-win_x64  ./mvnw verify                 (compilación, *Test, arquitectura, bloqueo, cobertura)
JAVA_HOME=D:\java\zulu25.36.205-ca-jdk25.0.4.1-win_x64  ./mvnw verify -Pintegration   (además *IT del incremento)
```

## 5. Spikes

Los items de `ticketing.architecture.v2.md` §13 que afectan al backend. Las verificaciones `CONFIRMED` de `ticketing.local-environment.v1.md` no se repiten; solo se cubre la parte que ese artefacto deja pendiente.

| ID | TO_VERIFY # | What to verify | Method | Success criterion | Alternative (ADR) | Needed by |
|---|---|---|---|---|---|---|
| SPK-001 | 23 (parte de build) | Build multi-módulo con Maven Wrapper 3.9.16, JDK 25 y el plugin de Spring Boot 4.x | Crear el wrapper y el POM padre con los cuatro módulos; `mvnw verify` con JDK 25; empaquetado ejecutable de `bootstrap` | Compila los cuatro módulos con `release 25` y genera el jar ejecutable sin errores | ADR-034: "cambia herramienta, no estructura"; como `ENV-001` fija Maven por decisión humana → **escalar** | INC-001 |
| SPK-002 | 24 | Herramienta de cobertura con Java 25 (clases versión 25), informe agregado multi-módulo y comprobación de umbral | Prueba trivial instrumentada; informe agregado en `bootstrap`; `check` con umbral | Informe agregado correcto y el build falla deliberadamente por debajo del umbral en una prueba de control | ADR-038: si no es compatible, `NFR-013` queda bloqueado → **escalar** | INC-001 |
| SPK-003 | 24 | JUnit (versión gestionada por el BOM de Spring Boot 4.x), Mockito (mock maker inline y carga del agente en JDK 25) y reactor-test | Pruebas mínimas con `@ExtendWith`, mocks de clases finales, `StepVerifier` con tiempo virtual | Pruebas en verde sin avisos de carga dinámica de agentes no gestionados | `TC-014` las fija → **escalar**; versión de JUnit según `IV-002` | INC-001 |
| SPK-004 | 24 | Librería de pruebas de arquitectura con clases de Java 25 | Regla de dependencia entre módulos que falla deliberadamente y luego pasa | Lee clases versión 25 y detecta la violación | ADR-034: reglas internas por revisión (la frontera la sigue imponiendo el compilador) | INC-001 |
| SPK-005 | 24 | Detector de llamadas bloqueantes en JDK 25 | Prueba canaria con llamada bloqueante en `Schedulers.parallel()` | El detector señala la llamada; el resto de pruebas en verde | Sin alternativa en ADR-038 → **escalar** | INC-001 |
| SPK-006 | 24 + entorno §5 | Testcontainers sobre Docker Desktop en Windows con las imágenes fijadas (DynamoDB Local 3.3.1, LocalStack 4.14.0) | `*IT` mínima que arranca ambos contenedores y opera contra ellos | Arranque, operación y limpieza (ryuk) correctos | ADR-038: integración contra el Docker Compose levantado (depende del agente Platform; se registraría como dependencia) | INC-005 |
| SPK-007 | 19 | SDK oficial de AWS v2 con Java 25: clientes asíncronos de DynamoDB y SQS con cliente HTTP asíncrono; política de retry por defecto | Llamadas contra los contenedores; inspección de la configuración de retry efectiva | Operaciones correctas sin bloqueo; modo e intentos de retry por defecto documentados | Si no es compatible: conflicto con el stack obligatorio → **escalar** | INC-005 |
| SPK-008 | 1, 2 (parte Java SDK) | `TransactWriteItems` de 14 items con el SDK asíncrono; exposición de `CancellationReasons` posicionales y del item `ALL_OLD` del item que no cumplió la condición | Transacción de 14 items sobre 14 particiones con un fallo de condición inducido en una posición | Motivos posicionales y item devuelto accesibles desde el SDK (el límite de 14 items ya está `CONFIRMED` en el entorno) | ADR-023/ADR-039: lectura de respaldo de los items señalados | INC-005 |
| SPK-009 | 3 | Exclusión entre transacciones multi-partición concurrentes y frente a escrituras condicionales simples en DynamoDB Local | N transacciones simultáneas sobre Ticket solapados y transacción terminal frente a `AP-011`, repetidas | Exactamente un ganador por Ticket en todas las repeticiones; ningún parcial | ADR-023: las pruebas de concurrencia de integración se ejecutan contra AWS; la de unidad con el doble en memoria se mantiene. ADR-025: si falla en el servicio, revisar diseño → escalar | INC-005 |
| SPK-010 | 4 | Escritura por lotes: 25 items por solicitud y `UnprocessedItems` | Lote de 100 Ticket en 4 solicitudes de 25 con reintento de no procesados | Todas las claves escritas; reintento correcto | Cambia el número de solicitudes por lote, no el diseño | INC-005 |
| SPK-011 | 5 | Lectura por lotes consistente de 100 claves en DynamoDB Local | `BatchGetItem` con `ConsistentRead` sobre 100 claves | 100 claves devueltas con lectura consistente aceptada | ADR-024/ADR-039: lecturas individuales consistentes en local | INC-005 |
| SPK-012 | 9 | Recuento con selección de solo cantidad y paginación por tamaño leído | Query `COUNT` sobre un shard con miles de items | Recuento exacto agregando páginas | ADR-040: ajustar la concurrencia del recuento | INC-005 |
| SPK-013 | 10 | TTL en DynamoDB Local | Habilitar TTL en la tabla de prueba y escribir registros de idempotencia con `ttl` | El atributo se acepta; la lógica no depende del borrado (se documenta el comportamiento) | Ninguno funcional | INC-005 |
| SPK-014 | 11 | Retirada de un item de un GSI disperso al eliminar el atributo de clave | Escribir y luego `REMOVE` de `GSI3PK`/`GSI4PK`; consultar el índice | El item deja de aparecer en el índice | Más candidatas obsoletas; sin impacto en corrección | INC-005 |
| SPK-015 | 8 (tamaño de item) | Item Event con la definición compacta máxima (100 secciones, 2.000 filas, 500 rangos) por debajo del límite de item | Escribir el Event máximo en DynamoDB Local y medir su tamaño | Escritura aceptada con margen frente a 400 KB | ADR-024: "reducir los límites de la definición", que cambiaría `VAL-013` (funcional) → **escalar** | INC-005 |
| SPK-016 | 14 (parte SDK) | Límites y atributos de SQS desde el SDK contra LocalStack 4.14.0: 10 mensajes por recepción, espera 20 s, visibilidades 60/120 s y cambios, retenciones 1 h / 1 día / 14 días, `ApproximateReceiveCount` | Colas creadas por el fixture de prueba con los atributos de `ticketing.messaging.v2.md` §1; recepción y cambio de visibilidad por SDK | Atributos aceptados y observables desde el SDK (la parte CLI ya está `CONFIRMED`) | ADR-029: ajuste de parámetros; un valor distinto del aprobado requiere `IV-*` | INC-006 |
| SPK-017 | 16 | Latencia real de publicación frente al timeout de 500 ms por intento | Medición de N publicaciones contra LocalStack (percentiles) | p99 claramente inferior a 500 ms | ADR-026: valor configurable (el valor por defecto no cambia; se documenta la medición) | INC-006 |
| SPK-018 | 17 | Clasificación de errores del cliente SQS: reintentables (timeout, conexión, 5xx, throttling) y no reintentables (cola inexistente, acceso denegado, mensaje inválido) | Inducir cada caso contra LocalStack (cola inexistente, contenedor detenido, mensaje inválido) | Tabla de excepciones del SDK por clase | ADR-026/ADR-035: ajusta la clasificación | INC-006 |
| SPK-019 | 18 | Resilience4j (circuit breaker) y su módulo de Reactor con Java 25 y la versión de Reactor del BOM de Spring Boot 4.x; transiciones con tiempo controlado | Circuito con los valores de ADR-035; cerrado → abierto → semiabierto → cerrado con reloj/tiempo virtual | Transiciones correctas y deterministas | ADR-035: circuito mínimo propio en el adaptador con las mismas propiedades | INC-006 |
| SPK-020 | 20 (parte web) | Spring Boot 4.x: Resource Server reactivo con emisor y URL de claves configurados por separado y validadores propios; Problem Details en WebFlux; límite de cuerpo en memoria y respuesta 413 | Aplicación de prueba con `WebTestClient` y JWT simulados | Cada capacidad disponible con su configuración identificada | Solo implementación | INC-008 |
| SPK-021 | 20 + ADR-030 | Cliente HTTP reactivo de Spring Boot 4.x: timeouts de respuesta y conexión, clasificación de 4xx/5xx/timeout | Llamadas contra un servidor de prueba reactivo con latencia y fallos simulados | Timeouts efectivos sin bloqueo | Solo implementación del adaptador | INC-007 |
| SPK-022 | 21 | Claims del token de acceso de Cognito: grupos, tipo de token, cliente, sujeto, ausencia de audiencia | Según la respuesta a `IV-006` | Nombres de claims confirmados y validadores configurables | ADR-032: ajustar validadores | INC-008 |
| SPK-023 | 26 | Librería de limitación por tasa compatible con WebFlux, Java 25 y no bloqueante | Limitador por sujeto con rechazo inmediato (sin espera) | Rechazo inmediato con tiempo de espera calculable para `Retry-After` | ADR-032: componente propio | INC-008 |
| SPK-024 | 20 (parte operativa) | Puerto de gestión separado con salud en el puerto de aplicación; apagado ordenado; exportación de métricas y trazas | Aplicación de prueba con ambos puertos y señal de parada | Salud expuesta en el puerto de aplicación, métricas solo en el de gestión, apagado que completa lo que está en vuelo | Solo implementación | INC-010 |
| SPK-025 | 12 | Duración del aprovisionamiento de 50.000 Ticket en DynamoDB Local | Prueba de integración de componentes con el Event máximo | Medición documentada | ADR-024: ajustar paralelismo y tamaño de lote (valores aprobados → requeriría `IV-*`) | INC-010 |

Items de §13 sin spike del backend:

| # | Motivo |
|---|---|
| 15 | `CONFIRMED` en `ticketing.local-environment.v1.md` §4.1 (LocalStack 4.14.0 sin token, redrive, contador, visibilidad, long polling). No se repite; la parte SDK la cubre `SPK-016` |
| 6, 7 | Límites de throughput y facturación de AWS: sin acción en el código (divisores configurables); verificación en AWS por Platform/IaC |
| 13, 25 | Latencia bajo carga y herramienta de carga: QA / Resilience |
| 22 | Emisor OIDC local: Platform |
| 23 (parte de imagen) | Imagen base de Java 25: Platform (Dockerfile) |
| 27–30 | Plataforma AWS, warm throughput, captura de cambios, tarifas: Platform/IaC |

## 6. Increments

Criterio de terminado común a todos los incrementos (contrato §3.5): compilación de todos los módulos con JDK 25; todas las `*Test` en verde; las `*IT` del incremento en verde; pruebas de arquitectura en verde; detector de bloqueo activo y sin detecciones; cobertura agregada ≥ 90 % según §4.1; informe `implementation/increments/INC-NNN.report.md` con resultado `DONE`.

### INC-001 — Esqueleto del build y puertas de calidad

- Goal: build multi-módulo verificable con todas las puertas activas desde el primer día.
- Modules: `ticketing/` (POM padre, wrapper), los cuatro módulos con contenido mínimo; clase de arranque en `bootstrap`.
- Components / ports: `CMP-021` (solo esqueleto de arranque).
- Spec IDs: `TC-001`, `TC-002`, `TC-007`, `TC-013`, `TC-014`, `TC-015`, `NFR-003`, `NFR-012`, `NFR-013`, `DEL-001`, `DEL-003`.
- ADRs: ADR-034, ADR-038, ADR-035 (regla de ubicación de retry/circuit breaker).
- AP / API / MSG: ninguno.
- Spikes: SPK-001, SPK-002, SPK-003, SPK-004, SPK-005.
- Tests: pruebas de arquitectura (§4.2) con casos de control que fallan deliberadamente en un fixture aislado; prueba canaria del detector de bloqueo; prueba de control de la puerta de cobertura.
- Done criteria: criterio común; wrapper fijado a Maven 3.9.16; `release 25`; perfiles de unidad e integración separados.
- Depends on: aprobación del plan; `IV-001`, `IV-002`.

### INC-002 — Dominio

- Goal: todas las reglas de negocio en `domain`, sin dependencias, probadas de forma unitaria.
- Modules: `domain`.
- Components / ports: `CMP-009`.
- Contenido: Event con ciclo de aprovisionamiento (`DS-011`..`DS-013`, `ST-011`..`ST-013`) e inmutabilidad desde `ENABLED`; definición compacta con validaciones `VAL-006`..`VAL-009`, `VAL-013` y límites configurables (capacidad 50.000, 100 secciones, 2.000 filas, 1.000 asientos por fila, 500 rangos; rangos de cortesía existentes y no solapados); generación de `ticketId` `<sección>-<fila>-<asiento>` (`BR-033`) y orden determinista de claves y lotes de 100; `availabilityShards = min(32, max(1, ceil(capacity / 2000)))`, shards `RESV#` (8), `REVERSAL#` (4), `PENDQ#` (8) con función de hash estable definida en el dominio (ADR-022, ADR-034); máquina de Ticket (`DS-001`..`DS-005`, `ST-001`..`ST-005`; `SOLD` y `COMPLIMENTARY` finales); máquina de Order (`DS-006`..`DS-010`, `ST-006`..`ST-010`) con guardián `CREATED`, guardas temporales (`expiresAt = reservedAt + 10 min` como constante no configurable según spec §13.1; confirmar exige `expiresAt > ahora`; expirar exige `expiresAt <= ahora`; margen de corte 15 s, `BR-029`); validación de compra 1..10 sin repetidos (`VAL-012`) y formato de `Idempotency-Key`; hash canónico del contenido de compra y de creación de Event (ADR-027); regla de una Order activa por cliente y Event (`BR-024`, identidad del bloqueo); criterio de cuarentena y política de marca de reverso (cuándo se marca, calendario 10 s, 30 s, 1 min, 2 min, 5 min y luego 10 min, máximo 10 intentos; ADR-025); `paymentAttemptId = <orderId>-1` (ADR-027); causas funcionales (`PAYMENT_DECLINED`, `PROCESSING_UNAVAILABLE`, `PROCESSING_FAILED`, `RESERVATION_EXPIRED`); Event pasado (`BR-025`) y visibilidad (`BR-022`, `BR-027`); construcción de registros de auditoría con el catálogo de ADR-031; jerarquía cerrada de errores de dominio (ADR-035) con el orden de precedencia de `BR-031`; políticas de decisión puras de las reglas de consumo (`ticketing.messaging.v2.md` §5.1 y §5.2) dado el estado leído y el instante.
- Spec IDs: invariantes 1..16 de §5.1; `DS-001`..`DS-013`; `ST-001`..`ST-013`; `BR-001`..`BR-016`, `BR-020`..`BR-022`, `BR-024`..`BR-031`, `BR-033`; `VAL-001`, `VAL-003`, `VAL-005`..`VAL-009`, `VAL-012`, `VAL-013`, `VAL-014`, `VAL-016`; `ERR-010`, `ERR-011`; `FR-003`, `FR-013`, `FR-014` (construcción).
- ADRs: ADR-003, ADR-008, ADR-022, ADR-024, ADR-025, ADR-027, ADR-031, ADR-032, ADR-034, ADR-035.
- AP / API / MSG: ninguno directamente (alimenta todos).
- Spikes: ninguno.
- Tests: unitarias JUnit (parametrizadas para máquinas de estado, límites y bordes temporales con reloj fijo).
- Done criteria: criterio común.
- Depends on: INC-001.

### INC-003 — Casos de uso del rol `api` y puertos

- Goal: orquestación reactiva de las seis operaciones síncronas con dobles en memoria.
- Modules: `application`.
- Components / ports: `CMP-003`, `CMP-004`, `CMP-005`, `CMP-006`. Puertos de entrada: Create Event, Get Event provisioning status, List Events, Get Event availability, Start purchase, Get Order. Puertos de salida (definición completa de ADR-034): Event catalog, Ticket inventory, Order lifecycle store, Order reader, Idempotency store, Order queue publisher (incluye consulta de disponibilidad del publicador, sin tipos de circuit breaker), Provisioning queue publisher, Clock, Id generator. Dobles en memoria de todos los puertos de salida que reproducen condiciones, bloqueo de Order activa, cuarentena y marca de reverso (ADR-038, consecuencias).
- Contenido: creación de Event con idempotencia (`AP-022` antes de validar el negocio; repetición 200 con estado actual; clave reutilizada), escritura `AP-001`, publicación `MSG-002` sin cambiar la respuesta si falla (ADR-024); estado de aprovisionamiento; listado de futuros `ENABLED` con `soldOut`; disponibilidad con recuento cacheado 1 s compartido, página por cursor, filtro por sección y Event pasado informativo (ADR-040); compra con precedencia `BR-031` completa (validación → idempotencia → Event → pasado → `UNKNOWN_TICKETS` contra la definición → circuito de publicación → transacción con relectura de idempotencia prioritaria → bloqueo → Ticket inexistente → no disponible → conflicto persistente), publicación con presupuesto, `enqueuedAt` sin hacer fallar la compra, compensación "Fallar (encolado)", doble fallo 503, republicación en la repetición (ADR-026, ADR-027); consulta de Order propia (lectura fuerte solo del item Order; inexistente = ajena; cuarentena invisible).
- Spec IDs: `FR-001`, `FR-002`, `FR-004`, `FR-005`, `FR-006`, `FR-009`, `FR-010`, `FR-012`, `FR-016`, `FR-017`, `FR-019` (propiedad), `FR-020`, `FR-021`, `FR-022`, `FR-024`; `BR-017`, `BR-018`, `BR-019`, `BR-022`, `BR-023`, `BR-024`, `BR-025`, `BR-027`, `BR-031`, `BR-032`; `VAL-002`, `VAL-010`, `VAL-011`, `VAL-014`, `VAL-015`, `VAL-016`; `ALT-002`, `ALT-003`, `ALT-006`, `ALT-007`, `ALT-010`, `ALT-013`; `ERR-001`, `ERR-002`, `ERR-006`, `ERR-007`, `ERR-009`, `ERR-012`, `ERR-013`, `ERR-014`, `ERR-015`, `ERR-019`, `ERR-020`, `ERR-021`; `ST-001`, `ST-006`, `ST-009` (encolado), `ST-011`.
- ADRs: ADR-023, ADR-024, ADR-026, ADR-027, ADR-032, ADR-035, ADR-040, ADR-034.
- AP / API / MSG: lógica de uso de `AP-001`, `AP-004`..`AP-006`, `AP-008`..`AP-011`, `AP-015` (encolado), `AP-020`..`AP-023` a través de puertos; `API-001`..`API-006` (casos de uso); producción de `MSG-001` y `MSG-002` a través de puertos.
- Spikes: ninguno.
- Tests: unitarias con Mockito, reactor-test y dobles en memoria; pruebas de mecanismo con doble en memoria (§8).
- Done criteria: criterio común.
- Depends on: INC-002; `IV-004`.

### INC-004 — Casos de uso del rol `worker`

- Goal: procesamiento de Orders, expiración, barrido, reversos, aprovisionamiento y limpieza como casos de uso puros.
- Modules: `application`.
- Components / ports: `CMP-007`, `CMP-008`, `CMP-015`, `CMP-022`, `CMP-023`, `CMP-024`. Puertos de entrada: Process Order, Provision Event, Expire Reservations, Republish pending Orders, Reverse payments, Clean up provisioning. Uso de Payment gateway (autorizar, cancelar; resultados tipados incluido "dependencia no disponible"). Resultado tipado de procesamiento de mensaje (eliminar, posponer hasta, reintentar con backoff, venenoso) que el adaptador de consumo traduce.
- Contenido: reglas 1..15 de `ticketing.messaging.v2.md` §5.1 en su orden (margen de corte, inicio de pago `AP-012`, lease vigente/vencido `AP-013`, autorización con plazo limitado por `expiresAt`, confirmación `AP-014`, aprobación tardía `AP-032` o expiración con auditoría, rechazo, fallo definitivo sin reverso, transitorios, última recepción con marca de reverso, cuarentena `AP-031`); reglas 1..10 de §5.2 (lease, comprobación antes de cada lote, reanudación desde `provisionedBatches`, verificación con hasta 3 reescrituras, habilitación, `FAILED` en la última recepción); expiración con guardas y cuarentena (ADR-028); barrido con umbral 30 s, exclusión por margen de corte y omisión con circuito abierto (ADR-026); reversos con calendario persistido, confirmación, reprogramación y agotamiento (ADR-025); limpieza: estancados > 3 min, republicación hasta 3 veces o `FAILED`, purga de Events `FAILED` (ADR-024).
- Spec IDs: `FR-001`, `FR-003`, `FR-007`, `FR-008`, `FR-011`, `FR-013`, `FR-014`, `FR-015`, `FR-016`, `FR-017`, `FR-023`; `BR-002`, `BR-003`, `BR-011`, `BR-013`, `BR-020`, `BR-026`, `BR-028`, `BR-029`, `BR-030`, `BR-034` (tratamiento del rechazo por cancelación previa); `VAL-001`, `VAL-003`, `VAL-009`; `ALT-001`, `ALT-004`, `ALT-005`, `ALT-008`, `ALT-009`, `ALT-011`, `ALT-012`; `ERR-003`, `ERR-004`, `ERR-005`, `ERR-008`, `ERR-016`, `ERR-017`, `ERR-018`; `ST-002`..`ST-005`, `ST-007`..`ST-010`, `ST-012`, `ST-013`; §12.1 (cuarentena retenida).
- ADRs: ADR-008, ADR-024, ADR-025, ADR-026, ADR-027, ADR-028, ADR-029, ADR-030 (clasificación), ADR-031.
- AP / API / MSG: lógica de uso de `AP-002`, `AP-003`, `AP-010`, `AP-012`..`AP-016`, `AP-018`, `AP-024`..`AP-033`; consumo de `MSG-001` y `MSG-002`; producción de `MSG-001` (barrido) y `MSG-002` (estancados).
- Spikes: ninguno.
- Tests: unitarias con reloj inyectado y tiempo virtual; dobles en memoria y de Payment gateway; pruebas de mecanismo (§8).
- Done criteria: criterio común.
- Depends on: INC-003; `IV-004`.

### INC-005 — Adaptador de DynamoDB

- Goal: todos los access patterns y transacciones de `ticketing.data-model.v2.md` con exactamente los items y condiciones de §5.
- Modules: `infrastructure`.
- Components / ports: `CMP-010` implementa Event catalog, Ticket inventory, Order lifecycle store, Order reader, Idempotency store.
- Contenido: cliente asíncrono de bajo nivel con mapeo explícito (ADR-039); transacciones con solicitud del item que falló la condición y resultado tipado con motivo por item (idempotencia, bloqueo, Ticket inexistente, Ticket no disponible, guardián de Order, conflicto); reintento de conflicto de la transacción completa máximo 2 con jitter (ADR-023, ADR-035); escritura y borrado por lotes con reintento de no procesados (hasta 4 solicitudes en paralelo de 25); lectura por lotes consistente; recuento paralelo por shards y sondeo con parada en el primer resultado en oleadas de 4; cursores opacos validados (versión, Event, filtro, shard, última clave); índices dispersos con entrada y salida exactas de `GSI1`..`GSI4`; TTL de idempotencia; ninguna creación de tablas por la aplicación (el fixture de prueba crea la tabla según §2 del modelo).
- Spec IDs: `TC-004`, `TC-011`, `NFR-004`, `NFR-006`, `FR-010`, `FR-013`, `FR-014`, `BR-001`, `BR-006`, `BR-007`, `BR-009`, `BR-010`, `VAL-004`, `VAL-005`.
- ADRs: ADR-022, ADR-023, ADR-025, ADR-027, ADR-031, ADR-032, ADR-039, ADR-040; addendum (tamaños 14/13/12/3).
- AP / API / MSG: `AP-001`..`AP-006`, `AP-008`..`AP-018`, `AP-020`..`AP-033` (`AP-007` y `AP-019` retirados: no se implementan).
- Spikes: SPK-006, SPK-007, SPK-008, SPK-009, SPK-010, SPK-011, SPK-012, SPK-013, SPK-014, SPK-015.
- Tests: unitarias del adaptador con cliente simulado (construcción exacta de cada solicitud: items, expresiones, condiciones, índices); batería de contrato de puertos ejecutada contra el doble en memoria (unitaria) y contra DynamoDB Local (`*IT`) para garantizar que ambos se comportan igual; `*IT` de concurrencia (sobreventa, misma clave, una Order activa, confirmar frente a expirar).
- Done criteria: criterio común; cada operación de §5 cubierta por al menos una prueba unitaria y una `*IT`.
- Depends on: INC-004.

### INC-006 — Adaptadores de SQS

- Goal: publicación con presupuesto y circuit breaker, y los dos bucles reactivos de consumo.
- Modules: `infrastructure`.
- Components / ports: `CMP-011` (Order queue publisher y Provisioning queue publisher), `CMP-012` (bucle de Orders), `CMP-025` (bucle de aprovisionamiento), `CMP-026` (circuit breaker de publicación).
- Contenido: serialización de `MSG-001` y `MSG-002` versión 1 con sus atributos (`messageType`, `schemaVersion`, contexto de traza, `publisher`, `correlationId`); versión o tipo desconocido = venenoso; publicación con composición timeout 500 ms → circuit breaker (20/10/50 %/10 s/2) → retry (3 intentos, 2 s) y sin reintentos del SDK apilados; clasificación de errores (SPK-018); exposición del estado del circuito al puerto; bucle de Orders (20 s, 10 mensajes, concurrencia 16, backoff por visibilidad 5/15/30/60 s con jitter, posponer hasta el fin del lease, `ApproximateReceiveCount` para la última recepción, pausa/reanudación gobernada por una compuerta que en INC-007 se conecta al circuito del Payment Mock, recepción limitada en semiabierto, apagado ordenado); bucle de aprovisionamiento (1 mensaje, concurrencia 1, heartbeat a 120 s cada 30 s con progreso, parada sin progreso en 60 s, backoff 30/60/120/240 s).
- Spec IDs: `TC-005`, `TC-009`, `TC-010`, `FR-005`, `FR-007`, `FR-017`, `ALT-004`, `ERR-004`, `ERR-005`, `ERR-015`, `NFR-003`.
- ADRs: ADR-026, ADR-029, ADR-035, ADR-039.
- AP / API / MSG: `MSG-001`, `MSG-002`.
- Spikes: SPK-016, SPK-017, SPK-018, SPK-019.
- Tests: unitarias con cliente simulado y tiempo virtual (presupuesto, circuito, backoff, heartbeat, pausa); `*IT` contra LocalStack (publicar, recibir, cambiar visibilidad, contador de recepciones, redrive a DLQ creada por el fixture, heartbeat).
- Done criteria: criterio común.
- Depends on: INC-004.

### INC-007 — Adaptador de pago con circuit breaker

- Goal: autorizar y cancelar contra el contrato del Payment Mock con clasificación de resultados y pausa del consumo.
- Modules: `infrastructure`.
- Components / ports: `CMP-013` (Payment gateway), `CMP-026` (circuit breaker del Payment Mock y eventos de estado), conexión de la compuerta de pausa del bucle de Orders (`CMP-012`) y del proceso de reversos.
- Contenido: tipos propios del adaptador (sin código compartido con `payment-mock`); `X-Api-Key` desde entorno; `Idempotency-Key = paymentAttemptId`; cuerpo `paymentAttemptId`, `orderId`, `eventId`, `customerRef` (sujeto), `ticketIds`; autorización con timeout 3 s, hasta 2 reintentos solo transitorios con backoff y jitter, mismo `paymentAttemptId`, plazo limitado por `expiresAt` menos el margen de aplicación (ADR-008); cancelación con una llamada y timeout 3 s; clasificación `APPROVED` / `DECLINED` (incluido `ATTEMPT_CANCELLED`) / 4xx definitivo / timeout, conexión, 5xx o circuito abierto = transitorio o "dependencia no disponible"; circuito 20/10/50 % de fallos o de llamadas > 2 s/15 s/3; rechazos y errores de contrato no abren el circuito.
- Spec IDs: `FR-015`, `FR-017`, `FR-023`, `TC-017`, `BR-020`, `BR-034`, `ALT-004`, `ALT-005`, `ALT-009`, `ERR-005`, `ERR-008`.
- ADRs: ADR-030, ADR-035, ADR-039, ADR-008, ADR-034 (regla de frontera 7).
- AP / API / MSG: `API-101` (autorizar), `API-102` (cancelar), consumidos. `API-103`..`API-111` no los consume `ticketing` (API de control y salud del mock: Payment Mock / QA).
- Spikes: SPK-021.
- Tests: unitarias contra un servidor HTTP de prueba reactivo que implementa el contrato `payment-mock.openapi.v1.yaml` (aprobación, rechazo, 4xx, 5xx, latencia, cancelación con los tres estados, cancelación previa al cobro); circuito con tiempo controlado; pausa y reanudación del bucle y de los reversos.
- Done criteria: criterio común.
- Depends on: INC-006; `IV-003`.

### INC-008 — API HTTP, seguridad y traducción de errores

- Goal: `API-001` a `API-006` conformes con `ticketing.openapi.v2.yaml`.
- Modules: `infrastructure`.
- Components / ports: `CMP-001`, `CMP-002`, `CMP-016`, `CMP-017`.
- Contenido: rutas bajo `/api/v1`; validación sintáctica y de cabeceras (`Idempotency-Key` 16..64 `[A-Za-z0-9_-]`); Resource Server (firma asimétrica, emisor, expiración con 60 s, tipo de token de acceso, cliente permitido; emisor, URL de claves y nombres de claims configurables); mapeo de `cognito:groups` a `ADMIN` / `CUSTOMER`; autorización por operación (ADR-032) con 401/403; Problem Details con `code`, `traceId` y `errors`, mapeo exhaustivo y precedencia de ADR-035; 202 con `Location`, 200 con `Idempotency-Replayed: true`, 201 con `Location`; 503 con `Retry-After`; límite de cuerpo 256 KB → 413; limitador por sujeto en `API-004` → 429 con `Retry-After`; respuestas sin detalle técnico; no se registran tokens ni cabeceras de autorización.
- Spec IDs: `FR-018`, `FR-019`, `NFR-010`, `TC-003`, `TC-008`, `TC-009`, `TC-016`, `BR-023`, `VAL-011`, `ERR-006`, `ERR-009`, `ERR-020`, `ALT-007`.
- ADRs: ADR-032, ADR-035, ADR-027, ADR-024, ADR-040, ADR-033 (identidades simuladas en pruebas).
- AP / API / MSG: `API-001`, `API-002`, `API-003`, `API-004`, `API-005`, `API-006`.
- Spikes: SPK-020, SPK-022, SPK-023.
- Tests: capa web reactiva con cliente de pruebas reactivo y JWT simulados para las cinco identidades de ADR-033 (`admin`, `customer-a`, `customer-b`, `admin-customer`, `no-groups`); validación de peticiones y respuestas contra el OpenAPI v2 (según `IV-003`); pruebas de mapeo exhaustivo de cada error de dominio.
- Done criteria: criterio común; cada código y estado HTTP de OpenAPI v2 cubierto.
- Depends on: INC-003; `IV-003`, `IV-005`, `IV-006`.

### INC-009 — Procesos periódicos del worker

- Goal: cuatro disparadores independientes y aislados.
- Modules: `infrastructure`.
- Components / ports: `CMP-014` (dispara Expire Reservations, Republish pending Orders, Reverse payments, Clean up provisioning).
- Contenido: periodicidades 5 s / 10 s / 10 s / 60 s y concurrencias 16 / 8 / 4 / 2 (ADR-028); sin solape por proceso (ciclo que no termina se omite); recorrido aleatorio de shards con jitter; fallo de ciclo registrado con espera creciente acotada; reversos pausados con el circuito del Payment Mock abierto; todo no bloqueante.
- Spec IDs: `FR-011`, `BR-030`, `NFR-003`, `ERR-003`.
- ADRs: ADR-028, ADR-035, ADR-026, ADR-025, ADR-024.
- AP / API / MSG: invoca `AP-016`, `AP-018`, `AP-028`, `AP-029` a través de los casos de uso.
- Spikes: ninguno.
- Tests: unitarias con tiempo virtual (periodicidad, no solape, aislamiento: un barrido lento no retrasa la expiración; demora de expiración ≤ 15 s con reloj controlado; pausa de reversos).
- Done criteria: criterio común.
- Depends on: INC-004, INC-007.

### INC-010 — Bootstrap, roles y observabilidad

- Goal: aplicación ejecutable con roles `api` y `worker`, configuración por entorno y señales operativas.
- Modules: `bootstrap`, `infrastructure`.
- Components / ports: `CMP-021` (composición, activación de roles, propiedades con valores por defecto de los ADR —Anexo A—, adaptadores de Clock e Id generator con UUID aleatorios), `CMP-018` (logs estructurados sin tokens, métricas técnicas y de negocio de `ticketing.aws-target.v2.md` §7, trazas propagadas por atributos de mensaje, puerto de gestión interno, salud en el puerto de aplicación), `CMP-026` (métrica de estado de cada circuito).
- Contenido: apagado ordenado (`api` completa solicitudes en curso incluida la publicación; `worker` detiene bucles y completa lo que está en vuelo); secretos solo por entorno (API key del mock, credenciales de AWS); sin secretos en configuración versionada.
- Spec IDs: `TC-002`, `TC-006` (preparación de la imagen única, sin Dockerfile), `TC-012` (parámetros), `NFR-005`, `NFR-007`.
- ADRs: ADR-034, ADR-037, ADR-032 (gestión y secretos), ADR-035, ADR-039.
- AP / API / MSG: todos, de extremo a extremo dentro de la JVM.
- Spikes: SPK-024, SPK-025.
- Tests: `*IT` de integración de componentes por rol dentro de la JVM con DynamoDB Local, LocalStack y el servidor de prueba del Payment Mock: compra feliz (`MF-003`), rechazo de pago, fallo con reverso, expiración con liberación, aprovisionamiento completo y fallido, mensaje duplicado con un único PaymentAttempt; unitarias de composición y configuración.
- Done criteria: criterio común.
- Depends on: INC-005, INC-006, INC-007, INC-008, INC-009; `IV-007`.

### INC-011 — Cierre

- Goal: estado `IMPLEMENTATION_COMPLETE`.
- Modules: todos.
- Components / ports: —
- Contenido: ejecución completa (`verify` y `verify -Pintegration`); puerta del 90 % agregada; comprobación automatizada de que cada `AC-*` asignado a `ticketing` en §7 aparece en el nombre visible de al menos una prueba; revisión de que ningún secreto ni credencial está versionado; inventario final de exclusiones de cobertura; notas de entrega para Platform y QA (variables de entorno, puertos, recursos esperados, identidades).
- Spec IDs: `NFR-013`, `DEL-001`, `DEL-003`, `EVAL-007`.
- ADRs: ADR-038, ADR-034.
- AP / API / MSG: —
- Spikes: ninguno.
- Tests: suite completa.
- Done criteria: criterio común y matriz §7 sin huecos.
- Depends on: INC-010.

### 6.1 Asignación de componentes

| CMP | Increment(s) |
|---|---|
| CMP-001 | INC-008 |
| CMP-002 | INC-008 |
| CMP-003 | INC-003 |
| CMP-004 | INC-003 |
| CMP-005 | INC-003 |
| CMP-006 | INC-003 |
| CMP-007 | INC-004 |
| CMP-008 | INC-004 |
| CMP-009 | INC-002 |
| CMP-010 | INC-005 |
| CMP-011 | INC-006 |
| CMP-012 | INC-006 (pausa conectada en INC-007) |
| CMP-013 | INC-007 |
| CMP-014 | INC-009 |
| CMP-015 | INC-004 |
| CMP-016 | INC-008 |
| CMP-017 | INC-008 |
| CMP-018 | INC-010 |
| CMP-019 | Fuera de alcance (Payment Mock) |
| CMP-020 | Fuera de alcance (Platform) |
| CMP-021 | INC-001 (esqueleto), INC-010 |
| CMP-022 | INC-004 |
| CMP-023 | INC-004 |
| CMP-024 | INC-004 |
| CMP-025 | INC-006 |
| CMP-026 | INC-006 (publicación SQS), INC-007 (Payment Mock), INC-010 (métricas) |

### 6.2 Asignación de AP, API y MSG

| ID | Increment | Nota |
|---|---|---|
| AP-001 .. AP-006 | INC-005 (lógica en INC-003 / INC-004) | |
| AP-007 | — | Retirado (sucesor AP-020) |
| AP-008 .. AP-018 | INC-005 (lógica en INC-003 / INC-004) | |
| AP-019 | — | Retirado (sucesor AP-027) |
| AP-020 .. AP-033 | INC-005 (lógica en INC-003 / INC-004) | |
| API-001 .. API-006 | INC-008 (casos de uso en INC-003) | |
| API-101, API-102 | INC-007 | Consumidos por `CMP-013` |
| API-103 .. API-111 | — | No consumidos por `ticketing`; Payment Mock / QA |
| MSG-001 | INC-006 (producción en INC-003/INC-004, consumo en INC-004) | |
| MSG-002 | INC-006 (producción en INC-003/INC-004, consumo en INC-004) | |

## 7. Traceability matrix

Cada `AC-*` aparece exactamente una vez. "Test level" es el nivel que verifica el criterio; las notas indican niveles complementarios.

| Spec ID | Increment | Test level | Notes |
|---|---|---|---|
| AC-001 | INC-010 | Integración de componentes (`*IT`) | Unitarias: definición y `ticketId` (INC-002), `CMP-003` (INC-003), `CMP-022` (INC-004); 202 con `Location` en capa web (INC-008); `AP-001`/`AP-002`/`AP-003`/`AP-025` en INC-005 |
| AC-002 | INC-003 | Unitaria de caso de uso | `AP-004`/`AP-005` en `*IT` (INC-005); contrato `API-002` (INC-008) |
| AC-003 | INC-003 | Unitaria de caso de uso | 201 `CREATED` / `FAILED` en capa web (INC-008) |
| AC-004 | INC-003 | Unitaria de caso de uso | Transacción de 14 items en `*IT` (INC-005) |
| AC-005 | INC-004 | Unitaria de caso de uso | `AP-012` en `*IT` (INC-005); flujo en INC-010 |
| AC-006 | INC-003 | Unitaria de caso de uso | Causa funcional y cuarentena invisible; contrato `API-005` (INC-008) |
| AC-007 | INC-005 | Integración DynamoDB Local (`*IT`) | Mecanismo con doble en memoria (INC-003); si SPK-009 es `DIFFERENT`, alternativa de ADR-023 |
| AC-008 | INC-004 | Unitaria con reloj inyectado | Plazo ≤ 15 s con tiempo virtual (INC-009); `AP-015`/`AP-016` en INC-005; flujo en INC-010 |
| AC-009 | INC-004 | Unitaria con reloj inyectado | Condición `expiresAt <= ahora` en `*IT` (INC-005) |
| AC-010 | INC-003 | Unitaria de caso de uso | `AP-020`/`AP-021` en `*IT` (INC-005); contrato `API-003` (INC-008) |
| AC-011 | INC-002 | Unitaria de dominio | Atributo único `state` en INC-005 |
| AC-012 | INC-002 | Unitaria de dominio | Ninguna condición acepta `SOLD` como origen (unitarias del adaptador, INC-005) |
| AC-013 | INC-002 | Unitaria de dominio | `COMPLIMENTARY` solo en "Escribir lote" (INC-004, INC-005) |
| AC-014 | INC-005 | Integración DynamoDB Local (`*IT`) | Cancelación sin parciales en todas las transacciones; dobles en memoria (INC-003, INC-004) |
| AC-015 | INC-005 | Integración DynamoDB Local (`*IT`) | Auditoría en la misma transacción y lectura `AP-017`; construcción por catálogo (INC-002) |
| AC-016 | INC-003 | Unitaria de caso de uso | Cancelación por Ticket no disponible en `*IT` (INC-005) |
| AC-017 | INC-002 | Unitaria de dominio | Nada escrito (INC-003); 400 en capa web (INC-008) |
| AC-018 | INC-003 | Unitaria de caso de uso | La consulta de disponibilidad solo usa puertos de lectura; ningún cambio en el doble en memoria |
| AC-019 | INC-004 | Unitaria de caso de uso | `AP-014` en `*IT` (INC-005); flujo en INC-010 |
| AC-020 | INC-004 | Unitaria de caso de uso | `AP-015` rechazo en `*IT` (INC-005); flujo en INC-010 |
| AC-021 | INC-004 | Unitaria de caso de uso | Última recepción con marca de reverso y 4xx sin reverso; `AP-015` en INC-005 |
| AC-022 | INC-003 | Unitaria de caso de uso | Presupuesto de publicación (INC-006) |
| AC-023 | INC-004 | Unitaria de caso de uso | Un único PaymentAttempt según el doble del Payment Mock en INC-010 |
| AC-024 | INC-004 | Unitaria de caso de uso | `Idempotency-Key = paymentAttemptId` (INC-007); `AP-013` en INC-005 |
| AC-025 | INC-004 | Unitaria de caso de uso | `AP-032` en `*IT` (INC-005) |
| AC-026 | INC-008 | Capa web reactiva | No existe operación de modificación ni de cortesía posterior; inmutabilidad (INC-002) y condición de `AP-003` (INC-005) |
| AC-027 | INC-008 | Capa web reactiva | Misma respuesta 404 para inexistente, ajena y malformada; propiedad en INC-003 |
| AC-028 | INC-008 | Capa web reactiva | Cinco identidades de ADR-033; bloqueo por identidad `CUSTOMER` (INC-003) |
| AC-029 | Fuera de alcance | Prueba de carga | QA / Resilience. El backend aporta las pruebas deterministas de concurrencia (§8) |
| AC-030 | Fuera de alcance | Prueba de carga | QA / Resilience |
| AC-031 | Fuera de alcance | Prueba de carga | QA / Resilience |
| AC-032 | INC-002 | Unitaria de dominio | Nada creado (INC-003); 400 en capa web (INC-008) |
| AC-033 | INC-002 | Unitaria de dominio | 400 en capa web (INC-008) |
| AC-034 | INC-004 | Unitaria de caso de uso | `AP-032` en INC-005; una única cancelación en INC-010 |
| AC-035 | Fuera de alcance | Pruebas del Payment Mock y extremo a extremo | Payment Mock (comportamiento del proveedor) y QA. Lado `ticketing`: tratamiento de `DECLINED`/`ATTEMPT_CANCELLED` (INC-004) y doble de contrato (INC-007) |
| AC-036 | INC-003 | Unitaria de caso de uso | Ningún registro persistido en el rechazo, `*IT` (INC-005) |
| AC-037 | INC-003 | Unitaria de caso de uso | 409 en capa web (INC-008) |
| AC-038 | INC-003 | Unitaria de caso de uso | Contrato `API-003` (INC-008) |
| AC-039 | INC-003 | Unitaria de caso de uso | `API-006` solo `ADMIN` (INC-008) |
| AC-040 | INC-004 | Unitaria de caso de uso | `AP-026`/`AP-027` en INC-005; flujo fallido en INC-010 |
| AC-041 | INC-003 | Unitaria de caso de uso | 200 con `Idempotency-Replayed` (INC-008) |
| AC-042 | INC-003 | Unitaria de caso de uso | 422 en capa web (INC-008) |
| AC-043 | INC-003 | Unitaria de caso de uso | Condición del bloqueo en `*IT` (INC-005) |
| AC-044 | INC-005 | Integración DynamoDB Local (`*IT`) | Mecanismo con doble en memoria (INC-003) |
| AC-045 | INC-003 | Unitaria de caso de uso | Borrado del bloqueo en toda transición terminal en INC-005 |
| AC-046 | INC-003 | Unitaria de caso de uso | Circuito de publicación (INC-006); 503 con `Retry-After` (INC-008) |
| AC-047 | INC-003 | Unitaria de caso de uso | Mapeo y precedencia HTTP (INC-008) |
| AC-048 | INC-003 | Unitaria de caso de uso | Cursor del adaptador (INC-005); 400 (INC-008) |
| AC-049 | INC-004 | Unitaria de caso de uso | Condición de `AP-012` en INC-005 |
| AC-050 | INC-004 | Unitaria de caso de uso | Carrera confirmar/expirar en INC-005 |
| AC-051 | INC-004 | Unitaria de caso de uso | Sin guarda de `startsAt` en la confirmación (condición de `AP-014` en INC-005) |

### 7.2 Requisitos no AC por incremento (resumen)

| Spec ID | Increment(s) |
|---|---|
| FR-001, FR-003 | INC-002, INC-003, INC-004, INC-005 |
| FR-002, FR-012, FR-020, FR-022 | INC-003, INC-005, INC-008 |
| FR-004, FR-006, FR-010, FR-016, FR-021, FR-024 | INC-003, INC-005 |
| FR-005 | INC-003, INC-004, INC-006 |
| FR-007, FR-008, FR-011, FR-015, FR-023 | INC-004, INC-005, INC-006, INC-007, INC-009 |
| FR-009 | INC-003, INC-008 |
| FR-013, FR-014 | INC-002, INC-005 |
| FR-017 | INC-002 a INC-007 |
| FR-018, FR-019 | INC-008 (propiedad en INC-003) |
| NFR-003 | INC-001 (detector), todos |
| NFR-004 | INC-003, INC-005 |
| NFR-005 | INC-002, INC-005, INC-010 |
| NFR-006 | INC-005 |
| NFR-010 | INC-008, INC-010 |
| NFR-012, NFR-013 | INC-001, INC-011 |
| NFR-015 | INC-003, INC-004, INC-005 |
| NFR-001, NFR-002, NFR-007 | QA / Resilience y Platform (el backend no los verifica) |
| TC-001, TC-002, TC-007, TC-013, TC-014, TC-015 | INC-001 |
| TC-003, TC-008, TC-016 | INC-008 |
| TC-004, TC-011 | INC-005 |
| TC-005, TC-010 | INC-006 |
| TC-009 | INC-006, INC-007, INC-008 |
| TC-017 | INC-007 |
| TC-006, TC-012 | INC-010 (parámetros); contenedores y Compose: Platform |

## 8. Mechanism tests (ADR-038)

| Mechanism | Increment | Test level |
|---|---|---|
| Sobreventa: N solicitudes simultáneas sobre el mismo Ticket y conjuntos solapados (un ganador, N − 1 rechazos, ningún parcial) | INC-003 (doble en memoria) y INC-005 (DynamoDB Local) | Unitaria concurrente / `*IT` |
| Carrera de misma `Idempotency-Key` (misma Order; contenido distinto → una Order y `IDEMPOTENCY_KEY_REUSED`; nunca `TICKETS_UNAVAILABLE` para la perdedora) | INC-003 y INC-005 | Unitaria concurrente / `*IT` |
| Reentrega del aprovisionamiento (tras habilitar y reservar, la reentrega no obtiene lease ni modifica Ticket; lease ajeno vencido se reclama) | INC-004 y INC-005 | Unitaria / `*IT` |
| Cancelación anticipada (autorización posterior `DECLINED`/`ATTEMPT_CANCELLED`; una cancelación por intento) | INC-007 | Unitaria contra el doble de contrato; la inspección real del mock la verifica QA extremo a extremo |
| Barrido frente a ruta síncrona (< 30 s no republica; > 30 s sin `enqueuedAt` sí; tiempo restante < margen de corte no) | INC-004 | Unitaria con reloj inyectado |
| Una Order activa por cliente y Event (simultáneas: una Order y `ACTIVE_ORDER_EXISTS`; tras terminal, nueva compra posible) | INC-003 y INC-005 | Unitaria concurrente / `*IT` |
| Circuit breaker (cerrado → abierto → semiabierto → cerrado; rechazos y errores de contrato no abren; pausa y reanudación) | INC-006 (publicación) e INC-007 (Payment Mock y pausa) | Unitaria con tiempo virtual / controlado |
| Carreras temporales (confirmar y expirar en ambos órdenes y simultáneos; aprobación tardía; marca de reverso) | INC-004 y INC-005 | Unitaria / `*IT` |
| Idempotencia de mensajes (entregas repetidas y simultáneas; un único PaymentAttempt) | INC-004 e INC-010 | Unitaria / `*IT` de componentes |
| Cuarentena (cancelación por condición de Ticket con Order `CREATED`: cuarentena, salida de `RESV#`, auditoría, sin escritura sobre Ticket; bloqueo retenido) | INC-004 y INC-005 | Unitaria / `*IT` |
| Detección de bloqueo | INC-001 (activo en todos los incrementos) | Detector en toda prueba reactiva |

## 9. Implementation validations

| ID | Question | Options | Recommendation | Blocking |
|---|---|---|---|---|
| IV-001 | ¿Qué librerías concretas se usan donde ningún ADR las fija? | (a) JaCoCo, ArchUnit, BlockHound, Testcontainers (módulos de LocalStack y genérico), Awaitility, validador de OpenAPI `swagger-request-validator-core`, servidor de prueba de Reactor Netty (sin dependencia nueva) para los dobles del Payment Mock, cliente HTTP asíncrono Netty del SDK de AWS (por defecto del SDK), Caffeine para cachés acotados en memoria; (b) mismas herramientas de calidad con WireMock como doble HTTP y cachés propios sin Caffeine; (c) otra combinación indicada por el humano | (a), sujeto a los spikes SPK-002 a SPK-006, SPK-019 y SPK-023 | Sí (HIGH, INC-001) |
| IV-002 | Si el BOM de Spring Boot 4.x gestiona JUnit 6 (Jupiter), ¿se acepta como cumplimiento de `TC-014` o se fuerza JUnit 5.x? | (a) Aceptar la versión gestionada por el BOM (misma plataforma Jupiter; el requerimiento dice "frameworks como JUnit 5"); (b) forzar JUnit 5.x con riesgo de incompatibilidad con el soporte de pruebas de Spring 7 | (a); si SPK-003 muestra que el BOM gestiona 5.x, la pregunta queda sin efecto | Sí (HIGH, INC-001) |
| IV-003 | ¿Cómo acceden las pruebas de contrato de `CODE_REPO` a `ticketing.openapi.v2.yaml` y `payment-mock.openapi.v1.yaml`, que residen en `SPEC_REPO`? | (a) Copia literal versionada en recursos de prueba de `ticketing` con su SHA-256 registrado y prueba que compara contra el original si `SPEC_REPO` está disponible; (b) ruta externa configurable (el build depende de la presencia de `SPEC_REPO`); (c) submódulo de git (exige tocar la raíz de `CODE_REPO`, prohibido para este agente) | (a) | No (MEDIUM; antes de INC-007) |
| IV-004 | Valores por defecto que la arquitectura usa sin cuantificar (§9.1). ¿Se aprueban los propuestos? | (a) Aprobar la tabla §9.1; (b) aprobar con cambios; (c) devolver al Architect | (a) | No (MEDIUM; antes de INC-003) |
| IV-005 | Limitador por tasa por sujeto en `API-004` (ADR-032): valor del límite y librería | (a) Resilience4j RateLimiter (misma familia aprobada para el circuit breaker), 10 solicitudes por 10 s por sujeto, rechazo inmediato, `Retry-After` = segundos hasta el siguiente permiso; (b) Bucket4j con el mismo límite; (c) componente propio no bloqueante | (a), sujeto a SPK-023 | No (MEDIUM; antes de INC-008) |
| IV-006 | ¿Cómo se obtiene la evidencia del SPK-022 (claims del token de acceso de Cognito), si el agente no puede acceder a servicios externos? | (a) El humano aporta el payload decodificado de un token de acceso de un user pool de prueba (sin firma ni secretos); (b) se autoriza al agente a consultar la documentación oficial de AWS; (c) se acepta como contrato el de ADR-033 con nombres de claims configurables y la verificación contra Cognito real queda a QA/Platform (`RISK-013`) | (a); si no es posible, (c) | No (MEDIUM; antes de INC-008) |
| IV-007 | Tecnología de exportación de métricas, trazas y formato de logs (ADR-037 no la fija) | (a) Micrometer con endpoint Prometheus en el puerto de gestión, Micrometer Tracing con puente OpenTelemetry y exportación OTLP configurable (desactivada por defecto en local), logs estructurados JSON nativos de Spring Boot; (b) registro directo de CloudWatch en Micrometer; (c) solo métricas en el endpoint de gestión, sin exportación de trazas | (a) | No (MEDIUM; antes de INC-010) |

### 9.1 Valores no cuantificados en la arquitectura (propuesta de `IV-004`)

Cada valor se implementará como propiedad configurable. Ninguno contradice un valor aprobado.

| Parámetro | Fuente que lo menciona sin valor | Propuesta | Justificación |
|---|---|---|---|
| Margen de aplicación del plazo de la autorización | ADR-008 punto 4 | 2 s | Deja tiempo para la transacción de confirmación antes de `expiresAt` dentro del margen de corte de 15 s |
| Visibilidad corta de mensajes venenosos | `ticketing.messaging.v2.md` §5.1 y §5.2 reglas 1–2; ADR-029 | 10 s (ambas colas) | Llegan a la DLQ en ≈ 5 recepciones sin ocupar capacidad |
| Espera creciente acotada tras error de un bucle de consumo | ADR-035, ADR-039 | 1 s inicial, ×2, máximo 30 s | Continuidad sin tormenta de errores |
| Espera creciente acotada tras fallo de ciclo periódico | ADR-028 regla 3 | Expiración: 1 s inicial, máximo 5 s; resto: periodo propio, máximo 5 × periodo | La expiración conserva el plazo de 15 s |
| Backoff entre intentos de publicación | ADR-026, ADR-035 ("con jitter") | 100 ms base, ×2, jitter 50 %, siempre dentro del presupuesto de 2 s | 3 × 500 ms + backoff ≤ 2 s |
| Backoff del reintento de conflicto transaccional | ADR-023, ADR-035 ("con jitter", máximo 2) | 25 ms base, máximo 200 ms, jitter 50 % | Mantiene la ruta de compra bajo el objetivo de 1 s |
| Backoff de los reintentos de autorización | ADR-035 ("backoff y jitter") | 200 ms base, máximo 1 s, jitter 50 % | 3 × 3 s + backoff < margen de corte de 15 s |
| `Retry-After` de 503 por conflicto persistente o doble fallo | ADR-035, OpenAPI v2 | 1 s | Reintento rápido con la misma clave |
| `Retry-After` de 503 por circuito de publicación abierto | OpenAPI v2 ("remaining open time") | Tiempo restante del circuito, mínimo 1 s | Literal del contrato |
| Reintentos del SDK de DynamoDB | ADR-035 ("acotado") | Modo estándar del SDK con el máximo por defecto verificado en SPK-007 (sin retry de aplicación adicional) | No apilar reintentos |
| Reintentos del SDK en el cliente de publicación SQS | ADR-035 ("no se apilan") | 0 (el retry lo gobierna la política de 3 intentos) | Literal de ADR-035 |
| Tamaño del caché de Events `ENABLED` | ADR-023, ADR-040 (`AP-006` cacheable) | 10.000 entradas, sin expiración (inmutables) | Memoria acotada |
| Concurrencia del sondeo `soldOut` entre Events de una página | ADR-040 ("concurrencia acotada") | 8 | Página de hasta 100 Events |
| Concurrencia del recuento por shards de un Event | ADR-039, ADR-040 ("concurrencia acotada") | Todos los shards del Event (≤ 32) | Latencia de `AC-030` |
| Vigencia del registro de idempotencia | ADR-027 ("mínimo 24 h") | 24 h exactas en `ttl` | Cumple el mínimo |
| Tiempo de apagado ordenado | ADR-037 ("superior a 2 s y a 30 s") | 35 s | Supera el tope de procesamiento de 30 s |
| Puertos | ADR-032 (puerto interno), OpenAPI v2 (`localhost:8080/api/v1`) | Aplicación 8080 con base `/api/v1`; gestión 8081 | Coherente con el contrato; se comunica a Platform |

## 10. Risks

| Risk | Impact | Mitigation | Related RISK-/ADR |
|---|---|---|---|
| Incompatibilidad de cobertura, Mockito, ArchUnit, BlockHound o Resilience4j con Java 25 / Spring Boot 4.x | Bloqueo de INC-001 o cambio de herramienta | Spikes en INC-001 e INC-006 antes de escribir lógica; alternativas de ADR o escalado | RISK-012; ADR-034, ADR-035, ADR-038 |
| DynamoDB Local no reproduce la exclusión transaccional o el GSI disperso | Pruebas de integración no concluyentes | SPK-009 y SPK-014; batería de contrato compartida doble/adaptador; alternativa de ADR-023 | RISK-007; ADR-023, ADR-025 |
| Divergencia entre el doble en memoria y DynamoDB | Pruebas unitarias verdes con comportamiento real distinto | Misma batería de contrato de puertos ejecutada contra ambos | ADR-038 |
| Puerta del 90 % activa desde INC-001 | Incrementos más lentos | Pruebas escritas junto al código; adaptadores con cliente simulado | NFR-013; ADR-038 |
| Copia del OpenAPI desalineada del original | Prueba de contrato contra un contrato obsoleto | `IV-003` (a): hash registrado y comparación | ADR-035 |
| Fixtures de prueba (tabla, índices, colas) distintos de los que crea `infra-init` | Comportamiento local distinto | Fixtures derivados literalmente de data model §2 y messaging §1; nota a Platform | ADR-036 |
| Pruebas dependientes del tiempo inestables | Falsos fallos | Reloj inyectado, tiempo virtual, plazos acotados con espera activa solo en `*IT` | RISK-006; ADR-008, ADR-028 |
| Aprovisionamiento de 50.000 Ticket lento en DynamoDB Local | `*IT` largas | Medición en SPK-025; Event máximo solo en una prueba de componentes | RISK-011; ADR-024 |
| Testcontainers en Windows (named pipe) | Integración no ejecutable | SPK-006; alternativa de ADR-038 (Compose del agente Platform) | RISK-007 |
| Espacio en C: (82 %) por el repositorio local de Maven | Fallo de descarga de dependencias | Vigilancia; mover `.m2` es una opción de usuario fuera del proyecto | — |

## 11. Dependencies on other agents

| Agente | Qué necesita el backend | Qué entrega el backend |
|---|---|---|
| Platform | `infra-init` equivalente a los fixtures de prueba (tabla `ticketing` con `GSI1`..`GSI4`, TTL en `ttl`, cuatro colas con atributos y redrive de `ticketing.messaging.v2.md` §1); `local-idp` con los claims de ADR-033 y las cinco identidades; Dockerfile con imagen base de Java 25; Docker Compose; variables de entorno | Lista de variables y propiedades (Anexo A, `IV-004`), puertos (8080 aplicación con `/api/v1`, 8081 gestión), rutas de salud, selección de rol, secretos esperados por entorno (API key del mock, credenciales) |
| Payment Mock | Servicio conforme a `payment-mock.openapi.v1.yaml` (incluida la cancelación anticipada `AC-035`, `X-Api-Key`, idempotencia por `paymentAttemptId`) | Ninguna dependencia de código; solo el contrato |
| QA / Resilience | Prueba de carga (`AC-029`..`AC-031`), extremo a extremo `MF-001`..`MF-004`, escenarios de resiliencia de ADR-038, invariantes posteriores a la carga, verificación de `AC-035` con la inspección del mock | Pruebas deterministas de concurrencia y mecanismos; identificadores de prueba trazables |
| Documentation | README (`DEL-002`) y colección (`DEL-004`) | Notas de configuración y de endpoints en los informes de incremento |
| Architect | Solo si un spike resulta `DIFFERENT` sin alternativa o si se rechaza `IV-004` | — |

## 12. Coherencia entre fuentes

No se detectó ninguna contradicción entre fuentes autoritativas que requiera un `IV-*` bloqueante. Observaciones registradas:

1. **Doble fallo de publicación y compensación** (ADR-026, `PurchaseServiceUnavailable` de OpenAPI v2): responde 503 sin Order ID y deja la Order en `CREATED`, que el barrido republica. La spec (`ALT-006`, `FR-006`) describe el fallo definitivo de encolado con la Order `FAILED`. No es contradicción: la spec no trata la imposibilidad de escribir la compensación y la respuesta humana a ADR-006 (autoridad 1) aprueba expresamente este caso.
2. **"JUnit 5" en `TC-014`** frente a la versión de JUnit que gestione el BOM de Spring Boot 4.x: tensión potencial, pendiente de SPK-003, planteada como `IV-002`.
3. **13 frente a 14 items** (ADR-003, ADR-025): cerrada por el addendum (14 / 13 / 12 / 3).
4. **OpenAPI v2 complementa** a la spec v5 en códigos (`TICKETS_UNAVAILABLE`, `UNKNOWN_TICKETS`, `EVENT_NOT_FOUND`), `provisionedTickets` (derivado de `provisionedBatches` y del tamaño de lote, sin atributo nuevo), 413, 429, formato de `Idempotency-Key` y tamaño de página por defecto 50; no la contradice.
5. **Valores mencionados sin cuantificar** en ADR-008, ADR-028, ADR-029, ADR-035, ADR-040 y messaging v2: no son contradicciones; se proponen en `IV-004`.

## Anexo A — Propiedades configurables con valor aprobado

Se implementan como propiedades cuyo valor por defecto es el aprobado (contrato §3.8). La duración de la Reservation (10 min) no es configurable según la spec §13.1 y se implementa como constante del dominio.

| Grupo | Propiedad (valor por defecto) | Fuente |
|---|---|---|
| Compra | Máximo de Ticket por Order (10) | ADR-003, FG-001 |
| Event | Capacidad máxima (50.000); secciones (100); filas totales (2.000); asientos por fila (1.000); rangos de cortesía (500) | FG-002, ADR-024 |
| Shards | Divisor de `availabilityShards` (2.000) y máximo (32); `RESV#` (8); `REVERSAL#` (4); `PENDQ#` (8); oleadas del sondeo (4) | ADR-022, data model §2/§4 |
| Aprovisionamiento | Tamaño de lote (100); escrituras por lotes en paralelo (4); reintentos de verificación (3); lease (60 s); umbral de estancado (3 min); republicaciones máximas (3) | ADR-024 |
| Temporales | Periodo de expiración (5 s); demora máxima objetivo (15 s, umbral de métrica); margen de corte (15 s) | AV-003, ADR-008, ADR-028 |
| Procesos periódicos | Expiración 5 s / 16; barrido 10 s / 8 (umbral 30 s); reversos 10 s / 4; limpieza 60 s / 2 | ADR-026, ADR-028 |
| Publicación SQS | Timeout por intento (500 ms); intentos (3); presupuesto (2 s) | ADR-026, ADR-035 |
| Cola de Orders | Espera (20 s); mensajes (10); visibilidad (60 s); tope de procesamiento (30 s); lease de pago (45 s); recepciones máximas (5); backoff (5/15/30/60 s); concurrencia (16) | ADR-029 |
| Cola de aprovisionamiento | Espera (20 s); mensajes (1); visibilidad (120 s); heartbeat (30 s); sin progreso (60 s); recepciones máximas (5); backoff (30/60/120/240 s); concurrencia (1) | ADR-029 |
| Pago | Timeout de autorización (3 s); reintentos (2); timeout de cancelación (3 s) | ADR-035 |
| Reversos | Calendario (10 s, 30 s, 1 min, 2 min, 5 min, luego 10 min); intentos máximos (10) | ADR-025 |
| Circuito Payment Mock | Ventana (20); mínimo (10); fallos (50 %); llamada lenta (2 s, 50 %); abierto (15 s); pruebas (3) | ADR-035 |
| Circuito publicación SQS | Ventana (20); mínimo (10); fallos (50 %); abierto (10 s); pruebas (2) | ADR-035 |
| DynamoDB | Reintentos por conflicto transaccional (2) | ADR-023, ADR-035 |
| HTTP | Límite de cuerpo (256 KB); tolerancia de reloj JWT (60 s); página de disponibilidad (50, máx. 100); listado (20, máx. 100); caché del recuento (1 s) | ADR-032, ADR-040, OpenAPI v2 |
| Idempotencia | Vigencia mínima (24 h) | ADR-027 |
