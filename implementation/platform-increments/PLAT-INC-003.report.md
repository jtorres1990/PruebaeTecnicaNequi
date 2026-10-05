---
artifact: platform-increment-report
increment: PLAT-INC-003
result: DONE
code_revision: "CODE_REPO main 31b39ac + untracked platform files (docker-compose.yml, platform/local-idp/**, platform/verify/identity.sh); ticketing/ has uncommitted changes of the Backend Developer (INC-010), untouched; no commits by this agent"
verified_at: 2026-10-04
plan: implementation/platform.implementation-plan.v1.md
review: human-review/platform.implementation-plan-review.yaml (APPROVED)
---

# PLAT-INC-003 — `local-idp`, cinco identidades y tokens de carga

## 1. Implemented

| Archivo (`CODE_REPO`) | Contenido |
|---|---|
| `platform/local-idp/src/LocalIdp.java` | Emisor propio, solo biblioteca estándar de Java 25 (`com.sun.net.httpserver`, `java.security`, `java.net.http`); modos `serve`, `generate`, `verify` y `probe` (sonda de salud) |
| `platform/local-idp/Dockerfile` | Multi-etapa: `javac --release 25 -Xlint:all -Werror` + `jar` en la JDK fijada; runtime en la JRE fijada; usuario `localidp` uid/gid 10001; `/tokens` propiedad de 10001 para el volumen; jar `0444`; `ENTRYPOINT` exec con `-XX:MaxRAMPercentage=75 -XX:+ExitOnOutOfMemoryError` |
| `platform/verify/identity.sh` | Verificador del contrato de identidad (plan §8.3), 40 comprobaciones |
| `docker-compose.yml` | Servicio `local-idp`; servicio `load-token-generator` en el perfil `load`; volumen `load-tokens` |

Contrato implementado (`PLAT-IV-003` a):

| Elemento | Valor |
|---|---|
| Puerto | 9000 en el contenedor; `127.0.0.1:${LOCAL_IDP_HOST_PORT:-9000}` |
| Emisor | Fijo por configuración `LOCAL_IDP_ISSUER` (por defecto `http://local-idp:9000`), independiente del host de la solicitud |
| Discovery | `GET /.well-known/openid-configuration` (`issuer`, `jwks_uri`, `token_endpoint`, algoritmos, claims) |
| JWKS | `GET /.well-known/jwks.json`: una clave RSA 2048, `use=sig`, `alg=RS256`, `kid` = huella RFC 7638 |
| Emisión | `POST /token` (`application/x-www-form-urlencoded`): `identity=<admin\|customer-a\|customer-b\|admin-customer\|no-groups>` **o** `sub=<[A-Za-z0-9._-]{1,128}>&groups=<subconjunto de ADMIN,CUSTOMER>`; `expires_in` opcional (defecto 3.600, máximo 86.400) → `{access_token, token_type: "Bearer", expires_in}`; `Cache-Control: no-store`; errores 400 `invalid_request`, 405, 404 |
| Token | Cabecera `{kid, alg: RS256}`; claims `sub`, `cognito:groups` (array; ausente si no hay grupos), `iss`, `client_id` (`ticketing-local-client`, ficticio), `token_use=access`, `iat`, `exp`, `jti` (UUID); sin `aud` |
| Claves | Generadas en memoria al arrancar; nunca persistidas ni en la imagen; reiniciar el servicio rota la clave |
| Seguridad | Sin autenticación (solo loopback del host y red del proyecto); sin logs por solicitud y nunca de tokens |

Modo `generate` (`load-token-generator`, `PLAT-IV-009` a): pide al emisor en ejecución `LOAD_TOKEN_COUNT` (por defecto 1.200; mínimo 1.001) tokens `CUSTOMER` con sujetos `load-customer-<lote>-NNNNN` y vigencia `LOAD_TOKEN_TTL_SECONDS` (por defecto 7.200), 32 en vuelo; comprueba sujeto, grupo y unicidad de `sub` y `jti`; escribe de forma atómica `/tokens/customer-tokens.csv` (cabecera `sub,access_token`) y `/tokens/manifest.json` (sin tokens), ambos `0644`.

Modo `verify`: obtiene el JWKS de la URL indicada, reconstruye las claves y verifica criptográficamente la firma RS256 de cada token (stdin, `--csv` o `--issue <form>`), además de `iss`, `exp`, `token_use` y `client_id`. Imprime solo un resumen de claims (jti truncado), nunca el token.

## 2. Resources and services

