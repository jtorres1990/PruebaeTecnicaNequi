---
artifact: documentation-implementation-plan
schema_version: 1.0
feature: ticketing-event-processing
version: 1

agent:
  name: documentation-engineer
  version: 1.0
  mode: planning

status: READY_FOR_HUMAN_PLAN_REVIEW
generated_at: 2026-10-05

repositories:
  spec_repo: "D:\\Nequi\\PruebaeTecnicaNequi (main ac1d406)"
  code_repo: "D:\\Nequi\\ticketing-platform (main 13050c6, árbol limpio)"

preconditions_verified:
  architecture: "architecture/ticketing.architecture.v2.md -> status READY_FOR_DEVELOPMENT, development_can_start true"
  architecture_review: "human-review/ticketing.architecture-review.yaml -> review.status APPROVED, gate.pending_blocking_items []"

sources:
  normative:
    - requirements/Prueba2026.md
    - feature-spec/ticketing.feature-spec.v5.md
    - human-review/ticketing.functional-review.yaml
    - human-review/ticketing.architecture-review.yaml
    - architecture/ticketing.architecture.v2.md
    - architecture/adr/ticketing.adr-registry.v1.md
    - architecture/ticketing.data-model.v2.md
    - architecture/ticketing.messaging.v2.md
    - architecture/ticketing.openapi.v2.yaml
    - architecture/payment-mock.openapi.v1.yaml
    - architecture/ticketing.aws-target.v2.md
    - architecture/ticketing.consolidation-addendum.v1.md
    - "ADR ACCEPTED: ADR-003, ADR-008, ADR-022 a ADR-040 (leídos para este plan: ADR-036, ADR-038; el resto se lee en DOC-INC-004)"
  execution_and_evidence:
    - implementation/ticketing.local-environment.v1.md
    - implementation/ticketing.implementation-plan.v1.md
    - implementation/increments/INC-001..INC-010.report.md (INC-011 no ejecutado)
    - implementation/payment-mock.implementation-plan.v1.md
    - implementation/payment-mock-increments/PM-INC-001..PM-INC-006.report.md
    - implementation/platform.implementation-plan.v1.md
    - implementation/platform-increments/PLAT-INC-001..PLAT-INC-007.report.md (incluye PLAT-INC-005.r2)
    - "human-review/ticketing.implementation-review.{1..7}.yaml, payment-mock.implementation-plan-review.yaml, platform.implementation-plan-review.yaml, platform.implementation-review.{1,2}.yaml"
  code_repo_inspected_read_only:
    - README.md, run-local.sh, .env.example, .gitignore, .gitattributes, docker-compose.yml
    - postman/ticketing.postman_collection.json, postman/ticketing-local.postman_environment.json
    - platform/verify/*.sh (cabeceras de uso)
    - ticketing/**/contracts/*.openapi*.yaml y payment-mock/**/contracts/*.yaml (copias verificadas por SHA-256)
  reference_only_not_modified:
    - documents/documento-arquitectura-ticketing.docx
    - presentations/arquitectura-ticketing-prueba-tecnica.pptx
    - visualizations/ticketing-architecture-blueprint.drawio
    - visualizations/dynamodb-partitions-gsi.html

counts:
  increments: 6
  documentation_validations: 8
  blocking_documentation_validations: 1
  doc_issues: 9
  missing_evidence_items: 12

gate:
  review_artifact: human-review/documentation.implementation-plan-review.yaml
  blocking_items: [PLAN, DOC-IV-001]
  implementation_can_start: false
---

# Plan de implementación de la documentación — Ticketing Event Processing Platform

Este plan define cómo producir `DEL-002` (README) y `DEL-004` (colección de solicitudes) y la documentación de apoyo, a partir de las fuentes vigentes y de la evidencia real de Backend, Payment Mock y Platform. No contiene documentación entregable. En esta invocación no se escribió en `CODE_REPO`, no se levantó el entorno y no se ejecutaron solicitudes.

Instrucción humana vinculante incorporada: **el entregable incluye dos README equivalentes, uno en español y otro en inglés** (§3.2 y `DOC-IV-001`).

---

## 1. Scope and audiences

### 1.1 Audiencias y lo que necesita cada una

| Audiencia | Necesita, en este orden | Punto de entrada |
|---|---|---|
| Quien instala y ejecuta | Requisitos con versiones verificadas → `.env` → `docker compose up -d --wait` → comprobación de salud → parada y limpieza → problemas conocidos | README §5 a §8 y §22 |
| Quien demuestra los flujos | Tokens → reinicio y configuración del Payment Mock → flujos principales con la colección o curl → escenarios de rechazo, fallo, idempotencia y seguridad → qué no se puede demostrar y por qué | README §9 a §12, `docs/collection-guide.md`, `docs/demo-guide.md` |
| Quien evalúa | Qué está implementado y verificado y qué no → decisiones con su alternativa descartada y su consecuencia → concurrencia, idempotencia, seguridad y resiliencia → limitaciones y cambios para producción → topología AWS diseñada (no implementada) | README §2, §14 a §20, `docs/architecture.md`, `docs/diagrams.md`, `docs/traceability.md` |

### 1.2 En alcance

- `CODE_REPO/README.md` y `CODE_REPO/README.en.md` (nombres sujetos a `DOC-IV-001`).
- `CODE_REPO/docs/**` (siete documentos, §3.3).
- `CODE_REPO/postman/**` (evolución de la colección y del environment existentes).
- `CODE_REPO/examples/**`: no se prevé crear nada. Solo se usaría si el humano lo aprueba en una revisión posterior.
- En `SPEC_REPO`: este plan, su revisión, los informes `implementation/documentation-increments/DOC-INC-NNN.report.md` y, si hiciera falta, `human-review/documentation.implementation-review.<n>.yaml`.

### 1.3 Fuera de alcance

Código, pruebas, POM y wrappers; `docker-compose.yml`, Dockerfile, `.env.example`, `.gitignore`, `.gitattributes`; `run-local.sh` y `platform/**`; contratos OpenAPI, ADR y especificación; Terraform y AWS; ejecución de la carga y certificación E2E o de resiliencia; PPTX; publicación externa; commit y push. Los defectos ajenos se registran como `DOC-ISSUE-*` (§12).

---

## 2. Evidence inventory

### 2.1 Estado real por agente

| Agente | Estado | Evidencia | Restricción para la documentación |
|---|---|---|---|
| Backend `ticketing` | INC-001 a INC-010 `DONE`. **INC-011 (cierre) no ejecutado** | `INC-010.report.md` §3: `clean verify -Pintegration` en verde, 129 `*IT`, cobertura agregada **99,46 %** (5.498/5.528 líneas, JaCoCo, solo pruebas sin contenedores) | No afirmar `IMPLEMENTATION_COMPLETE`. Citar cifras solo con su informe y fecha |
| Payment Mock | `PAYMENT_MOCK_COMPLETE` | `PM-INC-006.report.md`: 256 pruebas en verde (3 builds), API-101 a API-111 cubiertas, handoff §7 | Tolerancia PM-IV-016 en `API-103`/`API-104` mientras el Architect no publique la versión 1.0.1 del contrato |
| Platform | `LOCAL_PLATFORM_COMPLETE` | `PLAT-INC-007.report.md`: `docker compose up -d --wait` desde un clon limpio con checkout CRLF (exit 0, 2 min 3 s); `environment.sh` 65/0; `resources.sh` 0 fallos; `identity.sh` 38/0; newman de punta a punta en verde | `load-test` falla a propósito hasta que QA entregue su arnés |
| QA / Resilience | **No ejecutado** | Ninguna | E2E, carga (AC-029 a AC-031) y escenarios de ADR-038 se documentan como procedimientos con estado `NOT_YET_VALIDATED_BY_QA` |
| Cloud / IaC | No ejecutado | `ticketing.aws-target.v2.md` §11 (handoff de diseño) | Terraform y AWS se presentan como `Designed, not implemented` |

### 2.2 Inventario de afirmaciones

Etiquetas de estado (§3.1 del contrato del agente): **IV** = Implemented and verified; **IP** = Implemented, verification pending; **DN** = Designed, not implemented; **OD** = Optional / differential; **OS** = Out of scope; **KL** = Known limitation.

| # | Componente o afirmación | Fuente normativa | Propietario | Estado | Informe o comando que lo verifica | Dónde se documenta |
|---:|---|---|---|---|---|---|
| 1 | Operaciones `API-001` a `API-006` | OpenAPI v2 | Backend | IV | INC-008 (pruebas web validadas contra el contrato), INC-010 (`RoleComponentIT`), newman en PLAT-INC-006/007 | README §11, colección 02–08 |
| 2 | Consumidores y procesos periódicos del `worker` | ADR-024 a ADR-029 | Backend | IV dentro de la JVM; IP en Compose | INC-009, INC-010 (`RoleComponentIT`, `ProvisioningFailureIT`, `ProvisioningCapacitySpikeIT`) | README §3, `docs/architecture.md` |
| 3 | Cobertura ≥ 90 % | ADR-038, AV-006 | Backend | IV (99,46 %) | INC-010 §3; se reejecuta `./mvnw verify` en DOC-INC-002 | README §13 |
| 4 | Pruebas de integración `-Pintegration` | ADR-038 | Backend | IV (129 `*IT`) | INC-010 §3 | README §13 |
| 5 | Cierre del backend (INC-011) | Plan del backend | Backend | No ejecutado | — | README §2 (`DOC-ISSUE-005`) |
| 6 | Payment Mock `API-101` a `API-111` | Payment Mock OpenAPI v1 | Payment Mock | IV | PM-INC-006 §2 y §4 | README §10, colección 01, 05, 06, 09 |
| 7 | Esquema `OutcomeRule` insatisfacible para un validador estricto | Payment Mock OpenAPI v1 | Architect | KL | PM-INC-001 §3 (PM-IV-016) | README §19, `docs/collection-guide.md` (`DOC-ISSUE-001`) |
| 8 | Topología Compose: 7 servicios y perfil `load` | ADR-036 | Platform | IV | PLAT-INC-007 §4.3 | README §4, §7 |
| 9 | Tabla, `GSI1` a `GSI4`, TTL y cuatro colas creadas por `infra-init` | ADR-036, data model v2, messaging v2 | Platform | IV | `resources.sh` (PLAT-INC-007) | README §8 |
| 10 | `local-idp`, cinco identidades y sujetos arbitrarios | ADR-033 | Platform | IV | `identity.sh` 38/0 (PLAT-INC-007) | README §9 |
| 11 | Lote de más de 1.000 tokens `CUSTOMER` | ADR-033, ADR-036 | Platform | IV (1.200 tokens) | PLAT-INC-007 §4.3 | `docs/operations.md` |
| 12 | Servicio `load-test` | ADR-036, ADR-038 | QA | No implementado (provisional, exit 1) | PLAT-INC-007 §4.3 | README §2, §18 (`DOC-ISSUE-006`) |
| 13 | Colección Postman de 30 solicitudes | DEL-004, AV-004 | Documentation (base creada por el humano) | IV como flujo feliz, con defectos documentales (§2.3) | newman: 30 solicitudes, 42 aserciones, 0 fallos (PLAT-INC-006/007; 31/43 cuando el sondeo repite una solicitud) | Colección y `docs/collection-guide.md` |
| 14 | `run-local.sh` (opción B) | PLAT-IV-015, platform.implementation-review.1 | Humano | IP | Usado por Platform para levantar dependencias (PLAT-INC-007 §2); `smoke` se reverifica en DOC-INC-002 | README §21, `docs/operations.md` |
| 15 | Escenarios de resiliencia 1 a 3 | ADR-038 | QA | `NOT_YET_VALIDATED_BY_QA` | Solo evidencia parcial de Platform sobre pausa y reinicio de LocalStack (PLAT-INC-007 §3) | README §18, `docs/resilience.md` |
| 16 | Carga AC-029 a AC-031 | ADR-038 | QA | No ejecutado | — | README §2, §19 |
| 17 | E2E sobre Compose de MF-001 a MF-004 | ADR-038 | QA | No certificado (solo newman de humo) | — | README §2 |
| 18 | Logs ECS, métricas y trazas | ADR-037, aws-target v2 §7 | Backend | IV con huecos | INC-010 SPK-024, §6.3; huecos en §6.1.6 | README §17, `docs/operations.md` (`DOC-ISSUE-004`) |
| 19 | JWT, roles, propiedad, límite de cuerpo y límite por sujeto | ADR-032 | Backend | IV | INC-008 (SPK-020, SPK-023) | README §16, colección 08 |
| 20 | Apagado ordenado | ADR-037 | Backend y Platform | IV | INC-010 SPK-024; PLAT-INC-007 §4.3 (4,8 s) | README §7 |
| 21 | Endurecimiento de imágenes, digests fijados, barrido de secretos | ADR-032, ADR-036 | Platform | IV | PLAT-INC-007 §5 | README §16 |
| 22 | Topología AWS objetivo | ADR-037, aws-target v2 | Architect (diseño) | DN | — | README §20, `docs/architecture.md` |
| 23 | Terraform | EVAL-011 | Cloud / IaC | DN (no implementado) | — | README §2, §20 |
| 24 | Alternativa de LocalStack con token no versionado | ADR-036 | Platform | No soportada por Compose (imagen fijada por digest, sin variable de token) | Inspección de `docker-compose.yml` | `docs/operations.md` (`DOC-ISSUE-007`, `DOC-IV-008`) |
| 25 | Clonado en Windows: ruta versionada de 141 caracteres | — | Backend | KL | PLAT-INC-007 §3 | README §5, §22 (`DOC-ISSUE-002`) |
| 26 | `customerRef` limitado a 128 caracteres en el mock; `sub` sin límite en `ticketing` | Payment Mock OpenAPI v1 | Backend / Architect | KL | Inspección del contrato y del adaptador `PaymentWireFormat` | README §19 (`DOC-ISSUE-003`) |

### 2.3 Documentación existente en `CODE_REPO` y tratamiento propuesto

Toda la documentación existente la creó el humano (commits `bf5ebc6`, `0e52ae2`, `5259066`, `13050c6`). Se preserva lo correcto y se corrige o mueve el resto. No se elimina nada sin que su contenido correcto quede en el nuevo lugar.

#### `README.md` (inglés, 98 líneas)

| Contenido existente | Correcto | Acción |
|---|---|---|
| Descripción y tabla de carpetas | Sí | Se conserva en README §1 y §4 (ambos idiomas) |
| Requisitos (JDK 25, Docker con Compose v2, `sh` y `curl`) | Parcial: la opción A solo requiere Docker, porque el build de la imagen ejecuta las pruebas | Se precisa por opción y se añaden las versiones verificadas (DOC-INC-002) |
| Nota de rutas largas en Windows | Sí | Se conserva en §5 y §22; se enlaza `DOC-ISSUE-002` |
| `cp .env.example .env` y `*_HOST_PORT` | Sí | Se conserva y se añade el equivalente PowerShell |
| Opción A (Compose) | Sí | Pasa a §7 como inicio rápido. `down -v` se presenta como limpieza explícita y `down` como parada |
| `sh platform/verify/environment.sh` | No: el script es `#!/usr/bin/env bash` y Platform lo documenta con `bash` | Se corrige a `bash platform/verify/environment.sh` |
| Opción B (`run-local.sh`) y advertencia de exclusión | Sí | Pasa a §21 (resumen) y a `docs/operations.md` (detalle) |
| Llamadas manuales con curl | Sí, pero incompletas: crean un Event y no lo siguen hasta `ENABLED` ni compran | Se amplían a MF-001 a MF-004 con sondeo explícito (DOC-INC-003) |
| Tabla de operaciones y roles | Sí | Se amplía con `API-xxx`, códigos principales y enlace al contrato |
| Enlace al contrato en `ticketing/infrastructure/src/test/resources/contracts/ticketing.openapi.v2.yaml` | Sí: es una copia verificada por SHA-256 | Se conserva y se añade la copia del contrato del mock (§10.4) |
| Sección Postman con `npx newman run …` | Parcial: `npx` sin versión resuelve la última de npm, que no se ha verificado | Se sustituye por `newman run …` con la versión verificada, y se presenta la interfaz de Postman como vía principal |
| Salud y métricas | Sí | Se conserva en §8 y §17 |
| Build y pruebas | Sí | Se conserva en §13 con cifras de evidencia |
| Configuración | Sí | Se amplía en §6 y en la tabla completa de `docs/operations.md` |

Destino del texto inglés: se convierte en la base de `README.en.md` (§3.2). `README.md` pasa a ser la versión en español con el mismo índice.

#### `postman/ticketing.postman_collection.json` (30 solicitudes, 6 carpetas)

| Hallazgo | Tipo | Acción en DOC-INC-003 |
|---|---|---|
| Cinco descripciones citan IDs de operación equivocados: aprovisionamiento como `API-002` (es `API-006`), listado como `API-003` (es `API-002`), disponibilidad como `API-004` (es `API-003`), compra como `API-005` (es `API-004`), consulta de Order como `API-006` (es `API-005`) | Defecto documental propio | Corregir según `x-id` del OpenAPI v2 |
| El sondeo de la Order considera terminales `CONFIRMED`, `FAILED` y `EXPIRED`, pero no `REJECTED` | Defecto propio | Usar el enum completo de `OrderStatus` terminal |
| La carpeta 5 dice que con `declinePercentage=100` la Order queda `FAILED` | Defecto propio: un rechazo produce `REJECTED` con `PAYMENT_DECLINED` (ST-008, AC-020) | Corregir y convertirlo en el escenario 05 con regla explícita |
| El environment versiona un valor en `paymentMockApiKey` (`type: secret`) | Contradice la regla "ninguna variable secreta tiene valor inicial versionado" | Dejar el valor vacío y explicar cómo cargarlo desde `.env` |
| La descripción de la colección solo menciona `./run-local.sh start` | Incompleta | Mencionar la opción A como principal |
| El control del Payment Mock está al final y sin reglas (`API-103` a `API-106`) ni inspección de cancelaciones (`API-109`) | Incumple AV-004 (configurar el mock antes de cada escenario) | Reordenar a 00–09 (§5) |
| Los tokens quedan en variables de colección tras la ejecución | Riesgo de exportarlos | Añadir limpieza final de variables de token |
| Aserción `Order CONFIRMED` dentro de la rama `else` | Ya corregido por el humano en `5259066` | Se conserva |
| 30 solicitudes, compra feliz, repetición idempotente, casos 400/401/403/404/409/422 | Correctos y validados con newman | Se conservan, renumerados en las carpetas nuevas |

#### Otros archivos

| Archivo | Observación | Acción |
|---|---|---|
| `postman/ticketing-local.postman_environment.json` | `baseUrl`, `idpUrl` y `paymentMockUrl` correctos para los puertos por defecto | Se conserva. Se vacía la clave y se añaden variables de sondeo (§5.3) |
| `run-local.sh` | Fuera de alcance de escritura. El bucle de `smoke` no trata `REJECTED` como terminal; usa `API_PORT`/`WORKER_PORT` no documentados en `.env.example` | Solo se documenta lo que hace. `DOC-ISSUE-008` |
| `.env.example` | Fuente de las variables de Platform, sin secretos | Se documenta sin copiar valores sensibles |
| `.run/` | Salida local de `run-local.sh`, ignorada por Git | Se menciona en §21 |
| Material del humano en `SPEC_REPO` (docx, pptx, drawio, html) | Referencia | No se modifica. Su enlace depende de `DOC-IV-006` |

### 2.4 Comandos, puertos, salud y variables reales (inspección sin ejecución)

| Elemento | Valor real | Fuente |
|---|---|---|
| Arranque completo | `docker compose up -d --wait` (raíz) | PLAT-INC-007 §4.3 |
| Parada y limpieza | `docker compose stop`; `docker compose down`; `docker compose down -v` (también elimina el volumen `load-tokens` del perfil `load`) | PLAT-INC-006 §8, PLAT-INC-007 |
| Reconstrucción | `docker compose build --no-cache` (unos 2 min 15 s) | PLAT-INC-007 §8 |
| Perfil de carga | `docker compose --profile load up -d` | PLAT-INC-007 §8 |
| Verificadores | `bash platform/verify/environment.sh [--env-file f] [-p proyecto]`; `docker compose run --rm --no-deps -v ./platform/verify:/verify:ro --entrypoint sh infra-init /verify/resources.sh`; `bash platform/verify/identity.sh --skip-restart` (sin la opción reinicia `local-idp` e invalida los tokens); `bash platform/verify/infra-init-negative.sh` (proyecto aislado) | Cabeceras de los scripts |
| Recuperación tras reiniciar un emulador | `docker compose up -d --wait infra-init && docker compose wait infra-init` | PLAT-IV-011, PLAT-INC-007 §3 |
| Puertos de host, todos en `127.0.0.1` | 8000 DynamoDB Local, 4566 LocalStack, 9000 `local-idp`, `PAYMENT_MOCK_HOST_PORT` (8090 por defecto) Payment Mock, 8080 API, 8081 gestión. El `worker` no publica puertos | `.env.example`, `docker-compose.yml` |
| Salud | API: `GET :8080/readyz` y `/livez`; gestión: `GET :8081/actuator/health` y `/actuator/prometheus`; mock: `GET /health` sin API key; IdP: `/.well-known/openid-configuration`, `/.well-known/jwks.json` | INC-010 §6.5.4, PLAT-INC-003/004 |
| Tokens | `POST http://localhost:9000/token` con `identity=<admin|customer-a|customer-b|admin-customer|no-groups>` o `sub=<id>&groups=CUSTOMER[,ADMIN]&expires_in=<s>`; `iss` siempre `http://local-idp:9000` | PLAT-INC-003 §8 |
| Límite por sujeto | `API-004`: 10 solicitudes por 10 s y sujeto; el filtro actúa antes de leer el cuerpo, así que también cuenta las solicitudes inválidas | INC-008 §1 |
| Build | `ticketing/`: `./mvnw verify` (sin Docker, con la puerta de cobertura), `./mvnw verify -Pintegration` (Docker); `payment-mock/`: `./mvnw verify` | INC-010 §3, PM-INC-006 §4 |
| Opción B | `./run-local.sh start|smoke|stop|down` | `run-local.sh` |
| Variables | Las de `.env.example` (Platform) y las de INC-010 §6.5.3 (aplicación) | PLAT-INC-001/006, INC-010 |

Herramientas locales disponibles para verificar en la implementación, sin instalar nada: Python 3.13 con `pyyaml` y `jsonschema`; Node 14.21.3; newman 5.3.2 (`node C:/Users/zombr/node_modules/newman/bin/newman.js`); Docker 29.8.0 con Compose v5.5.1; Git 2.39.2. **No hay validador de Mermaid local** (§11, punto 10).

---

## 3. Documentation architecture

### 3.1 Principios aplicados

1. Una sola entrada canónica (`README.md`) y una versión inglesa equivalente.
2. Los README cubren todo `DEL-002` y la ruta operativa completa. Lo extenso vive en `docs/**`, enlazado.
3. Los contratos se enlazan, no se copian. Las copias de los OpenAPI que ya existen en el repositorio están verificadas por SHA-256 contra `SPEC_REPO` (`TicketingContractCopyTest`, `PaymentContractCopyTest`, `ContractCopyTest`).
4. Toda afirmación lleva una etiqueta de estado (§2.2). Las etiquetas se escriben en inglés, idénticas en ambos idiomas, con una leyenda traducida.
5. Ninguna afirmación de QA. El estado máximo alcanzable es `DOCUMENTATION_COMPLETE_WITH_PENDING_QA`.

### 3.2 Estrategia bilingüe (instrucción humana; decisión en `DOC-IV-001`, `DOC-IV-002` y `DOC-IV-003`)

**Propuesta recomendada:**

| Aspecto | Propuesta |
|---|---|
| Entrada canónica | `README.md` en **español**: coincide con el contrato del agente (prosa principal en español), con el idioma del enunciado y con el de la reunión técnica. GitHub lo muestra por defecto |
| Versión inglesa | `README.en.md` en **inglés**, construido a partir del README actual del humano |
| Enlace entre ambos | La primera línea de cada archivo es el selector de idioma: `**Español** · [English](README.en.md)` en uno y `[Español](README.md) · **English**` en el otro |
| Precedencia ante una divergencia | Prevalece `README.md`. La divergencia es un defecto que se corrige en el mismo incremento |

**Cómo se garantiza que no diverjan:**

1. **Mismo índice.** Ambos tienen las mismas 23 secciones H2 numeradas (§4) y las mismas subsecciones H3, en el mismo orden. Solo cambia el texto del título.
2. **Mismos comandos.** Todo bloque de código es idéntico byte a byte en ambos archivos y aparece en el mismo orden. Los comentarios dentro de los bloques se escriben en inglés, que es el idioma del código, en los dos README.
3. **Mismos enlaces, IDs y tablas.** Ambos tienen el mismo conjunto de destinos de enlace (salvo el selector de idioma), los mismos IDs citados por sección (`API-*`, `ADR-*`, `MF-*`, `AC-*`, `DEL-*`, `EVAL-*`, `DOC-ISSUE-*`), el mismo número de tablas y de filas por tabla y las mismas etiquetas de estado.
4. **Comprobación de equivalencia.** Un script Python en el scratchpad de la sesión (no se entrega) compara estos cinco aspectos y falla ante cualquier diferencia. Se ejecuta en cada incremento que toque un README y en DOC-INC-006. Su salida resumida va al informe del incremento.
5. **Edición en pareja.** Ningún incremento modifica un README sin modificar el otro en la misma unidad de trabajo.

Las cifras se localizan (`99,46 %` frente a `99.46%`). La comprobación compara IDs y estructura, no la puntuación decimal.

**`docs/**` en un solo idioma (español).** Recomendación de `DOC-IV-002`, con esta justificación:

- Los README contienen en ambos idiomas todo lo exigido por `DEL-002` y la ruta operativa completa: instalación, configuración, Docker, decisiones condensadas con su alternativa y consecuencia, ejemplos y limitaciones. Un lector en inglés puede instalar, ejecutar, demostrar y entender los trade-offs sin leer `docs/`.
- `docs/**` es material de profundidad (unas 7 páginas largas). Traducirlo duplicaría el mantenimiento y el riesgo de divergencia sin añadir cobertura de `DEL-002`.
- El contrato del agente fija el español como idioma de la prosa principal, y la evaluación es en español.
- En `README.en.md`, cada enlace a `docs/` se marca como "(Spanish)".

Alternativas en `DOC-IV-002`: `docs/es/` y `docs/en/` completos, o `docs/` en inglés.

**Colección.** Un único archivo de colección (`DEL-004`). Recomendación de `DOC-IV-003`:

- Nombres de carpeta y solicitud con el ID y un título breve en español: `04.01 API-004 Iniciar compra`.
- Descripciones en español seguidas de una línea `EN:` con su equivalente inglés.
- Nombres de prueba basados en códigos del contrato (`202 Accepted`, `status CONFIRMED`, `code TICKETS_UNAVAILABLE`), legibles en ambos idiomas.
- `docs/collection-guide.md` en español; los README de ambos idiomas resumen su uso.

### 3.3 Mapa de archivos entregables

Nombres de archivo en inglés (regla de rutas del contrato). Contenido según `DOC-IV-002`.

| Archivo | Propósito | Incremento |
|---|---|---|
| `README.md` | Entrada canónica en español | 001 → 006 |
| `README.en.md` | Versión inglesa equivalente | 001 → 006 |
| `docs/traceability.md` | Matriz de cobertura documental (§9) y bitácora resumida de estado de verificación por afirmación | 001, actualizado en cada incremento |
| `docs/operations.md` | Variables completas, puertos, salud, logs, métricas, perfil `load`, opción B, alternativa de LocalStack con token y problemas conocidos en detalle | 002, 005 |
| `docs/collection-guide.md` | Uso de la colección carpeta por carpeta: precondiciones, resultado esperado, tiempos, límite por sujeto y ejecución con newman | 003 |
| `docs/architecture.md` | Guía de arquitectura y decisiones (§6.3 del contrato), seguridad, concurrencia, observabilidad, AWS diseñado, limitaciones y cambios para producción | 004 |
| `docs/diagrams.md` | Diagramas Mermaid derivados de la arquitectura v2 | 004 |
| `docs/demo-guide.md` | Guía para la reunión técnica | 005 |
| `docs/resilience.md` | Procedimientos de ADR-038 con estado `NOT_YET_VALIDATED_BY_QA` | 005 |
| `postman/ticketing.postman_collection.json` | Colección v2.1 reorganizada (mismo nombre) | 003 |
| `postman/ticketing-local.postman_environment.json` | Template de environment sin secretos (mismo nombre) | 003 |

No se crean README dentro de `ticketing/` ni de `payment-mock/`.

### 3.4 Referencias a `SPEC_REPO`

`CODE_REPO` (`github.com/jtorres1990/ticketingNequi`) y `SPEC_REPO` (`github.com/jtorres1990/PruebaeTecnicaNequi`) son repositorios distintos. Los ADR, la arquitectura v2, los OpenAPI originales, los informes de evidencia y el material del humano solo existen en `SPEC_REPO`. La estrategia de enlaces se decide en `DOC-IV-006`. La recomendación es que la documentación de `CODE_REPO` sea autosuficiente (resumen de cada decisión con su ID) y que añada enlaces absolutos a `SPEC_REPO` solo si el humano confirma que los evaluadores tendrán acceso. Nunca se usan rutas locales absolutas.

---

## 4. README outline

Ambos README tienen estos 23 H2, numerados y en este orden. Columna "Contrato": ítem de §6.1 del contrato del agente que cubre.

| # | Español (`README.md`) | English (`README.en.md`) | Contenido | Contrato | Fuente principal | Incr. |
|---:|---|---|---|---|---|---|
| — | Selector de idioma, propuesta de valor en 2–3 frases, mapa de navegación | Same | — | 1 | — | 001 |
| 1 | Propósito y alcance | Purpose and scope | Problema, solución, qué queda fuera (§3.2 de la spec) | 1 | Feature spec v5 §1–§3 | 001 |
| 2 | Estado de implementación y verificación | Implementation and verification status | Tabla por componente con etiqueta de estado y evidencia (informe y fecha). INC-011 pendiente; QA no ejecutado; Terraform no implementado | 2 | §2 de este plan | 001, 006 |
| 3 | Arquitectura en resumen | Architecture at a glance | Diagrama de contenedores (Mermaid de arquitectura v2 §4), una imagen y dos roles, colas, mock independiente | 3 | Arquitectura v2 §1, §4 | 004 |
| 4 | Estructura del repositorio | Repository layout | Tabla de carpetas (contenido existente) | 4 | README actual | 001 |
| 5 | Requisitos previos | Prerequisites | Por opción, con versiones verificadas; rutas largas en Windows | 5 | DOC-INC-002 | 002 |
| 6 | Configuración | Configuration | `.env` desde `.env.example`, variables relevantes con propósito, obligatoriedad y ejemplo seguro; secretos | 6 | `.env.example`, INC-010 §6.5.3 | 002 |
| 7 | Inicio rápido con Docker Compose | Quick start with Docker Compose | `up -d --wait`, tiempos medidos, `ps`, `logs`, `stop`, `down`, `down -v` con su efecto | 8 | PLAT-INC-006/007 | 002 |
| 8 | Verificación del entorno | Environment verification | `readyz`, `livez`, salud del mock e IdP, `environment.sh`, `resources.sh` | 9 | PLAT-INC-007 | 002 |
| 9 | Identidades y tokens locales | Local identities and tokens | Cinco identidades, sujetos arbitrarios, roles, vigencia; reiniciar `local-idp` invalida los tokens | 10 | ADR-033, PLAT-INC-003 | 002 |
| 10 | Control del Payment Mock | Payment Mock control | Reinicio, defaults, reglas, inspección; estado en memoria; sin campos nuevos en la compra (AV-004) | 11 | ADR-030, PM-INC-006 §7.2 | 002 |
| 11 | API y ejemplos | API and examples | Tabla `API-001` a `API-006` (método, ruta, rol, respuestas principales) y curl de MF-001 a MF-004 con sondeo | 12, 13 | OpenAPI v2 | 003 |
| 12 | Colección de solicitudes | Request collection | Importación, variables, orden de carpetas, newman y qué no demuestra | 13 | DEL-004 | 003 |
| 13 | Build y pruebas | Build and tests | Comandos de `ticketing` y `payment-mock`, cobertura con su fuente, qué no sustituye la colección | 7 | INC-010, PM-INC-006 | 002 |
| 14 | Decisiones arquitectónicas y trade-offs | Architecture decisions and trade-offs | Tabla condensada: problema, decisión, alternativa descartada, consecuencia, ADR | 14 | ADR vigentes | 004 |
| 15 | Concurrencia, atomicidad e idempotencia | Concurrency, atomicity and idempotency | Transacción por transición, guardián Order, una Order activa, idempotencia de compra, creación, mensajes y pagos | 15 | ADR-023, 025, 027, 032 | 004 |
| 16 | Seguridad y secretos | Security and secrets | Resource Server, roles, propiedad, límites, secretos, endurecimiento | 16 | ADR-032, PLAT-INC-007 §5 | 004 |
| 17 | Observabilidad y operación | Observability and operations | Logs ECS, métricas, trazas, puerto de gestión, huecos conocidos | 17 | INC-010 §6.3 | 004 |
| 18 | Resiliencia | Resilience | Circuit breakers, DLQ, barrido, expiración; tres escenarios `NOT_YET_VALIDATED_BY_QA` | 18 | ADR-035, ADR-038 | 005 |
| 19 | Limitaciones conocidas | Known limitations | Arquitectura v2 §17 vigente (sin el punto 15, ya superado) y limitaciones locales | 19 | Arquitectura v2 §17 | 004 |
| 20 | Cambios para producción | Production changes | Tabla de §17 y topología AWS diseñada; Terraform no implementado | 20 | aws-target v2 | 004 |
| 21 | Modo desarrollo con JVM locales | Development mode with local JVMs | `run-local.sh` y exclusión con la opción A | — | `run-local.sh`, PLAT-IV-015 | 002 |
| 22 | Solución de problemas | Troubleshooting | Solo fallos observados en informes o en la verificación (§6.4) | 21 | PLAT-INC-005 a 007 | 002, 005 |
| 23 | Referencias | References | Contratos, `docs/`, ADR según `DOC-IV-006` | 22 | — | 004 |

Reglas de redacción: primero el resultado y después el mecanismo; términos de dominio (Event, Ticket, Order, Reservation) definidos en §1; bloques `sh` (Git Bash, Linux o macOS) y `powershell` solo donde difieren (copiar `.env` y cargar variables); comandos con directorio de ejecución; sin rutas personales; sin `latest`; objetivos de carga presentados como objetivos de prueba y nunca como SLA.

---

## 5. Collection design

### 5.1 Formato y evolución (`DOC-IV-004`)

Recomendación: mantener **Postman Collection v2.1** y evolucionar en el lugar los dos archivos existentes. Se conservan los nombres de archivo y las 30 solicitudes validadas, renumeradas en las carpetas 00–09. Los README añaden curl equivalentes solo para MF-001 a MF-004. No se crea una segunda colección.

### 5.2 Carpetas

| Orden | Carpeta | Contenido previsto (operación → resultado esperado según el contrato) | Origen | Ejecución por defecto |
|---:|---|---|---|---|
| 00 | Environment and health | `API-111` → 200; `GET {{apiRootUrl}}/readyz` → 200; generación de `runId`; tokens de las cinco identidades y de sujetos `CUSTOMER` por escenario (`sub=demo-{{runId}}-<escenario>`) | 5 existentes + nuevas | Sí |
| 01 | Payment Mock control | `API-110` → 204; `API-105` → 204; `API-107` `APPROVED`, 0 % → 200; `API-103` → `[]` | 2 existentes + nuevas | Sí |
| 02 | Event provisioning | `API-001` → 202 con `Location` y captura de `eventId`; repetición → 200 con `Idempotency-Replayed: true`; `API-006` sondeado hasta `ENABLED` o `FAILED`; `API-006` con `CUSTOMER` → 403; capacidad distinta de los asientos → 400 `VALIDATION_ERROR`; capacidad 50.001 → 400 | 3 existentes + nuevas | Sí |
| 03 | Catalog and availability | `API-002` incluye el Event con `soldOut` false; `API-003` con `pageSize` pequeño → `nextCursor` → página siguiente; filtro por sección; cursor inválido → 400; sección inexistente → 400; Event inexistente → 404 | 3 existentes + nuevas | Sí |
| 04 | Happy purchase | `API-004` → 201 `CREATED` con captura de `orderId` y `ticketIds`; `API-005` sondeado hasta estado terminal → `CONFIRMED`; `API-108` del intento `<orderId>-1` → `invocations` 1 y `APPROVED`; `API-003` sin esos Ticket | 4 existentes + nueva | Sí |
| 05 | Declined payment | `API-110`; `API-104` `DECLINE` con un único matcher (`ticketId` de un asiento reservado para el escenario) → 201 con `ruleId`; `API-004` → 201; `API-005` → `REJECTED` con `PAYMENT_DECLINED`; Ticket de nuevo en `API-003`; `API-106` → 204 | Nueva | Sí |
| 06 | Failure behaviours | Rápido: `DEFINITIVE_ERROR` → `FAILED` con `PROCESSING_FAILED`, sin cancelación (`API-109` → 404). Lento: `TRANSIENT_THEN_OUTCOME` con 1 fallo y `APPROVED` → `CONFIRMED` tras la reentrega | Nueva | Rápido sí; lento según `DOC-IV-005` |
| 07 | Idempotency | Repetición de compra → 200 con `Idempotency-Replayed`; misma clave y otro contenido → 422 `IDEMPOTENCY_KEY_REUSED` (compra y creación de Event); segunda compra con otra clave mientras la primera sigue en `CREATED` (regla `LATENCY` < 3 s) → 409 `ACTIVE_ORDER_EXISTS` | 1 existente + nuevas | Sí |
| 08 | Security and ownership | Sin token → 401; `no-groups` → 403; `CUSTOMER` crea Event → 403; `admin` sin grupo `CUSTOMER` compra → 403; `admin-customer` compra → 201; Order ajena → 404 igual que inexistente y que identificador mal formado; sin `Idempotency-Key` → 400; Ticket repetido → 400; más de 10 Ticket → 400; Ticket vendido → 409 `TICKETS_UNAVAILABLE`; Ticket de cortesía → 409; Ticket inexistente → 422 `UNKNOWN_TICKETS`; 11 solicitudes rápidas de un sujeto dedicado → 429 `RATE_LIMITED` con `Retry-After` | 9 existentes + nuevas | Sí |
| 09 | Cancellation inspection | MF-005: regla `TRANSIENT_THEN_OUTCOME` con fallos suficientes para agotar las 5 recepciones → `FAILED` con `PROCESSING_FAILED` y reverso; `API-109` → `received` 1 y `REGISTERED_BEFORE_CHARGE`; `API-108` → un único `paymentAttemptId` | Nueva | Según `DOC-IV-005` |
| 99 | Cleanup | Restablece el mock (`API-110`) y borra de la colección las variables de token | Nueva | Sí |

Opcional y lento, sujeto a `DOC-IV-005`: Event con `startsAt` cercano (≈ 20 s) para demostrar `EVENT_NOT_ON_SALE` (AC-037) y la disponibilidad informativa de un Event pasado (AC-038). Solo se incluye si se verifica de forma estable.

No se incluyen solicitudes directas a `API-101` y `API-102` en el flujo principal: el flujo público es `API-004` y el `worker` llama al mock. Si se documentan, van en una subcarpeta técnica separada, identificada como contrato aislado y fuera de la ejecución por defecto.

### 5.3 Variables

| Ámbito | Variable | Valor versionado |
|---|---|---|
| Environment | `baseUrl` (`http://localhost:8080/api/v1`), `apiRootUrl` (`http://localhost:8080`), `idpUrl` (`http://localhost:9000`), `paymentMockUrl` (`http://localhost:8090`) | Valores por defecto de `.env.example` |
| Environment | `paymentMockApiKey` (`type: secret`) | **Vacío**. Se carga desde `.env` (en la interfaz) o con `--env-var` (newman) |
| Environment | `pollIntervalMs` (1000), `maxPolls` (30), `slowMaxPolls` (valor que se calibrará en DOC-INC-003) | No sensibles |
| Colección (en ejecución) | `runId`, tokens, `eventId`, `orderId`, `ticketIds`, `paymentAttemptId`, `ruleId`, claves de idempotencia | Vacíos; la carpeta 99 borra los tokens |

### 5.4 Reglas de diseño

- Cada ejecución crea su propio Event y sus sujetos `CUSTOMER`, así que no depende de ejecuciones anteriores. El mock se aísla con `API-110` antes de cada escenario que lo configura.
- El presupuesto de `API-004` (10 por 10 s y sujeto) se respeta con un sujeto por escenario. Solo el escenario de 429 lo agota.
- Sondeo acotado con `pollIntervalMs` y `maxPolls`. Al agotarse, el mensaje de la prueba dice qué estado se esperaba y cuál se observó. No hay esperas fijas largas.
- Aserciones separadas para la respuesta síncrona (201 `CREATED`) y para el estado final asíncrono (`API-005`).
- Aserciones de forma: campos `required` y enumerados del esquema OpenAPI de cada respuesta, con esquemas mínimos derivados del contrato. Las respuestas de `API-103`/`API-104` se comprueban con combinadores resueltos por `DOC-ISSUE-001`.
- Ningún campo adicional en las solicitudes. El resultado del pago se elige solo con reglas del mock (AV-004).
- `paymentAttemptId` se deriva como `<orderId>-1` según la descripción del parámetro en el contrato del mock. Se explica como inspección técnica, porque `API-005` no lo expone.
- Scripts legibles, sin `eval`, sin descargas y sin hosts distintos de las variables locales.

### 5.5 Cobertura que la colección no puede ofrecer

| Comportamiento | Motivo | Dónde se documenta |
|---|---|---|
| Sobreventa bajo concurrencia (AC-007, AC-029) | Requiere solicitudes simultáneas y verificación de invariantes | README §13 y §15 (pruebas deterministas del backend), §18/§19 (carga QA pendiente) |
| 503 con cola indisponible (AC-046) | Requiere detener o pausar LocalStack | `docs/resilience.md`, escenario 3 |
| Expiración tras 10 minutos (AC-008) | Duración real de la Reservation | Pruebas del backend con reloj inyectado (`RoleComponentIT.expirationReleasesTheTickets`) |
| Aprobación tardía tras `expiresAt` (AC-034, AC-050) | La latencia máxima del mock es 60 s y la Reservation dura 10 min | Pruebas del backend (INC-009/010) |
| 413 con cuerpo de más de 256 KB | Posible con un script que genere el cuerpo; se decide en DOC-INC-003 según su claridad | `docs/collection-guide.md` |

---

## 6. Diagram and demo strategy

### 6.1 Diagramas (`docs/diagrams.md`)

Se reutilizan los diagramas Mermaid de la arquitectura v2. Cada uno indica su sección de origen y no añade nodos ni flechas.

| Diagrama | Origen |
|---|---|
| Contexto del sistema | Arquitectura v2 §3 |
| Contenedores y dependencias | Arquitectura v2 §4 (también en README §3) |
| Creación y aprovisionamiento de Event | Arquitectura v2 §7.8 |
| Compra y pago: flujo feliz y rechazo | Arquitectura v2 §7.1 y §7.3 |
| Indisponible y Order activa existente | Arquitectura v2 §7.2 |
| Expiración | Arquitectura v2 §7.6 |
| Carrera pago y expiración, y reverso | Arquitectura v2 §7.7 y §7.9 |
| Circuit breaker del Payment Mock | Arquitectura v2 §7.11 |
| Topología AWS (diseño) | aws-target v2 §1, rotulado "Designed, not implemented" |

Se traducen solo etiquetas de texto cuando hace falta. Los nombres de servicio, cola e índice se conservan. La validación de sintaxis no es posible con herramientas locales (§11, punto 10). Se hace una revisión manual nodo a nodo contra el original, y el informe lo declara.

### 6.2 Guía de demostración (`docs/demo-guide.md`)

Orden: problema y criterios (2 min) → arquitectura con el diagrama de contenedores → arranque previo (`docker compose up -d --wait` antes de la reunión y `environment.sh`) → flujo feliz (carpetas 00–04) → rechazo determinista (05) → idempotencia y Order activa (07) → seguridad y propiedad (08) → evidencia de pruebas y cobertura (cifras con su informe) → decisiones y trade-offs → limitaciones y producción → preguntas previsibles, cada una con el documento o ADR que la responde.

Plan de contingencia sin Docker: recorrer los diagramas, los curl documentados y los informes de evidencia, sin presentar salidas inventadas. La guía no contiene respuestas memorizadas sin fuente.

### 6.3 Resiliencia (`docs/resilience.md`)

Tres procedimientos de ADR-038, todos con estado **`NOT_YET_VALIDATED_BY_QA`**. Cada uno tiene precondiciones, preparación del mock, comandos, señales que observar (métricas `ticketing_circuit_state`, `ticketing_sqs_failures_total`; logs `circuit.opened`; `API-108`), resultado esperado según ADR-038 y restauración.

1. **Detener `ticketing-worker` a mitad de un pago.** Regla `LATENCY` para mantener el pago en curso. `docker compose stop` hace un apagado ordenado que completa lo que está en vuelo, así que para simular una caída se usa `docker compose kill ticketing-worker`. Después, `docker compose start ticketing-worker`. Al vencer el lease (45 s), se reanuda el mismo `paymentAttemptId`.
2. **Detener `payment-mock`.** Apertura del circuito, pausa del consumo y de los reversos, expiración activa. Restauración con `docker compose start payment-mock`; el estado del mock se pierde.
3. **Detener `localstack`.** Se usa `docker compose pause/unpause localstack`, que conserva las colas (PLAT-IV-011). Respuesta 503 con `Retry-After` sin cambiar el inventario. Tras un `restart`, hay que volver a ejecutar `infra-init`.

Si el humano lo autoriza (`DOC-IV-007`), Documentation ejecuta los comandos solo para comprobar que existen y que las señales se observan. El estado sigue siendo `NOT_YET_VALIDATED_BY_QA`.

### 6.4 Solución de problemas: solo fallos observados

| Síntoma | Origen de la evidencia |
|---|---|
| `Filename too long` al clonar en Windows | PLAT-INC-007 §3 |
| Puerto 8090 ocupado en el host | PM-INC-006 §7.1, PLAT-INC-004 |
| `infra-init` falla y `api`/`worker` no arrancan | PLAT-INC-006 §4.5 |
| Tras reiniciar un emulador desaparecen la tabla o las colas | PLAT-INC-007 §3 |
| `run-local.sh` y Compose completo a la vez: conflicto de puertos y de colas | PLAT-INC-006 §8 |
| Tokens rechazados tras reiniciar `local-idp` | PLAT-INC-003 §8 |
| `.env` con CRLF leído por un `sh` que no es de MSYS (WSL) | PLAT-INC-007 §3 (riesgo, no observado) |
| Compose falla si falta `PAYMENT_MOCK_API_KEY` | `docker-compose.yml` (`${…:?}`) |

Se añaden los fallos que aparezcan durante la verificación, con su fecha.

---

## 7. Verification gates

| Gate | Comprobación | Herramienta | Cuándo |
|---|---|---|---|
| G1 Estático | Enlaces locales de Markdown resolubles (archivos y anclas); JSON de colección y environment parseable; ninguna referencia a archivos inexistentes; ninguna ruta personal (`C:\Users`, `D:\`); sin `latest`; sin versiones v1 de la arquitectura ni ADR `SUPERSEDED` presentados como vigentes; ningún comando destructivo ambiguo | Script Python (scratchpad) | Cada incremento |
| G2 Secretos | Sin `eyJ` (JWT), `AKIA`, `PRIVATE KEY`, cabeceras `Authorization` completas ni el valor real de `PAYMENT_MOCK_API_KEY` (leído de `.env` por el script sin imprimirlo) en ningún entregable ni salida guardada; `paymentMockApiKey` vacío en el template | Script Python | Cada incremento |
| G3 Contrato | Cada solicitud de la colección y cada curl del README corresponde a un `operationId` (método y ruta); cabeceras obligatorias presentes; cuerpos válidos contra el esquema con `additionalProperties: false`; cada status asertado existe en las respuestas de la operación; campos capturados existen en el esquema; enumerados correctos | Python con `pyyaml` y `jsonschema` sobre las copias del contrato en `CODE_REPO` (SHA-256 coincidente con `SPEC_REPO`) | 003, 006 |
| G4 Equivalencia bilingüe | §3.2: índice, bloques de código, enlaces, IDs, tablas y etiquetas | Script Python | Cada incremento que toque los README |
| G5 Ejecutable | `docker compose config --quiet`; `up -d --wait`; `environment.sh`; `resources.sh`; `identity.sh --skip-restart`; curl de tokens, mock y MF-001 a MF-004; `./mvnw verify` en ambos proyectos; `run-local.sh start/smoke/stop` si no coincide con la opción A; newman con las carpetas por defecto y, por separado, las lentas; parada | Docker, Git Bash, JDK 25, newman 5.3.2 | 002, 003, 005, 006 |
| G6 Estados | Ninguna afirmación `PASS`, `QA_PASSED`, `validated` o equivalente sobre E2E, carga o resiliencia; cada comando lleva su estado de verificación; objetivos de carga nunca presentados como SLA | Revisión con script y lectura | 005, 006 |
| G7 Diagramas | Cada nodo y flecha existe en la fuente citada | Revisión manual (sin validador local) | 004 |

Política de evidencia: para cada comando o escenario el informe registra fecha, revisión de `CODE_REPO`, entorno, comando, resultado y limitación. Si algo no se puede ejecutar, se conserva el comando del handoff aprobado, rotulado "no verificado" con su dependencia, sin una salida inventada.

Acciones sobre el entorno que requieren `DOC-IV-007`: `docker compose down -v` (del proyecto `ticketing-platform`), `pause/unpause`, `stop/start/kill` de servicios concretos. Nunca se hace limpieza global de Docker, eliminación de imágenes ajenas ni `builder prune`. El entorno se deja como se encontró: hoy solo las dependencias, levantadas por PLAT-INC-007.

---

## 8. Increments

### DOC-INC-001 — Inventario, estructura, trazabilidad y esqueleto documental

- **Fuentes:** este plan; el README y la colección actuales; feature spec v5 §1–§3; §2 de este plan.
- **Dependencias:** plan aprobado y `DOC-IV-001` respondida.
- **Archivos:** `README.md` (español), `README.en.md` (inglés, a partir del README actual), `docs/traceability.md`.
- **Trabajo:** crear los dos README con los 23 H2 de §4. Ubicar todo el contenido correcto existente en su sección, traducido al español en `README.md`, y aplicar las correcciones de §2.3 que no requieren ejecución (comando `bash`). Las secciones que dependen de incrementos posteriores muestran contenido mínimo veraz y un marcador explícito del tipo "Pendiente de DOC-INC-00N". Ningún comando existente se pierde. Crear la matriz de §9 con estado inicial. Preparar los scripts G1, G2 y G4 en el scratchpad.
- **Validaciones:** G1, G2, G4; ninguna afirmación nueva sin fuente.
- **Terminado cuando:** ambos README son navegables y equivalentes, el contenido correcto del humano está preservado, la matriz existe y el informe registra el mapa "contenido existente → sección nueva".

### DOC-INC-002 — README operativo: requisitos, configuración, Compose, verificación, tokens, mock y build

- **Fuentes:** `.env.example`, `docker-compose.yml`, `run-local.sh`, PLAT-INC-001 a 007, INC-010 §6.5, PM-INC-006 §7, ADR-033, ADR-036.
- **Dependencias:** DOC-INC-001; `DOC-IV-007` (autorización de acciones) y `DOC-IV-008` (alternativa de LocalStack).
- **Archivos:** README §5–§10, §13, §21, §22 (parcial); `docs/operations.md`.
- **Trabajo:** versiones verificadas de los requisitos; tabla de variables (propósito, obligatoriedad, ejemplo seguro) sin copiar valores de `.env`; comandos `sh` y PowerShell donde difieren; inicio, salud, logs, parada y limpieza; tokens; control del mock con curl; build y pruebas; opción B; alternativa de LocalStack según `DOC-IV-008`.
- **Ejecución (G5):** `docker compose config --quiet`; `down -v` del proyecto si está autorizado; `up -d --wait` con tiempo medido; `environment.sh`; `resources.sh`; `identity.sh --skip-restart`; curl de `readyz`, `livez`, `/health` y tokens sin imprimirlos; `API-110` y `API-107` con la clave cargada desde `.env`; `./mvnw verify` en `ticketing/` y `payment-mock/` con JDK 25; `-Pintegration` solo si el humano lo pide, porque tarda unos 4 min; opción B (`start`, `smoke`, `stop`) con la opción A detenida; parada.
- **Validaciones:** G1, G2, G4, G5.
- **Terminado cuando:** cada comando de estas secciones tiene estado de verificación y el entorno queda como se encontró.

### DOC-INC-003 — Colección de solicitudes y environment template

- **Fuentes:** OpenAPI v2, Payment Mock OpenAPI v1, AV-004, ADR-038, PM-INC-006 §7.2, INC-008 §6 (límite por sujeto), §5 de este plan.
- **Dependencias:** DOC-INC-002; `DOC-IV-003`, `DOC-IV-004` y `DOC-IV-005`.
- **Archivos:** `postman/ticketing.postman_collection.json`, `postman/ticketing-local.postman_environment.json`, `docs/collection-guide.md`, README §11 y §12.
- **Trabajo:** reorganizar en 00–09 y 99, corregir los defectos de §2.3, añadir los escenarios nuevos, curl de MF-001 a MF-004 en README §11 con sondeo explícito, y calibrar `slowMaxPolls` con tiempos medidos.
- **Ejecución:** newman 5.3.2 con las carpetas por defecto, dos ejecuciones consecutivas para demostrar que no depende de datos previos; carpetas lentas por separado; curl de §11 ejecutados. La clave se pasa desde `.env` sin imprimirla. Las salidas se barren con G2.
- **Validaciones:** G1, G2, G3, G4, G5.
- **Terminado cuando:** la colección importa (JSON v2.1 válido), la ejecución por defecto termina con 0 fallos, cada solicitud tiene operación OpenAPI y estado esperado, y los escenarios no reproducibles están rotulados con su motivo.

### DOC-INC-004 — Arquitectura, decisiones, diagramas, seguridad y límites

- **Fuentes:** arquitectura v2 completa; addendum; ADR `ACCEPTED` (ADR-003, ADR-008, ADR-022 a ADR-040); data model v2; messaging v2; aws-target v2; INC-008 y INC-010 (seguridad y observabilidad implementadas).
- **Dependencias:** DOC-INC-001; `DOC-IV-002` y `DOC-IV-006`. Puede ejecutarse en paralelo con 002 y 003.
- **Archivos:** `docs/architecture.md`, `docs/diagrams.md`; README §3, §14–§17, §19, §20, §23.
- **Trabajo:** cada decisión con problema, opción elegida, alternativa descartada, consecuencia principal y ADR. Temas: Clean Architecture y módulos; DynamoDB de una tabla con Ticket en particiones propias; transacciones todo o nada y guardián Order; índices dispersos y sharding; publicación directa, compensación y barrido; SQS al menos una vez, idempotencia y DLQ; aprovisionamiento asíncrono; carrera pago-expiración y reversos; circuit breakers y pausa de consumo; JWT, roles, propiedad y control de abuso; topología local y AWS diseñada. Las métricas diseñadas que no se producen se marcan `Designed, not implemented` (`DOC-ISSUE-004`). EVAL-011 se presenta como diseño y handoff.
- **Validaciones:** G1, G2, G4, G7; ningún ADR `SUPERSEDED` como vigente.
- **Terminado cuando:** EVAL-002 a EVAL-014 tienen su sección (§9.4) y cada afirmación sobre concurrencia, consistencia, seguridad o límites enlaza su fuente.

### DOC-INC-005 — Guía de demostración, resiliencia documentada y solución de problemas

- **Fuentes:** ADR-035, ADR-038, PLAT-IV-011, PLAT-INC-007 §3 y §8, INC-010 §6.6, colección de DOC-INC-003.
- **Dependencias:** DOC-INC-003 y DOC-INC-004; `DOC-IV-007` para ejecutar los comandos de resiliencia.
- **Archivos:** `docs/demo-guide.md`, `docs/resilience.md`, README §18 y §22, `docs/operations.md` (solución de problemas en detalle).
- **Trabajo:** según §6.2–§6.4.
- **Ejecución (si está autorizada):** comandos de los tres procedimientos a nivel de comando y señal. El informe los registra como "comando verificado; escenario `NOT_YET_VALIDATED_BY_QA`".
- **Validaciones:** G1, G2, G4, G6.
- **Terminado cuando:** los tres procedimientos están completos con su estado, y la guía de demostración solo usa material existente y verificado.

### DOC-INC-006 — Verificación final de comandos, enlaces, contratos y estados

- **Dependencias:** DOC-INC-001 a 005.
- **Trabajo:** ejecutar G1–G7 sobre el conjunto completo; ciclo limpio `down -v` → `up -d --wait` → `environment.sh` → newman por defecto → curl de README → parada, si está autorizado; actualizar README §2 y `docs/traceability.md` con el estado final; checklist de §13 del contrato del agente.
- **Terminado cuando:** todos los gates en verde o con su excepción justificada (G7 sin herramienta), `DEL-002` y `DEL-004` trazados por completo y estado global `DOCUMENTATION_COMPLETE_WITH_PENDING_QA`.

### Orden y paralelismo

```text
001 → 002 → 003 → 005 → 006
001 → 004 ───────↗
```

---

## 9. Traceability

### 9.1 Entregables

| ID | Cubierto por | Incremento |
|---|---|---|
| DEL-002 | `README.md` y `README.en.md` §1–§23: descripción (§1–§3), instalación (§5, §7), configuración (§6), comandos Docker (§7, §8), decisiones (§14 y `docs/architecture.md`), ejemplos de endpoints (§11) | 001–006 |
| DEL-004 | `postman/*.json` (carpetas 00–09 y 99), curl de README §11, `docs/collection-guide.md` | 003 |

### 9.2 Flujos y operaciones

| ID | Colección | README | Otros |
|---|---|---|---|
| MF-001 | 02 | §11 (curl con sondeo) | `docs/diagrams.md` (§7.8) |
| MF-002 | 03 | §11 | Diagrama de contenedores |
| MF-003 | 04, 05, 06, 07 | §11 | `docs/diagrams.md` (§7.1–§7.4) |
| MF-004 | 04, 08 | §11 | — |
| MF-005 | 09 (según `DOC-IV-005`) | §15, §18 | `docs/diagrams.md` (§7.7, §7.9); evidencia en JVM: `RoleComponentIT.failureWithReversal` |
| API-001 | 02, 07, 08 | §11 | — |
| API-002 | 03 | §11 | — |
| API-003 | 03, 04, 05 | §11 | — |
| API-004 | 04–09 | §11 | — |
| API-005 | 04–09 | §11 | — |
| API-006 | 02, 08 | §11 | — |
| API-101, API-102 | Solo subcarpeta técnica opcional (§5.2) | §10 (explicación) | — |
| API-103, API-104, API-105, API-106 | 01, 05, 06, 07, 09 | §10 | `DOC-ISSUE-001` |
| API-107 | 01 | §10 | — |
| API-108 | 04, 09 | §10 | Resiliencia 1 |
| API-109 | 06, 09 | §10 | — |
| API-110 | 01, 05, 06, 99 | §10 | — |
| API-111 | 00 | §8 | — |

### 9.3 Criterios de aceptación demostrados por la colección

Demostración documental, no certificación QA: AC-001, AC-002, AC-003, AC-004, AC-006, AC-010, AC-016, AC-017, AC-019, AC-020, AC-021 (error definitivo), AC-023 y AC-024 (inspección de un único intento), AC-027, AC-028, AC-032, AC-033, AC-039, AC-041, AC-042, AC-043, AC-048. Si `DOC-IV-005` lo permite, también AC-037 y AC-038. Los AC restantes se remiten a las pruebas del backend o a QA en `docs/traceability.md`.

### 9.4 Criterios de evaluación

| EVAL | Tratamiento | Ubicación |
|---|---|---|
| EVAL-001 | Estado real por requisito y flujos demostrables | README §2, §11, §12; colección |
| EVAL-002 | Decisiones con alternativa y consecuencia | README §14; `docs/architecture.md` |
| EVAL-003 | Transacción multi-partición, guardián, carreras; pruebas deterministas del backend | README §15; `docs/architecture.md`; colección 07, 08 |
| EVAL-004 | Aprovisionamiento y compra asíncronos, al menos una vez, idempotencia | `docs/diagrams.md`; colección con sondeo |
| EVAL-005 | Circuit breakers, DLQ, barrido, expiración; escenarios `NOT_YET_VALIDATED_BY_QA` | README §18; `docs/resilience.md` |
| EVAL-006 | Resource Server, roles, propiedad | README §16; colección 08 |
| EVAL-007 | `.env` no versionado, ejemplo ficticio, imágenes y logs sin secretos (evidencia de PLAT-INC-007 §5); gestor de secretos en AWS (diseño) | README §6, §16 |
| EVAL-008 | Idempotencia, una Order activa, límite por sujeto, límite de cuerpo, control de bots (diseño) | README §15, §16; colección 07, 08 |
| EVAL-009 | Estructura, índice equivalente en dos idiomas, guía de demostración | Ambos README; `docs/demo-guide.md` |
| EVAL-010 | Diagramas de contexto, contenedores y secuencias | `docs/diagrams.md`; README §3 |
| EVAL-011 | **Designed, not implemented**: handoff de aws-target v2 §11; sin código de Terraform | README §2, §20; `docs/architecture.md` |
| EVAL-012 | Topología AWS diseñada (red, IAM, secretos, escalado) | README §20; `docs/architecture.md` |
| EVAL-013 | Observabilidad implementada frente a diseñada; costes, aislamiento y gobernanza (diseño) | README §17; `docs/operations.md`; `docs/architecture.md` |
| EVAL-014 | Limitaciones, riesgos y cambios para producción | README §19, §20 |

### 9.5 Arquitectura v2 §16, §17 y ADR-038

- §16: cada fila de EVAL se resuelve en §9.4.
- §17, limitaciones 1 a 14: README §19, cada una con su ADR o riesgo. La limitación 15 se omite porque la feature spec v5 ya incorporó las aclaraciones (`DOC-ISSUE-009`).
- §17, tabla de cambios en producción: README §20.
- ADR-038, escenarios de resiliencia 1–3: `docs/resilience.md` y README §18, con estado `NOT_YET_VALIDATED_BY_QA`.

---

## 10. Documentation validations

Detalle y opciones en `human-review/documentation.implementation-plan-review.yaml`.

| ID | Pregunta | Recomendación | Bloquea | Necesaria antes de |
|---|---|---|---|---|
| DOC-IV-001 | Entrada canónica y enlace entre los dos README | `README.md` en español (canónico) + `README.en.md`, selector de idioma en la primera línea, mismo índice y comprobación de equivalencia | **Sí** | DOC-INC-001 |
| DOC-IV-002 | Idioma de `docs/**` | Solo español; README en inglés autosuficiente y enlaces marcados "(Spanish)" | No | DOC-INC-004 |
| DOC-IV-003 | Idioma de la colección | Un solo archivo: nombres con ID y título en español, descripciones en español con línea `EN:`, pruebas nombradas con códigos del contrato | No | DOC-INC-003 |
| DOC-IV-004 | Formato y evolución de la colección | Postman v2.1, evolucionar en el lugar los dos archivos, carpetas 00–09 y 99, curl solo para MF-001 a MF-004 | No | DOC-INC-003 |
| DOC-IV-005 | Escenarios lentos (reintento transitorio, reverso de MF-005, Event pasado) | Carpetas separadas fuera de la ejecución por defecto, con sondeo acotado configurable | No | DOC-INC-003 |
| DOC-IV-006 | Referencias a `SPEC_REPO` (ADR, arquitectura, informes, material del humano) | Documentación autosuficiente con IDs + enlaces absolutos a GitHub solo si los evaluadores tienen acceso; si no, solo IDs | No | DOC-INC-004 |
| DOC-IV-007 | Autorización de acciones sobre el entorno durante la verificación | Autorizar `down -v` del proyecto, `pause/unpause localstack` y `stop/start/kill` de `ticketing-worker` y `payment-mock` | No | DOC-INC-002 |
| DOC-IV-008 | Cómo documentar la alternativa de LocalStack con token (ADR-036) sin soporte en Compose | Ejemplo conceptual rotulado y no ejecutado + `DOC-ISSUE-007` a Platform | No | DOC-INC-002 |

---

## 11. Known missing evidence

| # | Evidencia ausente | Propietario | Efecto en la documentación | Referencia |
|---:|---|---|---|---|
| 1 | Certificación E2E sobre Compose de MF-001 a MF-004 | QA | README §2: la colección demuestra, no certifica | `DOC-ISSUE-006` |
| 2 | Carga AC-029 a AC-031 (servicio `load-test` provisional) | QA | Objetivos de prueba sin resultado; nunca SLA | `DOC-ISSUE-006` |
| 3 | Escenarios de resiliencia 1–3 de ADR-038 | QA | `NOT_YET_VALIDATED_BY_QA` | `DOC-ISSUE-006` |
| 4 | Cierre del backend (INC-011) | Backend | Sin `IMPLEMENTATION_COMPLETE`; cifras de INC-010 con su fecha | `DOC-ISSUE-005` |
| 5 | Versión 1.0.1 del contrato del mock (`OutcomeRule`) | Architect | Nota en §10 y §19; comprobación con combinadores resueltos | `DOC-ISSUE-001` |
| 6 | Terraform (EVAL-011) | Cloud / IaC | `Designed, not implemented` | — |
| 7 | Despliegue y medición en AWS (EVAL-012, RISK-008) | Cloud / IaC, QA | Solo diseño | — |
| 8 | Claims frente a un user pool real de Cognito (RISK-013) | QA / Platform | Limitación declarada | INC-008 §6 |
| 9 | Mecanismo ejecutable para la alternativa de LocalStack con token | Platform | Ejemplo conceptual no verificado | `DOC-ISSUE-007` |
| 10 | Validación de sintaxis Mermaid | — (sin herramienta local) | Revisión manual; el humano puede comprobar el renderizado en GitHub | — |
| 11 | Reproducción de MF-005 en Compose | Documentation (intento en DOC-INC-003) | Hoy solo hay evidencia dentro de la JVM | INC-010 `RoleComponentIT.failureWithReversal` |
| 12 | Métricas diseñadas que la aplicación no produce | Backend | `Designed, not implemented` en observabilidad | `DOC-ISSUE-004` |

---

## 12. Risks and handoffs

### 12.1 DOC-ISSUE detectados

| ID | Propietario | Severidad | Descripción | Tratamiento documental |
|---|---|---|---|---|
| DOC-ISSUE-001 | Architect | MEDIUM | `OutcomeRule = allOf(OutcomeRuleInput, {ruleId})` en `payment-mock.openapi.v1.yaml` es insatisfacible para un validador estricto, porque `additionalProperties: false` choca entre las ramas. El mock aplica la tolerancia aprobada PM-IV-016 en `API-103`/`API-104`. Falta publicar la versión 1.0.1 con `OutcomeRule` como objeto propio (recomendación en PM-INC-001 §3) | Se documenta como limitación conocida. Los ejemplos siguen la intención del contrato (`ruleId`, `match`, `behaviour`) y no se presenta la desviación como contrato nuevo |
| DOC-ISSUE-002 | Backend | LOW | La ruta versionada más larga mide 141 caracteres (fixtures de arquitectura). En Windows, sin `core.longpaths`, el clon falla con `Filename too long` si la carpeta de destino supera unos 117 caracteres | Requisito y problema conocido en README §5 y §22. Corrección opcional del backend: acortar esas rutas |
| DOC-ISSUE-003 | Backend (decisión del Architect) | LOW | El contrato del mock limita `customerRef` a 128 caracteres, y `ticketing` envía el `sub` del JWT sin limitarlo. Un `sub` de más de 128 caracteres (posible con sujetos arbitrarios del IdP local) produciría un error de contrato del mock. Ese comportamiento no está definido por ningún contrato | Limitación conocida en README §19. No se documenta ningún resultado concreto como correcto |
| DOC-ISSUE-004 | Backend | LOW | Métricas de aws-target v2 §7 que la aplicación no produce (INC-010 §6.1.6): profundidad y antigüedad de colas y DLQ (las da el servicio SQS), "reversos pendientes" derivado de contadores, "Orders expiradas sin encolar" no producida (se produce "expiradas sin PaymentAttempt") | README §17 separa lo implementado de lo diseñado |
| DOC-ISSUE-005 | Backend | MEDIUM | INC-011 (cierre del backend) no ejecutado | README §2: "INC-001 a INC-010 DONE; cierre pendiente". Al cerrarse, se actualizan las cifras |
| DOC-ISSUE-006 | QA / Resilience | HIGH | No hay evidencia de E2E, carga ni resiliencia. `load-test` termina con exit 1 a propósito | Estados `NOT_YET_VALIDATED_BY_QA` y estado máximo `DOCUMENTATION_COMPLETE_WITH_PENDING_QA` |
| DOC-ISSUE-007 | Platform | MEDIUM | ADR-036 exige que el README documente la alternativa "LocalStack actual con token gratuito no versionado". Compose fija `localstack/localstack:4.14.0@sha256:…` sin variable de imagen ni de `LOCALSTACK_AUTH_TOKEN`, y `.env.example` no la contempla. No hay mecanismo soportado | Según `DOC-IV-008` |
| DOC-ISSUE-008 | Humano (autor de `run-local.sh`) | LOW | `run-local.sh smoke` no trata `REJECTED` como terminal (espera hasta 60 s antes de fallar); usa `API_PORT`, `API_MANAGEMENT_PORT`, `WORKER_PORT` y `WORKER_MANAGEMENT_PORT`, que no están en `.env.example`, en lugar de `TICKETING_API_HOST_PORT`; riesgo de CRLF en `.env` con un `sh` que no es de MSYS (regla opcional `.env.example text eol=lf`, PLAT-INC-007 §8) | Se documenta el comportamiento real sin modificar el script |
| DOC-ISSUE-009 | Architect | LOW | La limitación 15 de arquitectura v2 §17 ("la especificación v4 aún no incorpora las aclaraciones") quedó superada por feature spec v5, que incorporó 14 de 15 (la 15 sigue condicionada y no activada, `ticketing.local-environment.v1.md` §4.1) | No se repite como limitación vigente |

### 12.2 Riesgos del plan

| Riesgo | Mitigación |
|---|---|
| Divergencia entre los dos README | §3.2: edición en pareja y comprobación G4 en cada incremento |
| Los escenarios del mock con reintentos tardan más de lo estimado (lease de 45 s, backoff 5/15/30/60 s, circuito) | Medición en DOC-INC-003, `slowMaxPolls` calibrado, carpetas lentas fuera de la ejecución por defecto |
| El límite de 10 compras por 10 s y sujeto rompe la colección | Un sujeto por escenario (§5.4) |
| newman 5.3.2 con Node 14 difiere de versiones actuales | Se documenta la versión verificada y no se usa `npx` sin versión |
| Enlaces a `SPEC_REPO` inaccesibles para el evaluador | `DOC-IV-006` |
| El entorno levantado por Platform se altera | Se registra el estado inicial y se restaura. Las acciones destructivas dependen de `DOC-IV-007` |
| QA llega después | Los estados pendientes se actualizan con su informe. Este agente no produce esa evidencia |

### 12.3 Handoffs

| Destino | Contenido |
|---|---|
| Architect | `DOC-ISSUE-001`, `DOC-ISSUE-009`; decisión sobre `DOC-ISSUE-003` |
| Backend | `DOC-ISSUE-002`, `DOC-ISSUE-003`, `DOC-ISSUE-004`, `DOC-ISSUE-005` |
| Platform | `DOC-ISSUE-007` |
| QA / Resilience | `DOC-ISSUE-006`; `docs/resilience.md` y la colección como punto de partida; la colección no sustituye su suite |
| Humano | `DOC-ISSUE-008`; respuestas a `DOC-IV-001` a `DOC-IV-008`; versionar los cambios (este agente no hace commit ni push) |
