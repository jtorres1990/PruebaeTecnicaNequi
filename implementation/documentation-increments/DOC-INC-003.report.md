---
artifact: documentation-increment-report
increment: DOC-INC-003
result: DONE
code_revision: "CODE_REPO main 90f8ed2 + cambios sin commit de este agente: README.md (M), README.en.md, docs/{traceability,operations,collection-guide}.md, postman/ticketing-demo.postman_collection.json, postman/ticketing-demo.postman_environment.json (nuevos)"
spec_revision: "SPEC_REPO main 9b78fdd"
verified_at: 2026-10-05
environment: "Stack completo de Compose levantado en DOC-INC-002; newman 5.3.2 sobre Node.js 14.21.3; Payment Mock en el host 18090; clave cargada de .env sin imprimirla"
---

# DOC-INC-003 — Colección de solicitudes y environment template

## 1. Documents produced

| Archivo | Cambio |
|---|---|
| `postman/ticketing-demo.postman_collection.json` | **Nuevo** (DOC-IV-004 b). Postman Collection v2.1, 103 solicitudes en las carpetas 00 a 09 y 99: 00 entorno y salud (7), 01 control del mock (4), 02 aprovisionamiento (7), 03 catálogo y disponibilidad (7), 04 compra confirmada (7), 05 rechazo (9), 06 fallos (12), 07 idempotencia y Order activa (9), 08 seguridad y propiedad (19), 09 escenarios lentos en tres subcarpetas (6 + 9 + 6), 99 limpieza (1). Nombres `NN.MM API-xxx Título` en español y descripciones con línea `EN:` (DOC-IV-003 b) |
| `postman/ticketing-demo.postman_environment.json` | **Nuevo**. `baseUrl`, `apiRootUrl`, `idpUrl`, `paymentMockUrl`, `paymentMockApiKey` (`secret`, **vacío**), `pollIntervalMs` 1000, `maxPolls` 30, `slowMaxPolls` 300 |
| `docs/collection-guide.md` | Nuevo: preparación, ejecución por defecto (Runner y newman), construcción, tabla por carpeta, escenarios lentos con tiempos medidos, lo que no demuestra, problemas conocidos, correspondencia con operaciones y diferencias con la colección previa |
| `README.md`, `README.en.md` | §2 (fila de la colección), §11 (tabla `API-001` a `API-006` con respuestas principales y recorrido curl de MF-001 a MF-004 con sondeo), §12 (colección principal y previa, newman, resultados) |
| `docs/traceability.md` | Flujos, operaciones y AC demostrados |

**No modificados** (DOC-IV-004 b): `postman/ticketing.postman_collection.json` (sha256 `f670a864…`) y `postman/ticketing-local.postman_environment.json` (`d1bb68d2…`), iguales antes y después.

Defectos de la colección previa corregidos en la nueva (plan §2.3): IDs de operación según `x-id`; estados finales completos y rechazo `REJECTED` con `PAYMENT_DECLINED` mediante regla explícita; clave vacía en el environment; borrado de tokens en la carpeta 99; un sujeto `CUSTOMER` por escenario; control del mock antes de cada escenario.

La colección se genera con un script del scratchpad de la sesión (no se entrega); el entregable es el JSON.

## 2. Sources and traceability

OpenAPI v2 y Payment Mock OpenAPI v1 (copias en `CODE_REPO` con el mismo SHA-256 normalizado que `SPEC_REPO`: `2b403755…` y `5360ffee…`); AV-004; ADR-029, ADR-030, ADR-032, ADR-035, ADR-038; PM-INC-006 §7.2; INC-008 §1 y §6. Respuestas humanas: DOC-IV-003 (b), DOC-IV-004 (b), DOC-IV-005 (a).

## 3. Commands and scenarios verified

Exploración previa con curl (sin imprimir tokens) para fijar las aserciones: cursor de disponibilidad en base64url; `DEFINITIVE_ERROR` → `FAILED`/`PROCESSING_FAILED` en 0,3 s, `API-108` con 1 invocación y sin `result`, `API-109` 404; 2 fallos transitorios → `CONFIRMED` en menos de 2 s con 3 invocaciones; compra feliz `CONFIRMED` en 138 ms; cuerpo de 270 KB → 413 `PAYLOAD_TOO_LARGE` (también sin token); límite de tasa → 429 `RATE_LIMITED` con `Retry-After: 8`, contando también las compras previas del sujeto; 401 con `WWW-Authenticate: Bearer`; Order ajena y malformada con el mismo cuerpo salvo `traceId`.

Calibración de escenarios lentos con curl: transitorio de 4 fallos → `CONFIRMED` en 47 s, 5 invocaciones; fallos sostenidos → `FAILED` en 117 s (7 invocaciones), cancelación `REGISTERED_BEFORE_CHARGE` 21 s después; Event a 20 s → 409 `EVENT_NOT_ON_SALE`, disponibilidad 200, ausente del listado. `slowMaxPolls` = 300 (unos 5 min, más del doble del caso más lento).

