---
artifact: documentation-increment-report
increment: DOC-INC-002
result: DONE
code_revision: "CODE_REPO main 90f8ed2 + cambios sin commit de este agente: README.md (M), README.en.md, docs/traceability.md, docs/operations.md (nuevos)"
spec_revision: "SPEC_REPO main 9b78fdd"
verified_at: 2026-10-05
environment: "Windows 10 Pro 19045; Docker 29.8.0; Compose v5.5.1; Git Bash 5.2.12; curl 7.87.0; Windows PowerShell 5.1.19041; JDK Zulu 25.0.4.1; .env local sin modificar (PAYMENT_MOCK_HOST_PORT=18090), clave nunca impresa"
---

# DOC-INC-002 — README operativo: requisitos, configuración, Compose, verificación, tokens, mock y build

## 1. Documents produced

| Archivo | Cambio |
|---|---|
| `README.md`, `README.en.md` | §2 (filas de build y entorno con la evidencia nueva), §5 (requisitos por uso con versiones verificadas), §6 (bloques `sh` y `powershell`, tabla resumida de variables), §7 (arranque, estado, logs, parada y limpieza con tiempos medidos), §8 (verificadores y salud con resultados), §9 (identidades, tokens en variables, PowerShell), §10 (control del Payment Mock, tabla `API-101` a `API-111`, comportamientos), §13 (resultados de build con revisión y fuente), §21 (requisito de JDK 25 y parada previa de los servicios de Compose, resultado verificado), §22 (fallos observados), §23 (referencias operativas) |
| `docs/operations.md` | Nuevo: servicios y puertos, variables de `.env` (propósito, obligatoriedad, ejemplo seguro sin copiar la clave), variables de la aplicación (INC-010 §6.5.3), ciclo de vida con estado de verificación, verificadores, salud y métricas, recuperación de emuladores, `run-local.sh`, alternativa de LocalStack con token (DOC-IV-008 a) y solución de problemas en detalle |

## 2. Sources and traceability

`.env.example`, `docker-compose.yml`, `run-local.sh`, cabeceras de `platform/verify/*.sh`; PLAT-INC-003 §8, PLAT-INC-006 §4 y §8, PLAT-INC-007 §3, §4 y §8; INC-010 §3 y §6.5; PM-INC-006 §4 y §7; ADR-030, ADR-033, ADR-036. Respuestas humanas: PLAN (sin `-Pintegration`), DOC-IV-007 (a), DOC-IV-008 (a).

## 3. Commands and scenarios verified

Todos desde `D:\Nequi\ticketing-platform` (2026-10-05, revisión `90f8ed2`).

| Comando | Resultado |
|---|---|
| `docker compose --env-file .env.example config --quiet`; `docker compose config --quiet`; `docker compose --profile load config --quiet` | exit 0 los tres; 7 servicios por defecto |
| Estado inicial | Solo dependencias (`dynamodb-local`, `localstack`, `local-idp`, `payment-mock` sanos; `infra-init` `Exited (0)`) |
| `docker compose down -v` (autorizado) | exit 0; 0 contenedores, 0 volúmenes y 0 redes del proyecto. Recursos ajenos (`dynamodb-local` ajeno, `dynamodb_default`, volúmenes ajenos) intactos |
| `docker compose up -d --wait` | exit 0 en **24,8 s**, 0 `pull`; 6 servicios `healthy`, `infra-init` `Exited (0)` |
| `bash platform/verify/environment.sh` | **65 / 0** |
| `docker compose run --rm --no-deps -v ./platform/verify:/verify:ro --entrypoint sh infra-init /verify/resources.sh` | **Falla en Git Bash** (`C:/Program Files/Git/verify/resources.sh: No such file or directory`, rc 127): conversión de rutas de MSYS. Con `MSYS_NO_PATHCONV=1`: **38 PASS, 0 fallos**. Documentado como fallo observado |
| `bash platform/verify/identity.sh --skip-restart` | **38 / 0** en 24 s. Deja el contenedor `load-token-generator` y el volumen `ticketing-platform_load-tokens` |
| `environment.sh` tras `identity.sh` | **63 / 2** ("load profile containers", "project volumes"). Limpieza documentada `docker compose --profile load rm -s -f load-token-generator` + `docker volume rm ticketing-platform_load-tokens` → 65 / 0 |
| Salud por curl | `:8080/readyz` 200 `{"status":"UP"}`, `/livez` 200, `:8081/actuator/health` 200, `/actuator/prometheus` 200, `:8080/actuator/health` 401, `:18090/health` 200, discovery y JWKS 200 |
| Tokens | `identity=admin` → 200 con `access_token`, `token_type`, `expires_in` 3600 (token no impreso); `sub=…&groups=CUSTOMER&expires_in=600` → 200; `identity=root` → 400 |
| Control del mock (curl) | reset 204, defaults 200, regla 201 (`rule-1`), lista 200, borrar regla 204, borrar todas 204, autorización inexistente 404, sin clave 401 |
| Bloques `sh` de README §8, §9 y §10 extraídos del archivo y ejecutados tal cual (solo `8090` → `18090` en §8, como indica el comentario) | exit 0 los tres; salidas sin tokens impresos salvo el JSON del token de §9, guardado y borrado después |
| Bloques `powershell` de README §9 y §10 ejecutados tal cual; §6 `Copy-Item` probado contra un destino temporal (sin tocar `.env`) | exit 0; tokens de 713 y 724 caracteres (no impresos); reset 204 |
| `curl -X POST …` en PowerShell 5.1 | Falla: `curl` es alias de `Invoke-WebRequest` (documentado) |
| `JAVA_HOME=<JDK 25> ./mvnw -B -o verify` en `ticketing/` | `BUILD SUCCESS` en 107 s: 99 + 277 + 369 + 24 + 1 = **770 pruebas, 0 fallos, 0 omitidas**; cobertura agregada JaCoCo **99,49 %** (5.505 / 5.533 líneas: domain 98,18 %, application 99,85 %, infrastructure 99,60 %); puerta en verde |
| `JAVA_HOME=<JDK 25> ./mvnw -B -o verify` en `payment-mock/` | `BUILD SUCCESS` en 35 s: **256 pruebas, 0 fallos** |
| `.\mvnw.cmd -v` en PowerShell (ambos proyectos) | Maven 3.9.16 sobre Java 25.0.4.1 |
| `./mvnw verify -Pintegration` | **No ejecutado** (decisión humana del PLAN). Se cita INC-010: 129 `*IT`, revisión `69a2c23` |
| `docker compose stop ticketing-api ticketing-worker` | 5,3 s, exit 143 de ambos (apagado ordenado) |
| `JAVA_HOME=<JDK 25> ./run-local.sh start` | exit 0 en 20 s (`api` :8080/:8081, `worker` :8082/:8083) |
| `./run-local.sh smoke` | exit 0 en 3 s: `SMOKE OK: Order … CONFIRMED`; 0 `eyJ` en la salida |
| `./run-local.sh stop` | `api stopped`, `worker stopped`; `:8080` sin respuesta |
| `docker compose up -d --wait` (restaurar) | exit 0 en 20,5 s |
| `docker compose stop` (todo) | exit 0 en **7,2 s** |
| `docker compose up -d --wait` tras `stop` | exit 0 en **28,2 s**; `infra-init` se vuelve a ejecutar; `GET /events` pasa de 1 Event a 0 (datos efímeros confirmados) |
| `docker compose logs --tail 2 ticketing-api ticketing-worker` | Líneas JSON ECS |

