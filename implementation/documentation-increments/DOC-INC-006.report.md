---
artifact: documentation-increment-report
increment: DOC-INC-006
result: DONE
code_revision: "CODE_REPO main 90f8ed2 + cambios sin commit de este agente: README.md (M); README.en.md, docs/{architecture,collection-guide,demo-guide,diagrams,operations,resilience,traceability}.md, postman/ticketing-demo.postman_collection.json, postman/ticketing-demo.postman_environment.json (nuevos)"
spec_revision: "SPEC_REPO main 9b78fdd + implementation/documentation-increments/DOC-INC-001..006.report.md (nuevos, sin commit)"
verified_at: 2026-10-05
overall_status: DOCUMENTATION_COMPLETE_WITH_PENDING_QA
---

# DOC-INC-006 — Verificación final de comandos, enlaces, contratos y estados

## 1. Documents produced

| Archivo | Cambio |
|---|---|
| `README.md`, `README.en.md` | §12 (fila de la ejecución final y nota sobre la variación de conteos por sondeo), §23 (referencias a `demo-guide.md` y `resilience.md`; enlace de directorio de `SPEC_REPO` con `/tree/`; textos de enlace en inglés en `README.en.md`) |
| `docs/traceability.md` | Versión final: entregables, flujos, operaciones, AC demostrados, EVAL-001 a EVAL-014, arquitectura v2 §16/§17, escenarios de ADR-038 y resumen de la evidencia |
| `docs/collection-guide.md`, `docs/demo-guide.md` | Resultado de la ejecución final y duración medida del bloque curl de README §11 (0,8 s; antes "unos 5 s", sin medir) |

Conjunto entregable final en `CODE_REPO`: 2 README (541 líneas cada uno, 23 H2), 7 documentos en `docs/` (1.433 líneas), colección (103 solicitudes) y environment nuevos; colección y environment previos **sin cambios** (sha256 `f670a864…`, `d1bb68d2…`).

## 2. Sources and traceability

Todas las fuentes de DOC-INC-001 a DOC-INC-005. Matriz final en `CODE_REPO/docs/traceability.md`:

- **DEL-002**: README §1 a §23 en ambos idiomas: descripción (§1 a §3), instalación (§5, §7), configuración (§6), comandos Docker (§7, §8), decisiones (§14 y `docs/architecture.md`), ejemplos de endpoints (§11). Además las secciones exigidas por ADR-036: tokens (§9), configuración del mock (§10), alternativa de LocalStack con token (`docs/operations.md` §9) y escenarios de resiliencia (§18, `docs/resilience.md`).
- **DEL-004**: colección principal, curl de README §11 y `docs/collection-guide.md`.
- **MF-001 a MF-005**, **API-001 a API-006** y **API-103 a API-111**: demostrados (colección y curl); **API-101** y **API-102** de forma indirecta.
- **EVAL-001 a EVAL-014**: con ubicación y estado; EVAL-011 y EVAL-012 como `Designed, not implemented`.
- **Arquitectura v2 §16 y §17**: limitaciones 1 a 14 en README §19 (la 15 omitida por DOC-ISSUE-009) y cambios de producción en README §20.
- **ADR-038 escenarios 1 a 3**: `docs/resilience.md` y README §18, `NOT_YET_VALIDATED_BY_QA`.

## 3. Commands and scenarios verified

Ciclo limpio autorizado (DOC-IV-007 a), 2026-10-05, revisión `90f8ed2`:

| Paso | Resultado |
|---|---|
| `docker compose down -v` | exit 0 en 7,5 s; 0 contenedores del proyecto |
| `docker compose up -d --wait` | exit 0 en **27,7 s** (imágenes existentes, sin `pull` ni build) |
| `bash platform/verify/environment.sh` | **65 / 0** |
| newman 5.3.2, carpetas por defecto (00 a 08 y 99) | **96 solicitudes ejecutadas / 0 fallos, 244 aserciones / 0 fallos**, 14,3 s |
| Bloque `sh` de README §11 extraído y ejecutado tal cual | exit 0 en **0,8 s**: `provisioning: ENABLED`, `order: CONFIRMED`, 404 para otro cliente |
| Restauración: `docker compose --profile load down -v --remove-orphans`; `docker compose up -d --wait dynamodb-local localstack infra-init local-idp payment-mock`; espera a `infra-init` `exited 0` | Estado igual al encontrado al empezar (§8) |

