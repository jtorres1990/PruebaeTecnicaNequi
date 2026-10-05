---
artifact: platform-increment-report
increment: PLAT-INC-001
result: DONE
code_revision: "CODE_REPO main 31b39ac + untracked platform files of this increment (.gitignore, .env.example, docker-compose.yml, platform/.gitattributes); ticketing/ has uncommitted changes of the Backend Developer (INC-010), untouched; no commits by this agent"
verified_at: 2026-10-04
plan: implementation/platform.implementation-plan.v1.md
review: human-review/platform.implementation-plan-review.yaml (APPROVED)
---

# PLAT-INC-001 — Contrato de configuración, estructura y spikes de imágenes y herramientas

## 1. Implemented

| Archivo (`CODE_REPO`) | Contenido |
|---|---|
| `.gitignore` (raíz, nuevo) | Solo `.env`, `.env.*` y la excepción `!.env.example` (adición mínima; no existía) |
| `.env.example` | Variables de Platform de `PLAT-IV-007` con valores inequívocamente ficticios: `AWS_REGION`, `AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY`, `TICKETING_TABLE_NAME`, `ORDERS_QUEUE_NAME`, `ORDERS_DLQ_NAME`, `PROVISIONING_QUEUE_NAME`, `PROVISIONING_DLQ_NAME`, `DYNAMODB_HOST_PORT`, `LOCALSTACK_HOST_PORT`, `LOCAL_IDP_HOST_PORT`, `PAYMENT_MOCK_HOST_PORT`, `PAYMENT_MOCK_API_KEY`, `LOCAL_IDP_ISSUER`, `LOCAL_IDP_CLIENT_ID`, `LOCAL_IDP_DEFAULT_TTL_SECONDS`, `LOCAL_IDP_MAX_TTL_SECONDS`, `LOAD_TOKEN_COUNT`, `LOAD_TOKEN_TTL_SECONDS` |
| `docker-compose.yml` | Esqueleto: `name: ticketing-platform`, `services: {}`, red `ticketing-net`; sin `container_name` |
| `platform/.gitattributes` | LF obligatorio para todo `platform/**` (scripts, Java, JSON, Dockerfile) |

Las variables de puertos de `ticketing-api` (8080/8081) y todas las del backend se añaden en PLAT-INC-006 con los nombres literales del handoff de INC-010; no se inventan aquí.

## 2. Resources and services

Ningún servicio todavía (esqueleto). Proyecto Compose `ticketing-platform`, red `ticketing-net`.

## 3. Spikes

### PLAT-SPK-001 — Imágenes JDK/JRE 25 (`PLAT-IV-001` a: Eclipse Temurin sobre Ubuntu) — CONFIRMED con alternativa de healthcheck

Consulta al registro oficial (Docker Hub, `library/eclipse-temurin`) y descarga solo de los candidatos aprobados.

| Uso | Imagen fijada | Verificado |
|---|---|---|
| Build (JDK) | `eclipse-temurin:25.0.4.1_1-jdk-noble@sha256:589ff4cc3f71aab462e7048a47a0d10edf57fbccde3fceea2281e610bf5880b4` | `openjdk version "25.0.4.1" 2026-08-18 LTS`, `Temurin-25.0.4.1+1`; Ubuntu 24.04.5 LTS (noble); linux/amd64; 400 MB |
| Runtime (JRE) | `eclipse-temurin:25.0.4.1_1-jre-noble@sha256:d9a39a23634650173f1e2bbc176227af9728587ecf0f4b62d53e9355cd7a19ab` | Misma versión; Ubuntu 24.04.5 LTS; linux/amd64; 318 MB; `Config.User` vacío (root por defecto, se fija uid no root en las imágenes derivadas) |

- Mantenimiento: ambos tags publicados/reconstruidos el 2026-10-04 (índice multi-arquitectura; el digest fijado es el del índice).
- Ejecución no root: `docker run --user 10001:10001 … id` → `uid=10001 gid=10001`; `java -version` funciona; `/tmp` es `drwxrwxrwt`; `useradd` disponible para crear el usuario en la etapa final.
- Cliente HTTP: **no** hay `curl`, `wget` ni `nc` en la JRE (`command -v` → missing). Módulos `java.net.http` y `jdk.httpserver` presentes. Conforme a la respuesta de `PLAT-IV-001`, el healthcheck de las imágenes Java propias usa una **sonda JDK compilada en la etapa de build** (sin instalar paquetes).
- Se eligió el sufijo explícito `-noble` (Ubuntu 24.04 LTS) en lugar del tag sin sufijo, que ahora apunta a `resolute`; el tag con codename evita que una revisión de Ubuntu cambie con el mismo tag Java.

