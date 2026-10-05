---
artifact: platform-increment-report
increment: PLAT-INC-006
result: DONE
code_revision: "CODE_REPO main 35524f9 (35524f98afb474a6f1316866cb85672e3d604ca7); uncommitted changes from this agent only: docker-compose.yml (sha256 4aa119c7…9a6bf) and .env.example (sha256 f7b80966…64824); ticketing/ and payment-mock/ unchanged; no commits by this agent"
verified_at: 2026-10-05
plan: implementation/platform.implementation-plan.v1.md
review: human-review/platform.implementation-plan-review.yaml (APPROVED)
resumed_by: human-review/platform.implementation-review.1.yaml (increment_can_resume.PLAT-INC-006 true; PLAT-IV-015 a)
new_validations: human-review/platform.implementation-review.2.yaml (PLAT-IV-016, non-blocking)
---

# PLAT-INC-006 — Servicios de aplicación, orden de arranque y perfil `load`

## 1. Implemented

| Archivo (`CODE_REPO`) | Cambio |
|---|---|
| `docker-compose.yml` | Ancla `x-ticketing-common-env` (variables literales de INC-010 §6.5.3 comunes a ambos roles) y ancla `x-ticketing-app` (imagen, endurecimiento, memoria, apagado y salud). Servicios nuevos: `ticketing-api`, `ticketing-worker` y `load-test` (perfil `load`). Comentario de cabecera con la topología por defecto. No cambia ningún servicio existente |
| `.env.example` | `TICKETING_API_HOST_PORT=8080`, `TICKETING_API_MANAGEMENT_HOST_PORT=8081` y `#TICKETING_WORKER_ID=` comentado (opcional). Solo valores ficticios o por defecto |

Diseño aplicado:

- **Una imagen, dos roles** (ADR-034, ADR-036). Ambos servicios usan `image: ticketing-platform/ticketing:local` y `pull_policy: never`. Solo `ticketing-api` declara `build: ./ticketing`. El rol cambia solo por `TICKETING_ROLE=api|worker`.
- **Variables.** Son las literales de INC-010 §6.5.3; no se inventó ningún nombre.
  - `TICKETING_DYNAMODB_TABLE` toma `${TICKETING_TABLE_NAME:-ticketing}`, la misma variable que alimenta a `infra-init`.
  - Las URL de cola se derivan de `AWS_REGION` y de los nombres de cola de `infra-init`: `http://localstack:4566/queue/<región>/000000000000/<cola>`.
  - Solo en `api`: emisor `${LOCAL_IDP_ISSUER}`, JWKS por la URL de red `http://local-idp:9000/.well-known/jwks.json` y `client_id` `${LOCAL_IDP_CLIENT_ID}`.
  - Solo en `worker`: `TICKETING_PAYMENT_BASE_URL=http://payment-mock:8090` y `TICKETING_PAYMENT_API_KEY=${PAYMENT_MOCK_API_KEY:?}`, la misma variable de `.env` que usa el mock.
  - `TICKETING_WORKER_ID=${TICKETING_WORKER_ID:-}`. Vacío significa `worker-<aleatorio>` por proceso, así que los identificadores siguen siendo únicos al escalar.
- **Salud.** `HealthProbe` sobre `http://localhost:8080/readyz` con `interval 10s`, `timeout 5s`, `retries 3`, `start_period 30s` y `start_interval 1s`.
- **Recursos y endurecimiento** (PLAT-IV-012 a): `stop_grace_period: 40s`, `mem_limit: 768m`, `read_only: true`, `tmpfs: /tmp` (montado `noexec`), `cap_drop: [ALL]`, `no-new-privileges:true` y `restart: "no"`.
- **Puertos** (PLAT-IV-008 a). El `api` publica `127.0.0.1:${TICKETING_API_HOST_PORT:-8080}:8080` y `127.0.0.1:${TICKETING_API_MANAGEMENT_HOST_PORT:-8081}:8081`. El `worker` no publica nada.
- **Dependencias** (ADR-036):
  - `ticketing-api`: `infra-init` `service_completed_successfully` y `local-idp` `service_healthy`.
  - `ticketing-worker`: `infra-init` `service_completed_successfully` y `payment-mock` `service_healthy`.
