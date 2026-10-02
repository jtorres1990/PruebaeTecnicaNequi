---
artifact: messaging
schema_version: 1.0
feature: ticketing-event-processing
version: 1
source:
  feature_spec: feature-spec/ticketing.feature-spec.v4.md
  architecture: architecture/ticketing.architecture.md
status: READY_FOR_HUMAN_ARCHITECTURE_REVIEW
generated_at: 2026-10-01
---

# Messaging — Ticketing Event Processing

Decisiones que gobiernan este documento: [ADR-006](adr/ADR-006-persistence-plus-enqueue.md), [ADR-007](adr/ADR-007-idempotency.md), [ADR-008](adr/ADR-008-payment-versus-expiration-race.md), [ADR-010](adr/ADR-010-sqs-operational-policy.md), [ADR-020](adr/ADR-020-aws-integration-technology.md). Este documento no repite sus alternativas ni su justificación.

Los valores numéricos son valores iniciales configurables. Los límites del servicio están marcados `TO_VERIFY` en ADR-010.

## 1. Queues

| Cola | Tipo | Propósito | Productor | Consumidor | Source IDs |
|---|---|---|---|---|---|
| `ticketing-orders` | SQS Standard | Trabajo asíncrono de procesamiento de una Order | `CMP-011` (rol `api`) | `CMP-012` (rol `worker`) | TC-005, FR-005, FR-007 |
| `ticketing-orders-dlq` | SQS Standard | Retención de mensajes que agotaron sus recepciones | SQS, por política de redrive | Ninguno automático; inspección y redrive manual | TC-010, ERR-005, ERR-008 |

Los nombres físicos se parametrizan por entorno. En local las colas viven en LocalStack; en AWS en Amazon SQS, con cifrado en reposo.

| Atributo | `ticketing-orders` | `ticketing-orders-dlq` |
|---|---|---|
| Visibility timeout | 60 s | 60 s |
| Espera de recepción (long polling) | 20 s | No aplica |
| Retención | 1 hora | 14 días |
| Política de redrive | Hacia la DLQ con `maxReceiveCount = 5` | — |
| Retardo de entrega | 0 | 0 |

## 2. Message contracts

### MSG-001

`OrderProcessingRequested`, versión de esquema 1.

- Producer: `CMP-005` (caso de uso de inicio de compra) mediante `CMP-011` (adaptador publicador de SQS), después del commit de la transacción de reserva. También en la ruta de repetición cuando la Order está en `CREATED` sin `enqueuedAt` (ADR-006).
- Consumer: `CMP-012` (adaptador consumidor de SQS), que invoca a `CMP-007` (caso de uso de procesamiento de Order).
- Schema:

  Cuerpo (JSON):

  | Campo | Tipo | Obligatorio | Descripción |
  |---|---|---|---|
  | `schemaVersion` | integer | Sí | Valor `1` |
  | `messageType` | string | Sí | Valor `OrderProcessingRequested` |
  | `orderId` | string (UUID) | Sí | Order a procesar |
  | `eventId` | string (UUID) | Sí | Event de la Order; informativo, para diagnóstico |
  | `occurredAt` | string (instante UTC) | Sí | Instante de creación de la Order |
  | `correlationId` | string | Sí | Identificador de correlación de la solicitud original |

  Atributos del mensaje:

  | Atributo | Descripción |
  |---|---|
  | `messageType` | Igual que en el cuerpo; permite descartar sin deserializar |
  | `schemaVersion` | Igual que en el cuerpo |
  | Contexto de traza | Propagación de la traza distribuida |

  Reglas del contrato:

  - El mensaje es una referencia: no transporta Ticket, estado ni datos del cliente. El consumidor siempre lee la Order como fuente de verdad (`AP-010`).
  - No contiene datos personales, tokens ni el identificador del cliente.
  - Evolución: los campos nuevos son opcionales; un cambio incompatible incrementa `schemaVersion`. Un consumidor que recibe una versión desconocida trata el mensaje como venenoso.

- Idempotency identity: `orderId`. El identificador de mensaje de SQS no es la identidad de la operación, porque cambia en cada republicación (ADR-007).
- Source IDs: FR-005, FR-007, FR-017, TC-005, TC-010, MF-003, AC-003, AC-005, AC-023.

No existen otros contratos de mensaje en esta versión. Si se aprueba FG-003, el reverso de pago se resuelve con una marca en la Order y un índice de trabajo (ADR-008), sin un mensaje nuevo.

## 3. Delivery and retry policy

