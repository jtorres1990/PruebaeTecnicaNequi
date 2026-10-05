---
artifact: platform-implementation-plan
schema_version: 1.0
feature: ticketing-event-processing
version: 1

agent:
  name: platform-engineer
  version: 1.0
  mode: planning

source:
  feature_spec:
    artifact: feature-spec/ticketing.feature-spec.v5.md
    version: 5
  architecture:
    artifact: architecture/ticketing.architecture.v2.md
    version: 2
  adr_registry: architecture/adr/ticketing.adr-registry.v1.md
  governing_adrs: [ADR-033, ADR-036]
  related_adrs: [ADR-022, ADR-024, ADR-029, ADR-030, ADR-032, ADR-034, ADR-035, ADR-038, ADR-039, ADR-040]
  data_model: architecture/ticketing.data-model.v2.md
  messaging: architecture/ticketing.messaging.v2.md
  openapi: [architecture/ticketing.openapi.v2.yaml, architecture/payment-mock.openapi.v1.yaml]
  aws_target: architecture/ticketing.aws-target.v2.md          # solo paridad local/AWS
  addendum: architecture/ticketing.consolidation-addendum.v1.md
  local_environment: implementation/ticketing.local-environment.v1.md
  backend_plan: implementation/ticketing.implementation-plan.v1.md   # INC-001..INC-008 DONE, INC-009 en curso, INC-010/011 pendientes
  backend_reviews: [human-review/ticketing.implementation-plan-review.yaml, human-review/ticketing.implementation-review.1..5.yaml]
  payment_mock_plan: implementation/payment-mock.implementation-plan.v1.md
  payment_mock_handoff: implementation/payment-mock-increments/PM-INC-006.report.md   # PAYMENT_MOCK_COMPLETE

repositories:
  # Inspeccionados en fb9b2d5 / f39b4eb; durante la planificación el humano avanzó ambos HEAD
  # (INC-009 y payment-mock con commit). Ningún commit es de este agente.
  spec_repo: { path: 'D:\Nequi\PruebaeTecnicaNequi', branch: main, commit_at_start: fb9b2d5, commit_at_end: 12b7c62 }
  code_repo: { path: 'D:\Nequi\ticketing-platform', branch: main, commit_at_start: f39b4eb, commit_at_end: 31b39ac }

status: READY_FOR_HUMAN_PLAN_REVIEW
human_validation_required: true
increments: 7
spikes: 14
platform_validations: 13
blocking_items: 9            # PLAN + 8 PLAT-IV
implementation_can_start: false

generated_at: 2026-10-04
---

# Platform Implementation Plan v1 — Ticketing Event Processing

## 0. Precondición y fuentes

Verificada el 2026-10-04, sin mutar ningún repositorio:

| Condición | Valor observado | Resultado |
|---|---|---|
| `ticketing.architecture.v2.md` `status` / `development_can_start` / `blocking_items` | `READY_FOR_DEVELOPMENT` / `true` / `0` | PASS |
| `ticketing.architecture-review.yaml` `review.status` | `APPROVED` | PASS |
| `gate.pending_blocking_items` / `gate.development_can_start` | `[]` / `true` | PASS |
| `ticketing.local-environment.v1.md` `status` | `READY` | PASS |
| ADR-033 y ADR-036 en el registro | `ACCEPTED` (sucesores de ADR-014 y ADR-017) | PASS |
| Docker Desktop / Compose | Engine 29.8.0 `linux/amd64`, Compose v5.5.1, Buildx 0.37.1, contexto `desktop-linux` | PASS |
| `CODE_REPO` | `D:\Nequi\ticketing-platform`; `ticketing/` y `payment-mock/` pertenecen a otros agentes | PASS |
| `implementation/platform.implementation-plan.v1.md` | No existía → MODO 1 | PASS |

Fuentes leídas completas: arquitectura v2; registro de ADR; ADR-022, 024, 029, 030, 032, 033, 034, 036, 038, 039 (ADR-035 y ADR-040 revisados en lo relativo a salud, apagado y puertos); data model v2; messaging v2; aws-target v2 (§10 y §11 para paridad); addendum; local-environment v1; plan del backend y sus revisiones 1..6; informes INC-001..INC-008 (notas a Platform); plan, revisión e informes PM-INC-001..006 del Payment Mock; contratos de agente de backend, Payment Mock y QA (límites de propiedad). De la feature spec v5: TC-001..TC-017, §22.1 y DEL-001..DEL-005. No se usó ningún ADR `SUPERSEDED` ni artefacto v1.

Decisiones vinculantes aplicadas: TC-006, TC-012, DEL-005; ADR-036 (topología, `infra-init` idempotente, misma imagen para `api` y `worker`, salud y dependencias explícitas, variables y ejemplo ficticio, multi-etapa y no root, datos efímeros, LocalStack fijado); ADR-033 (emisor local con claims de forma Cognito, sujetos arbitrarios, cinco identidades, más de 1.000 tokens `CUSTOMER`); ADR-034 (builds e imágenes independientes); IV-004 del backend (puertos 8080 con `/api/v1` y 8081 de gestión, apagado ordenado de 35 s); IV-006 (c) del backend (contrato de claims de ADR-033 con nombres configurables; verificación contra Cognito real a QA/Platform, `RISK-013`).

## 1. Scope and ownership

### 1.1 Rutas que este agente escribirá (solo tras la aprobación)

| Ruta en `CODE_REPO` | Contenido |
|---|---|
| `docker-compose.yml` | Topología completa de §4; crece por incremento |
| `docker-compose.*.yml` | Ninguno previsto. Solo si una respuesta humana aprueba un overlay (p. ej. persistencia opcional, `PLAT-IV-011` opción b) |
| `.env.example` | Variables de plataforma con valores inequívocamente ficticios |
| `.gitignore` (raíz) | Hoy no existe; se crea solo con `.env` y artefactos generados de `platform/` (adición mínima) |
| `platform/.gitattributes` | Fin de línea LF para scripts y Dockerfiles de `platform/**` |
| `platform/infra-init/**` | Imagen derivada, scripts, definiciones literales de tabla y colas |
| `platform/local-idp/**` | Emisor OIDC local y modos `generate` (`load-token-generator`) y `verify` (según `PLAT-IV-002`) |
| `platform/verify/**` | Verificadores de contrato de recursos, identidad, salud y seguridad |
| `ticketing/Dockerfile`, `ticketing/.dockerignore` | Únicos archivos en la carpeta del backend |
| `payment-mock/Dockerfile`, `payment-mock/.dockerignore` | Únicos archivos en la carpeta del Payment Mock |

En `SPEC_REPO`: este plan, su revisión, `implementation/platform-increments/PLAT-INC-NNN.report.md` y, si surgen, `human-review/platform.implementation-review.<n>.yaml`.

### 1.2 Fuera de alcance

Código, POM, wrapper, recursos, configuración Spring y pruebas de `ticketing/**` y `payment-mock/**`; Terraform/`infra/**`; AWS real; `.github/**`; `load-tests/**` y `qa/**` (QA); README final y colección (Documentation); pruebas E2E, resiliencia de negocio y carga; commits y push.

### 1.3 Convivencia

- Al inicio, INC-009 del backend estaba en curso en `ticketing/**`; al cierre de esta planificación ya tiene commit (`3df9b19`) y el backend sigue trabajando en el árbol (cambios sin commit en `bootstrap/pom.xml`, `domain/**`, pruebas de `infrastructure`). Platform no toca esa carpeta salvo `ticketing/Dockerfile` y `ticketing/.dockerignore`, que hoy no existen.
- `payment-mock/` estaba completo sin commit al inicio; al cierre tiene commit (`31b39ac`). Se lee; no se modifica.
- En `SPEC_REPO` solo se crean los dos entregables de esta invocación.

## 2. Current environment

Inspección de solo lectura (sin construir, descargar ni levantar):