Se usó `-o` (offline) en Maven porque las dependencias ya estaban en la caché local; el comando documentado es `./mvnw verify`.

## 4. Static validation

| Gate | Resultado |
|---|---|
| G1 | 0 fallos (un enlace a `resilience.md`, aún inexistente, se dejó como texto) |
| G2 | 0 fallos sobre entregables y las 13 salidas guardadas (logs de verificadores, builds y bloques del README). El patrón `PRIVATE KEY` se afinó a una cabecera PEM real porque los propios verificadores imprimen el nombre de la comprobación |
| G4 | 0 fallos: 23 H2, 12 bloques de código idénticos, 14 tablas con las mismas filas, mismos enlaces, IDs y etiquetas por sección |
| G5 | Ver §3 |
| G6 | 0 hallazgos |

## 5. Unverified claims and dependencies

- Tiempo del primer build (unos 2 min) citado de PLAT-INC-007; no se reconstruyeron imágenes.
- Postman (aplicación) no verificado; solo newman (DOC-INC-003).
- `platform/verify/infra-init-negative.sh` no ejecutado (documentado como tal).
- Efectos de los comportamientos del mock en la Order (tabla de README §10) según ADR-030; se verifican con la colección en DOC-INC-003.
- Alternativa de LocalStack con token: `Designed, not implemented`, no ejecutada (DOC-IV-008 a).

## 6. DOC-ISSUE items

| ID | Cambio |
|---|---|
| DOC-ISSUE-008 | Actualizado: el `smoke` ya trata `REJECTED` como final (commit `90f8ed2`); siguen las variables de puerto propias del script no documentadas en `.env.example` |
| DOC-ISSUE-010 (nuevo) | Propietario: Platform. Severidad LOW. (a) El uso documentado de `resources.sh` en su cabecera falla en Git Bash sin `MSYS_NO_PATHCONV=1`; (b) `identity.sh` deja el contenedor `load-token-generator` y el volumen `load-tokens`, y `environment.sh` ejecutado después informa 2 fallos. Tratamiento documental: prefijo y limpieza acotada documentados en README §8 y `docs/operations.md` §5. Corrección opcional de Platform: incluir el prefijo en la cabecera y limpiar los restos al final de `identity.sh` |

## 7. Deviations

- Entorno al terminar el incremento: **todo el stack levantado** (necesario para DOC-INC-003). La restauración al estado inicial (solo dependencias) se hace al final de DOC-INC-006.
- El volumen `load-tokens` creado por `identity.sh` se eliminó con `docker volume rm ticketing-platform_load-tokens` (volumen del proyecto, mismo efecto que el `down -v` autorizado).

## 8. Handoff

Siguiente: DOC-INC-003. Platform: DOC-ISSUE-010.