- **`load-test`** (PLAT-IV-010 a):
  - Perfil `load`, sobre la imagen local `ticketing-platform/infra-init:local` con `pull_policy: never`, como mero portador de `sh`.
  - Su comando provisional escribe en stderr que el arnés de QA no se ha entregado y termina con `exit 1`. Nunca simula una prueba.
  - Depende de `ticketing-api` `service_healthy` y de `load-token-generator` `service_completed_successfully`.
  - Monta `load-tokens` en `/tokens` en solo lectura y lleva el mismo endurecimiento que los demás.
  - El montaje de `./load-tests` queda pendiente: ver §6 (desviación 1) y `PLAT-IV-016`.

## 2. Resources and services

| Recurso | Estado al cierre |
|---|---|
| Proyecto Compose `ticketing-platform` | Levantado **como estaba al empezar** (flujo de `run-local.sh`): `dynamodb-local`, `localstack`, `local-idp` y `payment-mock` `healthy`, e `infra-init` `Exited (0)`. Sin contenedores `ticketing-*` ni `load-*`. Red `ticketing-platform_ticketing-net`. Sin volúmenes del proyecto. Datos efímeros recién creados: se perdieron los datos previos del entorno de `run-local.sh` al hacer el `down -v` exigido para arrancar desde limpio |
| Imagen `ticketing-platform/ticketing:local` | **`sha256:4f8a52fef8410141bd23540c3868c3902c2380e82f742a613db5eb8d527d4425`**, 383.596.829 B, `Created` 2026-10-05T14:37:17Z. Ver §4.2: mismas capas que `2d698ffb…` (r2) y etiquetas de Compose añadidas |
| Imagen anterior `sha256:2d698ffb…3ded` | Eliminada con `docker rmi ticketing-platform/ticketing:local` para probar el build desde Compose |
| Proyectos aislados `plat-spk013` (spike) y `plat-inc006-neg` (prueba negativa) | Eliminados (`down -v`). Sus imágenes de spike (etiqueta `plat-spike=PLAT-INC-006`) también se eliminaron por ID. Quedan 0 contenedores, redes e imágenes de ambos |
| Directorio `CODE_REPO/load-tests/` | Lo creó vacío Docker Desktop en la primera prueba del perfil `load` (§6, desviación 1). Se eliminó con `rmdir` tras comprobar que estaba vacío y fuera de Git |
| Recursos ajenos | Contenedor `dynamodb-local` (`7c60ccb8…`, `exited` desde 2026-10-04T21:00:17Z) y red `dynamodb_default` (`13f6d7a5…`) sin cambios: mismo ID y estado antes y después. Volumen anónimo `d72a66f1…`, `portainer_data` y 3 imágenes huérfanas del 1 y 2 de octubre sin tocar |
| `run-local.sh`, `README.md`, `postman/`, `.env` | No modificados. sha256 de `.env` y de `run-local.sh` idénticos antes y después |

## 3. Spikes

### PLAT-SPK-013 — Mismo image ID para `api` y `worker` y `pull_policy` — CONFIRMED

Proyecto aislado `plat-spk013`, con una imagen derivada de `infra-init:local` y Compose v5.5.1.

| Variante | `up` completo con la imagen ausente | `up b` solo con la imagen ausente |
|---|---|---|
| `a` con `build`, `b` solo con `image` (política por defecto) | Un build; mismo ID en `a` y `b` | `b` intenta `pull` del registro: `pull access denied` |
| `a` y `b` con `build`, sin `pull_policy` | Intenta `pull` del registro antes de construir (2 veces); luego construye; mismo ID | Intenta `pull` y luego construye |
| Ambos con `build` y `pull_policy: build` | Un build y mismo ID, pero **cada `up` reconstruye y recrea** los contenedores | — |
| **Elegida:** `a` con `build`, ambos con `pull_policy: never` | **Un build, sin intentos de `pull`, mismo ID** | **Error explícito** "No such image", sin acceso al registro |

