---
artifact: platform-increment-report
increment: PLAT-INC-005
attempt: r2
result: DONE
code_revision: "CODE_REPO main 8b7d655 (8b7d6556681cff0d673cb51eded49e1c538b42d2); ticketing/ consumed at 8b7d655; only uncommitted changes: this agent's untracked ticketing/Dockerfile (sha256 040c733d…f452696) and ticketing/.dockerignore (sha256 1948d9cb…c489cd), unchanged since the BLOCKED attempt; no commits by this agent"
verified_at: 2026-10-05
plan: implementation/platform.implementation-plan.v1.md
review: human-review/platform.implementation-plan-review.yaml (APPROVED)
resumes: implementation/platform-increments/PLAT-INC-005.report.md (BLOCKED, not modified)
unblocked_by: human-review/platform.implementation-review.1.yaml (APPROVED; PLAT-IV-014 CONFIRMED (a), PLAT-IV-015 CONFIRMED (a))
---

# PLAT-INC-005 (r2) — Imagen única de `ticketing` (roles `api` y `worker`)

Este informe registra la reanudación de PLAT-INC-005 tras `PLAT-IV-014` (a). El informe anterior (`PLAT-INC-005.report.md`, `BLOCKED`) se conserva sin cambios. Lo que allí consta como diseño de la imagen (§1) sigue vigente; aquí solo se registra lo que se ha verificado de nuevo.

## 1. Implemented

| Archivo (`CODE_REPO`) | Estado en r2 |
|---|---|
| `ticketing/Dockerfile` | **Sin cambios** respecto del entregado en la ejecución `BLOCKED` (mtime 2026-10-05 08:38:49, anterior a `8b7d655`). Mantiene `PLAT-IV-006` (a): `sh ./mvnw -B -ntp verify`, sin `JAVA_TOOL_OPTIONS`, sin `-DskipTests`, sin `ARG` ni `ENV` |
| `ticketing/.dockerignore` | **Sin cambios** (lista de permitidos `.mvn/`, `mvnw`, `pom.xml`, `*/pom.xml`, `*/src/`) |

El desbloqueo proviene solo del commit `8b7d655` del humano, que cambia únicamente código de prueba en `ticketing/`:

- `WebStackSpikeTest` y `PaymentHttpClientSpikeTest` aceptan cualquier event loop `reactor-http-`.
- `WebApiConfigurationTest` firma los tokens antes del pipeline reactivo.

El mismo commit modifica también `README.md` y `run-local.sh`. Este agente no tocó ninguno de los dos.

Imagen oficial creada: **`ticketing-platform/ticketing:local`** = `sha256:2d698ffb6b2e11963aaa532310c80a9128c6c775efc5b0abec1f7ae0f84d3ded`.

## 2. Resources and services

| Recurso | Estado |
|---|---|
| Imagen `ticketing-platform/ticketing:local` | `sha256:2d698ffb…3ded`, 383.596.829 B (**384 MB**), creada 2026-10-05T14:37:17Z (build 5 de §4) |
| Imágenes intermedias de los builds 1 a 4 de esta reanudación (`ae39733c`, `61eb8b62`, `285b19f0`, `e685ba5b`) | Eliminadas por ID tras comprobar que ningún contenedor las usaba |
| Imagen `da7974ac82f5` (build del humano sobre `8b7d655`, sin etiqueta tras la reconstrucción) | **Se conserva**: no la creó este agente |
| Contenedores de verificación `plat-r2-norole`, `plat-r2-norole2`, `plat-r2-api` y `plat-r2-worker`, y el cliente efímero `--rm` (todos con la etiqueta `plat-spike=PLAT-INC-005-r2`) | Todos eliminados; quedan 0 contenedores con la etiqueta `plat-spike` |
| Caché de BuildKit `ticketing-m2` (repositorio Maven, sin secretos) y caché de capas del builder | Se conservan |
| Proyecto Compose `ticketing-platform` | Sin cambios: `dynamodb-local`, `localstack`, `local-idp` y `payment-mock` siguen `healthy`. Se ejecutó una vez `infra-init` (`run --rm --no-deps`, solo validación). No se tocaron el contenedor ajeno `dynamodb-local` ni la red `dynamodb_default` |

