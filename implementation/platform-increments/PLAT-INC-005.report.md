---
artifact: platform-increment-report
increment: PLAT-INC-005
result: BLOCKED
code_revision: "CODE_REPO main 0e52ae2 (clean tree at start); ticketing/ consumed at 0e52ae2 (INC-010 committed); only change: this agent's untracked ticketing/Dockerfile and ticketing/.dockerignore; no commits by this agent"
verified_at: 2026-10-05
plan: implementation/platform.implementation-plan.v1.md
review: human-review/platform.implementation-plan-review.yaml (APPROVED)
blocking_review: human-review/platform.implementation-review.1.yaml
---

# PLAT-INC-005 — Imagen única de `ticketing` (roles `api` y `worker`)

## 1. Implemented

| Archivo (`CODE_REPO`) | Contenido |
|---|---|
| `ticketing/Dockerfile` | Multi-etapa. Build: JDK Temurin fijada por digest; copia `.mvn/`, `mvnw`, `pom.xml` y `pom.xml` + `src/` de `domain`, `application`, `infrastructure` y `bootstrap`; normaliza CR de `mvnw`; `sh ./mvnw -B -ntp verify` (sin `-DskipTests`, `PLAT-IV-006` a) con el repositorio Maven y la distribución del wrapper en un montaje de caché de BuildKit (`id=ticketing-m2`, independiente de `payment-mock-m2`; nunca en capas); compila la sonda de salud JDK `HealthProbe` (`-Xlint:all -Werror`). Runtime: JRE Temurin fijada por digest, usuario `ticketing` uid/gid 10001, `ticketing.jar` (artefacto del handoff INC-010 §6.5.1) y la sonda en `0444`, `EXPOSE 8080 8081`, `STOPSIGNAL SIGTERM`, `ENTRYPOINT` exec con `-XX:MaxRAMPercentage=75 -XX:+ExitOnOutOfMemoryError` (`PLAT-IV-012`). Sin rol por defecto: el rol se elige solo con `TICKETING_ROLE` |
| `ticketing/.dockerignore` | Lista de permitidos: `.mvn/`, `mvnw`, `pom.xml`, `*/pom.xml`, `*/src/` (fuera `target/`, IDE, logs, `.git`, `.gitattributes`, cualquier `.env`) |

No se modificó ningún otro archivo de `ticketing/` ni de `payment-mock/`, ni `docker-compose.yml`, `.env.example` o `.gitignore`. El Dockerfile no referencia `payment-mock`. La sonda `HealthProbe` se duplica textualmente en cada Dockerfile (ADR-034: sin código compartido entre proyectos).

**El incremento queda `BLOCKED`**: con el Dockerfile tal como se aprobó (`PLAT-IV-006` a), `sh ./mvnw -B verify` falla en Linux por tres pruebas unitarias del backend que dependen del sistema operativo (§3, §7). La imagen oficial `ticketing-platform/ticketing:local` **no se creó**.

## 2. Resources and services

Ningún servicio nuevo en Compose (eso es PLAT-INC-006). Imagen prevista: `ticketing-platform/ticketing:local` (no creada, ver §7).

Recursos temporales del spike, todos eliminados al terminar: imagen `ticketing-platform/ticketing:spk-002-diagnostic` y contenedores `plat-spk002-api`, `plat-spk002-worker` y `plat-spk014-api-exec`, con la etiqueta `plat-spike`. Quedan la caché de BuildKit `ticketing-m2` (repositorio Maven, sin secretos) y la caché de capas del builder.

## 3. Spikes

### PLAT-SPK-002 — Build real de `ticketing` en contenedor y arranque por rol — DIFFERENT (build) / CONFIRMED (runtime)

**Build con la política aprobada (Dockerfile entregado, sin cambios):** `FAILURE`.

| Ejecución (valores por defecto de Linux) | `domain` | `application` | `infrastructure` | `bootstrap` | Puerta de cobertura | Fallos |
|---|---|---|---|---|---|---|
| 1 (`docker build`, Dockerfile entregado; caché fría) | 99/0 | 277/0 | 369, **3 fallos**, 2 omitidas | no ejecutado | no ejecutada | 3 |
| 2 (diagnóstico, `-Dmaven.test.failure.ignore=true`) | 99/0 | 277/0 | 369, **2 fallos**, 2 omitidas | 24/0 | 1/0 (verde) | 2 |
| 3 (diagnóstico, ídem) | 99/0 | 277/0 | 369, **2 fallos**, 2 omitidas | 24/0 | verde | 2 |
| 4 (diagnóstico, ídem) | 99/0 | 277/0 | 369, **3 fallos**, 2 omitidas | 24/0 | verde | 3 |