La opción elegida evita dos cosas: descargar de Docker Hub una imagen con el mismo nombre que la local, y recrear los contenedores en cada `up`. En el entorno real también fue así (§4.2): un build y el mismo ID en ambos roles.

### PLAT-SPK-008 (`api` / `worker`) — Salud real — CONFIRMED

- **Estado de la sonda.** Ambos roles pasan a `healthy` solo cuando `/readyz` responde 200. Sanos unos 2 s después de arrancar el contenedor (`Started TicketingApplication in 4,6 s`) y unos 4 s después de un `restart`.
- **Comprobación desde la red.** `worker` `:8080/readyz` 200, `:8080/livez` 200, `:8081/actuator/health` 200 y `:8080/api/v1/events` 401.
- **Comportamiento en el apagado.** `/readyz` deja de estar disponible antes de detener el trabajo (INC-010 §6.5.4). Lo implementa el backend; Platform no lo indujo por separado.

### PLAT-SPK-014 — Apagado ordenado y endurecimiento en Compose — CONFIRMED

Se cumple con todo el endurecimiento de §1:

- **`docker compose stop`:** `ticketing-api` tarda 4.795 ms y `ticketing-worker` 4.547 ms, ambos con exit 143 y `OOMKilled=false`. En los logs aparece "Commencing graceful shutdown…" y "Graceful shutdown complete".
- **`down` del entorno completo:** 7.761 ms.

## 4. Verification

Todo se ejecutó desde `D:\Nequi\ticketing-platform`, con `.env` local (`PAYMENT_MOCK_HOST_PORT=18090`) sin modificar y sin imprimir la API key.

### 4.1 Estáticas

| Comprobación | Resultado |
|---|---|
| `docker compose --env-file .env.example config --quiet` | exit 0 |
| `docker compose --env-file .env config --quiet` / `--profile load config --quiet` | exit 0 / exit 0 |
| `config --services` (por defecto) | `dynamodb-local`, `localstack`, `local-idp`, `payment-mock`, `infra-init`, `ticketing-api`, `ticketing-worker` (7) |
| `--profile load config --services` | Los 7 anteriores más `load-token-generator` y `load-test` (9) |
| Configuración resuelta con `.env.example` y perfil `load` | 0 `eyJ`, 0 `PRIVATE KEY`, 0 `AKIA`, 0 `:latest`, 0 apariciones de la API key real. Secretos solo ficticios (`fictitious…`). Imágenes externas con tag y `@sha256`; propias `ticketing-platform/*:local` |
| Sin variables obligatorias | `config` falla con el mensaje de la variable que falta (`AWS_*`, `PAYMENT_MOCK_API_KEY`) |

### 4.2 Arranque completo desde limpio (`docker compose up -d --wait`)

Preparación:

- `docker compose --env-file .env --profile load down -v --remove-orphans`, limitado al proyecto `ticketing-platform`.
- `docker rmi ticketing-platform/ticketing:local`.

Resultado de `docker compose --env-file .env up -d --wait`:

- **exit 0 en 32,6 s.** Compose construyó `ticketing-platform/ticketing:local` una sola vez y no intentó ningún `pull`.
- **Build desde la caché de capas.** BuildKit reutilizó todas las capas (`CACHED`), incluida la de `sh ./mvnw -B -ntp verify` del build 5 de PLAT-INC-005 r2: 770 pruebas, 0 fallos.
- **Contexto sin cambios.** Ese build corresponde al mismo contexto: `ticketing/` sin cambios desde `8b7d655`, el Dockerfile con sha256 `040c733d…` y `.dockerignore` con `1948d9cb…`.
- **Nuevo ID de imagen** `4f8a52fe…` (antes `2d698ffb…`). Tiene las mismas capas y la misma fecha `Created`. Solo cambian las etiquetas que Compose añade (`com.docker.compose.project`/`service`/`version`).

