---
artifact: platform-increment-report
increment: PLAT-INC-007
result: DONE
code_revision: "CODE_REPO main 5259066 (árbol limpio al empezar); cambios sin commit, solo de este agente: docker-compose.yml (M, sha256 f0189a0d…3d53879) y platform/verify/environment.sh (nuevo, sha256 fafda89f…1f40846); ningún commit de este agente"
verified_at: 2026-10-05
plan: implementation/platform.implementation-plan.v1.md
review: human-review/platform.implementation-plan-review.yaml (APPROVED)
decisions_applied: human-review/platform.implementation-review.2.yaml (APPROVED; PLAT-IV-016 (a), pull_policy never autorizado, down -v del proyecto autorizado)
new_validations: none
overall_status: LOCAL_PLATFORM_COMPLETE
---

# PLAT-INC-007 — Verificación limpia, seguridad, reproducibilidad y handoffs

## 1. Implemented

| Archivo (`CODE_REPO`) | Cambio |
|---|---|
| `docker-compose.yml` | `pull_policy: never` en `payment-mock`, `infra-init`, `local-idp` y `load-token-generator` (autorizado en `platform.implementation-review.2.yaml`). Con esto las 7 definiciones que usan imágenes propias `ticketing-platform/*:local` nunca consultan un registro. Comentario de cabecera actualizado. Sin otros cambios: mismos servicios, variables, puertos, dependencias y salud |
| `platform/verify/environment.sh` (nuevo) | Verificador del entorno levantado (§8.1 y §8.4 del plan). Hace 65 comprobaciones; con valores reales comprueba, sin imprimirlos:<br>• Compose resuelto con `.env.example` sin secretos;<br>• 7 servicios por defecto y 9 con el perfil `load`;<br>• salud y `infra-init` con salida 0;<br>• mismo image ID en `api` y `worker`;<br>• puertos aprobados, todos en `127.0.0.1`, y `worker` sin puertos;<br>• sin volúmenes ni binds, y `restart: no`;<br>• endurecimiento y uid 10001 de las imágenes propias;<br>• imágenes externas con tag y digest, sin `:latest`;<br>• ni la API key ni JWT en el historial, la configuración de imagen o los logs.<br>Uso: `bash platform/verify/environment.sh [--env-file <f>] [-p <proyecto>]`. El código de salida es el número de fallos |

No se modificó ningún otro archivo. Siguen igual (sha256 antes = después) `.env`, `.env.example`, `.gitattributes`, `.gitignore`, `README.md`, `run-local.sh`, `postman/*`, `ticketing/Dockerfile`, `payment-mock/Dockerfile` y ambos `.dockerignore`. Tampoco se tocó el resto de `ticketing/` y `payment-mock/`.

PLAT-IV-016 (a) aplicado: `load-test` sigue montando solo `load-tokens` y no crea `load-tests/`; se verificó que no existe tras usar el perfil `load`.

## 2. Resources and services