Pruebas que fallan (código del backend, no modificado):

| Prueba | Frecuencia | Causa |
|---|---|---|
| `PaymentHttpClientSpikeTest.responseTimeout` (línea 83) | 4/4 | `assertThat(thread.get()).startsWith("reactor-http-nio")`. En Linux, Reactor Netty usa el transporte nativo `epoll` (`reactor-http-epoll-N`); en Windows solo existe NIO |
| `WebStackSpikeTest.inMemoryBodyLimit` (línea 287) | 4/4 | Misma aserción `startsWith("reactor-http-nio")` |
| `WebApiConfigurationTest.configurableClaimNames` | 2/4 (intermitente) | `BlockingOperationError: java.io.FileInputStream#readBytes` en `Schedulers.parallel()`: el ayudante de pruebas `TestTokenIssuer.sign` firma RSA (cegado RSA → `SecureRandom`); en Linux el `SecureRandom` por defecto es `NativePRNG`, que recarga su búfer leyendo `/dev/urandom` con `FileInputStream`. Depende de en qué hilo toca recargar. En Windows no hay lectura de archivo |

Las 2 omitidas son `PaymentContractCopyTest` y `TicketingContractCopyTest` `copyMatchesTheOriginal`: `Assumption` por ausencia del `SPEC_REPO` dentro del contexto de build (diseño del backend; la comparación con el hash registrado sí se ejecuta y pasa). Además aparece en el log un `ERROR` de `ContextPropagationSupport` (`BlockingOperationError` de `RandomAccessFile` durante la carga de un `ServiceLoader`) que no hace fallar ninguna prueba.

**Diagnóstico de alternativa (no aplicada al Dockerfile):** con las JVM de prueba en condiciones equivalentes a Windows (`JAVA_TOOL_OPTIONS="-Dreactor.netty.native=false -Djava.security.egd=file:/dev/./urandom"` solo en el `RUN` de build) la suite completa pasa **4/4** (3 diagnósticos + imagen de spike): 99 + 277 + 369 (2 omitidas) + 24 + puerta de cobertura, `BUILD SUCCESS`, 0 fallos. Ese es el único cambio frente al Dockerfile entregado (`diff`: una línea) y no llega al runtime (`Config.Env` de la imagen sin `JAVA_TOOL_OPTIONS`).

**Runtime (imagen de spike, etapa final idéntica a la del Dockerfile entregado):**

| Comprobación | Resultado |
|---|---|
| Imagen | `sha256:079d4eef…20a97a4` (spike, eliminada); 383.596.829 B (**384 MB**); `Config.User=10001:10001`; `ENTRYPOINT` exec; `STOPSIGNAL SIGTERM`; `ExposedPorts` 8080, 8081 |
| Contenido | `/app/ticketing.jar` (65.986.553 B, `0444`) y `/opt/probe/HealthProbe.class`; nada más copiado del contexto |
| Sin `TICKETING_ROLE` | exit 1; el error nombra `TICKETING_ROLE` ("api or worker") |
| `api` y `worker` desde la misma imagen | Ambos contenedores con `Image` = `sha256:079d4eef8296b6c3c88a51180325ae8aa1bdb30fec4030553f46cfd1d20a97a4` (**mismo image ID**); rol solo por `TICKETING_ROLE=api` / `worker` |
| Arranque | Ambos `healthy` 7 s después de `docker run`; `Started TicketingApplication in 4.431 s` (api) / `4.207 s` (worker); `Netty started on port 8080` y `8081` |
| Proceso | `uid=10001(ticketing)`; `/proc/1/status` Uid/Gid 10001; PID 1 = `java -XX:MaxRAMPercentage=75 -XX:+ExitOnOutOfMemoryError -jar /app/ticketing.jar` |
| Salud (desde la red del proyecto) | `api` y `worker`: `:8080/readyz` 200 `{"status":"UP"}`, `:8080/livez` 200, `:8081/actuator/health` 200; `:8080/actuator/health`, `:8080/api/v1/events` sin token y `:8080/nope` → 401 |
| Contra INC-002..004 ya levantados | `api` con token `customer-a` de `local-idp` → `GET /api/v1/events` 200 (JWKS por `http://local-idp:9000`); `worker`: `/actuator/prometheus` con `ticketing_circuit_state` `payment-mock` y `sqs-publication` en `CLOSED`, `ticketing_log_dropped_total 0` |
| Logs | 0 `WARN`, 0 `ERROR` en ambos roles; 0 apariciones de la API key, `Bearer` o `eyJ` |
| Memoria | `api` 232,7 MiB / 768 MiB; `worker` 240 MiB / 768 MiB |