Orden real (`State.StartedAt` / `FinishedAt`, UTC, 2026-10-05):

| Servicio | Arranque | Sano / fin | Image ID |
|---|---|---|---|
| `dynamodb-local` | 14:52:02.34 | sano ≤ 14:52:05.1 | `9539afb50673` |
| `localstack` | 14:52:02.65 | sano ≤ 14:52:05.5 | `0e9f1067a26f` |
| `local-idp` | 14:52:02.95 | sano ≤ 14:52:05.0 | `90c0a973eacd` |
| `payment-mock` | 14:52:03.24 | sano ≤ 14:52:07.4 | `aedbe5e7bb25` |
| `infra-init` | 14:52:06.03 (tras ambos emuladores sanos) | **Exited (0)** 14:52:24.68 | `b787ede57259` |
| `ticketing-worker` | 14:52:25.02 (tras `infra-init` completado y `payment-mock` sano) | `healthy` | **`4f8a52fef841`** |
| `ticketing-api` | 14:52:25.29 (tras `infra-init` completado y `local-idp` sano) | `healthy` | **`4f8a52fef841`** |

- **Restart y publicaciones.** `restart=no` en todos.
- **Puertos publicados:**
  - `dynamodb-local`: `127.0.0.1:8000`
  - `localstack`: `127.0.0.1:4566`
  - `local-idp`: `127.0.0.1:9000`
  - `payment-mock`: `127.0.0.1:18090→8090`
  - `ticketing-api`: `127.0.0.1:8080` y `127.0.0.1:8081`
  - `ticketing-worker`: `{}`
- **Perfil `load`.** No se creó ningún contenedor `load-*`.

### 4.3 Salud, endurecimiento y logs

| Comprobación | `ticketing-api` | `ticketing-worker` |
|---|---|---|
| Health (Docker) | `healthy` | `healthy` |
| Usuario / `/proc/1/status` | `uid=10001 gid=10001`; Uid/Gid 10001 ×4; `CapEff 0`; `NoNewPrivs 1` | igual |
| Raíz / `/tmp` | `touch /app/x` → "Read-only file system"; `/tmp` escribible, `tmpfs rw,nosuid,nodev,noexec` | igual |
| `HostConfig` | `ReadonlyRootfs=true`, `CapDrop=[ALL]`, `no-new-privileges:true`, `Memory=805306368`, `StopTimeout=40`, `StopSignal=SIGTERM` | igual |
| Memoria en reposo (`docker stats`) | 254,7 MiB / 768 MiB | 237,7 MiB / 768 MiB |
| Logs tras arranque, E2E y reinicio | 14 líneas ECS; 0 WARN, 0 ERROR, 0 API key, 0 `Bearer`, 0 `eyJ` | igual |

Desde el host:

- `:8080/readyz` 200, `:8080/livez` 200, `:8081/actuator/health` 200, `:8081/actuator/prometheus` 200 y `:8080/actuator/health` 401.
- `GET /api/v1/events`: sin token 401; `customer-a` 200; `admin` 200; `no-groups` 403; firma alterada 401.

Desde la red, con un cliente efímero `--rm`:

- `ticketing-api:8080/readyz` 200.
- En el `worker`, `ticketing_circuit_state` `payment-mock` y `sqs-publication` = `CLOSED`.

Puertos del host 8082 y 8083: libres.

### 4.4 Flujo de punta a punta (colección Postman con newman)

Comando: `node C:/Users/zombr/node_modules/newman/bin/newman.js run postman/ticketing.postman_collection.json -e postman/ticketing-local.postman_environment.json --env-var paymentMockUrl=http://localhost:18090 --env-var paymentMockApiKey=<de .env>`. La clave no aparece en la salida.

