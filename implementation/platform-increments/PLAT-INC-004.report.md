---
artifact: platform-increment-report
increment: PLAT-INC-004
result: DONE
code_revision: "CODE_REPO main 31b39ac; payment-mock/ consumed at 31b39ac with a clean tree (git status: only this agent's untracked payment-mock/Dockerfile and payment-mock/.dockerignore); ticketing/ has uncommitted changes of the Backend Developer (INC-010), untouched; no commits by this agent"
verified_at: 2026-10-04
plan: implementation/platform.implementation-plan.v1.md
review: human-review/platform.implementation-plan-review.yaml (APPROVED)
---

# PLAT-INC-004 — Imagen independiente de `payment-mock`

## 1. Implemented

| Archivo (`CODE_REPO`) | Contenido |
|---|---|
| `payment-mock/Dockerfile` | Multi-etapa. Build: JDK fijada, copia `.mvn/`, `mvnw`, `pom.xml`, `src/`; normaliza CR de `mvnw`; `sh ./mvnw -B -ntp verify` (suite completa, sin `-DskipTests`, `PLAT-IV-006` a) con el repositorio Maven y la distribución del wrapper en un montaje de caché de BuildKit (`id=payment-mock-m2`, nunca en capas); compila una sonda de salud JDK (`HealthProbe`, heredoc, `-Xlint:all -Werror`). Runtime: JRE fijada, usuario `paymentmock` uid/gid 10001, `payment-mock.jar` y la sonda en `0444`, `EXPOSE 8090`, `ENTRYPOINT` exec con `-XX:MaxRAMPercentage=75 -XX:+ExitOnOutOfMemoryError` |
| `payment-mock/.dockerignore` | Lista de permitidos: solo `.mvn/`, `mvnw`, `pom.xml`, `src/` (fuera `target/`, IDE, logs, `.git`, cualquier `.env`) |
| `docker-compose.yml` | Servicio `payment-mock` |

No se modificó ningún otro archivo de `payment-mock/` (POM, wrapper, código, recursos, pruebas). El Dockerfile no referencia `ticketing` (solo el comentario que declara la independencia).

## 2. Resources and services

| Servicio | Imagen | Configuración | Host | Salud |
|---|---|---|---|---|
| `payment-mock` | `ticketing-platform/payment-mock:local` (353 MB) | `PAYMENT_MOCK_API_KEY: ${PAYMENT_MOCK_API_KEY:?…}` (solo desde `.env`); `SERVER_PORT` no se fija (8090); `read_only`, `tmpfs /tmp`, `cap_drop: ALL`, `no-new-privileges`, `mem_limit: 384m` (`PLAT-IV-012`) | `127.0.0.1:${PAYMENT_MOCK_HOST_PORT:-8090}:8090` (este equipo: 18090, `PLAT-IV-008`) | Sonda JDK `GET http://localhost:8090/health` → 200; intervalo 10 s, `start_period` 30 s, `start_interval` 1 s |

Sin dependencias (`depends_on`) según ADR-036.

## 3. Spikes

| Spike | Resultado | Evidencia |
|---|---|---|
| PLAT-SPK-003 Build real del mock en contenedor | CONFIRMED | `sh ./mvnw -B -ntp verify` en la JDK Temurin 25.0.4.1: `Tests run: 256, Failures: 0, Errors: 0, Skipped: 0`, `BUILD SUCCESS` (incluidos `BlockHoundCanaryTest`, `ArchitectureTest`, `RaceConditionTest`, `LogSecrecyTest`; los flags de BlockHound/Mockito del `argLine` funcionan en Linux). `mvnw` está en el índice con modo `100644`: se invoca con `sh`. El wrapper usa el `maven-wrapper.jar` versionado (sin `curl`/`wget` en la JDK) y descarga Maven 3.9.16 de Maven Central en la primera construcción. Build completo en 62 s (caché fría de Maven) |
| PLAT-SPK-008 (mock) | CONFIRMED | `healthy` en 4,7 s tras `up`; la sonda devuelve exit 1 ante respuesta ≠ 200 (`/nope` → 401) y ante puerto cerrado (`ConnectException`) |
| PLAT-SPK-014 (mock) | CONFIRMED | Raíz de solo lectura (`touch /app/x` → "Read-only file system"), `/tmp` escribible, `cap_drop: ALL`; `docker compose stop` en 2,6 s con "Commencing graceful shutdown … Graceful shutdown complete", código 143 (SIGTERM a la JVM en PID 1) |