Resumen de todo lo ejecutado en DOC-INC-002 a DOC-INC-006 (detalle en cada informe): `config --quiet` (3 variantes), dos ciclos `down -v`/`up -d --wait`, `stop`/`up` con datos efímeros, verificadores de Platform, `./mvnw verify` en ambos proyectos, `.\mvnw.cmd -v`, `run-local.sh start`/`smoke`/`stop`, bloques `sh` y `powershell` de README §6 a §11, cinco ejecuciones de newman sin fallos (tres por defecto, una lenta completa y una subcarpeta), tres procedimientos de resiliencia ejecutados dos veces (comando verificado; escenario `NOT_YET_VALIDATED_BY_QA`).

No verificado (y rotulado así en la documentación): importación en la aplicación Postman (solo newman), `platform/verify/infra-init-negative.sh`, alternativa de LocalStack con token (`Designed, not implemented`), `./mvnw verify -Pintegration` (no reejecutado por decisión humana; cifras de INC-010), primer build desde cero (citado de PLAT-INC-007), renderizado Mermaid en GitHub.

## 4. Static validation

| Gate | Herramienta | Resultado final |
|---|---|---|
| G1 Estático | `checks.py` (scratchpad) | **0 fallos**: enlaces y anclas locales de 9 documentos resolubles; 2 JSON parseables; sin rutas personales, `:latest`, `npx newman` sin versión, artefactos v1 ni ADR `SUPERSEDED` como vigentes, ni limpiezas globales de Docker. `down -v` aparece siempre con su efecto descrito |
| G2 Secretos | `checks.py` | **0 fallos** en 89 archivos (entregables y salidas guardadas de verificadores, builds, newman CLI, bloques del README y procedimientos): sin JWT, `AKIA`, claves PEM ni el valor real de `PAYMENT_MOCK_API_KEY`; `paymentMockApiKey` vacío; variables de token vacías. Los informes JSON de newman (con cabeceras) se borraron tras extraer los conteos |
| G3 Contrato | `g3.py` con `pyyaml` y `jsonschema` | **0 fallos**: 103 solicitudes y los curl de ambos README contra los dos OpenAPI (operación, cabeceras, estados asertados, cuerpos, parámetros de consulta, campos capturados, enumerados, ID citado = `x-id`). Cobertura: `API-001` a `API-006` y `API-103` a `API-111` |
| G4 Equivalencia bilingüe | `checks.py` | **0 fallos**: 23 H2 numerados, mismas H3, 14 bloques de código idénticos byte a byte, 24 tablas con las mismas filas, mismos destinos de enlace, IDs y etiquetas de estado por sección; selector de idioma en la primera línea de cada README. Barrido adicional: sin prosa en español en `README.en.md` fuera del selector |
| G5 Ejecutable | Docker, Git Bash, PowerShell, JDK 25, newman | Ver §3 |
| G6 Estados | `checks.py` y lectura | **0 hallazgos**: ninguna afirmación `PASS`, `QA_PASSED`, `RELEASED` ni `PRODUCTION_READY`; E2E, carga y resiliencia como `NOT_YET_VALIDATED_BY_QA` o `Designed, not implemented`; objetivos de carga presentados como objetivos de prueba, nunca como SLA |
| G7 Diagramas | Comparación con las fuentes | **16 de 16** bloques Mermaid idénticos a la arquitectura v2 o a aws-target v2. Sin validador Mermaid local (excepción justificada del plan) |

Checklist del contrato del agente (§13, modo 2): plan aprobado; cambios ajenos preservados (README del humano conservado en `README.en.md`, colección previa intacta); solo rutas permitidas (`README.md`, `README.en.md`, `docs/**`, `postman/` con archivos nuevos; informes en `SPEC_REPO`); ningún cambio en código, configuración, Compose, `.env`, `.env.example`, `run-local.sh` ni `platform/`; fuentes vigentes; enlaces y formatos válidos; ejemplos conformes a ambos OpenAPI; sin secretos; cada comando con estado de verificación; pendientes de QA identificados; informes con evidencia real. Ningún commit ni push.

## 5. Unverified claims and dependencies

