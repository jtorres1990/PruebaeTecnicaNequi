---
artifact: platform-increment-report
increment: PLAT-INC-002
result: DONE
code_revision: "CODE_REPO main 31b39ac + untracked platform files (docker-compose.yml, .env.example, platform/infra-init/**, platform/verify/resources.sh, platform/verify/infra-init-negative.sh); ticketing/ has uncommitted changes of the Backend Developer (INC-010), untouched; no commits by this agent"
verified_at: 2026-10-04
plan: implementation/platform.implementation-plan.v1.md
review: human-review/platform.implementation-plan-review.yaml (APPROVED)
---

# PLAT-INC-002 — `infra-init` idempotente para DynamoDB Local y LocalStack

## 1. Implemented

| Archivo (`CODE_REPO`) | Contenido |
|---|---|
| `platform/infra-init/Dockerfile` | Imagen derivada de `amazon/aws-cli:2.37.9@sha256:92de7572…160780`; copia script y definiciones, normaliza CR, `chmod` de solo lectura, `sh -n` en el build; `USER 10001:10001`, `HOME=/tmp` |
| `platform/infra-init/infra-init.sh` | Script POSIX (algoritmo del plan §6.3): espera acotada a operaciones reales, crea o valida tabla, `GSI1`..`GSI4`, TTL y las cuatro colas; compara forma canónica por conjuntos; diff preciso; códigos 0/1/2/3/4; nunca imprime credenciales |
| `platform/infra-init/resources/table.json` | Definición literal de la tabla (entrada de `create-table`), copiada de data model v2 §2.1–§2.2 |
| `platform/infra-init/resources/table.expected` | Forma canónica esperada (claves, tipos, billing, índices con claves y proyección, TTL) |
| `platform/infra-init/resources/queues.def` | Definición literal de las cuatro colas (messaging v2 §1, `PLAT-IV-013` a), DLQ primero |
| `platform/verify/resources.sh` | Verificador independiente del contrato de recursos con expectativas literales propias (no reutiliza `table.expected`) y marcas de creación |
| `platform/verify/infra-init-negative.sh` | Batería negativa en el proyecto Compose aislado `ticketing-platform-negtest` (puertos de host 18000/14566) |
| `docker-compose.yml` | Servicios `dynamodb-local`, `localstack`, `infra-init`; anclas de credenciales ficticias y nombres de recursos |
| `.env.example` | Credenciales ficticias cambiadas a alfanuméricas (ver §6) |

Códigos de salida de `infra-init`: 0 todo coincide; 1 recurso incompatible (no modifica ni borra, `PLAT-IV-005` a); 2 configuración; 3 emulador no disponible en el plazo (`INFRA_INIT_WAIT_TIMEOUT_SECONDS`, por defecto 60); 4 error AWS inesperado. Ningún `|| true` ni `exit 0` incondicional; la única tolerancia es el estado 1 de `grep` ("sin líneas"), tratado explícitamente.

Única acción sobre un recurso existente: habilitar TTL en `ttl` cuando está `DISABLED` (primera ejecución interrumpida), según el plan §6.3 paso 3. TTL habilitado sobre otro atributo = incompatibilidad.

## 2. Resources and services

| Servicio | Imagen | Configuración relevante | Host | Salud |
|---|---|---|---|---|
| `dynamodb-local` | `amazon/dynamodb-local:3.3.1@sha256:ff89bd48…bc0dab` | `-jar DynamoDBLocal.jar -inMemory -sharedDb`; uid 1000 de la imagen; `read_only`, `tmpfs /tmp:exec`, `cap_drop: ALL`, `no-new-privileges` | `127.0.0.1:${DYNAMODB_HOST_PORT:-8000}` | `ListTables` real con `curl` dentro del contenedor |
| `localstack` | `localstack/localstack:4.14.0@sha256:3ebc3759…bc364a` | `SERVICES=sqs`, `EAGER_SERVICE_LOADING=1`, `PERSISTENCE=0`, `SQS_ENDPOINT_STRATEGY=path`, `LOCALSTACK_HOST=localstack:4566`; sin token; `tmpfs /var/lib/localstack` (sin volumen anónimo); `no-new-privileges` | `127.0.0.1:${LOCALSTACK_HOST_PORT:-4566}` | `sqs` `available/running` en `/_localstack/health` **y** `ListQueues` real |
| `infra-init` | `ticketing-platform/infra-init:local` | `restart: "no"`; `depends_on` ambos `service_healthy`; `read_only`, `tmpfs /tmp`, `cap_drop: ALL`, `no-new-privileges` | — | Un solo uso; código de salida |