## 4. Verification

| Comando (en `D:\Nequi\ticketing-platform`) | Resultado |
|---|---|
| `docker build --check payment-mock` (también `platform/infra-init`, `platform/local-idp`) | "Check complete, no warnings found." |
| `docker compose --env-file <tmp> build --progress plain payment-mock` | exit 0; 256 pruebas, 0 fallos; contexto transferido 381 KB |
| `docker compose --env-file .env.example config --quiet` | exit 0 |
| `docker compose --env-file <tmp> up -d --wait payment-mock` | `Healthy` en 4,7 s; Spring `Started PaymentMockApplication in 2.094 seconds` |
| `curl http://127.0.0.1:18090/health` | 200 |
| `curl http://127.0.0.1:18090/control/rules` sin / con `X-Api-Key` | 401 / 200 |
| `docker compose run --rm --no-deps local-idp probe http://payment-mock:8090/health` (desde la red) | `200` |
| `docker compose exec payment-mock id` | `uid=10001(paymentmock)` |
| `docker compose --env-file <tmp sin clave> config` | exit 1: `required variable PAYMENT_MOCK_API_KEY is missing a value: set PAYMENT_MOCK_API_KEY in .env (see .env.example)` |
| `docker run --rm -e PAYMENT_MOCK_API_KEY= ticketing-platform/payment-mock:local` / sin la variable | exit 1, `APPLICATION FAILED TO START`, "Failed to bind properties under 'payment-mock'…", sin imprimir valores |
| Proyecto completo desde cero: `docker compose --env-file <tmp> --profile load down -v --remove-orphans`, luego `up -d --wait` | 4 servicios persistentes `healthy` en 4,8 s; `infra-init` termina después con código 0; `resources.sh` → `RESULT 0 failed check(s)` |

## 5. Security checks

- `Config.User=10001:10001`; proceso como `uid=10001`.
- `docker image inspect` `Config.Env`: solo `PATH`, `JAVA_HOME`, `LANG*`, `JAVA_VERSION`; ninguna clave. `docker history --no-trunc`: 0 coincidencias de `API_KEY` o `secret`. Sin `ARG`.
- Logs del contenedor: 0 apariciones de la API key usada.
- Imagen construida solo con el contexto `payment-mock/` (lista de permitidos); sin `.git`, `target/` ni archivos de entorno.
- Puerto publicado solo en `127.0.0.1`.
- Barrido estático de todos los archivos de Platform: sin `:latest`, toda imagen externa y todo `FROM` con `@sha256`, sin JWT, claves privadas, claves AWS reales ni `LOCALSTACK_AUTH_TOKEN`.

## 6. Deviations

1. Se pasa `-ntp` (sin progreso de transferencias) además de `-B`; no cambia qué se compila ni qué pruebas corren.
2. La sonda de salud se define en Compose (no como `HEALTHCHECK` de la imagen), igual que en `local-idp`.

## 7. Blockers

Ninguno.

## 8. Handoff to other agents

QA y Documentation:

| Elemento | Valor |
|---|---|
| Imagen | `ticketing-platform/payment-mock:local`, construida con `docker compose build payment-mock` (ejecuta las 256 pruebas) |
| Puerto | 8090 en la red (`http://payment-mock:8090`, servers del contrato); host `127.0.0.1:${PAYMENT_MOCK_HOST_PORT:-8090}` (18090 en este equipo) |
| Salud | `GET /health` → 200 sin API key |
| API key | `PAYMENT_MOCK_API_KEY` obligatoria en `.env` (no versionado); el `ticketing-worker` debe recibir el mismo valor (PLAT-INC-006) |
| Estado | En memoria: se pierde al reiniciar el contenedor (handoff PM-INC-006) |
| Nota de orquestación | `docker compose up -d --wait` no espera a que termine `infra-init` si ningún servicio persistente depende de él (hasta PLAT-INC-006); usar `docker wait ticketing-platform-infra-init-1` o `docker compose up infra-init` |