Ningún servicio nuevo en Compose: `ticketing-api` y `ticketing-worker` se añaden en PLAT-INC-006.

## 3. Spikes

### PLAT-SPK-002 — Build real de `ticketing` en contenedor y arranque por rol — CONFIRMED

El build con la política aprobada pasa en Linux de forma estable (§4): **5 de 5 builds completos con 0 fallos**, frente a 0 de 4 en el intento `BLOCKED`.

- **`WebApiConfigurationTest`**: antes fallaba de forma intermitente (2 de 4). Ahora ejecuta sus 5 pruebas sin fallos en las 5 ejecuciones.
- **Las dos pruebas de hilo**: `PaymentHttpClientSpikeTest` (6 pruebas) y `WebStackSpikeTest` (7) pasan 5 de 5.
- **Transporte**: las JVM de prueba usan ahora el transporte nativo por defecto de Linux; el log muestra hilos `reactor-http-epoll-N`. El comportamiento por defecto de Linux queda probado sin condiciones especiales.

El runtime sobre la imagen oficial se confirma en la tabla de §4.2.

### PLAT-SPK-014 — Apagado ordenado y endurecimiento — CONFIRMED

Ambos roles, con `--read-only --tmpfs /tmp --cap-drop ALL --security-opt no-new-privileges --memory 768m`:

- Arrancan y quedan sanos.
- Se apagan de forma ordenada con `SIGTERM` en unos 4,5 s, muy por debajo de 35 s.
- Con `tmpfs /tmp` (`noexec` por defecto) el runtime usa NIO: hilos `reactor-http-n…` 12 (`api`) y 10 (`worker`), 0 `reactor-http-e…`.

Tras `8b7d655` esto ya no condiciona la validez de las pruebas, porque ambos transportes están cubiertos (epoll en el build Linux, NIO en las ejecuciones del backend en Windows). Se mantiene el endurecimiento aprobado sin cambios.

## 4. Verification

### 4.1 Builds (en `D:\Nequi\ticketing-platform`)

Comando, idéntico en las cinco ejecuciones: `docker build --no-cache --progress plain -t ticketing-platform/ticketing:local ticketing`.

- `--no-cache` invalida solo las capas, de modo que `mvnw verify` se ejecuta realmente cada vez.
- La caché de Maven (`ticketing-m2`) se mantiene caliente.
- No se usa `--pull`.

| Build | Exit | Duración (reloj) | Maven `Total time` | `domain` | `application` | `infrastructure` | `bootstrap` | Puerta de cobertura (`report-aggregate` + check) | Resultado |
|---|---|---|---|---|---|---|---|---|---|
| 1 | 0 | 110 s | 01:41 min | 99 / 0 F / 0 S | 277 / 0 / 0 | 369 / 0 / **2 S** | 24 / 0 / 0 | 1 / 0 (verde) | `BUILD SUCCESS` |
| 2 | 0 | 110 s | 01:40 min | 99 / 0 / 0 | 277 / 0 / 0 | 369 / 0 / 2 S | 24 / 0 / 0 | 1 / 0 | `BUILD SUCCESS` |
| 3 | 0 | 115 s | 01:44 min | 99 / 0 / 0 | 277 / 0 / 0 | 369 / 0 / 2 S | 24 / 0 / 0 | 1 / 0 | `BUILD SUCCESS` |
| 4 | 0 | 114 s | 01:42 min | 99 / 0 / 0 | 277 / 0 / 0 | 369 / 0 / 2 S | 24 / 0 / 0 | 1 / 0 | `BUILD SUCCESS` |
| 5 (imagen final) | 0 | 114 s | 01:43 min | 99 / 0 / 0 | 277 / 0 / 0 | 369 / 0 / 2 S | 24 / 0 / 0 | 1 / 0 | `BUILD SUCCESS` |