| Aspecto | Política | Source IDs |
|---|---|---|
| Semántica de entrega | Al menos una vez; el orden no está garantizado y no se depende de él | TC-010 |
| Publicación | 3 intentos en total con backoff y jitter; después, fallo definitivo de encolado y Order `FAILED` (ADR-006) | FR-005, ALT-006, ERR-007, AC-022 |
| Recepción | Long polling de 20 s, hasta 10 mensajes por recepción | TC-005 |
| Concurrencia | Hasta 16 mensajes en procesamiento por instancia; no se reciben más mensajes de los que se pueden procesar | NFR-003 |
| Confirmación | El mensaje se elimina solo después de que el procesamiento alcanzó un resultado estable (ver §5) | TC-010 |
| Reintento | Por reentrega. Al fallar de forma transitoria, la visibilidad del mensaje se fija al siguiente escalón de backoff: 5 s, 15 s, 30 s, 60 s, con jitter | ALT-004, ERR-005 |
| Máximo de recepciones | 5; a la sexta, SQS mueve el mensaje a la DLQ | ERR-005, ERR-008 |
| Tope de procesamiento | 30 s por mensaje | — |
| Relación de tiempos | Tope de procesamiento (30 s) < lease de PaymentAttempt (45 s) < visibility timeout (60 s) | BR-020, AC-024 |
| Presupuesto total | Todos los reintentos terminan en pocos minutos, por debajo de los diez de la Reservation | BR-002 |

## 4. Dead-letter handling

| Aspecto | Política |
|---|---|
| Cuándo llega un mensaje a la DLQ | Tras superar 5 recepciones sin ser eliminado |
| Estado de la Order en ese momento | Caso normal: ya está en `FAILED`, porque el consumidor la cerró en la quinta recepción y liberó sus Ticket (`ST-009`, `AC-021`). Caso de caída del consumidor en la quinta recepción o de indisponibilidad de la persistencia: sigue en `CREATED` y el proceso de expiración la cierra como `EXPIRED` (`ST-010`). En ningún caso queda una Reservation indefinida |
| Mensajes venenosos | No corresponden a ninguna Order procesable; llegan a la DLQ sin efectos |
| Consumo de la DLQ | No hay consumidor automático. La DLQ es zona de retención para diagnóstico |
| Señal operativa | Alarma cuando la DLQ contiene mensajes; cada recepción fallida registra el motivo de forma estructurada |
| Redrive | Manual, de la DLQ a la cola principal. Es inocuo: el consumidor es idempotente y una Order terminal no produce efectos (`AC-025`) |
| Retención | 14 días |

## 5. Consumer processing rules

Reglas de `CMP-007` para cada mensaje recibido. El orden es significativo.

| # | Condición | Acción | Mensaje | Source IDs |
|---|---|---|---|---|
| 1 | Mensaje ilegible, `messageType` o `schemaVersion` desconocidos, o sin `orderId` | Registrar como venenoso | No eliminar; visibilidad corta | ERR-005 |
| 2 | La Order no existe (lectura consistente) | Registrar como venenoso | No eliminar; visibilidad corta | ERR-005 |
| 3 | La Order es terminal | Ninguna | Eliminar | AC-025, FR-017 |
| 4 | Order `CREATED`, sin PaymentAttempt, y el tiempo restante hasta `expiresAt` es inferior al margen de corte | Ninguna; si `expiresAt` ya pasó, ejecutar "Expirar" | Eliminar | BR-002, BR-003, ADR-008 |
| 5 | Order `CREATED`, sin PaymentAttempt | Ejecutar "Iniciar pago" (`ST-003`). Si la condición falla, releer y reevaluar desde la regla 3 | Continuar | AC-005, BR-020 |
| 6 | Order `CREATED`, con PaymentAttempt y lease vigente de otro consumidor | Ninguna | No eliminar; posponer la visibilidad hasta el fin del lease | AC-023, AC-024 |
| 7 | Order `CREATED`, con PaymentAttempt y lease vencido | Reclamar el lease de forma condicional; si se obtiene, continuar con el mismo PaymentAttempt; si no, aplicar la regla 6 | Continuar | AC-023, ALT-004 |
| 8 | Se posee el lease | Invocar el Payment Mock con el `paymentAttemptId` como clave de idempotencia, con plazo limitado por `expiresAt` | Continuar | FR-015, AC-024 |
| 9 | Resultado aprobado | Ejecutar "Confirmar" (`ST-004`, `ST-007`). Si falla por condición, releer: si la Order es terminal o `expiresAt` venció, aplicar ADR-008 | Eliminar | AC-019 |
| 10 | Resultado rechazado | Ejecutar "Rechazar" (`ST-005`, `ST-008`) | Eliminar | AC-020, ALT-005 |
| 11 | Error definitivo del proveedor | Ejecutar "Fallar" (`ST-005`, `ST-009`) | Eliminar | AC-021, ERR-008 |
| 12 | Error transitorio y quedan recepciones | Ninguna transición | No eliminar; backoff | ALT-004, ERR-005 |
| 13 | Error transitorio en la última recepción permitida | Ejecutar "Fallar" (`ST-005`, `ST-009`) | No eliminar; SQS lo mueve a la DLQ | AC-021, ERR-008 |

Reglas transversales:

- Un fallo de condición en cualquier transición no es un error: indica que otra transición ganó. El consumidor relee la Order y reevalúa.
- El consumidor nunca decide por el contenido del mensaje más allá de `orderId`.
- La regla 13 solo se aplica cuando el fallo es del propio procesamiento; si el lease pertenece a otro consumidor se aplica la regla 6.
- Las reglas 9 y 13 con un pago aprobado o de resultado desconocido dependen de FG-003.
- El margen de corte de la regla 4 depende de AV-003.
- Al apagar, el consumidor deja de recibir y termina los mensajes en vuelo; los no terminados reaparecen tras el visibility timeout.