Recursos creados (nombres por defecto, parametrizables por `TICKETING_TABLE_NAME`, `ORDERS_QUEUE_NAME`, `ORDERS_DLQ_NAME`, `PROVISIONING_QUEUE_NAME`, `PROVISIONING_DLQ_NAME`):

| Recurso | Valor verificado |
|---|---|
| Tabla `ticketing` | `PK` S HASH, `SK` S RANGE; `PAY_PER_REQUEST`; `ACTIVE` |
| `GSI1` | `GSI1PK` HASH / `GSI1SK` RANGE (S); INCLUDE `availabilityShards, capacity, createdAt, entityType, eventId, lastProgressAtMs, name, provisioningRepublishCount, provisioningStatus, startsAt, startsAtMs, venue` (12, = data model v2 §2.2) |
| `GSI2` | `GSI2PK` / `GSI2SK`; INCLUDE `row, seat, section, ticketId` |
| `GSI3`, `GSI4` | `GSI3PK`/`GSI3SK`, `GSI4PK`/`GSI4SK`; KEYS_ONLY |
| TTL | `ENABLED` sobre `ttl` |
| `ticketing-orders-dlq` | Standard; 60 / 0 / 1.209.600 / 0; sin redrive |
| `ticketing-event-provisioning-dlq` | Standard; 120 / 0 / 1.209.600 / 0; sin redrive |
| `ticketing-orders` | Standard; 60 / 20 / 3.600 / 0; `maxReceiveCount` 5 → `arn:aws:sqs:us-east-1:000000000000:ticketing-orders-dlq` |
| `ticketing-event-provisioning` | Standard; 120 / 20 / 86.400 / 0; `maxReceiveCount` 5 → `arn:aws:sqs:us-east-1:000000000000:ticketing-event-provisioning-dlq` |

(Orden de columnas: `VisibilityTimeout` / `ReceiveMessageWaitTimeSeconds` / `MessageRetentionPeriod` / `DelaySeconds`.)

## 3. Spikes

| Spike | Resultado | Evidencia |
|---|---|---|
| PLAT-SPK-004 AWS CLI contra ambos emuladores | CONFIRMED | `create-table` con 4 GSI INCLUDE/KEYS_ONLY y `PAY_PER_REQUEST` vía `--cli-input-json`, `update-time-to-live`, `describe-*` canónicos con `--query` (JMESPath `sort`, `sort_by`, `join`), `create-queue` con `RedrivePolicy`, `get-queue-attributes`; todo como uid 10001 con `HOME=/tmp` y raíz de solo lectura. Sin herramientas adicionales. Limitación: la salida `text` del CLI muestra el booleano JSON `false` como `False`; se usan literales de cadena |
| PLAT-SPK-008 (emuladores) | CONFIRMED | `dynamodb-local` quedó `unhealthy` mientras la operación fallaba realmente (primer intento con `/tmp` `noexec`: SQLite nativo no cargaba, HTTP 500 en `ListTables`) y `healthy` en ~4 s al corregirlo. LocalStack sano solo con SQS disponible y `ListQueues` respondido |
| PLAT-SPK-009 | CONFIRMED (PLAT-INC-001); reverificado: `infra-init` arranca solo tras ambos `service_healthy` |
| PLAT-SPK-010 URL de cola | CONFIRMED | Con `SQS_ENDPOINT_STRATEGY=path` y `LOCALSTACK_HOST=localstack:4566`: `http://localstack:4566/queue/us-east-1/000000000000/<cola>`, determinista. Desde un contenedor de la red: `send-message` → `receive-message` (cuerpo íntegro, `ApproximateReceiveCount` 1) → `delete-message`; cola vacía al final. La URL no depende de la clave de acceso mientras esta no sea un número de 12 dígitos (cuenta `000000000000`) |
| PLAT-SPK-011 DynamoDB Local compartido | CONFIRMED | Con `-sharedDb`, `list-tables` con otra clave (`anotherLocalKey`) y otra región (`eu-west-1`) ve `ticketing` |
| PLAT-SPK-012 reinicio y pausa | CONFIRMED | `pause localstack` → health sin respuesta (timeout); `unpause` → las 4 colas siguen. `restart localstack` → 0 colas; `restart dynamodb-local` → 0 tablas; volver a ejecutar `infra-init` → recrea todo y sale 0. Confirma `PLAT-IV-011` (a) |