Totales por build: 770 pruebas, 0 fallos, 0 errores y 2 omitidas.

**Las 2 omitidas** son `PaymentContractCopyTest` y `TicketingContractCopyTest` `copyMatchesTheOriginal`: una `Assumption` por ausencia del `SPEC_REPO` en el contexto de build. Es esperado y ya constaba en el informe anterior.

**Aviso de BlockHound no fallido.** En cada build aparece 1 `ERROR` de `reactor.core.publisher.ContextPropagationSupport`: `BlockingOperationError` de `RandomAccessFile#readBytes` al cargar un `ServiceLoader`. Ya figuraba en el informe anterior, no hace fallar ninguna prueba y no es el defecto de `PLAT-IV-014`.

Antes del build 1 el humano ya había construido la imagen una vez sobre `8b7d655`, también en verde. Con las del humano son 6 de 6 builds.

### 4.2 Ejecución sobre la imagen oficial `ticketing-platform/ticketing:local` (`sha256:2d698ffb…3ded`)

Montaje de la verificación:

- **Contenedores**: `docker run` en la red `ticketing-platform_ticketing-net`, contra los servicios de INC-002..004 ya levantados.
- **Endurecimiento**: el de §3.
- **Variables**: las literales del handoff INC-010 §6.5.3, pasadas por nombre (`-e VAR`) desde `.env`.
- **Cliente HTTP**: un contenedor efímero `--rm` de `ticketing-platform/infra-init:local` (con `curl`) en la misma red.
- **Puertos**: ninguno publicado al host.

| Comprobación | Resultado |
|---|---|
| Sin `TICKETING_ROLE` (resto de variables de ambos roles presentes; y de nuevo solo con `AWS_REGION`) | exit **1** las dos veces; mensaje: "Missing required configuration ticketing.role (environment variable TICKETING_ROLE): api or worker"; 0 apariciones de la API key en el log |
| Mismo image ID en ambos roles | `plat-r2-api` y `plat-r2-worker`: `Image` = `sha256:2d698ffb6b2e11963aaa532310c80a9128c6c775efc5b0abec1f7ae0f84d3ded` (= `ticketing-platform/ticketing:local`); el rol solo difiere en `TICKETING_ROLE=api` / `worker` |
| uid | `id`: `uid=10001(ticketing) gid=10001(ticketing) groups=10001(ticketing)` en ambos; `/proc/1/status` Uid/Gid `10001` ×4; PID 1 = `java -XX:MaxRAMPercentage=75 -XX:+ExitOnOutOfMemoryError -jar /app/ticketing.jar`; imagen `Config.User=10001:10001` |
| Arranque y salud | Sonda de Compose prevista (`java … -cp /opt/probe HealthProbe http://localhost:8080/readyz` con `docker exec`) en verde a los 6,2 s (`api`) y 7,1 s (`worker`) tras `docker run`. "Started TicketingApplication in 4.084 s" (`api`) / "3.774 s" (`worker`). Netty en 8080 y 8081 |
| Rutas de salud desde la red | `api` y `worker`: `:8080/readyz` 200 (`{"status":"UP"}`), `:8080/livez` 200, `:8081/actuator/health` 200; `:8080/actuator/health` 401 y `:8080/nope` 401 |
| Token aceptado | `api` `GET /api/v1/events`: sin token **401**; token `customer-a` de `local-idp` (`POST http://local-idp:9000/token`, JWKS `http://local-idp:9000/.well-known/jwks.json`) **200**; token `admin` **200**; mismo token con la firma alterada **401**. `worker` `GET /api/v1/events` con token válido 401 (el `worker` no expone la API). Los tokens no se imprimieron |
| Dependencias del `worker` | `/actuator/prometheus`: `ticketing_circuit_state` `payment-mock` y `sqs-publication` en `CLOSED` = 1 |
| Logs | 12 líneas ECS por rol; 0 `"level":"WARN"`, 0 `"level":"ERROR"`, 0 apariciones de la API key, `Bearer` o `eyJ`, también tras el apagado |
| Memoria | `api` 252,8 MiB / 768 MiB; `worker` 235,4 MiB / 768 MiB |
| Endurecimiento | `ReadonlyRootfs=true`, `CapDrop=[ALL]`, `no-new-privileges` (`NoNewPrivs: 1`), `CapPrm/CapEff/CapBnd = 0`, `Memory=805306368`; `touch /app/x` → "Read-only file system"; `/tmp` escribible; `PortBindings={}` |
| Apagado ordenado (`docker stop -t 40`) | `api` **4.517 ms**, `worker` **4.453 ms**; exit 143 (SIGTERM a la JVM en PID 1), `OOMKilled=false`; "Commencing graceful shutdown. Waiting for active requests to complete" y "Graceful shutdown complete" en ambos |
| Contenido de la imagen | `/app/ticketing.jar` (65.986.553 B, `0444`) y `/opt/probe/HealthProbe.class` (2.539 B, `0444`); nada más copiado del contexto |

