---
artifact: documentation-increment-report
increment: DOC-INC-001
result: DONE
code_revision: "CODE_REPO main 90f8ed2 (árbol limpio al empezar) + cambios sin commit de este agente: README.md (M), README.en.md (nuevo), docs/traceability.md (nuevo)"
spec_revision: "SPEC_REPO main 9b78fdd"
verified_at: 2026-10-05
plan: implementation/documentation.implementation-plan.v1.md
review: human-review/documentation.implementation-plan-review.yaml (APPROVED)
---

# DOC-INC-001 — Inventario, estructura, trazabilidad y esqueleto documental

## 1. Documents produced

| Archivo (`CODE_REPO`) | Cambio |
|---|---|
| `README.md` | Reescrito como entrada canónica en **español** (DOC-IV-001 a): selector de idioma en la primera línea, propuesta de valor, mapa de navegación por audiencia y los 23 H2 numerados del plan §4. §1 (propósito, términos del dominio, fuera de alcance) y §2 (leyenda de etiquetas y tabla de estado con evidencia) completos. El resto de secciones contiene el contenido correcto del README anterior, traducido, o un marcador explícito "Pendiente de DOC-INC-00N" |
| `README.en.md` | Nuevo, en **inglés**, construido a partir del README anterior del humano. Mismo índice, mismos bloques de código byte a byte, mismos enlaces, IDs, tablas y etiquetas |
| `docs/traceability.md` | Nuevo. Matriz inicial de DEL-002, DEL-004, MF-001 a MF-005, operaciones y EVAL-001 a EVAL-014 con su estado |

No se tocaron `postman/ticketing.postman_collection.json` ni `postman/ticketing-local.postman_environment.json` (sha256 `f670a864…` y `d1bb68d2…` sin cambios, DOC-IV-004 b). Copia del README original en el scratchpad de la sesión para el mapa de contenido (sha256 `3ed6c728…`).

Mapa "contenido existente → sección nueva" (nada se perdió):

| Contenido del README anterior | Ubicación nueva | Cambio |
|---|---|---|
| Título y descripción | Encabezado | Se añade la propuesta de valor |
| Tabla de carpetas | §4 | Se añaden `postman/` y `docs/` |
| Requisitos (JDK 25, Docker con Compose v2, `sh` y `curl`) | §5 | Sin cambio; la precisión por opción y las versiones quedan para DOC-INC-002 |
| Nota de rutas largas en Windows | §5 | Sin cambio (DOC-ISSUE-002 se enlaza en DOC-INC-002) |
| `cp .env.example .env` y `*_HOST_PORT` | §6 | Sin cambio |
| Opción A (`up -d --wait`, `down -v`, puertos, tiempos) | §7 | Sin cambio en los comandos |
| `sh platform/verify/environment.sh` | §8 | **Corregido** a `bash platform/verify/environment.sh` (el script es `#!/usr/bin/env bash`) |
| Opción B (`run-local.sh`) y advertencia de exclusión | §21 | Sin cambio |
| Curl de token | §9 | Sin cambio |
| Curl de creación de Event | §11 | Sin cambio |
| Tabla de operaciones y roles | §11 | Sin cambio; se enlaza la copia del contrato |
| Sección Postman | §12 | Se identifica como **colección previa** del autor; se retira `npx newman` sin versión (G1); la ejecución con newman verificado queda para DOC-INC-003 |
| Salud y métricas | §8 | Sin cambio |
| Build y pruebas | §13 | Sin cambio |
| Configuración (variables del backend y credenciales) | §6 | Sin cambio |

## 2. Sources and traceability

- Plan §2.3, §3.2 y §4; feature spec v5 §1 a §3 (propósito, términos, fuera de alcance); INC-008, INC-009, INC-010, PM-INC-006 y PLAT-INC-007 (filas de estado de §2).
- Respuestas humanas aplicadas: DOC-IV-001 (a); DOC-IV-004 (b) (la colección previa se conserva y se describe como previa; la nueva se anuncia como `postman/ticketing-demo.*`).

## 3. Commands and scenarios verified

Ninguno ejecutable en este incremento (sin cambios de comandos salvo `sh` → `bash`, verificado por la cabecera `#!/usr/bin/env bash` del script y por PLAT-INC-007, que lo ejecuta con `bash`).

## 4. Static validation

Script de comprobación en el scratchpad de la sesión (no se entrega), `checks.py`:

| Gate | Resultado |
|---|---|
| G1 Estático (enlaces y anclas locales, rutas personales, `:latest`, `npx newman`, artefactos v1 o ADR `SUPERSEDED` como vigentes, comandos destructivos globales) | 0 fallos tras corregir un enlace a un documento aún inexistente (`collection-guide.md`, ahora texto) |
| G2 Secretos (JWT, `AKIA`, claves privadas, valor real de `PAYMENT_MOCK_API_KEY` leído de `.env` sin imprimirlo) | 0 fallos |
| G4 Equivalencia bilingüe | 0 fallos: 23 H2 en ambos, 7 bloques de código idénticos, 6 tablas con las mismas filas, mismos destinos de enlace, IDs y etiquetas por sección. Dos diferencias corregidas en la primera ejecución: "Out of scope" usado como texto en inglés y "Implemented and verified" repetido en la leyenda inglesa |
| G6 Estados | 0 afirmaciones de QA, `PASS`, `RELEASED` o SLA |

Informativo: `down -v` aparece en §7 con su efecto ("removes everything (data is ephemeral)").

## 5. Unverified claims and dependencies

- §2 cita las cifras de INC-010 (99,46 %, 129 `*IT`) sin reejecutar; DOC-INC-002 ejecuta `./mvnw verify` sin integración (decisión humana del PLAN).
- Las secciones con marcador dependen de DOC-INC-002 a DOC-INC-005.

## 6. DOC-ISSUE items

Sin cambios respecto al plan (DOC-ISSUE-001 a DOC-ISSUE-009). DOC-ISSUE-008: el commit `90f8ed2` del humano hace que `run-local.sh smoke` trate `REJECTED` como estado final; el resto de la observación (variables `API_PORT`/`WORKER_PORT` no documentadas en `.env.example`, riesgo CRLF) sigue como nota.

## 7. Deviations

Ninguna.

## 8. Handoff

Siguiente: DOC-INC-002.
