---
artifact: documentation-increment-report
increment: DOC-INC-005
result: DONE
code_revision: "CODE_REPO main 90f8ed2 + cambios sin commit de este agente (README.md, README.en.md, docs/**, postman/ticketing-demo.*)"
spec_revision: "SPEC_REPO main 9b78fdd"
verified_at: 2026-10-05
environment: "Stack completo de Compose; acciones autorizadas por DOC-IV-007 (a): kill/start de ticketing-worker, stop/start de payment-mock, pause/unpause de localstack, solo en el proyecto ticketing-platform"
---

# DOC-INC-005 — Guía de demostración, resiliencia documentada y solución de problemas

## 1. Documents produced

| Archivo | Cambio |
|---|---|
| `docs/resilience.md` | Nuevo. Preparación común y los tres procedimientos de ADR-038 (detener `ticketing-worker` a mitad de un pago, detener `payment-mock`, detener `localstack`), cada uno con lo que se espera según ADR-038, comandos ejecutables, señales observadas en la verificación y restauración. Estado de los tres: **`NOT_YET_VALIDATED_BY_QA`** |
| `docs/demo-guide.md` | Nuevo. Preparación previa, orden de la presentación (12 bloques con material e idea clave), comandos de comprobación, 15 preguntas previsibles con respuesta breve y fuente, plan de contingencia sin Docker (sin salidas inventadas) y riesgos durante la demostración |
| `README.md`, `README.en.md` | §2 (fila de resiliencia con su estado), §18 (mecanismos con estado y los tres escenarios con lo esperado, lo observado y `NOT_YET_VALIDATED_BY_QA`), §22 (filas nuevas: restos del perfil `load`, 503 por circuito abierto, Orders en `CREATED` con el mock caído) |
| `docs/operations.md` | §6 enlaza la lectura de métricas del `worker`; §10 con tres fallos observados nuevos (503 con circuito abierto, Orders en `CREATED`, `localstack` `unhealthy` justo después de `unpause`) |
| `docs/collection-guide.md` | Enlace al procedimiento 3 |
| `docs/traceability.md` | EVAL-005 y EVAL-009 |

## 2. Sources and traceability

ADR-038 (escenarios 1 a 3), ADR-035, ADR-029, ADR-028, ADR-027; PLAT-IV-011 y PLAT-INC-007 §3 (pausa frente a reinicio de LocalStack); INC-009 (pausa de reversos con tiempo virtual); INC-010 §6.6 (señales para QA); colección de DOC-INC-003. Respuesta humana: DOC-IV-007 (a).

## 3. Commands and scenarios verified

Todas las acciones se limitaron al proyecto `ticketing-platform`. Resultado de cada una: **comando verificado; escenario `NOT_YET_VALIDATED_BY_QA`**.

| Procedimiento | Primera ejecución (scripts de exploración) | Segunda ejecución: bloques `bash` de `docs/resilience.md` extraídos y ejecutados tal cual (exit 0, 158 s) |
|---|---|---|
| Lectura de métricas del `worker` desde la red (`docker compose run --rm --no-deps -T --entrypoint sh infra-init -c 'curl … ticketing-worker:8081/actuator/prometheus'`, con `MSYS_NO_PATHCONV=1`) | `ticketing_circuit_state` de `payment-mock` y `sqs-publication` legibles | Igual |
| 1. `docker compose kill ticketing-worker` con una regla `LATENCY` de 2,5 s, a 1 s de la compra; `docker compose start ticketing-worker` | Tras el `kill`, `API-108`: 1 invocación sin resultado; Order `CONFIRMED` a los 62 s; `API-108`: 2 invocaciones del mismo `paymentAttemptId`, un único `APPROVED`; `worker` `healthy` | Igual: `CONFIRMED`, 2 invocaciones, un único resultado |
| 2. `docker compose stop payment-mock`, 5 compras de sujetos distintos, `docker compose up -d --wait payment-mock` | Circuito `payment-mock` `OPEN` a los 10 s; log `circuit.opened`; las 5 Orders `CREATED` a los 10 y a los 31 s; mock reiniciado sin reglas; las 5 `CONFIRMED` a los 76 s de detenerlo; circuito `CLOSED` | Circuito `OPEN`; 5 `CREATED`; 5 `CONFIRMED`; `CLOSED`. El recuento por estado se mostraba concatenado (faltaba un salto de línea en el helper `order_status`); corregido en el documento y comprobado después con un caso aislado |
| 3. `docker compose pause localstack`, 6 compras, `docker compose unpause localstack` | 4 compras 201 `FAILED` (`PROCESSING_UNAVAILABLE`, unos 1,8 s cada una), luego 503 `SERVICE_UNAVAILABLE` con `Retry-After: 10` en 8 ms; disponibilidad 10/10 antes y durante; `/readyz` 200; circuito `sqs-publication` `OPEN`; tras `unpause`, 201 `CREATED` a los 8 s; circuito `HALF_OPEN`; `localstack` `unhealthy` unos segundos y luego `healthy` | 2 compras 201 y 4 respuestas 503 con `Retry-After: 10`; `availableCount` igual (4) antes y durante; `/readyz` 200; 201 tras recuperar |
| Estado final | `docker compose ps`: todo `healthy`, `infra-init` `Exited (0)` (se vuelve a ejecutar al hacer `start` del `worker`) | Igual; mock restablecido (204) |

No observados (y así documentados): expiración y pausa de reversos durante la caída del mock, porque no había Reservation vencidas ni reversos pendientes; están verificados en las pruebas del backend con tiempo virtual.

## 4. Static validation

| Gate | Resultado |
|---|---|
| G1 | 0 fallos |
| G2 | 0 fallos en entregables y en las salidas guardadas de los procedimientos (sin tokens) |
| G3 | 0 fallos (sin cambios en la colección ni en los curl de los README) |
| G4 | 0 fallos: 23 H2, 14 bloques de código idénticos, 24 tablas |
| G6 | 0 hallazgos: los tres escenarios están rotulados `NOT_YET_VALIDATED_BY_QA` en `resilience.md` y README §2 y §18; ninguna afirmación `PASS` ni de validación de QA; los objetivos de carga no se presentan como SLA |

## 5. Unverified claims and dependencies

- Validación formal de los tres escenarios, repetición bajo carga e invariantes posteriores: QA (DOC-ISSUE-006).
- La guía de demostración no se ensayó en una reunión real; usa solo material verificado en DOC-INC-002 a DOC-INC-005. La duración del bloque curl de README §11 se mide en DOC-INC-006.

## 6. DOC-ISSUE items

Sin nuevos. DOC-ISSUE-006 sigue abierto (QA).

## 7. Deviations

- El procedimiento 1 usa `docker compose kill` (no `stop`), como preveía el plan §6.3, porque `stop` hace un apagado ordenado que termina el pago en curso.
- El procedimiento 2 restaura con `docker compose up -d --wait payment-mock` en lugar de `start`, para esperar a que el mock esté sano.

## 8. Handoff

Siguiente: DOC-INC-006. QA: `docs/resilience.md` como punto de partida de su validación.