| Elemento | Observado | Consecuencia |
|---|---|---|
| Docker | Engine 29.8.0, 12 CPU, 8,0 GB, overlay2; BuildKit/Buildx 0.37.1 | Compose builds con BuildKit (cache mounts, heredocs, `docker build --check`) disponibles |
| Compose | v5.5.1 | `depends_on.condition` `service_healthy` / `service_completed_successfully` se verifica en `PLAT-SPK-009` |
| Imágenes locales fijadas | `amazon/dynamodb-local:3.3.1` `sha256:ff89bd48…bc0dab`; `localstack/localstack:4.14.0` `sha256:3ebc3759…bc364a` (coinciden con local-environment v1) | Se reutilizan; no se reabre su verificación |
| Imágenes Java 25, AWS CLI u OIDC | Ninguna presente | Requieren `PLAT-SPK-001`, `-004`, `-005` y aprobación (`PLAT-IV-001`, `-002`, `-004`) |
| Contenedores/volúmenes ajenos | Contenedor detenido `dynamodb-local` (otro proyecto, red `dynamodb_default`), volumen `portainer_data` | Prohibido `container_name`; nombre de proyecto Compose explícito; nunca limpiar recursos ajenos |
| Puertos del host | 8090 ocupado por `WsToastNotification.exe` (PID 7728); 8000, 8080, 8081, 4566 libres | `PLAT-IV-008` |
| `CODE_REPO` raíz | Solo `ticketing/` y `payment-mock/`; sin `.gitignore`, `docker-compose*.yml`, `.env*` ni `platform/` | Sin solapamiento con rutas de Platform |
| `ticketing/` | Maven Wrapper 3.3.4 → Maven 3.9.16; Spring Boot 4.1.1; Enforcer Java `[25,26)`; JaCoCo y BlockHound/Mockito por `argLine`; `*IT` en el perfil `integration` (Testcontainers); `bootstrap` con `spring-boot-maven-plugin` y solo `TicketingApplication` (sin `application.yml`); `*Settings` en `infrastructure` | Artefacto, roles, variables y salud llegan con INC-010 |
| `ticketing/mvnw` | En el índice con modo `100644` (sin bit de ejecución) y LF; `core.autocrlf=true` y sin `.gitattributes` en `ticketing/` | Riesgo `R-09`: el Dockerfile invoca `sh ./mvnw` y normaliza CR en la etapa de build; recomendación al backend |
| `payment-mock/` | Wrapper propio (Maven 3.9.16); `.gitattributes` con `mvnw eol=lf`; artefacto `target/payment-mock.jar` | Handoff completo en PM-INC-006 |

## 3. Configuration contract

### 3.1 Principios

- Toda configuración entra por variables de entorno. Los secretos (API key del mock) solo desde `.env` no versionado; `.env.example` lleva valores ficticios marcados como tales.
- Las variables propias de Platform se fijan con `PLAT-IV-007`. Los **nombres de las variables del backend no se inventan**: se toman literalmente del handoff de INC-010; hasta entonces se planifica por parámetro lógico.
- Credenciales AWS locales: ficticias, por los nombres estándar del SDK (`AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY`, `AWS_REGION`), válidas solo contra los emuladores.
- `docker compose --env-file .env.example config` debe resolver sin valores reales.

### 3.2 Parámetros lógicos que la topología entrega a `ticketing` (aws-target v2 §11 y `*Settings` de INC-005..INC-008)

| Parámetro lógico | Rol | Valor local previsto | Clase / fuente | Nombre de variable |
|---|---|---|---|---|
| Rol activo | ambos | `api` / `worker` | CMP-021, ADR-034 | Handoff INC-010 |
| Nombre físico de la tabla | ambos | `ticketing` (parametrizable) | `DynamoDbAdapterSettings`; data model §2.1 | Handoff INC-010 |
| Endpoint DynamoDB | ambos | `http://dynamodb-local:8000` | `DynamoDbConnectionSettings.endpointOverride` | Handoff INC-010 |
| Región | ambos | `us-east-1` (ficticia local) | `DynamoDbConnectionSettings`, `SqsConnectionSettings` | Handoff INC-010 o `AWS_REGION` |
| Endpoint SQS | ambos | `http://localstack:4566` | `SqsConnectionSettings.endpointOverride` | Handoff INC-010 |
| URL cola Orders | ambos | Determinista según `PLAT-SPK-010` | `SqsPublisherSettings`, `OrderConsumerSettings` | Handoff INC-010 |
| URL cola aprovisionamiento | ambos | Determinista según `PLAT-SPK-010` | `SqsPublisherSettings`, `ProvisioningConsumerSettings` | Handoff INC-010 |
| Emisor esperado (`iss`) | `api` | Valor fijo de `PLAT-IV-003` | `AccessTokenSettings.issuer` | Handoff INC-010 |
| URL del JWK set | `api` | URL en la red de Compose de `PLAT-IV-003` | `AccessTokenSettings.jwkSetUri` | Handoff INC-010 |
| `client_id` permitidos | `api` | Cliente ficticio de `PLAT-IV-003` | `AccessTokenSettings.allowedClientIds` | Handoff INC-010 |
| URL base del Payment Mock | `worker` | `http://payment-mock:8090` (servers del contrato) | `PaymentGatewaySettings.baseUrl` | Handoff INC-010 |
| API key del Payment Mock (secreto) | `worker` | Misma variable de `.env` que alimenta al mock | `PaymentGatewaySettings.apiKey` | Handoff INC-010 |
| Credenciales AWS ficticias | ambos | Valores ficticios | Cadena por defecto del SDK | `AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY` |
| Puertos | `api` | 8080 (`/api/v1`) y 8081 (gestión) | IV-004 backend | Handoff INC-010 |
| Rutas de salud | ambos | — | ADR-032, SPK-024 backend | Handoff INC-010 |

Los demás valores ajustables (ADR-003, 024, 026, 028, 029, 032, 035, 040) conservan los valores por defecto aprobados que implementa el backend; Compose no los sobrescribe.

### 3.3 Parámetros de los demás servicios

| Servicio | Variable | Valor | Fuente |
|---|---|---|---|
| `payment-mock` | `PAYMENT_MOCK_API_KEY` | Obligatoria, desde `.env`; Compose falla con mensaje si falta (`${VAR:?}`) | PM-INC-006, PM-IV-004 |
| `payment-mock` | `SERVER_PORT` | No se fija (8090 por defecto) | PM-INC-006 |
| `infra-init` | Nombres físicos de tabla y colas, región, endpoints | Por defecto los nombres lógicos de data model §2.1 y messaging §1 | ADR-036 |
| `local-idp` | Emisor, cliente, vigencias, puerto | `PLAT-IV-003` | ADR-033 |
| `load-token-generator` | Número de tokens (por defecto 1.200), vigencia, identificador de lote | `PLAT-IV-009` | ADR-033, ADR-036 |

## 4. Local topology

### 4.1 Servicios (ADR-036; nombres exactos)