### PLAT-SPK-004 (parte de imagen) — AWS CLI v2 (`PLAT-IV-004` a) — CONFIRMED (imagen)

| Imagen fijada | Verificado |
|---|---|
| `amazon/aws-cli:2.37.9@sha256:92de75724b6a746951f0e8b915d86bbccd7cb55aff96cd0cb4f7017272160780` | `aws-cli/2.37.9 Python/3.14.6 … docker/x86_64.amzn.2023`; Amazon Linux 2023.12.20260930; ENTRYPOINT `/usr/local/bin/aws`; 457 MB; publicada 2026-10-02 |

Como uid 10001 con `HOME=/tmp`, `aws --version` funciona. La imagen trae `bash`, `sed`, `awk`, `sort`, `tr`, `curl`, `jq` y `python3`. La validación funcional contra los emuladores es parte de PLAT-INC-002.

### PLAT-SPK-005 (evidencia de la opción elegida) — emisor propio con la biblioteca estándar (`PLAT-IV-002` a) — CONFIRMED (viabilidad)

Programa de un solo archivo ejecutado como uid 10001 en la imagen JDK fijada: genera un par RSA 2048, firma un JWT RS256 con `cognito:groups` como array, reconstruye la clave pública desde módulo/exponente en base64url (forma JWKS) y verifica la firma → `verify=true`; `com.sun.net.httpserver.HttpServer` arranca → `httpserver=ok`. Sin dependencias externas. El emisor completo se implementa en PLAT-INC-003.

### PLAT-SPK-009 — `depends_on` con condiciones en Compose v5.5.1 — CONFIRMED

Proyecto Compose aislado `plat-spk-009` (directorio temporal fuera de `CODE_REPO`, eliminado con `down` al terminar; 0 contenedores restantes):

| Caso | Resultado observado |
|---|---|
| Emulador sano → init sale 0 → app | `up` exit 0; `init exited 0`, `app running` |
| init sale 3 (`restart: "no"`) | `up` exit 1: `service "init" didn't complete successfully: exit 3`; `app` queda `created` (nunca arranca) |
| Emulador nunca sano | `up` exit 1: `dependency failed to start: container plat-spk-009-emu-1 is unhealthy`; `init` y `app` quedan `created` |

`service_healthy` y `service_completed_successfully` se comportan como exige ADR-036.

## 4. Verification

| Comando (en `D:\Nequi\ticketing-platform`) | Resultado |
|---|---|
| `docker compose --env-file .env.example config` | exit 0; `name: ticketing-platform`, `services: {}` |
| `git check-ignore -v .env .env.local .env.example` | `.env` y `.env.local` ignorados (`.gitignore:3`, `:4`); `.env.example` re-incluido por `.gitignore:5` (`!.env.example`) |
| `grep -rn ":latest" docker-compose.yml .env.example platform/` | sin coincidencias |
| `grep -rniE "BEGIN (RSA )?PRIVATE|eyJ…|AKIA…|LOCALSTACK_AUTH_TOKEN" …` | sin coincidencias |
| `git status --short -- . ':!ticketing'` | solo `?? .env.example`, `?? .gitignore`, `?? docker-compose.yml`, `?? platform/` |

## 5. Security checks

- `.env` real ignorado por Git; el ejemplo solo contiene valores con `fictitious` en el propio valor.
- Credenciales AWS ficticias no numéricas: LocalStack las mapea a la cuenta por defecto `000000000000` (se reverifica en PLAT-INC-002).
- Sin tags flotantes: todas las imágenes externas registradas como `tag@sha256`.

## 6. Deviations

1. No se creó `.env` en `CODE_REPO`: no figura entre las rutas de escritura del agente. Las verificaciones de los incrementos siguientes usan un archivo de entorno temporal fuera del repositorio (`--env-file`). El humano crea su `.env` desde `.env.example` con `PAYMENT_MOCK_HOST_PORT=18090` (`PLAT-IV-008`) y una API key propia.
2. Los subdirectorios de `platform/` se crean con su contenido en cada incremento (Git no versiona directorios vacíos).

## 7. Blockers

Ninguno.

## 8. Handoff to other agents

- Documentation y QA: lista de variables de Platform = `.env.example` (sección 1). Copiar a `.env` y no versionar.
- PLAT-INC-004/005: bases Java fijadas arriba; healthcheck por sonda JDK compilada en la etapa de build.