### 4.3 Comprobaciones estáticas y de entorno

| Comando | Resultado |
|---|---|
| `docker build --check ticketing` | "Check complete, no warnings found." |
| `docker compose --env-file .env.example config --quiet` | exit 0 (Compose no cambia en este incremento) |
| `docker compose --env-file .env run --rm --no-deps infra-init` (antes de los contenedores de verificación) | exit 0, "done: all resources match the approved definitions" |
| `git status --short` (CODE_REPO) | solo `?? ticketing/.dockerignore` y `?? ticketing/Dockerfile`; `HEAD` = `8b7d6556681cff0d673cb51eded49e1c538b42d2` |

Se usó el `.env` local (no versionado, `PAYMENT_MOCK_HOST_PORT=18090`) sin modificarlo y sin imprimir la API key.

## 5. Security checks

- **Usuario**: `USER 10001:10001` en la etapa final; el proceso corre con uid/gid 10001 en ambos roles.
- **Metadatos de la imagen**: `Config.Env` solo contiene `PATH`, `JAVA_HOME`, `LANG`, `LANGUAGE`, `LC_ALL` y `JAVA_VERSION`; sin `JAVA_TOOL_OPTIONS`.
- **Historial**: `docker history --no-trunc` da 0 coincidencias de `api_key`, `secret`, `password` o `token`.
- **Barrido de los dos archivos** (`ticketing/Dockerfile` y `ticketing/.dockerignore`): 0 coincidencias de `eyJ`, claves privadas, claves AWS, `api_key`, `password`, `JAVA_TOOL_OPTIONS`, `skipTests`, `ARG` o `ENV`.
- **Imágenes base**: ambos `FROM` llevan tag inmutable y `@sha256` (`eclipse-temurin:25.0.4.1_1-jdk-noble@sha256:589ff4cc…` y `…-jre-noble@sha256:d9a39a23…`, de PLAT-SPK-001); 0 `:latest`.
- **Secretos en ejecución**: solo se pasan por nombre desde `.env`; 0 apariciones de la API key, `Bearer` o `eyJ` en los logs.
- **Puertos y privilegios**: ningún puerto publicado al host durante la verificación; capacidades eliminadas por completo y raíz de solo lectura sin errores.

## 6. Deviations