| Servicio | Imagen | Host | Salud / fin | Endurecimiento |
|---|---|---|---|---|
| `local-idp` | `ticketing-platform/local-idp:local` (sobre `eclipse-temurin:25.0.4.1_1-jre-noble@sha256:d9a39a23…7a19ab`) | `127.0.0.1:9000` | Sonda JDK: discovery **y** JWKS 200; sano en ~2,7 s | uid 10001, `read_only`, `tmpfs /tmp`, `cap_drop: ALL`, `no-new-privileges`, `mem_limit: 256m` |
| `load-token-generator` (perfil `load`) | Misma imagen, `command: generate` | — | Un solo uso, `restart: "no"`, `depends_on local-idp: service_healthy`; código 0 | Igual; volumen `load-tokens` en `/tokens` |

Volumen `load-tokens` (nombre real `ticketing-platform_load-tokens`): único volumen del proyecto; se elimina con `docker compose down -v`.

## 3. Spikes

| Spike | Resultado | Evidencia |
|---|---|---|
| PLAT-SPK-005 Emisor | CONFIRMED | Claims exactos, sujetos arbitrarios, vigencia configurable, discovery, JWKS, emisor fijo; firma verificada por el modo `verify` (verificador independiente que obtiene la clave del JWKS por HTTP) |
| PLAT-SPK-006 Host vs red | CONFIRMED | Discovery y JWKS 200 desde el host (`127.0.0.1:9000`) y desde la red (`http://local-idp:9000`); SHA-256 del JWKS idéntico (`302bd825d055c9aa` en la ejecución 1); tokens emitidos desde el host y desde la red con `iss=http://local-idp:9000` byte a byte igual |
| PLAT-SPK-007 > 1.000 tokens | CONFIRMED | 1.200 tokens en 896 ms (ejecución 2; 1.238 ms en la 1) dentro del contenedor; 2,4 s de pared con arranque del contenedor; 1.200 `sub` y `jti` distintos; los 1.200 verifican criptográficamente; sin tokens en logs ni manifiesto |
| PLAT-SPK-008 (IdP) | CONFIRMED | Healthcheck por sonda JDK (la JRE no trae `curl`/`wget`): `healthy` solo cuando discovery y JWKS responden 200 |
| PLAT-SPK-014 (IdP, parcial) | CONFIRMED | `read_only` + `tmpfs /tmp` + `cap_drop: ALL` operativos; `docker compose stop` en 0,6 s con el hook de apagado ejecutado (`local-idp: stopping`), código 143 (SIGTERM manejado por la JVM) |

## 4. Verification

| Comando (en `D:\Nequi\ticketing-platform`) | Resultado |
|---|---|
| `docker compose --env-file .env.example config --quiet` | exit 0 |
| `docker compose --env-file .env.example config --services` | `local-idp localstack dynamodb-local infra-init` (sin servicios del perfil `load`) |
| `docker compose --env-file .env.example --profile load config --services` | añade `load-token-generator` |
| `docker compose --env-file <tmp> build local-idp` | Built; `javac -Xlint:all -Werror` sin avisos |
| `docker compose --env-file <tmp> up -d --wait local-idp` | `Healthy` en 2,7 s |
| `bash platform/verify/identity.sh --env-file <tmp>` (2.ª ejecución) | `RESULT 40 passed, 0 failed` |

Detalle del verificador (2.ª ejecución):

| Bloque | Comprobaciones | Resultado |
|---|---|---|
| 1 Host | discovery `issuer` y `jwks_uri` = `http://local-idp:9000…`; JWKS RS256 | PASS |
| 2 Red | discovery y JWKS 200 desde un contenedor de la red; mismo SHA-256 de JWKS que el host | PASS |
| 3 Cinco identidades (emitidas en el host, verificadas con el JWKS) | `admin` `[ADMIN]`; `customer-a`, `customer-b` `[CUSTOMER]`; `admin-customer` `[ADMIN, CUSTOMER]`; `no-groups` sin claim; `token_use=access`, `client_id=ticketing-local-client`, `iss=http://local-idp:9000`, sin `aud`, vigencia 3.600 s; 5 `sub` y 5 `jti` distintos | PASS |
| 4 Sujeto arbitrario | `qa.arbitrary-subject_01` `[CUSTOMER]` con `expires_in=600`; sujeto de 128 caracteres aceptado | PASS |
| 5 Emitidos desde la red | `customer-a` y `network-subject`, mismo `iss` | PASS |
| 6 Rechazos | token vencido → `INVALID expired`; firma alterada → `INVALID signature`; carga útil falsificada con `ADMIN` → `INVALID signature`; emisor esperado distinto → `INVALID iss`; `identity=root`, sujeto de 129 caracteres, sujeto con espacio, grupo `ROOT`, `identity`+`sub`, `expires_in=86401` → 400; `GET /token` → 405; ruta desconocida → 404 | PASS |
| 7 Lote de carga | generador exit 0; 1.200 válidos, `distinct_sub=1200`, `distinct_jti=1200`; manifiesto `count:1200`, `issuer`, `client_id`, `batch`, `issued_at`, `expires_at` (+7.200 s); sin `eyJ` en manifiesto ni logs del generador ni de `local-idp`; archivos `644` propiedad de 10001 | PASS |
| 8 Rotación | tras `restart local-idp`, un token previo → `INVALID unknown kid`; el lote → `valid=0` (hay que regenerarlo) | PASS |

