---
id: ADR-029
title: SQS operational policy for the Orders and Event provisioning queues
status: ACCEPTED
priority: HIGH
supersedes: ADR-010
human_decision: CONFIRMED_WITH_CHANGE
source_ids: [TC-005, TC-009, TC-010, ALT-004, ERR-004, ERR-005, ERR-008, FR-007, FR-017, AC-021, AC-023, AC-024, AC-025, ST-009, BR-002]
related_adrs: [ADR-008, ADR-024, ADR-025, ADR-026, ADR-027, ADR-028, ADR-035, ADR-037, ADR-038, ADR-039]
---

# ADR-029 — SQS operational policy for the Orders and Event provisioning queues

Reemplaza a ADR-010. El detalle operativo está en `ticketing.messaging.v2.md`.

## Human decision applied

Respuesta humana vinculante a ADR-010 (CONFIRMED_WITH_CHANGE), incorporada íntegramente:

> Se confirma para la cola de Orders la cola Standard con DLQ, long polling de 20 s, visibility timeout de 60 s, tope de procesamiento de 30 s, lease de PaymentAttempt de 45 s y un máximo de 5 recepciones con backoff aplicado mediante la visibilidad del mensaje. En la última recepción, si el procesamiento vuelve a fallar, el consumidor cierra la Order como `FAILED` y libera sus Ticket. La DLQ es zona de retención para diagnóstico, con alarma y redrive manual. Se descartan el consumidor automático de la DLQ y la operación sin DLQ. Se añaden dos cambios.
>
> Cola de aprovisionamiento. La cola creada en ADR-004 debe tener su propia política: cola Standard con DLQ propia, visibility timeout de 120 s extendido mediante heartbeat mientras el aprovisionamiento progresa, máximo de 5 recepciones y concurrencia baja por instancia para no competir con la capacidad de las compras. Agotadas las recepciones, el Event queda en `FAILED` conforme a ADR-004.
>
> Capacidad de consumo. Una Order que espera en cola más allá del margen de corte de AV-003 expira sin intento de pago, por lo que la capacidad de consumo debe dimensionarse para que la espera en cola sea muy inferior a la vigencia de la Reservation. La concurrencia por instancia será configurable; se publicará la métrica de antigüedad del mensaje más antiguo de la cola de Orders con alarma a los 2 minutos; el worker escalará automáticamente según los mensajes pendientes; y la prueba de carga debe verificar que ninguna Order expira por espera en cola.
>
> Los valores indicados serán configurables y son los desplegados para esta implementación.

Decisiones aprobadas incorporadas: cola de aprovisionamiento de ADR-024; margen de corte de `AV-003`; marca de reverso al cerrar como `FAILED` con resultado desconocido (`FG-003`, ADR-025); pausa del consumo de Orders con el circuito del Payment Mock abierto (ADR-035); bucles de consumo propios con heartbeat (ADR-039); escalado por antigüedad del mensaje más antiguo (ADR-037); verificación en la prueba de carga (ADR-038).

## Context

`TC-005` fija SQS Standard y `TC-010` procesamiento al menos una vez con idempotencia de aplicación. `ALT-004` y `ERR-005` permiten reintentar fallos transitorios sin duplicar efectos; `ERR-008` y `AC-021` exigen cerrar la Order como `FAILED` ante fallo técnico definitivo. Existen ahora dos flujos asíncronos con perfiles distintos: Orders (muchos mensajes cortos, plazo de diez minutos) y aprovisionamiento (pocos mensajes largos).

## Options considered

### Option A — Cola principal más DLQ de retención por flujo; el consumidor cierra la entidad en la última recepción

- A favor: cierre inmediato al agotar reintentos; DLQ como evidencia; redrive inocuo; políticas independientes por flujo.
- En contra: una caída justo en la última recepción deja el cierre a la expiración o a la detección de estancados.

### Option B — Consumidor automático de la DLQ que cierra la entidad

- En contra: los mensajes venenosos reciclarían; se pierde la DLQ como evidencia. Descartada.

### Option C — Sin DLQ

- En contra: los mensajes venenosos bloquean capacidad indefinidamente. Descartada.

### Option D — Una sola cola para Orders y aprovisionamiento

- A favor: menos recursos.
- En contra: un aprovisionamiento largo comparte visibilidad y concurrencia con las compras; la decisión humana exige cola propia. Descartada.

## Decision

Se adopta la **Option A** con dos colas independientes. Valores configurables desplegados:

| Parámetro | `ticketing-orders` | `ticketing-event-provisioning` |
|---|---|---|
| Tipo | Standard | Standard |
| DLQ | `ticketing-orders-dlq` | `ticketing-event-provisioning-dlq` |
| Long polling | 20 s, hasta 10 mensajes | 20 s, 1 mensaje |
| Visibility timeout | 60 s | 120 s, extendido por heartbeat cada 30 s mientras progresa |
| Tope de procesamiento por mensaje | 30 s | Sin tope fijo mientras haya progreso; el heartbeat se detiene si no hay progreso en 60 s |
| Lease | PaymentAttempt, 45 s | Event, 60 s renovado antes de cada lote |
| `maxReceiveCount` | 5 | 5 |
| Backoff entre recepciones | 5 s, 15 s, 30 s, 60 s con jitter, por cambio de visibilidad | 30 s, 60 s, 120 s, 240 s con jitter |
| Concurrencia por instancia | Configurable, 16 por defecto | 1 |
| Retención principal / DLQ | 1 hora / 14 días | 1 día / 14 días |
| Cierre en la última recepción | Order → `FAILED` (`PROCESSING_FAILED`), libera Ticket, marca de reverso si el resultado del pago es desconocido | Event → `FAILED` (`PROVISIONING_FAILED`) |
| Productores | `api` (compra), `worker` (barrido) | `api` (creación), `worker` (detección de estancados) |