Hallazgo de entorno: DynamoDB Local 3.3.1 rechaza (`UnrecognizedClientException`) claves de acceso con caracteres no alfanuméricos; la primera ejecución con `fictitious-local-access-key` agotó la espera (código 3) como se esperaba de un fallo no transitorio. El mensaje de error de espera ahora incluye el último error recibido.

## 4. Verification

Archivo de entorno de prueba: copia temporal de `.env.example` fuera del repositorio con `PAYMENT_MOCK_HOST_PORT=18090` y una API key aleatoria (no se creó `.env` en `CODE_REPO`).

| Comando (en `D:\Nequi\ticketing-platform`) | Resultado |
|---|---|
| `docker compose --env-file .env.example config --quiet` | exit 0 |
| `docker compose --env-file <tmp> build infra-init` | Built (incluye `sh -n` del script) |
| `docker compose --env-file <tmp> up -d --wait dynamodb-local localstack` | ambos `healthy` (~4 s) |
| `docker compose --env-file <tmp> up infra-init` (1.ª ejecución) | crea tabla, habilita TTL, crea DLQ y colas; `exited with code 0` (~14 s) |
| `docker compose run --rm --no-deps -v ./platform/verify:/verify:ro --entrypoint sh infra-init /verify/resources.sh` | `RESULT 0 failed check(s)` (25 comprobaciones de tabla + conjunto exacto de 4 colas + URL, atributos y redrive de cada cola) |
| `docker compose up infra-init` (2.ª y 3.ª ejecución) | "exists: validating" en todos los recursos; `exited with code 0` |
| Verificador tras la 3.ª ejecución | `RESULT 0`; marcas idénticas: tabla `CreationDateTime 2026-10-05T04:24:32.633000+00:00`; colas `CreatedTimestamp` 1791174276 / 1791174278 / 1791174280 / 1791174283 (sin recreación) |
| `bash platform/verify/infra-init-negative.sh --env-file <tmp>` | `RESULT 14 passed, 0 failed` (detalle abajo); 0 contenedores del proyecto `ticketing-platform-negtest` al final |
| `bash -n` / `sh -n` de los tres scripts | OK |

Batería negativa (proyecto aislado; el principal no se tocó):

| Caso | Esperado | Observado |
|---|---|---|
| 1 `GSI2` KEYS_ONLY | exit 1, diff de `GSI2`; sin cambios; ninguna cola creada | exit 1; `expected: INDEX GSI2 … INCLUDE row,seat,section,ticketId` / `actual: … KEYS_ONLY -`; proyección sigue KEYS_ONLY; 0 colas |
| 2 Índice adicional `GSI5` | exit 1 | exit 1; diff en `ATTRIBUTES` e `INDEX GSI5`; los 5 índices siguen |
| 3 TTL sobre `expiresAt` | exit 1 | exit 1; `actual: TTL ENABLED expiresAt`; TTL sin cambios |
| 4 `ticketing-orders` con visibilidad 30 sin redrive | exit 1 | exit 1; diff de visibilidad, espera, retención y redrive; atributos sin cambios (`30 None`) |
| 5 Nombre físico `ticketing-alt` | exit 0 | exit 0; tabla `ticketing-alt` conforme |
| 6 DynamoDB Local detenido, espera 5 s | exit 3 acotado | exit 3 en 8 s con el último error ("Could not connect to the endpoint URL") |
| 7 Credencial vacía | exit 2 | exit 2 `AWS_SECRET_ACCESS_KEY is required` |

