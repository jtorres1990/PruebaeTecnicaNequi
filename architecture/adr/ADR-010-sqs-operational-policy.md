---
id: ADR-010
title: SQS operational policy
status: PROPOSED
priority: HIGH
source_ids: [TC-005, TC-009, TC-010, ALT-004, ERR-004, ERR-005, ERR-008, FR-007, FR-017, AC-021, AC-023, AC-024, AC-025, ST-009]
related_adrs: [ADR-006, ADR-007, ADR-008, ADR-009, ADR-016, ADR-020]
---

# ADR-010 — SQS operational policy

## Context

`TC-005` fija Amazon SQS Standard y `TC-010` exige procesamiento al menos una vez con idempotencia de aplicación. `ALT-004` y `ERR-005` permiten reintentar fallos transitorios sin duplicar pagos ni transiciones; `ERR-008` y `AC-021` exigen cerrar la Order como `FAILED` ante un fallo técnico definitivo. La especificación deja a arquitectura visibility timeout, polling, reintentos, DLQ y clasificación de errores (§19, §31).

El contrato del mensaje y el detalle operativo están en `ticketing.messaging.md`; este ADR registra la decisión.

## Options considered

### Option A — Cola principal más DLQ como zona de retención; el consumidor cierra la Order en su último intento

Reintentos por reentrega de SQS con backoff aplicado cambiando la visibilidad. En la última recepción permitida, si el procesamiento vuelve a fallar, el propio consumidor ejecuta la transición a `FAILED`. La DLQ conserva el mensaje para diagnóstico y tiene alarma; no se consume automáticamente.

- A favor: la Order se cierra en cuanto se agotan los reintentos; la DLQ conserva evidencia; el redrive manual es inocuo porque el consumidor es idempotente; solo hay un consumidor que mantener.
- En contra: si el consumidor muere justo en el último intento, la Order no pasa a `FAILED` sino que se cierra por expiración.

### Option B — Consumidor automático de la DLQ que cierra la Order

Un segundo listener sobre la DLQ ejecuta la transición a `FAILED` y elimina el mensaje.

- A favor: `FAILED` garantizado incluso si el consumidor principal muere en el último intento.
- En contra: los mensajes venenosos no resolubles a una Order quedarían reciclándose en la DLQ o habría que añadir una tercera cola; se pierde la DLQ como evidencia; un segundo consumidor que operar y probar.

### Option C — Sin DLQ, reintento hasta el fin de la retención

- A favor: mínima configuración.
- En contra: los mensajes venenosos bloquean capacidad de consumo indefinidamente y no hay señal operativa. No cumple la clasificación entre transitorio y definitivo que pide `ALT-004`.

## Decision

Se adopta la **Option A**. Valores iniciales configurables:

| Parámetro | Valor | Motivo |
|---|---|---|
| Tipo de cola | Standard | `TC-005` |
| Cola principal / DLQ | `ticketing-orders` / `ticketing-orders-dlq` (nombres parametrizados por entorno) | — |
| Polling | Long polling de 20 s, hasta 10 mensajes por recepción | Menos recepciones vacías |
| Concurrencia de procesamiento | 16 mensajes en vuelo por instancia, acotada (backpressure) | `NFR-003` |
| Visibility timeout | 60 s | Superior al tope de procesamiento |
| Tope de procesamiento por mensaje | 30 s | Presupuesto de pago más escrituras |
| Lease de PaymentAttempt | 45 s | Mayor que el tope de procesamiento y menor que el visibility timeout: cuando un mensaje reaparece tras una caída, el lease ya venció |
| Reintentos | `maxReceiveCount = 5` | Cierra en pocos minutos, muy por debajo de los diez de la Reservation |
| Backoff entre recepciones | 5 s, 15 s, 30 s, 60 s, con jitter, aplicado cambiando la visibilidad del mensaje al fallar | `ALT-004` |
| Retención de la cola principal | 1 hora | Un mensaje más antiguo que la Reservation ya no tiene efecto |
| Retención de la DLQ | 14 días | Diagnóstico |
| Redrive | Política de redrive de la cola principal hacia la DLQ al superar `maxReceiveCount`; redrive manual de la DLQ a la principal permitido | — |

### Clasificación de resultados del procesamiento