| Servicio | Imagen | Depende de | Host (según `PLAT-IV-008`) | Salud / finalización |
|---|---|---|---|---|
| `dynamodb-local` | `amazon/dynamodb-local:3.3.1@sha256:ff89…0dab`, `-jar DynamoDBLocal.jar -inMemory -sharedDb` | — | `127.0.0.1:8000` inspección | Operación `ListTables` real dentro del contenedor (`PLAT-SPK-008`) |
| `localstack` | `localstack/localstack:4.14.0@sha256:3ebc…364a`, `SERVICES=sqs`, sin token, sin socket de Docker | — | `127.0.0.1:4566` inspección | SQS `available/running` y `ListQueues` real (`PLAT-SPK-008`) |
| `local-idp` | Según `PLAT-IV-002` | — | `127.0.0.1:${LOCAL_IDP_HOST_PORT}` tokens | Discovery y JWKS responden 200 |
| `payment-mock` | Propia, desde `payment-mock/` | — | `127.0.0.1:${PAYMENT_MOCK_HOST_PORT}` → 8090 | `GET /health` 200 |
| `infra-init` | Según `PLAT-IV-004` | `dynamodb-local` y `localstack` `service_healthy` | No | Un solo uso; `restart: "no"`; código 0 solo si todo coincide |
| `ticketing-api` | Propia, desde `ticketing/` (imagen única) | `infra-init` `service_completed_successfully`; `local-idp` `service_healthy` | `127.0.0.1:8080`, `127.0.0.1:8081` | Ruta de salud de INC-010 en 8080 |
| `ticketing-worker` | **La misma imagen** que `ticketing-api` | `infra-init` `service_completed_successfully`; `payment-mock` `service_healthy` | Ninguno | Salud del rol `worker` según INC-010 |
| `load-token-generator` (perfil `load`) | La de `local-idp` en modo `generate` (si `PLAT-IV-002` a) | `local-idp` `service_healthy` | No | Un solo uso; manifiesto con conteo > 1.000 |
| `load-test` (perfil `load`) | Herramienta de QA (`PLAT-IV-010`) | `ticketing-api` `service_healthy`; `load-token-generator` `service_completed_successfully` | No | La define QA |

Orden resultante: (1) `dynamodb-local`, `localstack`, `local-idp`, `payment-mock`; (2) `infra-init`; (3) `ticketing-api`, `ticketing-worker`; (4) perfil `load`: `load-token-generator`, luego `load-test`. El perfil `load` no arranca por defecto.

### 4.2 Reglas de Compose

- Nombre de proyecto explícito (`name: ticketing-platform`) para acotar `down` y verificaciones; prohibido `container_name` (colisión con el contenedor ajeno `dynamodb-local`).
- Una red explícita (`ticketing-net`); los servicios se alcanzan por nombre. Solo se publican al host los puertos de §4.1, ligados a `127.0.0.1` (`PLAT-IV-008`). `ticketing-worker` no publica nada.
- Datos efímeros: DynamoDB Local `-inMemory`, LocalStack sin persistencia, sin volúmenes de datos. El único volumen es `load-tokens` del perfil `load` (`PLAT-IV-009`), eliminado con `down -v`.
- `ticketing-api` y `ticketing-worker` comparten una sola definición de build (un servicio construye, el otro referencia la misma `image:`); `PLAT-SPK-013` verifica el mismo image ID.
- `stop_grace_period` de `api`/`worker` mayor que el apagado ordenado aprobado de 35 s (`PLAT-IV-012`).
- Ningún `restart` que convierta un `infra-init` fallido en bucle; los servicios persistentes no reinician automáticamente salvo aprobación.
- Imágenes propias: `read_only: true` con `tmpfs: /tmp`, `cap_drop: [ALL]`, `security_opt: [no-new-privileges:true]` cuando el servicio lo soporte (`PLAT-SPK-014`); las excepciones de los emuladores se verifican y documentan.

## 5. Images and supply chain

| Imagen | Origen | Fijación | Usuario final | Notas |
|---|---|---|---|---|
| DynamoDB Local | Externa, ya fijada | tag + digest de local-environment v1 | El de la imagen (se inspecciona) | Sin cambios |
| LocalStack | Externa, ya fijada | tag + digest | El de la imagen (se inspecciona) | Sin token; alternativa con token solo documentada por Documentation |
| JDK 25 (build) y JRE 25 (runtime) | `PLAT-IV-001` | tag inmutable + digest verificado en `PLAT-SPK-001` | — / uid no root creado en la etapa final | Mismas bases para `ticketing`, `payment-mock` y, si aplica, `local-idp` |
| `infra-init` | `PLAT-IV-004` | tag + digest | No root | Scripts y definiciones copiados en una imagen derivada |
| `ticketing` | `ticketing/Dockerfile` | Etiqueta local `PLAT-IV-007` (no `latest`) | No root | Multi-etapa; `sh ./mvnw` con el wrapper del proyecto; política de pruebas `PLAT-IV-006`; artefacto según INC-010 |
| `payment-mock` | `payment-mock/Dockerfile` | Etiqueta local | No root | Multi-etapa; `payment-mock.jar`; límite ≥ 384 MB |
| `local-idp` | `PLAT-IV-002` | Etiqueta local o tag + digest | No root | — |

Reglas: sin `latest`; toda imagen externa `nombre:tag@sha256:digest`; caché de Maven por montaje de caché de BuildKit (nunca en capas); sin secretos en `ARG`, `ENV` de build ni capas; `.dockerignore` excluye `target/`, `.git`, IDE, logs y cualquier `.env`; `ENTRYPOINT` en forma exec para que la JVM reciba `SIGTERM`; no se instalan herramientas de compilación en runtime.

## 6. Resource definitions