| Ejecución | Solicitudes | Aserciones | Resultado |
|---|---|---|---|
| 1 | 32 / 0 fallos | 45 / **1 fallo** | exit 1. La Order quedó **`CONFIRMED`** en el segundo sondeo (~1 s). El fallo es la aserción `Order CONFIRMED` evaluada en el **primer** sondeo, cuando la Order aún estaba `CREATED` (ver nota) |
| 2 | 30 / 0 | 42 / 0 | **exit 0**. La Order estaba `CONFIRMED` en el primer sondeo |

En ambas ejecuciones pasan:

- tokens de las cinco identidades;
- creación del Event y su aprovisionamiento hasta `ENABLED`;
- catálogo y disponibilidad;
- compra con repetición idempotente (`Idempotency-Replayed: true`);
- `A-1-1` ya no disponible tras la venta;
- los casos de error 401, 403, 400, 404, 409 y 422;
- la API de control del Payment Mock.

El flujo de compra (`api` → SQS → `worker` → Payment Mock → `CONFIRMED`) funciona de punta a punta en Compose.

**Nota sobre la colección (no la modifica este agente).** En "Consultar Order (espera estado final)" la aserción `pm.test('Order CONFIRMED', …)` está fuera de la rama `else`. Se evalúa en cada sondeo, de modo que falla siempre que la Order no está confirmada en el primero, aunque se confirme después. Es un defecto de temporización de la colección. Corrección sugerida a Documentation: mover esa aserción dentro del `else`.

### 4.5 Fallo de `infra-init` → `api`/`worker` no arrancan

Proyecto aislado `-p plat-inc006-neg`, con puertos de host 28000/24566/29000/28090/28080/28081 y `TICKETING_TABLE_NAME=x` (nombre inválido):

- `up -d --wait` termina con exit 1: `service "infra-init" didn't complete successfully: exit 4`.
- Log de `infra-init`: `ValidationException … Invalid table/index name`.
- `ticketing-api` y `ticketing-worker` se quedan en `Created`, con `StartedAt=0001-01-01T00:00:00Z`: nunca arrancaron.
- Emuladores, IdP y mock quedaron sanos.
- El proyecto se eliminó con `down -v`.

### 4.6 Reinicio sin recreación e idempotencia de `up`

- **`docker compose restart ticketing-api ticketing-worker`.** Tardó 5,1 s y ambos volvieron a `healthy` en 4 s. Mismos ID de contenedor (`24b4c3adf086`, `83c27fc3a344`).
- **Segundo `docker compose up -d --wait`.**
  - Ningún `Recreate`: todos `Running`.
  - `infra-init` se volvió a ejecutar y terminó con `Exited (0)`, validando los recursos existentes.
  - Mismos ID de contenedor en los 7 servicios.
- **Datos tras reiniciar `api`/`worker`.** Se conservan: `GET /api/v1/events` devuelve los 2 Event creados por newman.

### 4.7 Perfil `load`

- **`docker compose --env-file .env --profile load up -d`** (exit 0):
  - `load-token-generator` arranca después de `local-idp` sano y termina con `Exited (0)`. Genera "1200 distinct tokens (distinct sub=1200, distinct jti=1200) in 1058 ms".
  - En el volumen hay 1.201 líneas (cabecera `sub,access_token` + 1.200), con 1.200 `sub` y 1.200 tokens distintos. `manifest.json` tiene `"count":1200` y 0 `eyJ`.
  - Un token del lote (línea 777) es aceptado por `ticketing-api` (`GET /api/v1/events` 200). No se imprimió.
- **`load-test`:**
  - Arranca a las 14:56:19.02, después de que el generador terminara (arrancó a las 14:56:06.96) y con el `api` sano.
  - Imprime "load-test: the QA load harness (load-tests/) has not been delivered yet; nothing was executed (PLAT-IV-010)." y termina con **exit 1**.
  - Corre con usuario `10001:10001`, raíz de solo lectura y `CapDrop=[ALL]`.
  - Montajes: solo `load-tokens` en `/tokens`, en solo lectura.