| Situación | Clase | Acción sobre el mensaje | Efecto sobre la Order |
|---|---|---|---|
| Procesado hasta estado terminal | Éxito | Eliminar | El que corresponda |
| Order ya terminal | Duplicado o tardío | Eliminar | Ninguno (`AC-025`) |
| PaymentAttempt activo con lease vigente de otro consumidor | Duplicado concurrente | No eliminar; posponer visibilidad hasta el fin del lease | Ninguno (`AC-024`) |
| Tiempo restante inferior al margen de corte | No procesable a tiempo | Eliminar | Ninguno; se cierra por expiración (`ADR-008`) |
| Payment Mock rechaza | Definitivo funcional | Eliminar | `REJECTED` |
| Payment Mock responde error de contrato o de autenticación | Definitivo técnico | Eliminar | `FAILED` |
| Timeout, conexión, indisponibilidad o saturación del Payment Mock; throttling, conflicto o indisponibilidad de DynamoDB | Transitorio | No eliminar; backoff | Ninguno hasta agotar reintentos |
| Transitorio en la última recepción permitida | Definitivo técnico por agotamiento | No eliminar; SQS lo mueve a la DLQ | `FAILED` (`AC-021`) |
| Mensaje ilegible, versión de esquema desconocida u Order inexistente | Venenoso | No eliminar; visibilidad corta para que llegue pronto a la DLQ | Ninguno; no hay Order que cerrar |
| Fallo de condición en una transición | Carrera resuelta | Releer la Order y reevaluar | El que haya ganado |

### Qué ocurre con la Order cuando su mensaje termina en la DLQ

- Caso normal: el consumidor ya la cerró como `FAILED` en la última recepción, liberando sus Ticket, antes de que SQS moviera el mensaje.
- Caso de caída en la última recepción, o indisponibilidad de DynamoDB que impide la transición: la Order sigue en `CREATED` y el proceso de expiración la cierra como `EXPIRED` (`ADR-009`). No queda una Reservation indefinida.
- Mensaje venenoso: no corresponde a ninguna Order.
- El mensaje permanece en la DLQ para diagnóstico. Una alarma se dispara cuando la DLQ tiene mensajes. Un redrive manual no produce efectos porque la Order ya es terminal.

## Rationale

- Aplicar el backoff con la visibilidad del mensaje mantiene el reintento en SQS y no en memoria: sobrevive a la caída del consumidor.
- Cerrar la Order en la última recepción libera el inventario en minutos en lugar de esperar al vencimiento, y cumple `AC-021`.
- Mantener la DLQ como zona de retención da una señal operativa clara y conserva los mensajes venenosos sin lógica adicional.
- La relación entre tope de procesamiento, lease y visibility timeout garantiza que un mensaje reentregado tras una caída pueda reanudarse sin esperar ni solaparse.

## Consequences

- Un duplicado concurrente consume una recepción del presupuesto de `maxReceiveCount`.
- En el caso de doble fallo, el estado final es `EXPIRED` en lugar de `FAILED` (limitación declarada, `RISK-014`).
- El consumidor necesita el contador aproximado de recepciones del mensaje.
- Los valores numéricos son puntos de partida; deben ajustarse con la prueba de carga.

## Risks and mitigations

| Risk | Mitigation |
|---|---|
| `RISK-014` mensajes en DLQ sin atender | Alarma por profundidad de la DLQ, registro estructurado del motivo, procedimiento de redrive |
| Reintentos que superan la vigencia de la Reservation | Presupuesto total de reintentos de pocos minutos; el proceso de expiración es la red de seguridad |
| Proveedor de pago degradado genera una tormenta de reintentos | Backoff con jitter, concurrencia acotada; evolución: circuit breaker en el adaptador de pago |

## Items to verify

| Item | Qué comprobar | Impacto si difiere |
|---|---|---|
| Límites de SQS: espera máxima de long polling, mensajes por recepción, visibilidad máxima, retención mínima | Valores vigentes | Ajuste de parámetros |
| Atributo de contador aproximado de recepciones y política de redrive en LocalStack | Soporte en la edición usada | Si el contador no está disponible en local, la regla de "última recepción" no puede probarse en local y el cierre queda a cargo de la expiración |
| SQS Standard en LocalStack sin licencia | Disponibilidad | Si no, se requiere otro emulador compatible con la API de SQS |

## Depends on

- `FG-003`: cierre como `FAILED` con PaymentAttempt de resultado desconocido.
- `AV-003`: margen de corte.