`infra-init` es la única pieza que crea recursos (data model v2 §7, aws-target v2 §10). Sus definiciones se escriben como archivos literales derivados de los documentos vigentes y coinciden con los fixtures del backend (`DynamoDbLocalSupport.createTable`, `LocalStackSqsSupport.createQueues`) y con el handoff a IaC (aws-target v2 §11 #5 y #6).

### 6.1 Tabla (data model v2 §2.1–§2.2)

| Elemento | Valor |
|---|---|
| Nombre | Lógico `ticketing`; físico parametrizable (por defecto `ticketing`) |
| Claves | `PK` String (HASH), `SK` String (RANGE) |
| Capacidad | On-demand (`PAY_PER_REQUEST`) |
| TTL | `ttl`, habilitado |
| Cifrado / PITR | No aplica en local (data model §7, aws-target §10) |

| Índice | Partition key | Sort key | Proyección |
|---|---|---|---|
| `GSI1` | `GSI1PK` (S) | `GSI1SK` (S) | INCLUDE: `entityType`, `eventId`, `name`, `venue`, `startsAt`, `startsAtMs`, `capacity`, `availabilityShards`, `provisioningStatus`, `createdAt`, `lastProgressAtMs`, `provisioningRepublishCount` |
| `GSI2` | `GSI2PK` (S) | `GSI2SK` (S) | INCLUDE: `ticketId`, `section`, `row`, `seat` |
| `GSI3` | `GSI3PK` (S) | `GSI3SK` (S) | KEYS_ONLY |
| `GSI4` | `GSI4PK` (S) | `GSI4SK` (S) | KEYS_ONLY |

Las listas INCLUDE se copian literalmente de data model v2 §2.2 en la definición y se comparan como conjuntos contra `DescribeTable`. El índice de reversos de pago es el rango `REVERSAL#` de `GSI3` (ADR-036, ADR-008): no hay índice adicional.

### 6.2 Colas (messaging v2 §1)

| Cola | Tipo | `VisibilityTimeout` | `ReceiveMessageWaitTimeSeconds` | `MessageRetentionPeriod` | `DelaySeconds` | `RedrivePolicy` |
|---|---|---:|---:|---:|---:|---|
| `ticketing-orders-dlq` | Standard | 60 | sin fijar (0) | 1.209.600 (14 d) | 0 | — |
| `ticketing-event-provisioning-dlq` | Standard | 120 | sin fijar (0) | 1.209.600 (14 d) | 0 | — |
| `ticketing-orders` | Standard | 60 | 20 | 3.600 (1 h) | 0 | `maxReceiveCount` 5 → ARN de `ticketing-orders-dlq` |
| `ticketing-event-provisioning` | Standard | 120 | 20 | 86.400 (1 d) | 0 | `maxReceiveCount` 5 → ARN de `ticketing-event-provisioning-dlq` |

Las DLQ se crean primero y su ARN alimenta la `RedrivePolicy`. "Hasta 10 mensajes" / "1 mensaje" son parámetros del consumidor, no atributos de cola. Sin consumidor de DLQ. Interpretaciones sometidas a `PLAT-IV-013`.

### 6.3 Algoritmo de `infra-init`

1. Espera acotada a que ambos emuladores acepten operaciones reales (además del `service_healthy`); falla con código distinto de 0 al agotar el plazo. Nunca `sleep` fijo como única comprobación.
2. Tabla: si no existe, crearla con §6.1 y esperar `ACTIVE`; si existe, validar claves, tipos, modo de capacidad, conjunto exacto de índices, claves y proyección de cada uno.
3. TTL: si está deshabilitado, habilitarlo en `ttl`; si está habilitado sobre otro atributo, incompatibilidad.
4. DLQ y colas principales: crear si no existen; si existen, comparar todos los atributos de §6.2 (con `RedrivePolicy` parseada y normalizada).
5. Tratamiento de incompatibilidades según `PLAT-IV-005` (recomendado: fallar con un diff preciso, sin modificar ni borrar).
6. Resumen sin secretos y código 0 solo si todo coincide. Prohibidos `|| true`, `exit 0` incondicional y equivalentes.

## 7. Identity contract

Contrato mínimo exigido por ADR-033, ADR-032 e INC-008 (`AccessTokenSettings`):

| Elemento | Valor |
|---|---|
| Firma | RS256 (algoritmo asimétrico aceptado por defecto por el backend), `kid` en la cabecera |
| Claims | `sub`, `cognito:groups` (array), `token_use = access`, `client_id`, `iss`, `exp`; además `iat` y `jti` (unicidad de tokens) — propuesta en `PLAT-IV-003` |
| Audiencia | No se emite ni se requiere |
| Emisor | Valor fijo e idéntico para tokens obtenidos desde el host o desde la red (`PLAT-SPK-006`); el backend recibe emisor y URL de JWKS por separado |
| Discovery / JWKS | Coherentes con el emisor y las claves activas |
| Vigencia | Configurable por solicitud con un máximo; para carga, mayor que duración + rampa (`PLAT-IV-009`) |
| Sujetos | Arbitrarios con grupo `CUSTOMER`; longitud ≤ 128 (límite de `customerRef` del contrato del mock) |
| Claves | Locales, efímeras y exclusivas del perfil local; nunca versionadas ni reutilizadas |

Identidades deterministas:

| Identidad | `sub` | `cognito:groups` |
|---|---|---|
| `admin` | `admin` | `["ADMIN"]` |
| `customer-a` | `customer-a` | `["CUSTOMER"]` |
| `customer-b` | `customer-b` | `["CUSTOMER"]` |
| `admin-customer` | `admin-customer` | `["ADMIN","CUSTOMER"]` |
| `no-groups` | `no-groups` | Claim ausente (comportamiento de Cognito sin grupos), sujeto a `PLAT-IV-003` |

Tokens de carga: lote previo de ≥ 1.200 tokens `CUSTOMER` con sujetos distintos (`load-customer-<lote>-<n>`), sin persistir en Git, sin imprimirse en logs; el backend limita `API-004` a 10 solicitudes por 10 s por sujeto (IV-005), lo que QA debe considerar al dimensionar el lote.

## 8. Quality gates

### 8.1 Estáticas (cada incremento que toque los archivos)

| Puerta | Comando / método |
|---|---|
| Compose resuelve sin secretos | `docker compose --env-file .env.example config` → 0 errores; la salida no contiene valores reales |
| Sintaxis de scripts | `sh -n` de cada script dentro de la imagen que lo ejecuta |
| Dockerfiles | `docker build --check` (comprobaciones integradas de BuildKit, sin imagen adicional) |
| Sin tags flotantes | Búsqueda de `:latest` e imágenes sin tag/digest en `docker-compose.yml` y Dockerfiles → 0 |
| Sin secretos | Búsqueda de claves privadas, `eyJ` (JWT), API keys y tokens en archivos de Platform, en `docker compose config`, en `docker history --no-trunc` y en `docker inspect` de imágenes → solo valores ficticios declarados |
| Usuario final | `docker inspect -f '{{.Config.User}}'` no vacío y `id -u` ≠ 0 en imágenes propias |
| Contexto de build | `.dockerignore` revisado; capas copiadas inspeccionadas con `docker history` |

### 8.2 Contrato de recursos

`DescribeTable` vs §6.1 (claves, tipos, modo, índices, proyecciones como conjuntos); `DescribeTimeToLive` = `ENABLED` sobre `ttl`; `ListQueues` = exactamente las cuatro; `GetQueueAttributes All` vs §6.2 con `RedrivePolicy` normalizada; segunda y tercera ejecución de `infra-init` con código 0 y marcas de creación sin cambios; pruebas negativas (tabla, índice, TTL y cola incompatibles; emulador ausente) en un proyecto Compose aislado.

### 8.3 Contrato de identidad

Discovery y JWKS desde host y desde un contenedor de la red; verificación criptográfica de la firma con la clave publicada (no solo decodificación); `iss`, `exp`, `client_id`, `token_use`, `sub`, `cognito:groups`; cinco identidades; sujeto arbitrario `CUSTOMER`; lote > 1.000 tokens distintos con tiempo medido; token vencido rechazado por el verificador.

### 8.4 Arranque integrado (tras INC-010)

Builds de ambas imágenes; arranque desde estado limpio; `infra-init` termina antes de que arranquen `api`/`worker`; mismo image ID en `api` y `worker`; salud de todos los servicios; un `infra-init` fallido impide arrancar dependientes; reinicio de servicios sin recreación destructiva; apagado ordenado dentro del periodo de gracia; perfil `load` no arranca por defecto. Solicitudes mínimas permitidas: salud, discovery, emisión, inspección de recursos y una solicitud autenticada por identidad para comprobar que `api` acepta los tokens del emisor (sin flujos de negocio).

## 9. Spikes

Lo ya `CONFIRMED` en local-environment v1 (DynamoDB Local 3.3.1, LocalStack 4.14.0 sin token, redrive, `ApproximateReceiveCount`, `ChangeMessageVisibility`, long polling) y en los spikes del backend (SPK-013 TTL, SPK-014 GSI disperso, SPK-016 atributos SQS) no se reabre; solo se reverifica dentro del entorno integrado.

| ID | Qué verificar | Método | Criterio de éxito | Alternativa | Incremento |
|---|---|---|---|---|---|
| PLAT-SPK-001 | Imágenes JDK/JRE 25 (`PLAT-IV-001`): existencia, digest, versión 25.0.x, mantenimiento, usuario no root creable, shell y cliente HTTP para healthcheck, tamaño | Consulta al registro oficial (manifiesto y digest), descarga solo del candidato aprobado, `java -version`, `id`, presencia de `curl`/`wget` | Tag y digest registrados; arranca como uid ≠ 0; mecanismo de healthcheck identificado | Siguiente opción de `PLAT-IV-001`; sonda JDK compilada en la etapa de build | 001 |
| PLAT-SPK-002 | Dockerfile multi-etapa de `ticketing` con el build real: `sh ./mvnw` en Linux (bit de ejecución, LF), agentes de Mockito/BlockHound del `argLine`, JaCoCo y puerta de cobertura en contenedor, artefacto de `bootstrap`, selección de rol | `docker build` del contexto `ticketing/` tras INC-010 | Imagen construida con la política de `PLAT-IV-006`; jar arranca en ambos roles | Revisión `platform.implementation-review.<n>` | 005 |
| PLAT-SPK-003 | Dockerfile de `payment-mock` con su build real (`verify`, 256 pruebas, BlockHound en contenedor), ejecución no root, `/health` | `docker build` del contexto `payment-mock/` | Build verde; contenedor sano como uid ≠ 0 | Revisión humana | 004 |
| PLAT-SPK-004 | Herramienta de `infra-init` (`PLAT-IV-004`) contra DynamoDB Local 3.3.1 y LocalStack 4.14.0: `create-table` con 4 GSI INCLUDE/KEYS_ONLY y `PAY_PER_REQUEST`, `update-time-to-live`, descripciones comparables, creación de colas con `RedrivePolicy`, ejecución no root (`HOME` escribible) | Contenedores efímeros del proyecto | Todas las operaciones y comparaciones expresables sin herramientas adicionales | Opción (b) de `PLAT-IV-004` | 001–002 |
| PLAT-SPK-005 | Emisor OIDC (`PLAT-IV-002`): claims exactos, sujetos arbitrarios, vigencia configurable, discovery, JWKS, emisor fijo, firma verificable | Prototipo o imagen candidata en contenedor | Token aceptado por un verificador independiente que obtiene la clave del JWKS | Otra opción de `PLAT-IV-002` | 001–003 |
| PLAT-SPK-006 | Emisor visto desde host y red: `iss` idéntico; JWKS alcanzable por URL de host y por URL de red | Tokens emitidos desde ambos lados | `iss` byte a byte igual; ambas URL sirven la misma clave | Alternativa de ADR-033 | 003 |
| PLAT-SPK-007 | Generación de > 1.000 tokens: tiempo, unicidad de `sub` y `jti`, tamaño del lote, sin tokens en logs | Generador con 1.200 tokens | < 60 s (objetivo), 1.200 únicos y verificables | Emisión por lotes en paralelo | 003 |
| PLAT-SPK-008 | Healthchecks reales: `ListTables` dentro de `dynamodb-local` (herramienta disponible en la imagen), SQS listo en `localstack`, discovery del IdP, `/health` del mock, salud de `api`/`worker` de INC-010 | `docker inspect` del estado de salud y fallos inducidos | Sano solo cuando la operación necesaria responde | Comprobación desde un contenedor auxiliar del proyecto | 002–006 |
| PLAT-SPK-009 | `depends_on` con `service_healthy` / `service_completed_successfully` en Compose v5.5.1; init fallido bloquea dependientes; `restart: "no"` | Proyecto Compose aislado de prueba | Dependiente no arranca si el init sale ≠ 0 | Revisión humana | 001–002 |
| PLAT-SPK-010 | URL de cola de LocalStack resoluble desde los contenedores y determinista para la configuración (`SQS_ENDPOINT_STRATEGY`, `LOCALSTACK_HOST`, cuenta `000000000000`) | `create-queue`/`get-queue-url` desde la red y uso por el SDK del backend en INC-006 | URL fija, alcanzable por nombre de servicio | Pasar la URL calculada por `infra-init` | 002 |
| PLAT-SPK-011 | DynamoDB Local compartido entre credenciales/regiones (`-sharedDb -inMemory`) para que `infra-init` y `ticketing` vean la misma tabla | Crear con unas credenciales y leer con otras | Tabla visible para ambos | Fijar las mismas credenciales en todos | 002 |
| PLAT-SPK-012 | Semántica de reinicio y pausa de emuladores: pérdida de colas/tabla al reiniciar; `pause`/`unpause` conserva estado | `restart` y `pause` de cada emulador en el proyecto | Comportamiento documentado para `PLAT-IV-011` y QA | — | 002 |
| PLAT-SPK-013 | Mismo image ID para `api` y `worker` construyendo una sola vez; `pull_policy` del servicio que no construye | `docker compose build/up` y `docker inspect` | IDs idénticos | Etiqueta común construida previamente | 006 |
| PLAT-SPK-014 | Apagado ordenado (`SIGTERM` a la JVM en PID 1, `stop_grace_period` > 35 s, código de salida) y compatibilidad con raíz de solo lectura + `tmpfs /tmp` y `cap_drop: ALL` | `docker compose stop` cronometrado; arranque endurecido | Parada completa dentro del periodo; servicios sanos endurecidos | Excepción documentada | 004–006 |

## 10. Increments

Criterio común de terminado: archivos solo en rutas de §1.1; puertas de §8 aplicables ejecutadas realmente; sin código ajeno modificado; sin secretos ni tags flotantes; recursos literales; informe `implementation/platform-increments/PLAT-INC-NNN.report.md` con `result: DONE`.

### PLAT-INC-001 — Contrato de configuración, estructura y spikes de imágenes y herramientas

- Objetivo: fijar variables, estructura de `platform/` e imágenes externas verificadas antes de escribir servicios.
- Rutas: `.env.example`, `.gitignore` (raíz, nuevo, mínimo), `platform/.gitattributes`, estructura de `platform/`; `docker-compose.yml` como esqueleto (`name`, red) si Compose lo acepta sin servicios, si no se crea en INC-002.
- IDs: TC-006, TC-012, DEL-005, ADR-036 (configuración por variables, ejemplo ficticio, secretos fuera del repositorio).
- Spikes: PLAT-SPK-001, PLAT-SPK-004 (parte de imagen), PLAT-SPK-005 (evidencia de la opción elegida), PLAT-SPK-009.
- Pruebas: `compose config` con `.env.example`; búsqueda de secretos y `latest`; registro de tag + digest de cada imagen aprobada.
- Terminado: imágenes de `PLAT-IV-001`, `-002`, `-004` fijadas con digest; `.env` ignorado por Git; sin valores reales en el ejemplo.
- Depende de: aprobación del plan; `PLAT-IV-001`, `-002`, `-004`, `-007`, `-008`.
- Handoff: lista de variables de plataforma a Documentation y QA.

### PLAT-INC-002 — `infra-init` idempotente para DynamoDB Local y LocalStack

- Objetivo: tabla, `GSI1`..`GSI4`, TTL y cuatro colas exactas, idempotentes y validadas.
- Rutas: `platform/infra-init/**`, `docker-compose.yml` (`dynamodb-local`, `localstack`, `infra-init`), `.env.example`.
- IDs: TC-004, TC-005, TC-012, DEL-005; ADR-036 (recursos de `infra-init`), ADR-022, ADR-029; data model v2 §2; messaging v2 §1; aws-target v2 §11 #5–#6.
- Recursos/servicios: §6; healthchecks de ambos emuladores.
- Spikes: PLAT-SPK-004, PLAT-SPK-008 (emuladores), PLAT-SPK-009, PLAT-SPK-010, PLAT-SPK-011, PLAT-SPK-012.
- Pruebas: §8.2 completo, incluida la batería negativa en un proyecto aislado; `infra-init` ejecutado tres veces; emulador detenido → fallo acotado; salida sin secretos.
- Terminado: recursos idénticos a §6 por comparación automatizada; idempotencia demostrada; incompatibilidad reportada y con código ≠ 0.
- Depende de: INC-001; `PLAT-IV-004`, `-005`, `-013`.
- Handoff: URL de colas, endpoints y región a Backend (para INC-010) y a QA; definiciones literales a Cloud/IaC.

### PLAT-INC-003 — `local-idp`, cinco identidades y tokens de carga

- Objetivo: emisor local con el contrato de §7 y generación previa de ≥ 1.200 tokens `CUSTOMER`.
- Rutas: `platform/local-idp/**`, `platform/verify/**` (verificador de identidad), `docker-compose.yml` (`local-idp`, `load-token-generator` con perfil `load`, volumen `load-tokens`).
- IDs: CMP-020, TC-016, TC-012, ADR-033 (todas sus filas), ADR-036 (`load-token-generator`), ADR-032 (validación del JWT), RISK-013.
- Spikes: PLAT-SPK-005, PLAT-SPK-006, PLAT-SPK-007, PLAT-SPK-008 (IdP).
- Pruebas: §8.3 completo; tokens no presentes en logs ni en el repositorio; el manifiesto no contiene tokens; reinicio del IdP rota la clave (comportamiento documentado).
- Terminado: cinco identidades y sujeto arbitrario verificados criptográficamente desde host y red; lote > 1.000 distinto y medido.
- Depende de: INC-001; `PLAT-IV-002`, `-003`, `-009`.
- Handoff: emisor, URL de JWKS y `client_id` a Backend (INC-010); contrato del endpoint de emisión y formato del lote a QA y Documentation.

### PLAT-INC-004 — Imagen independiente de `payment-mock`

- Objetivo: imagen propia del mock y su servicio en Compose.
- Rutas: `payment-mock/Dockerfile`, `payment-mock/.dockerignore`, `docker-compose.yml` (`payment-mock`).
- IDs: TC-017, TC-006, ADR-030 (contenedor propio), ADR-034 (build independiente), ADR-036, ADR-032 (API key por `.env`).
- Spikes: PLAT-SPK-003, PLAT-SPK-008 (mock), PLAT-SPK-014 (mock).
- Pruebas: build con la política de `PLAT-IV-006`; uid ≠ 0; sin API key → no arranca y no imprime valores; con API key → sano; `/health` 200 desde host (puerto de `PLAT-IV-008`) y red; clave ausente de logs, historial e imagen; raíz de solo lectura con `tmpfs /tmp`.
- Terminado: imagen construida solo desde `payment-mock/`, sin referencias a `ticketing`.
- Depende de: INC-001; `PLAT-IV-006`, `-008`. `payment-mock/` ya tiene commit (`31b39ac`): el informe registra la revisión consumida.
- Handoff: nombre de imagen, puertos y comprobación de salud a QA y Documentation.

### PLAT-INC-005 — Imagen única de `ticketing` (depende de INC-010)

- Objetivo: una imagen multi-etapa de `ticketing` ejecutable con los roles `api` y `worker`.
- Rutas: `ticketing/Dockerfile`, `ticketing/.dockerignore`.
- IDs: TC-006, TC-001, TC-002, ADR-034 (una imagen con dos roles), ADR-036 (multi-etapa, no root), ADR-037 vía aws-target §2 (imagen, apagado ordenado).
- Spikes: PLAT-SPK-002, PLAT-SPK-014.
- Pruebas: build con `PLAT-IV-006`; artefacto consumido y revisión Git registrados; uid ≠ 0; historial sin secretos; arranque de cada rol contra los servicios ya existentes (INC-002..004) hasta salud; `SIGTERM` respeta 35 s.
- Terminado: una sola imagen; rol seleccionado solo por la configuración documentada por INC-010.
- Depende de: INC-001; **INC-010 DONE con su handoff** (variables, selección de rol, rutas de salud, artefacto, puertos); preferiblemente INC-011 y commit del backend; `PLAT-IV-006`.
- Handoff: resultados de build y arranque a Backend si aparece un defecto.

### PLAT-INC-006 — Servicios de aplicación, orden de arranque y perfil `load` (depende de INC-010)

- Objetivo: topología completa de §4 con dependencias y salud.
- Rutas: `docker-compose.yml` (`ticketing-api`, `ticketing-worker`, `load-test`), `.env.example`.
- IDs: TC-012, DEL-005, ADR-036 (servicios, orden, reglas), ADR-032 (8081 solo local), ADR-033 (perfil `load`).
- Spikes: PLAT-SPK-008 (`api`/`worker`), PLAT-SPK-013, PLAT-SPK-014.
- Pruebas: §8.4; inyección de fallo de `infra-init` (proyecto aislado) → `api`/`worker` no arrancan; perfil `load` ausente por defecto y `load-token-generator` completado bajo `--profile load`; `load-test` según `PLAT-IV-010`.
- Terminado: todos los servicios base sanos o completados en el orden aprobado; `api` y `worker` con el mismo image ID; `worker` sin puertos publicados.
- Depende de: INC-002, INC-003, INC-004, INC-005; `PLAT-IV-010`, `-012`.
- Handoff: comandos y estados esperados a QA.

### PLAT-INC-007 — Verificación limpia, seguridad, reproducibilidad y handoffs

- Objetivo: demostrar el entorno desde un checkout limpio y cerrar con `LOCAL_PLATFORM_COMPLETE`.
- Rutas: `platform/verify/**`; correcciones menores en archivos de Platform si la verificación lo exige.
- IDs: TC-006, TC-012, DEL-005, ADR-036 completo, ADR-033 completo, ADR-032 (secretos), RISK-007, RISK-021.
- Pruebas: `docker compose down -v --remove-orphans` solo del proyecto `ticketing-platform`; checkout limpio exportado con `git archive` a un directorio temporal fuera de `CODE_REPO` + `.env` copiado de `.env.example` → build y `up`; barrido de seguridad de §8.1 sobre el entorno levantado; puertos publicados = los aprobados y en `127.0.0.1`; volúmenes del proyecto = solo `load-tokens` del perfil; reinicio/pausa según `PLAT-IV-011`.
- Terminado: checklist de §13 del contrato del agente cumplido.
- Depende de: INC-006; `PLAT-IV-011`; todo el código versionado por el humano (incluido `payment-mock/`); handoff de QA para `load-test` si existe.
- Handoff: §14 completo a QA, Documentation, Backend y Cloud/IaC.

### 10.1 Dependencias respecto de INC-010

| Incremento | ¿Depende de INC-010? | Motivo |
|---|---|---|
| PLAT-INC-001 a 004 | No | Usan solo la arquitectura, los handoffs ya entregados (INC-005..008, PM-INC-006) y emuladores |
| PLAT-INC-005 | Sí | Artefacto ejecutable, selección de rol, variables, puertos y salud |
| PLAT-INC-006 | Sí | Servicios `api`/`worker` y su salud |
| PLAT-INC-007 | Sí (transitiva) | Entorno completo desde checkout limpio |

## 11. Traceability

| ID / requisito | Incremento | Verificación |
|---|---|---|
| TC-006 (Docker) | 004, 005 | Imágenes propias multi-etapa y no root |
| TC-012 (Compose levanta app, DynamoDB Local, LocalStack y dependencias) | 002, 003, 004, 006, 007 | Arranque limpio integrado |
| DEL-005 (`docker-compose.yml`) | 002 → 006, 007 | `compose config` + arranque |
| CMP-020 (local identity provider) | 003 | §8.3 |
| ADR-033 — emisor OIDC simulado, solo local | 003 | Servicio solo en Compose local |
| ADR-033 — claims (`sub`, `cognito:groups`, tipo, cliente, emisor, expiración) | 003 | Verificación criptográfica y de claims |
| ADR-033 — cinco identidades deterministas | 003 | Un token por identidad verificado |
| ADR-033 — sujetos arbitrarios y > 1.000 tokens | 003 | Lote de 1.200 únicos |
| ADR-033 — vigencia configurable | 003 | Token vencido rechazado; vigencia de lote configurable |
| ADR-033 — emisor host/red y configuración separada | 003, 006 | PLAT-SPK-006; `api` acepta tokens |
| ADR-036 — siete servicios base | 002, 003, 004, 006 | `docker compose ps` |
| ADR-036 — `load-token-generator`, `load-test` (perfil) | 003, 006 | `--profile load` |
| ADR-036 — misma imagen `api`/`worker` | 005, 006 | Image ID idéntico |
| ADR-036 — `infra-init` un solo uso e idempotente | 002 | Tres ejecuciones; negativas |
| ADR-036 — salud y dependencias explícitas | 002, 006 | `depends_on` con condición; fallo inducido |
| ADR-036 — variables y ejemplo ficticio; secretos no versionados | 001, 007 | `.env.example`, `.gitignore`, barrido |
| ADR-036 — multi-etapa y no root | 003, 004, 005 | Inspección de usuario |
| ADR-036 — datos efímeros | 002, 007 | Sin volúmenes de datos |
| ADR-036 — LocalStack fijado sin token | 002 | Imagen con digest, arranque sin token |
| ADR-034 / ADR-030 — imagen separada de `payment-mock` | 004 | Build desde su carpeta |
| Tabla `ticketing` (`PK`/`SK`, on-demand) | 002 | `DescribeTable` |
| `GSI1`, `GSI2` INCLUDE literales | 002 | Comparación por conjuntos |
| `GSI3`, `GSI4` KEYS_ONLY | 002 | `DescribeTable` |
| TTL `ttl` | 002 | `DescribeTimeToLive` |
| `ticketing-orders` (60/20/1 h/0, redrive 5) | 002 | `GetQueueAttributes` |
| `ticketing-orders-dlq` (60/14 d/0) | 002 | `GetQueueAttributes` |
| `ticketing-event-provisioning` (120/20/1 d/0, redrive 5) | 002 | `GetQueueAttributes` |
| `ticketing-event-provisioning-dlq` (120/14 d/0) | 002 | `GetQueueAttributes` |
| aws-target v2 §11 #5–#6 (equivalencia con IaC) | 002 | Definiciones literales entregadas a Cloud/IaC |
| ADR-032 — gestión 8081 solo en local; secretos fuera | 006, 007 | Puertos publicados; barrido |
| ADR-032 — API key del mock por archivo no versionado | 004, 006 | `${PAYMENT_MOCK_API_KEY:?}` |
| IV-004 backend — 8080 `/api/v1`, 8081, apagado 35 s | 006 | Puertos, `stop_grace_period` |
| RISK-007, RISK-021 (emuladores, tag fijado) | 002, 007 | Reverificación integrada |
| RISK-013 (divergencia de tokens) | 003 | Contrato de claims; Cognito real fuera de alcance local |

## 12. Platform validations

| ID | Pregunta | Opciones | Recomendación | Bloquea | Necesaria antes de |
|---|---|---|---|---|---|
| PLAT-IV-001 | Imagen base exacta de Java 25 (build y runtime) y mecanismo de healthcheck | (a) Eclipse Temurin 25 JDK/JRE sobre Ubuntu (imagen oficial de Docker Hub); (b) Azul Zulu 25 JDK/JRE (mismo proveedor que `ENV-002`); (c) Amazon Corretto 25 (AL2023); en todas: healthcheck con cliente HTTP presente en la imagen, o sonda JDK compilada en la etapa de build si no lo hay | (a), tag `25.0.x` concreto + digest fijado tras PLAT-SPK-001; healthcheck con `curl` de la imagen si existe, si no sonda JDK (sin instalar paquetes) | Sí | INC-001 |
| PLAT-IV-002 | Implementación de `local-idp` | (a) Emisor mínimo propio en `platform/local-idp`, Java 25 solo con la biblioteca estándar (HTTP, RSA), sobre la imagen de PLAT-IV-001, con modos `serve`/`generate`/`verify`; (b) imagen de terceros de servidor OAuth2 simulado, fijada por digest, sujeta a PLAT-SPK-005; (c) servidor de identidad completo (Option D de ADR-033) con usuarios importados | (a): controla exactamente los claims de forma Cognito (`token_use`, `client_id`, `cognito:groups` array), fija el emisor con independencia del host de la solicitud y no añade dependencias externas; (b) suele derivar el emisor del host de la solicitud, lo que rompe la coherencia host/red | Sí | INC-001 |
| PLAT-IV-003 | Contrato operativo del emisor local | (a) Puerto 9000 en contenedor y `${LOCAL_IDP_HOST_PORT:-9000}` en `127.0.0.1`; emisor fijo `http://local-idp:9000`; `/.well-known/openid-configuration`; JWKS `/.well-known/jwks.json`; `POST /token` (form) con `identity=<una de las cinco>` o `sub=<1..128 [A-Za-z0-9._-]>&groups=<CUSTOMER,ADMIN>`, `expires_in` opcional (defecto 3.600 s, máximo 86.400 s); respuesta `{access_token, token_type: "Bearer", expires_in}`; `client_id` ficticio `ticketing-local-client`; claims de §7 más `iat` y `jti`; `no-groups` sin claim `cognito:groups`; sin autenticación (solo loopback y red local); (b) igual con `no-groups` emitiendo un grupo no reconocido; (c) otro contrato indicado por el humano | (a) | No | INC-003 |
| PLAT-IV-004 | Herramienta/imagen de `infra-init` | (a) AWS CLI v2 oficial fijada por tag + digest, imagen derivada con scripts POSIX y validación con `--query` (sugerencia de ADR-036); (b) Python slim fijada + `boto3` fijado; (c) programa Java con el SDK de AWS | (a), sujeta a PLAT-SPK-004 | Sí | INC-001 |
| PLAT-IV-005 | Recurso existente con esquema o atributos incompatibles | (a) Fallar con diff preciso y código ≠ 0, sin modificar ni borrar; remedio documentado: `docker compose down` (datos efímeros) y volver a levantar; (b) corregir automáticamente atributos mutables (`SetQueueAttributes`, TTL) y fallar solo en inmutables; (c) borrar y recrear | (a). Colas adicionales ajenas se ignoran; un índice adicional en la tabla es incompatibilidad | Sí | INC-002 |
| PLAT-IV-006 | Pruebas dentro del build de las imágenes | (a) La etapa de build ejecuta `sh ./mvnw -B verify`: en `ticketing` unitarias, arquitectura, BlockHound y puerta de cobertura (las `*IT` quedan fuera porque exigen un daemon de Docker; las verifica el backend por incremento); en `payment-mock` la suite completa; (b) `package -DskipTests` en la imagen tras un `verify -Pintegration` separado cuya revisión Git se registra; (c) (a) por defecto con una etapa opcional (b) seleccionable por `target` | (a) | Sí | INC-004 |
| PLAT-IV-007 | Variables de plataforma, nombres de imágenes propias y tratamiento de variables del backend | (a) Variables propuestas: `AWS_REGION=us-east-1`, `AWS_ACCESS_KEY_ID`/`AWS_SECRET_ACCESS_KEY` ficticias, `TICKETING_TABLE_NAME=ticketing`, `ORDERS_QUEUE_NAME`, `ORDERS_DLQ_NAME`, `PROVISIONING_QUEUE_NAME`, `PROVISIONING_DLQ_NAME` (por defecto los nombres lógicos), `PAYMENT_MOCK_API_KEY` (obligatoria, ficticia en el ejemplo), puertos de host `*_HOST_PORT`, `LOCAL_IDP_*`, `LOAD_TOKEN_COUNT=1200`, `LOAD_TOKEN_TTL_SECONDS`; imágenes propias `ticketing-platform/<servicio>:local`; variables del backend con los nombres literales del handoff de INC-010, nunca inventados; (b) otra convención | (a) | Sí | INC-001 |
| PLAT-IV-008 | Publicación de puertos al host (8090 ocupado en este equipo) | (a) Todos los puertos de host configurables con los valores del contrato por defecto (8080, 8081, 8090, 9000, 8000, 4566) y ligados a `127.0.0.1`; el `.env` de este equipo fija `PAYMENT_MOCK_HOST_PORT=18090`; el puerto interno del mock sigue en 8090; (b) mismo esquema con 18090 como valor por defecto del mock para todos; (c) no publicar el mock (contradice ADR-036, "Sí, API de control") | (a) | Sí | INC-002 |
| PLAT-IV-009 | Ubicación y formato del lote de tokens de carga | (a) Volumen Compose `load-tokens` (efímero), `/tokens/customer-tokens.csv` con cabecera `sub,access_token` y `/tokens/manifest.json` sin tokens (conteo, emisor, cliente, lote, emisión y vencimiento); `load-test` lo monta solo lectura; nada en el host; 1.200 tokens por defecto; vigencia por defecto 7.200 s configurable; (b) además, exportación a un directorio del host ignorado por Git bajo `platform/`; (c) otro formato indicado por QA | (a) | No | INC-003 |
| PLAT-IV-010 | `load-test` antes de que QA entregue su herramienta | (a) Definir el servicio en el perfil `load` con sus dependencias y montajes (`load-tests/` solo lectura, `load-tokens`), con un comando provisional que falla de inmediato indicando que falta el arnés de QA (nunca simula una prueba); se sustituye por la imagen y el comando que entregue QA; (b) no definirlo hasta el handoff de QA; (c) que QA proporcione ya imagen y comando | (a) | No | INC-006 |
| PLAT-IV-011 | Reinicio de emuladores efímeros (pierden tabla y colas) y escenario 3 de ADR-038 | (a) Mantener efímeros; documentar `docker compose pause/unpause localstack` para "cola detenida" (conserva estado) y, tras un reinicio, volver a ejecutar `infra-init` (idempotente); (b) overlay opcional de persistencia marcado como tal; (c) ganchos de inicio del emulador que recrean recursos (divide la responsabilidad de `infra-init`) | (a), confirmado por PLAT-SPK-012 | No | INC-007 |
| PLAT-IV-012 | Valores operativos derivados | (a) `stop_grace_period: 40s` para `api`/`worker` (35 s aprobados + margen); límites de memoria: `payment-mock` 384 MB (handoff), `api`/`worker` 768 MB cada uno, `local-idp` 256 MB, emuladores sin límite inicial y medidos en INC-007; JVM con `-XX:MaxRAMPercentage=75 -XX:+ExitOnOutOfMemoryError`; healthchecks cada 10 s con `start_period` acorde al arranque medido; (b) valores indicados por el humano | (a) | No | INC-006 |
| PLAT-IV-013 | Interpretaciones de atributos de recursos | (a) DLQ sin `ReceiveMessageWaitTimeSeconds` (0, "—" en messaging §1; igual que el fixture del backend); "hasta 10 / 1 mensaje" es parámetro del consumidor; sin `RedriveAllowPolicy` ni cifrado de colas en local; sin PITR ni cifrado de tabla en local; GSI sin capacidad provisionada (on-demand); `infra-init` no valida atributos no listados en §6 (p. ej. `MaximumMessageSize`); (b) otras | (a) | Sí | INC-002 |

## 13. Risks

| ID | Riesgo | Impacto | Mitigación |
|---|---|---|---|
| R-01 | La imagen JRE 25 elegida no trae cliente HTTP para healthchecks | Healthchecks más costosos o paquete extra | PLAT-SPK-001; sonda JDK sin paquetes (`PLAT-IV-001`) |
| R-02 | El build de `ticketing` en contenedor difiere del de Windows (agentes de Mockito/BlockHound, rutas, memoria del builder) | Imagen no construible | PLAT-SPK-002; caché de BuildKit; revisión humana si exige cambiar el POM (propiedad del backend) |
| R-03 | URL de cola de LocalStack con host no resoluble en la red | `api`/`worker` no publican ni consumen | PLAT-SPK-010 con estrategia de URL por ruta y host del servicio |
| R-04 | DynamoDB Local separa bases por credenciales/región | La aplicación no ve la tabla | `-sharedDb` (igual que el fixture del backend), PLAT-SPK-011 |
| R-05 | Reiniciar un emulador elimina recursos | Escenarios de resiliencia de QA fallan por motivos ajenos | PLAT-SPK-012, `PLAT-IV-011`, handoff a QA |
| R-06 | Emisor distinto visto desde host y red | 401 en una de las dos vías | Emisor fijo (`PLAT-IV-002` a, `PLAT-IV-003`); PLAT-SPK-006 |
| R-07 | Puerto 8090 del host ocupado | `payment-mock` no arranca si se publica 8090 | `PLAT-IV-008` |
| R-08 | 8 GB de Docker Desktop con perfil `load` y 50.000 Ticket | OOM o lentitud | Límites de `PLAT-IV-012`; medición en INC-007; RISK-008 de QA |
| R-09 | `ticketing/mvnw` sin bit de ejecución y riesgo de CRLF en un checkout de Windows (`core.autocrlf=true`, sin `.gitattributes` en `ticketing/`) | Build de imagen fallido | `sh ./mvnw` y normalización de CR solo en la etapa de build; recomendación al backend (`.gitattributes` y bit de ejecución) |
| R-10 | Cambios concurrentes del backend en `ticketing/` (árbol con cambios sin commit) | Revisión consumida no reproducible | Platform solo escribe los dos archivos permitidos por carpeta; registrar estado Git; pedir commit antes de INC-004/007 |
| R-11 | Retraso o ambigüedad del handoff de INC-010 | INC-005..007 bloqueados | Incrementos 001–004 independientes; revisión de implementación si falta un valor |
| R-12 | Colisión con recursos Docker ajenos (`dynamodb-local`, `dynamodb_default`) | Fallo de arranque o borrado accidental | Sin `container_name`; proyecto `ticketing-platform`; limpieza solo por proyecto |
| R-13 | Clave del IdP efímera: tokens previos invalidados al reiniciar `local-idp` | Carga con 401 | Orden del perfil `load`; documentado; regenerar el lote |
| R-14 | Limitador de `API-004` (10/10 s por sujeto) | Carga con 429 si faltan sujetos | Lote ≥ 1.200 configurable; nota a QA |
| R-15 | Deriva o retirada de imágenes externas | Build no reproducible | Tag + digest; verificación en INC-007 desde checkout limpio |

## 14. Dependencies and handoffs

| Agente | Platform necesita | Platform entrega |
|---|---|---|
| Backend (INC-010/011) | Nombres literales de variables por parámetro lógico de §3.2; mecanismo de selección de rol; rutas de salud de `api` y `worker` y si `worker` expone HTTP o gestión; artefacto ejecutable (ruta y nombre); flags de JVM obligatorios si los hay; confirmación de 8080/8081 y apagado de 35 s; recomendación (no bloqueante): `ticketing/.gitattributes` con `mvnw eol=lf` y bit de ejecución de `mvnw` | Endpoints DynamoDB/SQS, URL deterministas de colas, región y credenciales ficticias (INC-002); emisor, URL de JWKS y `client_id` (INC-003); Dockerfile y resultados de build (INC-005) |
| Payment Mock | Nada adicional (PM-INC-006 completo; código con commit `31b39ac`) | Imagen y servicio Compose (INC-004) |
| QA / Resilience | Herramienta, imagen fijada, comando y variables del arnés de carga en `load-tests/`; duración + rampa para la vigencia de tokens | Servicios, puertos, comandos Compose, contrato del emisor y cinco identidades, volumen y formato del lote, procedimiento de pausa/reinicio y reejecución de `infra-init`, límite por sujeto de `API-004` |
| Documentation | — | Variables (`.env.example`), comandos de arranque y parada, puertos, obtención de tokens, alternativa de LocalStack con token (solo documental), limitaciones locales |
| Cloud / IaC | — | Definiciones literales de tabla y colas de `infra-init` para mantener la equivalencia de aws-target v2 §11 |
| Humano | Respuestas a `PLAN` y `PLAT-IV-001`..`013`; commit de INC-010 e INC-011 cuando corresponda | Informes por incremento |

## 15. Source coherence

No se detectaron contradicciones normativas entre fuentes autoritativas. Observaciones:

1. Arquitectura v2, data model v2, messaging v2 y aws-target v2 citan la feature spec v4; la vigente es v5, que incorporó las aclaraciones 1–14 sin cambiar TC-005, TC-012 ni DEL-005 (aclaración 15 no aplicada). Sin impacto en Platform.
2. messaging v2 §1 mezcla en "Long polling" un atributo de cola (espera 20 s) y un parámetro de consumidor (10 / 1 mensaje), y marca "—" en las DLQ. Se interpreta como en el fixture del backend (`PLAT-IV-013`).
3. El handoff PM-INC-006 sugiere `-DskipTests package` para una imagen sin pruebas; el contrato de Platform exige no omitir pruebas salvo aprobación: `PLAT-IV-006`.
4. ADR-036 publica el mock al host ("API de control") y el contrato fija 8090, ocupado en este equipo: condición de entorno, no de fuentes (`PLAT-IV-008`).
5. El plan del backend asigna README y colección a "Documentation / Platform"; el contrato de Platform los asigna a Documentation. Platform solo entrega notas de handoff.
6. El contrato del mock limita `customerRef` a 128 caracteres y `ticketing` envía el `sub` sin límite (PM plan §11.3.5): los sujetos generados por Platform respetan ≤ 128.
7. La salud del `worker` es "comprobación de vida del contenedor" en AWS (aws-target §2) y ADR-036 exige salud en todos los servicios locales: el mecanismo concreto depende del handoff de INC-010.
