---
artifact: documentation-increment-report
increment: DOC-INC-004
result: DONE
code_revision: "CODE_REPO main 90f8ed2 + cambios sin commit de este agente (README.md, README.en.md, docs/**, postman/ticketing-demo.*)"
spec_revision: "SPEC_REPO main 9b78fdd"
verified_at: 2026-10-05
---

# DOC-INC-004 — Arquitectura, decisiones, diagramas, seguridad y límites

## 1. Documents produced

| Archivo | Cambio |
|---|---|
| `docs/architecture.md` | Nuevo. Vista general y las cuatro ideas del diseño; 16 decisiones con problema, decisión, alternativa descartada, consecuencia y ADR enlazado (Clean Architecture y mock independiente; tabla con Ticket en particiones propias; reserva multi-partición; una transacción por transición y guardián Order; índices dispersos con sharding y disponibilidad derivada; publicación directa, compensación y barrido; SQS al menos una vez, idempotencia y DLQ; aprovisionamiento asíncrono; carrera pago-expiración y reversos; procesos periódicos aislados; errores, retry y circuit breakers; seguridad y abuso; auditoría; topología local; pruebas; topología AWS diseñada); concurrencia e idempotencia con su evidencia; seguridad y observabilidad implementadas frente a diseñadas; AWS diseñado (cómputo, red, IAM, secretos, escalado, coste, aislamiento y gobernanza); limitaciones; cambios para producción; riesgos |
| `docs/diagrams.md` | Nuevo. 14 diagramas Mermaid copiados literalmente: contexto (§3), contenedores (§4), secuencias §7.1 a §7.11 de la arquitectura v2 y topología AWS (aws-target v2 §1, rotulada `Designed, not implemented`), cada uno con origen, ADR y relación con la colección |
| `README.md`, `README.en.md` | §3 (diagrama de contenedores literal y resumen), §14 (tabla de 14 decisiones con alternativa y consecuencia), §15 (concurrencia, atomicidad, idempotencia y evidencia con estado), §16 (controles de seguridad y secretos con estado), §17 (observabilidad implementada frente a diseñada), §19 (limitaciones con estado y fuente), §20 (AWS diseñado, Terraform no implementado, cambios para producción), §23 (referencias a `docs/`, contratos y `SPEC_REPO`) |
| `docs/traceability.md` | Filas EVAL-002 a EVAL-004 y EVAL-006 a EVAL-014 |

## 2. Sources and traceability

Arquitectura v2 completa (§1, §3, §4, §7, §8, §12, §16, §17); registro de ADR; ADR-003, ADR-008, ADR-022 a ADR-040 (secciones de opciones, decisión y consecuencias); aws-target v2 §1 a §11; data model v2 y messaging v2 (referencias); addendum de consolidación; INC-005 (`DynamoDbMechanismIT`), INC-006/007 (circuit breakers), INC-008 (seguridad), INC-010 (observabilidad y `RoleComponentIT`), PLAT-INC-003 (sujetos de hasta 128 caracteres), PLAT-INC-007 §5 (barrido de secretos). Respuestas humanas: DOC-IV-002 (a) (`docs/**` solo en español; enlaces marcados "(Spanish)" en `README.en.md`), DOC-IV-006 (a) (enlaces absolutos a `github.com/jtorres1990/PruebaeTecnicaNequi` además del resumen con ADR-ID).

Ningún ADR `SUPERSEDED` se presenta como vigente: la única mención de ADR-001 a ADR-021 está rotulada como reemplazada (G1). La limitación 15 de la arquitectura v2 §17 no se repite (DOC-ISSUE-009).

## 3. Commands and scenarios verified

Sin ejecución nueva sobre el entorno. Las afirmaciones de comportamiento citan pruebas de los informes del backend, del mock y de Platform, o las ejecuciones de DOC-INC-002/003.

## 4. Static validation

| Gate | Resultado |
|---|---|
| G1 | 0 fallos (enlaces y anclas locales; ningún ADR reemplazado como vigente) |
| G2 | 0 fallos |
| G3 | 0 fallos (sin cambios en ejemplos) |
| G4 | 0 fallos: 23 H2, 14 bloques de código idénticos (incluido el Mermaid de §3), 22 tablas con las mismas filas, mismos enlaces, IDs y etiquetas por sección |
| G6 | 0 hallazgos; los objetivos de carga se presentan como objetivos de prueba |
| G7 Diagramas | 16 bloques Mermaid (14 en `diagrams.md` y 1 en cada README), **los 16 idénticos byte a byte** a un bloque de la arquitectura v2 o de aws-target v2: ningún nodo ni flecha añadidos. La validez de sintaxis Mermaid no se pudo comprobar con una herramienta local (no existe); son los diagramas aprobados, y el humano puede confirmar su renderizado en GitHub |

## 5. Unverified claims and dependencies

- Renderizado Mermaid en GitHub: no verificado (sin herramienta local).
- Enlaces absolutos a `SPEC_REPO`: apuntan a archivos versionados en `main` (`git ls-files`); su accesibilidad depende de que el humano publique el repositorio (DOC-IV-006 a lo confirma).
- La topología AWS, Terraform, alarmas y panel se presentan como `Designed, not implemented`.

## 6. DOC-ISSUE items

Sin nuevos. DOC-ISSUE-001, DOC-ISSUE-003, DOC-ISSUE-004 y DOC-ISSUE-006 quedan reflejados en README §15, §17, §19 y en `docs/architecture.md` §5 y §7. DOC-ISSUE-009 aplicado (limitación 15 omitida).

## 7. Deviations

Ninguna. La tabla de decisiones del README (§14) tiene 14 filas; las 16 decisiones completas están en `docs/architecture.md` §2.

## 8. Handoff

Siguiente: DOC-INC-005.