Relación de tiempos de Orders: tope de procesamiento (30 s) < lease (45 s) < visibility timeout (60 s).

### Clasificación de resultados (Orders)

| Situación | Clase | Mensaje | Order |
|---|---|---|---|
| Procesado hasta estado terminal | Éxito | Eliminar | El que corresponda |
| Order terminal o en cuarentena | Duplicado, tardío o en revisión | Eliminar | Ninguno |
| Lease vigente de otro consumidor | Duplicado concurrente | Posponer visibilidad hasta el fin del lease | Ninguno |
| Tiempo restante inferior al margen de corte | No procesable a tiempo | Eliminar (si ya venció, ejecutar Expirar) | Se cierra por expiración |
| Payment Mock rechaza | Definitivo funcional | Eliminar | `REJECTED` |
| Error de contrato o autenticación del Payment Mock | Definitivo técnico | Eliminar | `FAILED`, sin reverso |
| Timeout, conexión, 5xx, circuito abierto del Payment Mock; throttling, conflicto o indisponibilidad de DynamoDB | Transitorio | No eliminar; backoff | Ninguno |
| Transitorio en la última recepción | Definitivo por agotamiento | No eliminar; pasa a la DLQ | `FAILED`, con reverso si había PaymentAttempt de resultado desconocido |
| Mensaje ilegible, versión desconocida, Order inexistente | Venenoso | No eliminar; visibilidad corta | Ninguno |
| Fallo de condición | Carrera resuelta | Releer y reevaluar | El que haya ganado |

### Qué ocurre con la Order cuando su mensaje termina en la DLQ

- Normal: ya está en `FAILED` (cerrada en la última recepción).
- Caída en la última recepción o DynamoDB indisponible: sigue en `CREATED` y la expiración la cierra como `EXPIRED` (con reverso si tenía PaymentAttempt).
- Venenoso: no hay Order. Alarma sobre la profundidad de la DLQ; redrive manual inocuo.

### Qué ocurre con el Event cuando su mensaje termina en la DLQ

- Normal: ya está en `FAILED` y la limpieza purga sus Ticket.
- Caída en la última recepción: sigue en `PROVISIONING`; la detección de estancados lo republica o lo marca `FAILED` (ADR-024).

### Capacidad de consumo de Orders

- Concurrencia por instancia configurable (16 por defecto).
- Métrica de antigüedad del mensaje más antiguo de la cola de Orders con alarma a los 2 minutos.
- Autoescalado del worker por mensajes pendientes por tarea y por esa antigüedad (ADR-037).
- La prueba de carga verifica que ninguna Order expira por espera en cola (ADR-038): ninguna Order `EXPIRED` sin PaymentAttempt habiendo sido encolada.

### Pausa del consumo

Con el circuito del Payment Mock abierto, el bucle de Orders deja de recibir (no consume recepciones de `maxReceiveCount`); en semiabierto recibe solo los mensajes de prueba permitidos; al cerrarse reanuda (ADR-035). El bucle de aprovisionamiento no depende del Payment Mock.

## Rationale

- El backoff por visibilidad sobrevive a la caída del consumidor.
- Cerrar en la última recepción libera inventario en minutos y cumple `AC-021`.
- Separar las colas evita que un aprovisionamiento largo consuma visibilidad y concurrencia de las compras.
- El heartbeat permite un visibility timeout corto sin cortar un aprovisionamiento que progresa.

## Consequences

- Un duplicado concurrente consume una recepción.
- Doble fallo en la última recepción: `EXPIRED` en lugar de `FAILED` (`RISK-014`).
- El consumidor necesita el contador aproximado de recepciones y el cambio de visibilidad.
- Durante una caída del Payment Mock la antigüedad de la cola crece y dispara el autoescalado sin efecto útil (acotado por el máximo de tareas, ADR-037).

## Risks and mitigations

| Risk | Mitigation |
|---|---|
| `RISK-014` mensajes en DLQ sin atender | Alarmas por profundidad de ambas DLQ; registro estructurado; redrive inocuo |
| Order que expira por espera en cola | Concurrencia configurable, alarma a los 2 minutos, autoescalado, verificación en la carga |
| Tormenta de reintentos ante un proveedor degradado | Circuit breaker y pausa del consumo (ADR-035), backoff con jitter |

## Items to verify

| Item | Qué comprobar | Impacto si difiere |
|---|---|---|
| Límites de SQS | Espera máxima de long polling, mensajes por recepción, visibilidad máxima, retención mínima y máxima | Ajuste de parámetros |
| LocalStack fijado a un tag anterior a la exigencia de token (ADR-036) | SQS Standard, redrive a DLQ, contador aproximado de recepciones, cambio de visibilidad | Token gratuito por archivo no versionado o ElasticMQ, previa revisión de `HV-010` |

## Depends on

- `AV-003`, `FG-003` (resueltos).