| Recurso | Estado al cierre |
|---|---|
| Proyecto `ticketing-platform` (CODE_REPO real) | **Como se encontró:** levantadas solo las dependencias, con el mismo comando que `run-local.sh start` (`up -d --wait dynamodb-local localstack infra-init local-idp payment-mock` y espera a `infra-init` `exited 0`, unos 12 s).<br>Estado: `dynamodb-local`, `localstack`, `local-idp` y `payment-mock` `healthy`, e `infra-init` `Exited (0)`. Sin `ticketing-*` ni `load-*`. Red `ticketing-platform_ticketing-net` y 0 volúmenes del proyecto.<br>Los datos son efímeros y nuevos: el `down -v` autorizado eliminó los anteriores |
| Imágenes propias | **Restauradas a los mismos ID que al empezar:**<br>• `ticketing` `4f8a52fe…4425`<br>• `payment-mock` `aedbe5e7…acdac0`<br>• `local-idp` `90c0a973…f7ddf4`<br>• `infra-init` `b787ede5…e869`<br>Antes de las pruebas se protegieron con etiquetas temporales `plat-inc007-backup/*:orig` y después se reetiquetaron; las temporales están eliminadas |
| Imágenes construidas desde el clon | 7 imágenes, todas eliminadas por etiqueta o ID:<br>• `772f3f6f` y `b6bf4212`, del primer `up`;<br>• `8cc0e61b`, `19a26f5e`, `5a264279`, `0380237a` y `d776f94a`, del build `--no-cache`.<br>Imágenes colgantes al cierre: solo las 3 ajenas del 1 y 2 de octubre (`d05895cb`, `8ae9e71b`, `71f047c9`), sin tocar |
| Clon temporal `C:\Users\zombr\AppData\Local\Temp\pi7\clone`, export `…\pi7\archive`, exportaciones de contexto y logs | Eliminados (`rm -rf …\pi7`). También se eliminó el primer intento de clon en el scratchpad (§6, desviación 1) |
| Caché de BuildKit | Quedan entradas de caché de capas de los builds del clon y la caché `id=ticketing-m2`/`payment-mock-m2`. No se hizo `builder prune` porque es una limpieza global, prohibida por el contrato. Son reutilizables y no pertenecen a ningún contenedor |
| Recursos ajenos | Sin cambios:<br>• contenedor `dynamodb-local` `7c60ccb8…` (`exited` desde 2026-10-04T21:00:17Z);<br>• red `dynamodb_default` `13f6d7a5…`;<br>• volumen anónimo `d72a66f1…` y `portainer_data`;<br>• imágenes de terceros |

## 3. Spikes

### Finales de línea con `core.autocrlf=true` — CONFIRMED (no rompe nada)

Configuración de git de este equipo: `core.autocrlf=true` (en `C:/Program Files/Git/etc/gitconfig`), `core.symlinks=false`, `core.longpaths` sin definir y git 2.39.2.windows.1.

Un `git clone` real de `D:\Nequi\ticketing-platform` (HEAD `5259066`) deja en el árbol de trabajo:

| Finales de línea en el clon | Archivos |
|---|---|
| **CRLF** (sin regla en `.gitattributes`) | 461 archivos, entre ellos `docker-compose.yml`, `.env.example`, `ticketing/Dockerfile`, `payment-mock/Dockerfile`, `ticketing/.dockerignore`, `payment-mock/.dockerignore`, `pom.xml`, `application.yaml`, `maven-wrapper.properties`, `*.java` y `postman/*.json` |
| **LF** (con regla) | `platform/**` completo (por `platform/.gitattributes`, `* text=auto eol=lf`), es decir, los Dockerfile de `infra-init` y `local-idp`, sus scripts, recursos y verificadores; `ticketing/mvnw` y `payment-mock/mvnw`; `run-local.sh`; los contratos `-text` |

`git archive HEAD` (exportado y extraído) produce **los mismos finales de línea que el clon**: aplica `core.autocrlf` y los atributos. `diff -rq` entre ambos solo muestra los dos archivos de este incremento.

Cada consumidor de un archivo CRLF se comprobó con un resultado real:

| Consumidor | Prueba | Resultado |
|---|---|---|
| Compose (YAML CRLF) | `config --quiet` con `.env.example` y con `.env`, también con `--profile load` | exit 0, 0 bytes CR en la configuración resuelta |
| Lectura de `.env` CRLF por Compose | `.env` copiado de `.env.example` conservando CRLF (48 CR) | Valores sin `\r`: `AWS_REGION: us-east-1`, URL de colas y puertos correctos |
| BuildKit, Dockerfile CRLF (heredoc `COPY <<'EOF'` y líneas con `\`) | `docker build --check` de los 4 Dockerfile del clon; build real | "no warnings"; los builds terminan y `HealthProbe` compila con `-Werror` |
| `.dockerignore` CRLF | Contexto exportado con `--output type=local` (sin crear imagen), con señuelos `target/sentinel.txt`, `.env` y `notes.log` | Señuelos excluidos (0); en `ticketing` solo `.mvn`, `mvnw`, `pom.xml` y los 4 módulos (369 archivos); en `payment-mock` solo `.mvn`, `mvnw`, `pom.xml` y `src` (85 archivos) |
| Maven Wrapper y fuentes CRLF | `sh ./mvnw -B -ntp verify` dentro del build | **ticketing:** 770 pruebas (99 + 277 + 369 + 24 + 1), 0 fallos, 2 omitidas y puerta de cobertura OK.<br>**payment-mock:** 256 pruebas, 0 fallos |
| `sh` de Git Bash cargando `.env` CRLF (`set -a; . ./.env`, como `run-local.sh`) | `od -c` de `AWS_REGION` y de una URL de cola | Sin `\r`: el `sh` de MSYS lo descarta |

**Conclusión: no hace falta ninguna regla nueva.** Ni `platform/.gitattributes` ni la raíz requieren cambios. Riesgo residual, no observable en este equipo: si un `sh` que no sea de MSYS (por ejemplo WSL) carga con `.` un `.env` CRLF sacado de un checkout de Windows, los valores llevarían `\r` y `run-local.sh` fallaría. Si el humano lo quiere evitar, la regla correspondiente en la raíz sería `.env.example text eol=lf`. No se registra como PLAT-IV porque no se ha observado ningún fallo.

### Ruta larga de Windows (MAX_PATH) — hallazgo para Documentation

El primer clon, creado en el scratchpad (ruta base de unos 125 caracteres), falló así:

```text
error: unable to create file ticketing/bootstrap/src/test/java/com/nequi/ticketing/architecture/fixture/infrastructure/adapter/in/scheduler/ForbiddenSchedulerFixture.java: Filename too long
fatal: unable to checkout working tree
```

- La ruta versionada más larga mide 141 caracteres.
- Con `core.longpaths` sin definir, en Windows el directorio de destino del clon debe medir unos 117 caracteres como máximo.
- El clon en `C:\Users\zombr\AppData\Local\Temp\pi7\clone` (44 caracteres) funcionó.
- No es un defecto de Platform y las rutas pertenecen al backend. Ver el handoff a Documentation (§8).

### Digest de imágenes externas (R-15) — CONFIRMED

`docker buildx imagetools inspect` (solo lectura del registro) de las 5 referencias externas: los 5 `tag@sha256` existen y **el tag sigue apuntando al mismo digest fijado**.

| Imagen | Digest fijado |
|---|---|
| `eclipse-temurin:25.0.4.1_1-jdk-noble` | `589ff4cc…80b4` |
| `eclipse-temurin:25.0.4.1_1-jre-noble` | `d9a39a23…19ab` |
| `amazon/aws-cli:2.37.9` | `92de7572…0780` |
| `amazon/dynamodb-local:3.3.1` | `ff89bd48…0dab` |
| `localstack/localstack:4.14.0` | `3ebc3759…364a` |

### PLAT-IV-011 — Pausa y reinicio de emuladores — CONFIRMED en el entorno integrado

Probado en el clon con `api` y `worker` en marcha:

- **Pausa.** `docker compose pause localstack`: tras unos 12 s la salud de `localstack` pasa a `unhealthy` y `ticketing-api` `/readyz` sigue en 200. Con `unpause` vuelven las 4 colas, `localstack` recupera `healthy`, y `api`/`worker` siguen `healthy` con 0 reinicios.
- **Reinicio.** `docker compose restart localstack` deja **0 colas** (`list-queues` → `None`).
- **Recuperación.** `docker compose up -d --wait infra-init` + `docker compose wait infra-init` tarda 13 s y termina con exit 0. Las 4 colas se recrean con "matches messaging v2". No se recrearon `api` ni `worker` (mismo ID de contenedor), y la colección newman posterior pasó: 30 solicitudes, 42 aserciones, 0 fallos.

## 4. Verification

### 4.1 Preparación

Se registraron los sha256 y los image ID iniciales. Las 4 imágenes propias se protegieron con etiquetas temporales. Después:

- **Proyecto real.** `docker compose --env-file .env --profile load down -v --remove-orphans` (autorizado) terminó con rc 0 en 2,8 s y dejó 0 contenedores, 0 volúmenes y 0 redes del proyecto.
- **Imágenes propias.** Con `docker rmi ticketing-platform/{ticketing,payment-mock,local-idp,infra-init}:local` dejó de haber imágenes `ticketing-platform/*` con nombre, como en una máquina limpia.

### 4.2 Estáticas (CODE_REPO, clon y `git archive`)

| Comprobación | Resultado |
|---|---|
| `docker compose --env-file .env.example config --quiet` | exit 0 en CODE_REPO, en el clon y en el export |
| `config --quiet` con `.env` y con `--profile load` | exit 0 / exit 0 (en los tres) |
| `pull_policy: never` resuelto | Presente en los 7 servicios con imagen propia |
| `docker build --check` de los 4 Dockerfile | "Check complete, no warnings found" (CODE_REPO LF y clon CRLF) |
| `bash -n platform/verify/environment.sh` | OK. LF en el clon (`attr/text eol=lf`) |
| Barrido de la configuración resuelta con `.env.example` (verificador) | 0 `eyJ`, 0 `PRIVATE KEY`, 0 `AKIA` y 0 apariciones de la clave real. 0 `:latest` en Compose y Dockerfiles. Todas las imágenes externas y los `FROM` con tag + digest |

### 4.3 Clon real → `docker compose up -d --wait` con las imágenes ausentes

El clon es `git clone` de CODE_REPO en `5259066` con la configuración de git de este equipo. Encima se aplicaron los 2 archivos de este incremento: se añadieron al índice **del clon** y se volvieron a extraer con `git checkout`, de modo que recibieron la misma conversión que un checkout real (`docker-compose.yml` CRLF; `environment.sh` LF).

El `.env` del clon se creó desde `.env.example` conservando CRLF, con una API key aleatoria (`secrets.token_hex`, sin imprimir) y `PAYMENT_MOCK_HOST_PORT=18090`; el resto de valores es el del ejemplo.

| Medida | Resultado |
|---|---|
| `docker compose --progress plain up -d --wait` | **exit 0 en 2 min 2,9 s**, con los builds incluidos |
| Intentos de `pull` | **0** (no aparece "pull" en todo el log) |
| Builds | 4 imágenes:<br>• `infra-init` y `local-idp` salieron de la caché con **los mismos ID que los originales** (`b787ede5`, `90c0a973`), porque su contexto es LF igual que en CODE_REPO;<br>• `payment-mock` y `ticketing` se reconstruyeron completas (las fuentes CRLF cambian el contexto), con `mvnw verify` real: 256 y 770 pruebas, 0 fallos |

Orden real (UTC, 2026-10-05):

| Servicio | Inicio | Fin / estado | Imagen |
|---|---|---|---|
| `dynamodb-local` | 15:14:23.29 | `healthy` | `9539afb50673` |
| `localstack` | 15:14:23.56 | `healthy` | `0e9f1067a26f` |
| `local-idp` | 15:14:23.83 | `healthy` | `90c0a973eacd` |
| `payment-mock` | 15:14:24.08 | `healthy` | `b6bf42127674` |
| `infra-init` | 15:14:27.91 (tras los emuladores sanos) | **exited 0** a las 15:14:45.64 | `b787ede57259` |
| `ticketing-api` | 15:14:45.96 (tras `infra-init` completado) | `healthy` | **`772f3f6fd23d`** |
| `ticketing-worker` | 15:14:46.23 (tras `infra-init` completado) | `healthy` | **`772f3f6fd23d`** |

Verificaciones sobre el entorno del clon:

| Verificador | Resultado |
|---|---|
| `bash platform/verify/environment.sh` | **67 passed, 0 failed** (la versión final cuenta 65: las 2 líneas de usuario de los emuladores pasaron a `INFO`) |
| `resources.sh` (dentro de la imagen `infra-init`, red del proyecto) | **0 failed**:<br>• tabla `ticketing` con `PK`/`SK` String, `PAY_PER_REQUEST`, `GSI1`..`GSI4` con claves y proyecciones literales y TTL `ENABLED ttl`;<br>• 4 colas exactas con 60/0/1209600/0, 120/0/1209600/0, 60/20/3600/0 + redrive 5 y 120/20/86400/0 + redrive 5 |
| `bash platform/verify/identity.sh --skip-restart` | **38 passed, 0 failed** en 23 s:<br>• discovery y JWKS desde el host y desde la red;<br>• las 5 identidades y un sujeto arbitrario, verificados criptográficamente;<br>• rechazos;<br>• lote de 1.200 tokens distintos, generado en 874 ms (2,5 s contando el arranque del contenedor), sin tokens en el manifiesto ni en los logs |
| newman de punta a punta (ejecución 1) | **rc 0: 31 solicitudes / 0 fallos, 43 aserciones / 0 fallos**. `Event ENABLED`, `Idempotency-Replayed: true` y `Order CONFIRMED` (en el segundo sondeo; la aserción corregida por el humano ya no falla). Apariciones de la clave en la salida: 0 |
| Perfil `load` | `docker compose --profile load up -d`:<br>• `load-token-generator` termina con exit 0, uid 10001, volumen `load-tokens` en `/tokens` con escritura;<br>• `load-test` arranca después y termina con **exit 1** y el mensaje provisional de PLAT-IV-010, uid 10001, `load-tokens` de solo lectura;<br>• **no se creó `load-tests/`** |
| Pausa y reinicio (PLAT-IV-011) | §3 |
| newman tras reiniciar LocalStack y volver a ejecutar `infra-init` (ejecución 2) | **rc 0: 30 / 0 y 42 / 0** |
| Reinicio sin recreación | `restart ticketing-api ticketing-worker` vuelve a `healthy` en 13,6 s. El segundo `up -d --wait` no recrea nada: 0 "Recreate" y los mismos ID de contenedor |
| Apagado ordenado | `stop ticketing-api ticketing-worker` tarda 4.842 ms; ambos exit 143, `OOMKilled=false`, con las líneas de apagado ordenado en el log |
| `--profile load down -v --remove-orphans` | rc 0 en 2,9 s. Quedan 0 contenedores, 0 volúmenes (incluido `load-tokens`) y 0 redes del proyecto |

### 4.4 Reproducibilidad: build completo sin caché de capas desde el clon

| Medida | Resultado |
|---|---|
| `docker compose --progress plain --profile load build --no-cache` | **exit 0 en 2 min 13,6 s**. Solo se reutilizaron la resolución de las bases fijadas por digest, el `WORKDIR` y el `COPY .mvn/`. Se ejecutaron de nuevo las 770 + 256 pruebas, con 0 fallos |
| `up -d --wait` con esas imágenes | exit 0 en 25,7 s, sin `pull` ni builds |
| `environment.sh` | 65 passed, 0 failed |
| `resources.sh` | 0 failed |
| newman (ejecución 3) | **rc 0: 31 / 0 y 43 / 0**; clave en la salida: 0 |
| `down -v` | 0 contenedores, 0 volúmenes y 0 redes |

**Reproducibilidad de los ID de imagen.**

- `infra-init` y `local-idp` dan el mismo ID cuando el contexto es idéntico y hay caché.
- Las imágenes Java reconstruidas sin caché tienen otro ID. El jar y las capas llevan marcas de tiempo, porque `project.build.outputTimestamp` no está fijado en los POM, que pertenecen al backend.
- La reproducibilidad demostrada es por tanto **funcional**: mismas bases fijadas por digest, mismas pruebas en verde y mismo comportamiento. No es bit a bit.

### 4.5 CODE_REPO real tras el cambio

- **`docker compose --env-file .env up -d --wait`** (imágenes originales, `.env` local con 18090): exit 0 en 26,1 s, 0 `pull` y 0 builds. Todos los servicios usan las imágenes originales.
- **`environment.sh`:** **65 passed, 0 failed**, con `api` y `worker` en `4f8a52fe…`, `payment-mock` en `127.0.0.1:18090→8090` y el resto de puertos por defecto en `127.0.0.1`.
- **Restauración:** después se hizo `down -v` y se volvió a levantar solo con las dependencias (§2).

### 4.6 Medición de memoria (PLAT-IV-012: emuladores sin límite, medidos en INC-007)

`docker stats --no-stream` en el clon, en reposo, después de newman y del lote de tokens:

| Servicio | Memoria | Límite |
|---|---|---|
| `dynamodb-local` | 175,4 MiB | ninguno |
| `localstack` | 125,9 MiB | ninguno |
| `local-idp` | 88,7 MiB | 256 MiB |
| `payment-mock` | 124,9 MiB | 384 MiB |
| `ticketing-api` | 271,6 MiB | 768 MiB |
| `ticketing-worker` | 263,3 MiB | 768 MiB |

En total son unos 1,05 GiB del 7,46 GiB disponible. Los emuladores se mantienen sin límite; su consumo bajo la carga de QA queda por medir (R-08).

## 5. Security checks

- **Secretos.**
  - **Origen:** la API key solo entra por `${PAYMENT_MOCK_API_KEY:?}` desde un `.env` no versionado. En el clon se usó una clave aleatoria nueva y la del `.env` real no se leyó ni se imprimió.
  - **Imágenes:** 0 apariciones de la clave, `eyJ`, `PRIVATE KEY`, `PAYMENT_MOCK_API_KEY=` o `AWS_SECRET_ACCESS_KEY=` en `docker history --no-trunc` y en `Config` de las 4 imágenes propias. Se revisaron tanto las construidas desde el clon (dos builds) como las originales.
  - **Logs y salidas:** en los logs de todos los servicios hay 0 apariciones de la clave, `eyJ`, `Bearer ` o `PRIVATE KEY`. En las 3 salidas de newman la clave aparece 0 veces. Los tokens solo se manejaron en variables o en el volumen efímero.
  - **Repositorio:** `.env` está ignorado; `git status` muestra solo ` M docker-compose.yml` y `?? platform/verify/environment.sh`.
- **Cadena de suministro.**
  - Ningún `:latest`.
  - Las 5 imágenes externas están fijadas con tag y digest, y se confirmó en el registro que el tag apunta al mismo digest.
  - Las 4 imágenes propias tienen `pull_policy: never`: nunca se resuelven contra Docker Hub. Hubo 0 intentos de `pull` con las imágenes ausentes.
- **Contenedores.**
  - Imágenes propias: `Config.User 10001:10001`, PID 1 con uid 10001, `ReadonlyRootfs=true`, `CapDrop=[ALL]` y `no-new-privileges:true`.
  - `dynamodb-local` corre como `dynamodblocal`.
  - `localstack` corre como root. Es la excepción documentada en PLAT-INC-002: solo SQS, sin socket de Docker y ligado a loopback.
  - `restart: no` en todos.
- **Puertos.** Solo los aprobados y todos en `127.0.0.1`: 8000, 4566, 9000, `PAYMENT_MOCK_HOST_PORT`→8090 y 8080/8081 del `api`. `worker` e `infra-init` no publican nada.
- **Datos.** Sin volúmenes ni binds en la topología por defecto; el único volumen es `load-tokens`, del perfil `load`, y se elimina con `down -v`.

## 6. Deviations

1. **Ubicación del clon.** El primer clon, en el scratchpad de la sesión, falló por MAX_PATH (§3). Se repitió en `C:\Users\zombr\AppData\Local\Temp\pi7\clone`, también fuera de CODE_REPO. El directorio parcial del primer intento se eliminó.
2. **El clon incluye los cambios sin commit de este incremento.** Son `docker-compose.yml` y `environment.sh`. Se aplicaron sobre `5259066` mediante el índice del clon temporal (sin commit), para obtener la misma conversión de finales de línea que tendrá el checkout cuando el humano los versione.
3. **Proyecto Compose del clon.** El clon usó el nombre de su archivo (`ticketing-platform`), igual que lo haría un usuario. Para que no coincidiera con el proyecto real, primero se bajó el real (`down -v`, autorizado) y al final se restauró.
4. **Imágenes del CODE_REPO real.** No se reconstruyeron. Se preservaron con etiquetas temporales y se reetiquetaron con sus ID originales, porque sus contextos no cambian en este incremento: solo cambia Compose.
5. **Doble build de `local-idp` con `--no-cache`.** `local-idp` y `load-token-generator` declaran ambos `build` sobre la misma imagen. Con caché producen el mismo ID; con `--no-cache` se construye dos veces y la última se queda con la etiqueta. Es inocuo y no se cambió. Mejora opcional: dejar solo `image` + `pull_policy: never` en `load-token-generator`, como `ticketing-worker`.
6. **Caché de BuildKit** no eliminada (§2).

## 7. Blockers

Ninguno. No hay PLAT-IV nuevos ni pendientes. No se creó `human-review/platform.implementation-review.3.yaml`.

## 8. Handoff to other agents

### QA / Resilience

- **Arranque.**
  - `cp .env.example .env`; poner cualquier `PAYMENT_MOCK_API_KEY` y, si el 8090 está ocupado, `PAYMENT_MOCK_HOST_PORT=18090`.
  - `docker compose up -d --wait`: unos 2 min la primera vez (incluye 770 + 256 pruebas en el build) y unos 26 s con las imágenes ya construidas.
  - Comprobación de estado: `bash platform/verify/environment.sh` (exit 0 = todo correcto).
- **Verificadores disponibles:**

  | Script | Comprueba |
  |---|---|
  | `platform/verify/environment.sh` | Topología, seguridad y puertos |
  | `platform/verify/resources.sh` | Tabla, índices, TTL y colas |
  | `platform/verify/identity.sh` | Emisor, cinco identidades y lote de tokens |
  | `platform/verify/infra-init-negative.sh` | Incompatibilidades, en un proyecto aislado |

- **Endpoints y estado esperado:**
  - API en `http://localhost:${TICKETING_API_HOST_PORT:-8080}/api/v1`; `/readyz` y `/livez` en 8080; gestión en `:8081/actuator/{health,prometheus}`.
  - `worker` solo es accesible desde la red, en `ticketing-worker:8080/8081`.
  - Tokens: `POST http://localhost:9000/token` con `identity=<admin|customer-a|customer-b|admin-customer|no-groups>` o con `sub=…&groups=CUSTOMER`.
- **Escenario "cola detenida" (ADR-038 escenario 3, PLAT-IV-011):**
  - `docker compose pause localstack` / `unpause` conserva las colas. Con la pausa, la salud de `localstack` pasa a `unhealthy` en unos 12 s, pero `api` `/readyz` sigue en 200.
  - Un `restart` de `localstack` o `dynamodb-local` borra colas o tabla. Remedio: `docker compose up -d --wait infra-init && docker compose wait infra-init`; tarda unos 13 s y no recrea `api`/`worker`.
- **Perfil `load`:**
  - `docker compose --profile load up -d` genera 1.200 tokens `CUSTOMER` distintos en el volumen `load-tokens` (`/tokens/customer-tokens.csv` con cabecera `sub,access_token` y `/tokens/manifest.json`), con vigencia de 7.200 s (`LOAD_TOKEN_TTL_SECONDS`).
  - Reiniciar `local-idp` invalida el lote: hay que regenerarlo.
  - `API-004` admite 10 solicitudes por 10 s por sujeto.
- **QA debe entregar para `load-test`** (PLAT-IV-010 a, PLAT-IV-016 a):
  - la carpeta `load-tests/` con scripts y umbrales;
  - la imagen de la herramienta, fijada por tag + digest;
  - `entrypoint`/`command` y variables (URL base `http://ticketing-api:8080/api/v1` y ruta del lote `/tokens/customer-tokens.csv`);
  - el montaje `./load-tests:/load-tests:ro`, que se añade al servicio cuando exista la carpeta;
  - la duración más la rampa, si superan 7.200 s.
  - Las dependencias (`ticketing-api` sano y `load-token-generator` completado) y el endurecimiento ya están declarados. Hoy el servicio termina con exit 1 a propósito.
- **Memoria en reposo:** unos 1,05 GiB en total (§4.6). Medir los emuladores bajo carga (R-08).
- **Exclusión con `run-local.sh`:** `docker compose up` (stack completo) y `run-local.sh start` no pueden usarse a la vez, por los puertos 8080/8081 y por consumir las mismas colas.

### Documentation

- **Clonado en Windows (nuevo).**
  - Clonar en una ruta corta, de no más de unos 117 caracteres, o activar `git config --global core.longpaths true`. La ruta versionada más larga es de 141 caracteres (fixtures de pruebas de arquitectura del backend) y el clon falla con "Filename too long".
  - Con `core.autocrlf=true` el checkout deja CRLF en Compose, en los Dockerfile de `ticketing/`/`payment-mock/` y en `.env.example`. Está verificado que funciona igual.
- **Verificación del entorno:** `bash platform/verify/environment.sh` (requiere Git Bash en Windows).
- **Comportamiento de las imágenes:** las imágenes propias nunca se descargan (`pull_policy: never`). Si faltan, `docker compose up` las construye. `docker compose build --no-cache` reconstruye todo en unos 2 min 15 s.
- **Sin cambios respecto de PLAT-INC-006:** variables de `.env.example`, puertos y comandos de parada. El README del humano ya documenta las opciones A y B.

### Cloud / IaC

Platform no implementa IaC. Definiciones literales que se mantienen equivalentes (aws-target v2 §11 #5–#6):

- **Tabla:**
  - `platform/infra-init/resources/table.json` (creación);
  - `platform/infra-init/resources/table.expected` (comparación): `PK`/`SK` String, `PAY_PER_REQUEST`, `GSI1` INCLUDE de 12 atributos, `GSI2` INCLUDE `row`, `seat`, `section`, `ticketId`, `GSI3`/`GSI4` KEYS_ONLY y TTL `ttl`.
- **Colas:** `platform/infra-init/resources/queues.def`, con las 4 colas Standard: visibilidad, espera, retención, retardo 0 y redrive `maxReceiveCount` 5 hacia su DLQ.
- **Diferencias locales (PLAT-IV-013 a):**
  - sin PITR, cifrado ni `RedriveAllowPolicy`;
  - cuenta `000000000000` y URL de LocalStack por ruta;
  - en AWS, las URL llegan por `TICKETING_SQS_ORDERS_QUEUE_URL` y `TICKETING_SQS_PROVISIONING_QUEUE_URL`.
- **Contrato de ejecución de la imagen `ticketing`:**
  - una sola imagen; rol por `TICKETING_ROLE=api|worker`;
  - uid 10001; compatible con raíz de solo lectura y `/tmp` en tmpfs;
  - salud `GET :8080/readyz`;
  - gestión en 8081, no pública;
  - apagado: `SIGTERM` con hasta 35 s por fase, por lo que el tiempo de parada de la tarea debe ser de al menos 40 s;
  - memoria: 768 MiB de límite local y unos 270 MiB medidos en reposo.
- La API key del mock y las credenciales no forman parte de la imagen.

### Backend (opcional, no bloquea)

- **Rutas largas:** acortar las rutas de los fixtures de pruebas de arquitectura (141 caracteres) para que el repositorio pueda clonarse en rutas más profundas sin `core.longpaths`.
- **Builds reproducibles:** si se quieren bit a bit, fijar `project.build.outputTimestamp` en los POM.

### Humano

- **Versionar** `docker-compose.yml` y `platform/verify/environment.sh`.
- **Opcional:** añadir `.env.example text eol=lf` en la raíz `.gitattributes`, solo si se quiere soportar `run-local.sh` con un `sh` que no sea de MSYS (por ejemplo WSL) sobre un checkout de Windows. En este equipo no hace falta.

## 9. Checklist de cierre (§13 del contrato)

| Criterio | Evidencia |
|---|---|
| Dos imágenes independientes construidas | `ticketing` desde `ticketing/` y `payment-mock` desde `payment-mock/`, desde el clon, con y sin caché |
| `api` y `worker` con la misma imagen | Mismo ID en el clon (`772f3f6f`), con `--no-cache` y en CODE_REPO (`4f8a52fe`) |
| `infra-init` idempotente y exacto | `resources.sh` con 0 fallos; volvió a ejecutarse tras el reinicio de LocalStack y en cada `up` con exit 0 |
| `local-idp` con su contrato y las cinco identidades | `identity.sh`: 38/0 |
| Más de 1.000 tokens `CUSTOMER` | 1.200 distintos en 874 ms |
| Entorno limpio levantado en orden | §4.3 |
| Todo sano o completado | `environment.sh` 65–67/0 |
| Datos efímeros | 0 volúmenes por defecto |
| Handoffs a QA y Documentation | §8 |

Estado global de Platform: **LOCAL_PLATFORM_COMPLETE**. Lo único pendiente es el arnés de QA para `load-test`, que no es de Platform.