1. **Cinco builds en lugar de tres.** Se pidieron al menos 3; se hicieron 5 para dar más peso a la evidencia sobre la antigua intermitencia (2 de 4). Con 5 de 5 verdes, la probabilidad de no ver fallos si la tasa siguiera en el 50 % es del 3 %.
2. **`--no-cache` en cada build**, para que la capa de `verify` no se reutilizara de la caché. No cambia el Dockerfile.
3. **Verificación con `docker run`.** Igual que en el intento anterior, se usaron contenedores `plat-r2-*` y un cliente efímero de la imagen `infra-init` en la red del proyecto, porque los servicios de Compose `ticketing-api`/`ticketing-worker` corresponden a PLAT-INC-006. Todos se eliminaron.
4. **Primera pasada del script de verificación.** Sus contadores de `WARN`/`ERROR` y de hilos estaban mal planteados para logs ECS (JSON) y para el truncado de `/proc/*/comm` a 15 caracteres. Se corrigieron los patrones y se repitió la verificación completa; las cifras de §4.2 corresponden a la segunda pasada. Las comprobaciones de salud, token, uid, image ID y apagado dieron el mismo resultado en ambas pasadas.
5. **Siguen vigentes las desviaciones 1 y 2 del informe anterior**: `-ntp` en `mvnw`, y la sonda de salud definida en Compose (PLAT-INC-006) en lugar de un `HEALTHCHECK` de imagen.

## 7. Blockers

Ninguno. `PLAT-IV-014` y `PLAT-IV-015` están resueltos (`CONFIRMED` (a)). No surgieron problemas que requieran decisión humana, así que no se creó `human-review/platform.implementation-review.2.yaml`.

## 8. Handoff to other agents

**Backend Developer:**

- **Build en Linux**: con `8b7d655`, el build de la imagen (`sh ./mvnw -B verify` en Linux, transporte epoll por defecto) pasa 5 de 5. No hay acciones pendientes.
- **Aviso de `ContextPropagationSupport`**: sigue apareciendo una vez por build y no falla. Si se quiere silenciar, es una mejora opcional de las pruebas, no un bloqueo.

**Platform, para PLAT-INC-006** (servicios `ticketing-api` y `ticketing-worker` en `docker-compose.yml`):

- **Imagen**: `image: ticketing-platform/ticketing:local`, con `build: ./ticketing` en un solo servicio, o en ambos con la misma etiqueta, para garantizar un único image ID.
- **Rol**: solo `TICKETING_ROLE=api` / `worker`.
- **Variables**: las literales de INC-010 §6.5.3, ya validadas aquí (`TICKETING_DYNAMODB_*`, `TICKETING_SQS_*`, `TICKETING_SECURITY_*` solo en `api`, `TICKETING_PAYMENT_*` y `TICKETING_WORKER_ID` solo en `worker`, `AWS_*` ficticias).
- **Healthcheck**: `java -XX:TieredStopAtLevel=1 -XX:+UseSerialGC -Xmx16m -cp /opt/probe HealthProbe http://localhost:8080/readyz`, con `start_period` ≥ 30 s. El arranque medido está entre 6 y 7 s hasta la sonda en verde.
- **Recursos y apagado**: `stop_grace_period: 40s`, `mem_limit: 768m`, `read_only: true`, `tmpfs: /tmp` (sin `exec`), `cap_drop: [ALL]`, `security_opt: [no-new-privileges:true]`.
- **Puertos**: el `api` publica `127.0.0.1:8080` y `127.0.0.1:8081`; el `worker` no publica nada.
- **Dependencias**: `api` → `infra-init` `service_completed_successfully` + `local-idp` `service_healthy`; `worker` → `infra-init` completado + `payment-mock` `service_healthy` (ADR-036).
- **`run-local.sh`** (`PLAT-IV-015` a): ya levanta solo `dynamodb-local localstack infra-init local-idp payment-mock`, así que los servicios de aplicación pueden ir en la topología por defecto sin perfil. Siguen siendo excluyentes a la vez: las JVM locales usan los puertos del host 8080 a 8083 y consumirían las mismas colas. Es un punto para Documentation.

**QA / Documentation:**

- **Imagen**: `ticketing-platform/ticketing:local` (384 MB), construida con `docker build -t ticketing-platform/ticketing:local ticketing` desde `CODE_REPO`. Un build completo con la caché de Maven caliente tarda unos 110 a 115 s, incluidas 770 pruebas.