- **`--profile load up -d --wait`:** exit 1 (`container ticketing-platform-load-test-1 exited (1)`). Es el fallo esperado y visible.
- **Limpieza:** se eliminaron los contenedores `load-*` del perfil. El volumen `load-tokens` se eliminó con el `down -v` final.

### 4.8 Apagado y compatibilidad con `run-local.sh`

- **`docker compose --env-file .env --profile load down -v --remove-orphans`.** exit 0 en 7.761 ms. Detuvo y eliminó los 7 contenedores, la red y el volumen `load-tokens`. Recursos ajenos intactos (§2).
- **Comando de dependencias de `run-local.sh` desde limpio.** Se ejecutó literalmente: `docker compose -f "$ROOT/docker-compose.yml" --env-file "$ROOT/.env" up -d --wait dynamodb-local localstack infra-init local-idp payment-mock`.
  - Terminó con exit 0 en 5 s.
  - No se creó ningún contenedor `ticketing-*`: 0 en `ps -a`.
  - Los puertos del host 8080 a 8083 siguen libres.
  - `infra-init` terminó con `Exited (0)` unos 12 s después.
  - El mismo comando se ejecutó también con el entorno previo levantado, antes del `down`: exit 0 y sin `ticketing-*`.
- **Conclusión:** el cambio de Compose no rompe `run-local.sh`.
- **Observación previa a este incremento, no causada por él.** `up --wait` informa `Healthy` para `infra-init` mientras sigue en ejecución y vuelve antes de que termine. En un entorno recién creado, `run-local.sh start` puede arrancar las JVM locales antes de que existan la tabla y las colas. Sugerencia al humano, en su script: añadir `docker compose … wait infra-init` tras el `up`. Los servicios `ticketing-*` de Compose no tienen este problema, porque usan `service_completed_successfully`.

## 5. Security checks

- **Secretos.** La API key solo entra por `${PAYMENT_MOCK_API_KEY:?}` desde `.env`. No aparece en la configuración resuelta con `.env.example`, ni en los logs de `api`/`worker`, ni en la salida de newman. Tampoco se imprimieron tokens: el lote se verificó dentro de un contenedor y un único token se usó sin mostrarlo.
- **Imágenes.**
  - Ningún `:latest`. Externas con tag y `@sha256`.
  - Las propias no se buscan en ningún registro (`pull_policy: never` en `ticketing-*` y `load-test`), lo que elimina el riesgo de suplantación por nombre en Docker Hub (§3, PLAT-SPK-013).
- **Contenedores.**
  - Usuario no root (10001) en `api`, `worker` y `load-test`.
  - `cap_drop: ALL`, `no-new-privileges`, raíz de solo lectura y `/tmp` `noexec`.
  - Límite de memoria de 768 MiB en `api` y `worker`.
- **Puertos.** Solo `127.0.0.1`. El `worker` y `load-test` no publican nada. La gestión 8081 se publica solo en local (ADR-032).
- **Repositorio.** `git status` muestra solo ` M .env.example` y ` M docker-compose.yml`. Ningún archivo de `ticketing/`, `payment-mock/`, `README.md`, `postman/` ni `run-local.sh` cambiado.

## 6. Deviations

1. **`load-test` sin el montaje `./load-tests` (PLAT-IV-010 a lo incluía).**
   - Se declaró primero como bind de solo lectura con `bind.create_host_path: false`.
   - Docker Desktop (Windows) **creó igualmente** `CODE_REPO/load-tests/` vacío al hacer `--profile load up`. Es una ruta de QA en la que este agente no puede escribir.
   - El directorio se eliminó (`rmdir`, vacío y no versionado) y el montaje se retiró. Queda documentado en el comentario del servicio para que QA lo añada al entregar su arnés.
   - El comportamiento del servicio no cambia: falla con su mensaje provisional.
   - Decisión registrada como `PLAT-IV-016` (no bloqueante) en `human-review/platform.implementation-review.2.yaml`.