### PLAT-SPK-014 — Apagado ordenado y endurecimiento — CONFIRMED (con hallazgo para PLAT-INC-006)

| Comprobación | Resultado |
|---|---|
| Endurecimiento | `--read-only`, `--tmpfs /tmp`, `--cap-drop ALL`, `no-new-privileges`, `--memory 768m`: ambos roles sanos; `touch /app/x` → "Read-only file system"; `/tmp` escribible |
| `docker stop -t 40` | `api` 4.606 ms, `worker` 4.483 ms; código 143 (SIGTERM a la JVM en PID 1), `OOMKilled=false`; "Commencing graceful shutdown. Waiting for active requests to complete" y "Graceful shutdown complete" en ambos. Dentro de 35 s + margen |
| Transporte de red según `/tmp` | **Hallazgo**: `--tmpfs /tmp` monta `noexec` por defecto; Netty extrae `libnetty_transport_native_epoll*.so` en `/tmp`, no puede mapearla como ejecutable (`/proc/1/maps`: solo `---p`/`r--p`) y recae en NIO sin avisar (hilos `reactor-http-nio`, 0 WARN). Con `--tmpfs /tmp:exec` se carga `epoll` (`r-xp`; hilos `reactor-http-epoll`). Es decir, con el endurecimiento aprobado (plan §4.2: `tmpfs: /tmp`) el runtime usa el **mismo transporte (NIO) que las pruebas del backend en Windows** |

## 4. Verification

| Comando (en `D:\Nequi\ticketing-platform`) | Resultado |
|---|---|
| `docker build --check ticketing` | "Check complete, no warnings found." |
| `docker build --progress plain -t ticketing-platform/ticketing:local ticketing` (caché de Maven fría) | **exit 1 tras 96 s**: `Tests run: 369, Failures: 3, Errors: 0, Skipped: 2` en `ticketing-infrastructure`; contexto transferido 2,16 MB |
| `docker build --output type=cacheonly -f <scratchpad>/Dockerfile.diag1 ticketing` ×3 (`-Dmaven.test.failure.ignore=true`) | 80 s, 106 s, 107 s; 2, 2 y 3 fallos (§3) |
| `docker build --no-cache --output type=cacheonly -f <scratchpad>/Dockerfile.diag2 ticketing` ×3 (JVM de prueba con NIO/DRBG) | 109 s, 109 s, 106 s; 0 fallos |
| `docker build -t ticketing-platform/ticketing:spk-002-diagnostic -f <scratchpad>/Dockerfile.spk002 ticketing` (Dockerfile entregado + `JAVA_TOOL_OPTIONS` en el `RUN`) | `BUILD SUCCESS` en 77 s (caché de Maven y bases calientes) |
| `docker run` sin rol / `api` / `worker`, comprobaciones de §3, `docker stop -t 40` | Ver §3 |
| `docker compose --env-file .env.example config --quiet` | exit 0 (Compose no cambia en este incremento) |
| `docker compose --env-file .env run --rm --no-deps infra-init` (antes de los spikes) | exit 0, "all resources match the approved definitions" |

Tiempos de build: con caché fría el build entregado falló a los 96 s, ya con Maven 3.9.16 y las dependencias descargados. Un build completo en verde con la caché caliente tarda entre 77 y 109 s, `verify` y la imagen incluidos (`--no-cache` de capas: 106 a 109 s).

## 5. Security checks