La 1.ª ejecución dio 36/40: el verificador montaba el volumen con `docker compose run -v load-tokens:…`, que crea un volumen `load-tokens` fuera del proyecto (vacío). Se corrigió el script para usar el nombre real `ticketing-platform_load-tokens` y se eliminó ese volumen creado por la prueba (`docker volume rm load-tokens`, identificado por nombre y sin etiquetas de otro proyecto).

## 5. Security checks

- Imagen `ticketing-platform/local-idp:local`: `Config.User=10001:10001`; `id` → `uid=10001(localidp)`; jar de solo lectura propiedad de root; 318 MB.
- `docker history --no-trunc`: 0 coincidencias de `secret`, claves privadas o `eyJ`. Ningún `ARG`; ninguna clave en capas (se generan al arrancar).
- Búsqueda de `eyJ…` y `BEGIN … PRIVATE` en `CODE_REPO` (excluidos `ticketing/` y `payment-mock/`): sin coincidencias. Los tokens solo existen en memoria del verificador y en el volumen efímero.
- Memoria en reposo de `local-idp`: 33,9 MiB de 256 MiB.
- Puerto ligado a `127.0.0.1`. El endpoint de emisión no tiene autenticación por diseño (`PLAT-IV-003` a): emisor solo local, nunca en AWS.

## 6. Deviations

1. `groups` es opcional para sujetos arbitrarios: si se omite, el token no lleva `cognito:groups` (mismo comportamiento que `no-groups`). El generador de carga siempre envía `groups=CUSTOMER`.
2. Variables internas no listadas en `PLAT-IV-007`, con valor fijo en Compose: `LOCAL_IDP_URL` (generador), `LOAD_TOKEN_OUTPUT_DIR` (`/tokens`), `LOCAL_IDP_PORT` (9000, no expuesta) y `LOAD_TOKEN_BATCH` (opcional; por defecto derivada del instante).
3. Se añadió el modo `probe` (sonda de salud) a los tres modos aprobados, por la ausencia de cliente HTTP en la JRE (`PLAT-IV-001`: sonda JDK).
4. La cabecera del token omite `typ`, igual que Cognito.

## 7. Blockers

Ninguno.

## 8. Handoff to other agents

Backend (INC-010), configuración del rol `api` (nombres de variables según su handoff):

| Parámetro lógico | Valor local en la red Compose |
|---|---|
| Emisor esperado (`iss`) | `http://local-idp:9000` |
| URL del JWK set | `http://local-idp:9000/.well-known/jwks.json` |
| `client_id` permitido | `ticketing-local-client` |
| Claims | `sub`, `cognito:groups` (array, ausente sin grupos), `token_use=access`, `client_id`, `iss`, `exp`, `iat`, `jti`; RS256 con `kid`; sin `aud` |

QA y Documentation:

- Obtener un token desde el host: `curl -s -X POST http://127.0.0.1:9000/token -H 'Content-Type: application/x-www-form-urlencoded' --data 'identity=customer-a'` (o `sub=<id>&groups=CUSTOMER[,ADMIN]&expires_in=<s>`). El `iss` es siempre `http://local-idp:9000`, también para tokens obtenidos desde el host.
- Lote de carga: `docker compose --profile load up load-token-generator` → volumen `ticketing-platform_load-tokens` con `customer-tokens.csv` (`sub,access_token`) y `manifest.json`. Reiniciar `local-idp` invalida todos los tokens previos y el lote: regenerarlo después de cualquier reinicio (R-13). Vigencia del lote: `LOAD_TOKEN_TTL_SECONDS` (7.200 por defecto) debe superar duración + rampa de la prueba; máximo del emisor `LOCAL_IDP_MAX_TTL_SECONDS` (86.400).
- `API-004` limita a 10 solicitudes por 10 s por sujeto (IV-005 backend): dimensionar `LOAD_TOKEN_COUNT` en consecuencia.
- Verificar tokens sin imprimirlos: `printf '%s\n' "$TOKEN" | docker compose run --rm --no-deps -T local-idp verify`.