| Ejecución (`node …/newman.js run postman/ticketing-demo.postman_collection.json -e postman/ticketing-demo.postman_environment.json --env-var paymentMockUrl=http://localhost:18090 --env-var paymentMockApiKey=<de .env> --folder …`) | Solicitudes ejecutadas | Aserciones | Fallos | Duración |
|---|---|---|---|---|
| Carpetas 00 a 08 y 99, primera | 95 (82 distintas + repeticiones) | 243 | 0 | 12 s |
| Carpetas 00 a 08 y 99, segunda seguida | 95 | 243 | 0 | 11,7 s |
| Carpetas 00, 01, 02, 09, 99 | 208 | 226 | 0 | 3 min 8,5 s (09.1: 43 sondeos; 09.2: 85 sondeos hasta `FAILED` y 19 hasta la cancelación) |
| Carpetas 00, 01, 02, `09.3 Event pasado`, 99 (subcarpeta por nombre) | 46 | 52 | 0 | 25,3 s |

Bloque `sh` de README §11 extraído del archivo y ejecutado tal cual: exit 0; `provisioning: ENABLED`, `order: CONFIRMED`, 404 para `customer-b`; 0 tokens en la salida. Se añadió `; echo` tras las respuestas JSON para separar las líneas.

newman se ejecutó como `node <ruta>/newman.js` (la misma versión 5.3.2 que el binario `newman` documentado). La aplicación Postman no se usó: la importación se apoya en que newman (postman-collection) analiza y ejecuta el archivo, no en una importación manual.

## 4. Static validation

| Gate | Resultado |
|---|---|
| G1 | 0 fallos; JSON de colección y environment parseables |
| G2 | 0 fallos en entregables, salidas CLI de newman y salidas de los bloques del README. `paymentMockApiKey` vacío en el template; variables de token vacías en la colección. Los informes JSON de newman, que contienen las cabeceras (tokens locales y la clave), se borraron del scratchpad tras extraer los conteos |
| G3 Contrato (script con `pyyaml` y `jsonschema` sobre las copias del contrato) | 0 fallos. Cada solicitud corresponde a una operación (método y ruta), salvo `/readyz` y el endpoint de token de `local-idp`, identificados como no contractuales; cabeceras obligatorias (`Authorization`, `Idempotency-Key`, `X-Api-Key`) presentes salvo en las pruebas negativas que las omiten a propósito; todos los estados asertados existen en la operación; cuerpos válidos contra los esquemas con `additionalProperties: false`; las 4 solicitudes inválidas a propósito (400) identificadas; reglas del mock con un único criterio; campos capturados presentes en los esquemas de respuesta; todo valor enumerado asertado existe en el contrato; el ID citado en cada nombre coincide con el `x-id`. Los curl de los README (`API-001`, `API-003`, `API-004`, `API-005`, `API-103`, `API-104`, `API-107`, `API-110`) también |
| G4 | 0 fallos: 23 H2, 13 bloques de código idénticos, 15 tablas |
| G6 | 0 hallazgos |

Cobertura de operaciones por la colección y los README: `API-001` a `API-006`, `API-103` a `API-111` (todas). `API-101` y `API-102` se ejercen de forma indirecta por el `worker`.

## 5. Unverified claims and dependencies

- Importación en la aplicación Postman y ejecución con su Runner: no verificadas en esta sesión (sin la aplicación); verificado con newman.
- La carpeta 07 depende de una ventana de 2,5 s; documentado que debe ejecutarse seguida.
- MF-005 se demuestra en su variante de resultado desconocido (`REGISTERED_BEFORE_CHARGE`). La variante `REVERSED` (pago aprobado no aplicado) no es reproducible con el mock sin cambiar contratos; queda en las pruebas del backend y del mock (documentado en `collection-guide.md` §6).
- 503 con la cola caída, expiración de 10 minutos y aprobación tardía: no demostrables con la colección (plan §5.5), documentados con su evidencia alternativa.

## 6. DOC-ISSUE items

Sin nuevos. DOC-ISSUE-001 documentado en `collection-guide.md` §7.

## 7. Deviations

- La colección es nueva y la previa se conserva (DOC-IV-004 b, distinta del plan original, que proponía evolucionarla en el lugar).
- No se incluye una subcarpeta técnica con `API-101`/`API-102`: el plan la dejaba como opcional y el flujo público ya las ejerce.
- La carpeta 09 agrupa los tres escenarios lentos en subcarpetas (09.1, 09.2, 09.3) en lugar de una carpeta solo de cancelaciones; cada subcarpeta se puede ejecutar por nombre.

## 8. Handoff

Siguiente: DOC-INC-004. Entorno: stack completo levantado (restauración al final de DOC-INC-006).