## 5. Security checks

- `infra-init` como `10001:10001` (`docker image inspect`), raíz de solo lectura, `cap_drop: ALL`, `no-new-privileges`.
- `docker history --no-trunc` de la imagen: 0 coincidencias de `secret`, claves ficticias o `api_key`; sin `ARG` ni `ENV` sensibles (solo `HOME`, `AWS_PAGER`).
- Logs de `infra-init`: 0 apariciones de las credenciales.
- Puertos publicados solo en `127.0.0.1`; `infra-init` no publica.
- Datos efímeros: DynamoDB `-inMemory`, LocalStack `PERSISTENCE=0` + `tmpfs`; `docker volume ls` sin volúmenes nuevos.
- Sin `:latest` en Compose ni en `platform/`.
- Excepción documentada: `localstack` se ejecuta como root (usuario por defecto de la imagen oficial; su entrypoint lo requiere) y sin `read_only`/`cap_drop`. Solo SQS, sin socket de Docker, ligado a loopback. `dynamodb-local` necesita `exec` en su `tmpfs /tmp` (biblioteca nativa SQLite).

## 6. Deviations

1. `.env.example`: `AWS_ACCESS_KEY_ID=fictitiousLocalAccessKey` y `AWS_SECRET_ACCESS_KEY=fictitiousLocalSecretKey` (antes con guiones), por la restricción alfanumérica de DynamoDB Local 3.x. Siguen siendo inequívocamente ficticias.
2. Variables adicionales de Platform no listadas en `PLAT-IV-007`, internas de `infra-init` y con valor por defecto en Compose: `DYNAMODB_ENDPOINT`, `SQS_ENDPOINT` (fijadas en Compose, no en `.env`) e `INFRA_INIT_WAIT_TIMEOUT_SECONDS` (opcional). No afectan a contratos.
3. `jq` (presente en la imagen fijada) solo se usa en la batería negativa para fabricar definiciones incompatibles; `infra-init` usa exclusivamente AWS CLI `--query` y utilidades POSIX.

## 7. Blockers

Ninguno.

## 8. Handoff to other agents

Backend (INC-010), valores para la configuración del rol `api`/`worker` dentro de la red Compose (los nombres de variables los fija el handoff de INC-010):

| Parámetro lógico | Valor local |
|---|---|
| Endpoint DynamoDB | `http://dynamodb-local:8000` |
| Endpoint SQS | `http://localstack:4566` |
| Región | `us-east-1` (`AWS_REGION`) |
| Credenciales | `AWS_ACCESS_KEY_ID` / `AWS_SECRET_ACCESS_KEY` ficticias de `.env`; **alfanuméricas** (DynamoDB Local) y **no** un número de 12 dígitos (fijaría otra cuenta en LocalStack) |
| Tabla | `ticketing` (`TICKETING_TABLE_NAME`) |
| URL cola Orders | `http://localstack:4566/queue/us-east-1/000000000000/ticketing-orders` |
| URL cola aprovisionamiento | `http://localstack:4566/queue/us-east-1/000000000000/ticketing-event-provisioning` |
| DLQ (solo inspección) | `…/ticketing-orders-dlq`, `…/ticketing-event-provisioning-dlq` |

Desde el host: DynamoDB `http://127.0.0.1:8000`, SQS `http://127.0.0.1:4566` (la URL de cola conserva el host `localstack`; los SDK envían al endpoint configurado).

QA: tras reiniciar `localstack` o `dynamodb-local` se pierden colas/tabla; volver a ejecutar `docker compose up infra-init` (idempotente). Para "cola detenida" usar `docker compose pause localstack` / `unpause` (conserva el estado). Remedio ante incompatibilidad: `docker compose down` del proyecto y volver a levantar.

Cloud/IaC: `platform/infra-init/resources/table.json` y `queues.def` son las definiciones literales para mantener la equivalencia de aws-target v2 §11 #5–#6 (en AWS se añaden cifrado y PITR, que no aplican en local).