2. **`pull_policy: never` en `ticketing-api`, `ticketing-worker` y `load-test`.** No figuraba en el plan; se añadió por el resultado de PLAT-SPK-013. No cambia contratos ni la topología. Los servicios `payment-mock`, `infra-init`, `local-idp` y `load-token-generator` (PLAT-INC-002 a 004) **no se tocaron**: con la política por defecto intentan un `pull` del registro antes de construir cuando falta su imagen local. Se propone corregirlo en PLAT-INC-007 (§8).
3. **El build de la imagen desde Compose reutilizó la caché de capas.** No se forzó `--no-cache`. La capa de `mvnw verify` procede del build 5 de PLAT-INC-005 r2 sobre un contexto idéntico. El ID cambió solo por las etiquetas de Compose.
4. **`load-test` usa la imagen local de `infra-init` como portador provisional de `sh`.** No se descargó ni se creó ninguna imagen nueva; QA la sustituye.
5. **Datos del entorno de `run-local.sh`.** El arranque desde limpio exigió `down -v` del proyecto. El entorno que levantó el humano, con datos efímeros, se recreó al final con el mismo comando de `run-local.sh`; los datos previos no se conservan.

## 7. Blockers

Ninguno. `PLAT-IV-016` no bloquea.

## 8. Handoff to other agents

**QA / Resilience:**

- **Arranque completo:** `docker compose up -d --wait`. Arranca en unos 33 s con la imagen ya en caché e `infra-init` tarda unos 18 s. Estado esperado:
  - 6 servicios `healthy`;
  - `infra-init` `Exited (0)`;
  - `api` y `worker` con el mismo image ID.
- **URL:**
  - API en `http://localhost:${TICKETING_API_HOST_PORT:-8080}/api/v1`;
  - gestión en `:8081/actuator/{health,prometheus}`;
  - salud en `:8080/readyz` y `/livez`;
  - el `worker` solo es accesible desde la red (`ticketing-worker:8080/readyz` y `:8081/actuator/*`).
- **Perfil `load`:**
  - `docker compose --profile load up -d` genera 1.200 tokens en el volumen `load-tokens` (`/tokens/customer-tokens.csv` y `/tokens/manifest.json`).
  - `load-test` es el punto de integración: QA sustituye `image`, `entrypoint`/`command` y añade `./load-tests:/load-tests:ro` cuando exista la carpeta (PLAT-IV-016).
  - Las dependencias ya están declaradas: `ticketing-api` sano y `load-token-generator` completado.
- **Parada:**
  - `docker compose stop` respeta el apagado ordenado; se midió en unos 4,5 s por rol.
  - `docker compose down -v` elimina datos y lote, porque son efímeros.
  - Reiniciar `api`/`worker` conserva los datos; reiniciar un emulador no (PLAT-IV-011).
- **Exclusión con `run-local.sh`:** `docker compose up` (stack completo) y `run-local.sh start` (JVM locales) son excluyentes a la vez, por los puertos 8080/8081 y por consumir las mismas colas.

**Documentation:**

- **Variables nuevas** de `.env.example`: `TICKETING_API_HOST_PORT`, `TICKETING_API_MANAGEMENT_HOST_PORT` y `TICKETING_WORKER_ID` (opcional).
- **README.** Hoy dice que `docker-compose.yml` contiene solo las dependencias; ahora incluye `ticketing-api` y `ticketing-worker`.
- **Colección Postman.** Corregir la aserción `Order CONFIRMED` (§4.4).

**Humano:**

- **`run-local.sh`.** Considerar `docker compose … wait infra-init` tras el `up --wait` (§4.8). No es un defecto de este incremento.

**Platform, para PLAT-INC-007:**

- `pull_policy: never` en `payment-mock`, `infra-init`, `local-idp` y `load-token-generator`.
- Verificación desde un checkout limpio (`git archive`).
- Barrido de seguridad de §8.1 sobre el entorno levantado.
- Reinicio y pausa de emuladores según PLAT-IV-011.
- Respuesta a PLAT-IV-016.