- Etapa final con `USER 10001:10001`; proceso con uid 10001 en ambos roles.
- `docker image inspect` `Config.Env`: solo `PATH`, `JAVA_HOME`, `LANG*`, `JAVA_VERSION`. Sin `ARG` ni `ENV` en el Dockerfile. `docker history --no-trunc`: 0 coincidencias de `api_key`, `secret`, `password` o `token`.
- Secretos solo en tiempo de ejecución, pasados por nombre (`-e VAR`) desde `.env`; 0 apariciones de la API key, `Bearer` o `eyJ` en los logs de ambos roles.
- Contexto de build por lista de permitidos (2,16 MB); la imagen solo contiene el jar y la sonda.
- Los dos `FROM` llevan tag y `@sha256` (los de PLAT-SPK-001). No hay `:latest`.
- Barrido de `ticketing/Dockerfile` y `ticketing/.dockerignore`: sin JWT, claves privadas ni claves AWS.
- Ningún puerto publicado al host en los spikes.

## 6. Deviations

1. Se añade `-ntp` a `-B`, igual que en PLAT-INC-004; no cambia qué se compila ni qué pruebas corren.
2. La sonda de salud se definirá en Compose (PLAT-INC-006) y no como `HEALTHCHECK` de la imagen, igual que en `payment-mock` y `local-idp`.
3. Los spikes usaron `docker run` con nombres `plat-spk*` sobre la red del proyecto porque los servicios de Compose `ticketing-api` y `ticketing-worker` pertenecen a PLAT-INC-006. Todos se eliminaron.

## 7. Blockers

**PLAT-IV-014 (bloqueante para PLAT-INC-005)** en `human-review/platform.implementation-review.1.yaml`: la política aprobada `PLAT-IV-006` a (`sh ./mvnw -B verify` dentro de la imagen) no puede pasar en Linux sin cambiar tres pruebas del backend o las condiciones de las JVM de prueba. Ninguna de las dos cosas puede decidirla este agente: §9.10, "una verificación requiere modificar aplicación" o "imagen sin alternativa aprobada".

Al reanudar, el resultado se registrará en un informe nuevo, sin sobrescribir este, con el nombre que indique el humano en la revisión.

**PLAT-IV-015 (bloqueante solo para PLAT-INC-006)**: coexistencia de `run-local.sh` (añadido por el humano) con los servicios de Compose `ticketing-api` y `ticketing-worker`; ver §8.

## 8. Handoff to other agents

**Backend Developer** (si el humano elige PLAT-IV-014 a):

- Hacer portables las aserciones `startsWith("reactor-http-nio")` de `PaymentHttpClientSpikeTest:83` y `WebStackSpikeTest:287`: aceptar el prefijo `reactor-http-` o fijar el transporte en la propia prueba.
- `WebApiConfigurationTest.configurableClaimNames`: firmar el token del `TestTokenIssuer` fuera del hilo que protege BlockHound, o evitar `NativePRNG` en el ayudante de pruebas.
- Reproducir: `docker build ticketing` desde `CODE_REPO`, o ejecutar `sh ./mvnw -B verify` en cualquier Linux x86_64 con JDK 25.
- Se omiten 2 pruebas por ausencia del `SPEC_REPO` en el contexto de build; es esperado.
- El runtime no requiere cambios: ambos roles arrancan, quedan sanos y se apagan de forma ordenada en contenedor.

**Platform, para PLAT-INC-006:**

- Mantener `tmpfs: /tmp` sin `exec`: el runtime queda en NIO, que es lo que validó el backend.
- Healthcheck con la sonda: `java -XX:TieredStopAtLevel=1 -XX:+UseSerialGC -Xmx16m -cp /opt/probe HealthProbe http://localhost:8080/readyz`, con `start_period` ≥ 30 s (arranque medido de unos 5 s).
- `stop_grace_period: 40s` y `mem_limit: 768m`.
- Variables literales del handoff INC-010 §6.5.3.

**Humano (`run-local.sh`):**

- Tras PLAT-INC-006, `docker compose up -d --wait`, que es lo que ejecuta `run-local.sh start`, levantará también `ticketing-api` (`127.0.0.1:8080/8081`) y `ticketing-worker`.
- Las JVM locales de `run-local.sh` usan 8080/8081 y 8082/8083: el `api` local no podría enlazar 8080/8081.
- Un segundo `worker` consumiría las mismas colas.
- `.gitignore` (`.run/`) y `.gitattributes` (`*.sh text eol=lf`) de la raíz no entran en conflicto con Platform.