- QA (DOC-ISSUE-006): carga AC-029 a AC-031, extremo a extremo certificado de MF-001 a MF-004 y validación de los tres escenarios de resiliencia. Por eso el estado máximo es `DOCUMENTATION_COMPLETE_WITH_PENDING_QA`.
- Backend (DOC-ISSUE-005): cierre INC-011 no ejecutado; las cifras de integración y cobertura de INC-010 se citan con su revisión `69a2c23`.
- Cloud/IaC: Terraform y despliegue en AWS no existen (EVAL-011, EVAL-012).

## 6. DOC-ISSUE items

| ID | Propietario | Severidad | Estado |
|---|---|---|---|
| DOC-ISSUE-001 | Architect | MEDIUM | Abierto: contrato del mock 1.0.1 (`OutcomeRule`) pendiente; documentado como limitación |
| DOC-ISSUE-002 | Backend | LOW | Abierto: rutas de 141 caracteres; documentado en README §5 y §22 |
| DOC-ISSUE-003 | Backend / Architect | LOW | Abierto: `customerRef` de 128 frente a `sub` sin límite; limitación documentada (no alcanzable en local) |
| DOC-ISSUE-004 | Backend | LOW | Abierto: métricas diseñadas no producidas; `Designed, not implemented` en README §17 |
| DOC-ISSUE-005 | Backend | MEDIUM | Abierto: INC-011 no ejecutado |
| DOC-ISSUE-006 | QA / Resilience | HIGH | Abierto: sin evidencia de E2E, carga ni resiliencia |
| DOC-ISSUE-007 | Platform | MEDIUM | Abierto: sin mecanismo soportado para LocalStack con token; ejemplo conceptual documentado |
| DOC-ISSUE-008 | Humano | LOW | Parcialmente resuelto (`90f8ed2`); quedan las variables de puerto propias de `run-local.sh` |
| DOC-ISSUE-009 | Architect | LOW | Aplicado en la documentación (limitación 15 omitida); el artefacto de arquitectura no cambia |
| DOC-ISSUE-010 | Platform | LOW | Abierto (DOC-INC-002): `resources.sh` necesita `MSYS_NO_PATHCONV=1` en Git Bash; `identity.sh` deja restos del perfil `load` que hacen fallar `environment.sh` |

No se creó ningún `DOC-IV` nuevo ni ninguna revisión `human-review/documentation.implementation-review.<n>.yaml`: no hubo decisiones bloqueantes.

## 7. Deviations

Ninguna en este incremento. Desviaciones acumuladas, todas por respuesta humana o registradas en su incremento: colección nueva en lugar de evolucionar la previa (DOC-IV-004 b); carpeta 09 con tres subcarpetas lentas; sin subcarpeta técnica de `API-101`/`API-102`.

## 8. Handoff

**Recursos Docker al terminar** (iguales al estado encontrado):

- Proyecto `ticketing-platform`: `dynamodb-local`, `localstack`, `local-idp` y `payment-mock` `healthy` (`payment-mock` en `127.0.0.1:18090`), `infra-init` `Exited (0)`; sin `ticketing-api`, `ticketing-worker` ni servicios `load-*`; red `ticketing-platform_ticketing-net`; 0 volúmenes del proyecto. Datos efímeros nuevos.
- Imágenes propias sin cambios: `ticketing` `4f8a52fef841`, `payment-mock` `aedbe5e7bb25`, `local-idp` `90c0a973eacd`, `infra-init` `b787ede57259`. Ninguna imagen construida ni eliminada.
- Recursos ajenos intactos: contenedor `dynamodb-local` (`Exited (143)`), red `dynamodb_default`, volúmenes `d72a66f1…` y `portainer_data`.
- Archivos de build ignorados por Git regenerados por la verificación: `ticketing/*/target`, `payment-mock/target`, `.run/*.log` (de `run-local.sh`).

**Humano**: revisar y versionar los cambios de `CODE_REPO` y los seis informes de `SPEC_REPO` (este agente no hace commit ni push).
**QA**: `docs/resilience.md` y la colección (carpetas 06 y 09) como punto de partida; la colección no sustituye su suite.
**Platform**: DOC-ISSUE-007 y DOC-ISSUE-010. **Backend**: DOC-ISSUE-002 a 005. **Architect**: DOC-ISSUE-001, 003 y 009.

Estado global: **`DOCUMENTATION_COMPLETE_WITH_PENDING_QA`**.
