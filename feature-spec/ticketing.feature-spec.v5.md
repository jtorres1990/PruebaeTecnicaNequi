---
artifact: feature-spec
schema_version: 1.0
feature: ticketing-event-processing
version: 5

agent:
  name: requirements-analyst
  version: 1.0

source:
  requirement: requirements/Prueba2026.md
  previous_specification:
    artifact: feature-spec/ticketing.feature-spec.v4.md
    version: 4
  human_review:
    artifact: human-review/ticketing.functional-review.yaml
    status: APPROVED
    decisions_incorporated: 21
  manual_clarification:
    id: HC-001
    reviewed_by: human
    reviewed_at: 2026-10-01
    decision: >
      Si cualquier ticket no está disponible durante la reserva inicial, la
      solicitud se rechaza sin crear Order ni retornar Order ID. REJECTED se
      reserva para una Order ya creada que recibe un rechazo funcional posterior.
  architecture_review:
    artifact: human-review/ticketing.architecture-review.yaml
    status: APPROVED
    reviewed_at: 2026-10-04
    role: Fuente vinculante de las respuestas humanas que originan las aclaraciones funcionales.
  functional_clarifications:
    artifact: architecture/ticketing.functional-clarifications.v1.md
    version: 1
    entries_total: 15
    entries_incorporated: 14
    entries_incorporated_ids: [1, 2, 3, 4, 5, 6, 7, 8, 9, 10, 11, 12, 13, 14]
    entries_conditional_not_applied: 1
    entries_conditional_ids: [15]

status: READY_FOR_ARCHITECTURE
human_validation_required: false
open_questions: 0
blocking_questions: 0
architecture_can_start: true

generated_at: 2026-10-04
---

# Feature Specification — Ticketing Event Processing

## 0. Cambios respecto a v4

Esta versión parte de la v4 completa y aplica las entradas 1 a 14 de `architecture/ticketing.functional-clarifications.v1.md`, todas originadas en respuestas humanas aprobadas en `human-review/ticketing.architecture-review.yaml`. No es un análisis nuevo: no se añaden decisiones, reglas ni supuestos que no estén en esas aclaraciones, en la v4 o en las respuestas humanas.

Convenciones de esta tabla:

- **(mod)**: el elemento existente conserva su ID y su texto cambia.
- **(new)**: elemento nuevo con el siguiente ID libre de su tipo.
- **(rev)**: elemento revisado sin cambios de texto; se lista para verificar la propagación.
- Ningún ID de la v4 se eliminó ni se reutilizó. Ningún elemento quedó reemplazado.

| # | Aclaración (ID de origen) | Elementos de la spec creados / modificados | Resumen del cambio |
|---|---|---|---|
| 1 | FG-001 (también ADR-003) | §5 `Order` e invariante 3 (mod); ST-001, ST-006 (mod); MF-003 (mod); §10 Iniciar compra (mod); FR-004 (mod); BR-014 (mod); VAL-012 (new); ERR-010 (new); AC-004 (mod); AC-032 (new); AC-016 (rev); §13.1 valores configurables (new) | Una Order solicita entre 1 y 10 Ticket del mismo Event, sin identificadores repetidos. 0, más de 10 o repetidos producen error de validación antes de reservar, sin Reservation, Order ni Order ID y sin cambiar el inventario. Máximo configurable; desplegado 10. |
| 2 | FG-002 | VAL-008 (mod); FR-001, FR-012 (mod); BR-026 (new, compartido con 6 y 11); ST-012 (new, compartido con 6); ERR-011 (new, compartido con 6); AC-001, AC-010, AC-017 (mod); AC-033 (new); §13.1 | Capacidad máxima de 50.000 Ticket por Event; superarla rechaza toda la creación sin crear el Event ni sus Ticket. El Event solo queda habilitado tras comprobar que los Ticket creados coinciden con `Event.capacity`. La disponibilidad es paginada. Límite configurable con nuevas pruebas de capacidad. |
| 3 | FG-003 (también ADR-008) | §4 Payment Mock (mod) y Proceso de reverso de pagos (new); §5 `PaymentAttempt` (mod) e invariante 15 (new); DS-009, DS-010 (mod); ST-007, ST-009, ST-010 (mod); CAP-005 (mod); CAP-009 (new); MF-004 (mod); MF-005 (new); §10 Procesar pago, Consultar Order (mod), Reversar pago (new); FR-009, FR-015, FR-017 (mod); FR-023 (new); BR-003 (mod); BR-028, BR-034 (new); ALT-001 (mod); ALT-008, ALT-009 (new); ERR-008 (mod); ERR-016, ERR-017 (new); AC-008, AC-019, AC-021, AC-025 (mod); AC-034, AC-035 (new); §18, §19 (mod) | Una Order que termina sin confirmarse (`EXPIRED`, o `FAILED` con resultado de pago desconocido) conserva su estado terminal y sus Ticket liberados; una aprobación tardía nunca la reabre ni la confirma. El sistema solicita la cancelación del PaymentAttempt con reintento acotado; si los agota, alerta y queda para revisión manual. Cancelación idempotente por `paymentAttemptId`, válida en cualquier estado; una cancelación anterior al cobro hace rechazar el cobro posterior. La marca de reverso no se expone. |
| 4 | FG-004 | FR-017 (mod); BR-019 (mod); ALT-003 (mod); §18 (mod); AC-036 (new); AC-016, AC-023 (rev) | Un rechazo síncrono no persiste Reservation, Order ni registro de idempotencia. Una repetición con la misma idempotency key se procesa como solicitud nueva y puede crear una Order; en ese caso la clave queda registrada con esa Order. |
| 5 | FG-005 | §5 invariante 16 (new); ST-001 (mod); MF-002, MF-003 (mod); FR-002 (mod); FR-022 (new); BR-022 (mod); BR-025 (new); VAL-014 (new); ALT-012 (new); ERR-012, ERR-013 (new); AC-002 (mod); AC-037, AC-038, AC-051 (new) | Un Event es pasado si `startsAt` es menor o igual al instante actual del servidor en UTC al recibir la compra. La compra se rechaza con `EVENT_NOT_ON_SALE` (HTTP 409) sin crear nada. La disponibilidad por identificador de un Event pasado sigue respondiendo de forma informativa. Una Reservation creada antes de `startsAt` puede confirmarse dentro de su vigencia. Un Event en `PROVISIONING` o `FAILED` se trata como inexistente. |
| 6 | ADR-004 (sucesor ADR-024) | §3.1 (mod); §5 `Event`, `Ticket` (mod) e invariantes 12 y 14 (new); §7A (new): DS-011, DS-012, DS-013, ST-011, ST-012, ST-013; DS-005 y nota de §6.2 (mod); CAP-001 (mod); MF-001 (mod); §10 Crear Event (mod) y Consultar estado de aprovisionamiento (new); FR-001, FR-003 (mod); FR-020 (new); BR-016, BR-021 (mod); BR-026, BR-027, BR-033 (new); VAL-009 (mod); VAL-013 (new); ERR-011, ERR-013, ERR-018 (new); AC-001, AC-017, AC-026, AC-028 (mod); AC-039, AC-040 (new); §19 (mod); §13.1 | Creación asíncrona del Event: validación completa, Event en `PROVISIONING` y respuesta inmediata HTTP 202 con `eventId` y estado de aprovisionamiento. El inventario se describe con una definición compacta (secciones, filas, asientos y rangos de cortesía) y el sistema genera `ticketId` = `<sección>-<fila>-<asiento>`. Límites de la definición configurables. Habilitación `ENABLED` solo tras verificar exactamente `capacity` Ticket; fallo definitivo lleva el Event a `FAILED`. Mientras no está `ENABLED` no se lista y la disponibilidad y la compra lo tratan como inexistente. El `ADMIN` consulta el estado con su progreso. |
| 7 | ADR-007 (sucesor ADR-027) | §10 Crear Event (mod); FR-017 (mod); BR-032 (new); VAL-016 (new); ALT-003 (mod); ERR-019 (new); AC-001 (mod); AC-041, AC-042 (new); §18, §20.2 (mod) | La creación de Event exige `Idempotency-Key` vinculada al `ADMIN` autenticado, con vigencia mínima de 24 horas. Repetir con la misma clave y contenido devuelve el mismo `eventId` y su estado actual (HTTP 200 con indicación de repetición) sin crear otro Event. La misma clave con contenido distinto se rechaza con `IDEMPOTENCY_KEY_REUSED` (HTTP 422) sin efectos. |
| 8 | ADR-013 (sucesor ADR-032) | §5 `Customer identity` (mod) e invariante 11 (new); DS-006 (mod); ST-006 (mod); CAP-004 (mod); MF-003 (mod); FR-004 (mod); FR-021 (new); BR-024 (new); §12.1 limitaciones aceptadas (new); ALT-013 (new); ERR-014 (new); AC-004, AC-028 (mod); AC-043, AC-044, AC-045 (new); §17, §20.2 (mod) | Un `CUSTOMER` no puede tener más de una Order en `CREATED` por Event. Una segunda compra se rechaza con `ACTIVE_ORDER_EXISTS` (HTTP 409) sin crear nada; una repetición con la misma clave sigue siendo repetición. Al terminar la Order activa, el cliente puede comprar de nuevo. Limitaciones aceptadas: no impide el acaparamiento con varias cuentas; una Order retenida para revisión manual por inconsistencia técnica mantiene el bloqueo. |
| 9 | ADR-016 (sucesor ADR-035) | §10 Iniciar compra y Crear Event (mod); MF-003 (mod); CAP-004 (mod); FR-006 (mod); FR-024 (new); BR-031 (new); ALT-006 (mod); ALT-010 (new); ERR-007 (mod); ERR-015, ERR-021 (new); AC-003, AC-022 (mod); AC-046, AC-047 (new); AC-016 (rev); §19 (mod) | Códigos observables `ACTIVE_ORDER_EXISTS` (409) y `EVENT_NOT_ON_SALE` (409). Con el sistema de colas indisponible, la compra se rechaza antes de reservar con `SERVICE_UNAVAILABLE` (503) e indicación de reintento, sin crear nada; con colas disponibles, el fallo definitivo de encolado sigue produciendo Order `FAILED`. Creación de Event: 202, repetición 200. Precedencia de rechazos explícita. |
| 10 | ADR-021 (sucesor ADR-040) | CAP-002 (mod); MF-002 (mod); §10 Consultar disponibilidad (mod); FR-012 (mod); VAL-015 (new); ERR-020 (new); AC-010, AC-011, AC-030 (mod); AC-048 (new); §13.1 | La disponibilidad devuelve la cantidad de Ticket `AVAILABLE` del Event y una página, por cursor, de Ticket `AVAILABLE` (identificador, sección, fila, asiento), tamaño máximo 100, filtrable por sección. Ya no lista Ticket en otros estados. La cantidad puede tener hasta 1 s de antigüedad e incluye `generatedAt`; sigue siendo informativa. Cursor inválido o sección inexistente: error de validación. |
| 11 | AV-002 | §3.2 (mod); §5 `Event` (mod) e invariantes 12 y 13 (new); §7A (new); CAP-001 (mod); FR-002 (mod); BR-022 (mod); BR-026, BR-027 (new, compartidos con 6); AC-001, AC-002, AC-026 (mod) | "Event habilitado" significa Event en `ENABLED`. El ciclo `PROVISIONING` → `ENABLED` / `FAILED` es visible para el `ADMIN`; para el `CUSTOMER` un Event no `ENABLED` no existe. No hay operación manual de habilitar o deshabilitar, y el Event es inmutable una vez habilitado. |
| 12 | AV-005 | §4 `ADMIN` (mod); MF-002 (mod); FR-019 (mod); FR-020 (new, compartido con 6); §20.2 (mod); AC-028 (mod) | Un `ADMIN` puede listar Events, consultar disponibilidad y consultar el estado de aprovisionamiento. No puede iniciar compras ni consultar Orders salvo que también pertenezca al grupo `CUSTOMER`. |
| 13 | AV-001 | MF-002 (mod); §10 Consultar Events (mod); FR-002 (mod); BR-022 (mod); AC-002 (mod) | El listado informa por Event un indicador de agotado (`soldOut`), no la cantidad exacta de disponibles; la cantidad se obtiene en la consulta de disponibilidad. |
| 14 | AV-003 | §5 invariante 15 (new, compartido con 3); ST-002, ST-003, ST-004, ST-007, ST-010 (mod); FR-011, FR-015 (mod); BR-002 (mod); BR-029, BR-030 (new); VAL-001 (mod); ALT-001 (mod); ALT-011 (new); ERR-003 (mod); AC-008, AC-009, AC-019 (mod); AC-049, AC-050 (new); §13.1 | Ninguna compra se confirma después de `expiresAt` (diez minutos desde la creación de la Reservation), aunque sus Ticket no se hayan liberado. Los Ticket de una Reservation vencida se liberan como máximo 15 s después de `expiresAt`. No se inicia un pago si faltan menos de 15 s para `expiresAt`; esa Order se cierra por expiración. Valores configurables; los indicados son los desplegados. |
| 15 | ADR-017 (sucesor ADR-036) — condicionada | §22.1 (new, solo nota). HV-010, TC-005, TC-012 y DEL-005 **sin cambios**. | No aplicada. Se registra como restricción técnica condicionada: si se activa el respaldo ElasticMQ, HV-010, TC-005, TC-012 y DEL-005 deben revisarse antes de aplicarse. |

Las secciones §25.2, §26, §27, §28, §29 y §30 se actualizaron para reflejar las 14 incorporaciones.

## 1. Purpose

Especificar el comportamiento funcional y no funcional de un backend reactivo de ticketing que gestione eventos, tickets individuales, disponibilidad, reservas temporales, órdenes y compras asíncronas, preservando la consistencia del inventario ante alta concurrencia.

Esta versión conserva las 21 decisiones aprobadas en la Human Functional Review y la aclaración humana `HC-001`, e incorpora 14 aclaraciones funcionales derivadas de la Human Architecture Review aprobada (ver §0 y §25.2). No define arquitectura de solución, modelo físico de datos, topología de infraestructura, clases, paquetes, endpoints definitivos ni políticas operativas concretas. Solo incluye los elementos observables (códigos HTTP, códigos de error, estados visibles, campos funcionales de respuesta) que fijan las aclaraciones.

## 2. Problem statement

El sistema actual presenta compras duplicadas, una misma ubicación vendida a más de una persona y timeouts durante picos de demanda. La solución debe mantener respuestas síncronas breves, delegar el procesamiento posterior a mensajería asíncrona y garantizar que ninguna ejecución concurrente o repetida produzca sobreventa, pagos duplicados ni ventas duplicadas.

- Objetivo de negocio: permitir compras de tickets sin sobreventa y con resultado consultable.
- Objetivo funcional: administrar tickets individuales, reservas con vencimiento, órdenes all-or-nothing y confirmación mediante un Payment Mock.
- Objetivo técnico explícito: backend reactivo y no bloqueante, persistencia NoSQL en DynamoDB y procesamiento asíncrono mediante Amazon SQS Standard.
- Resultado esperado: inventario consistente, operaciones idempotentes, autorización por roles y trazabilidad de estados.

## 3. Scope

### 3.1 In scope

- Creación y consulta de eventos.
- Creación asíncrona de eventos a partir de una definición compacta de inventario y consulta de su estado de aprovisionamiento por el `ADMIN` (aclaración 6).
- Tickets o ubicaciones individuales e inventario por evento.
- Consulta bajo demanda y paginada de disponibilidad (aclaración 10).
- Reservas temporales de máximo diez minutos.
- Órdenes de compra all-or-nothing de 1 a 10 tickets (aclaración 1).
- Una Order activa por `CUSTOMER` y Event (aclaración 8).
- Procesamiento asíncrono mediante Amazon SQS Standard.
- Payment Mock con resultados exitosos y fallidos deterministas y operación de cancelación (aclaración 3).
- Reverso de pagos aprobados o de resultado desconocido sin compra confirmada (aclaración 3).
- Control de concurrencia y prevención de sobreventa.
- Liberación de reservas expiradas.
- Idempotencia de solicitudes, mensajes, órdenes, pagos y creación de eventos (aclaraciones 4 y 7).
- Autenticación con JWT emitidos por Amazon Cognito.
- Autorización para `ADMIN` y `CUSTOMER`.
- Consulta del estado y resultado funcional de una Order.

### 3.2 Out of scope

- Frontend.
- UI de login, registro o recuperación de contraseña.
- Administración visual de usuarios.
- Reportes operativos, contables o de inventario.
- Disponibilidad por WebSocket, SSE o streaming continuo.
- Integración con un proveedor real de pagos.
- Operación manual para habilitar o deshabilitar un Event, y modificación de un Event ya habilitado (aclaración 11).
- Diseño físico de DynamoDB, parámetros operativos de SQS, topología AWS o estructura interna del Payment Mock.

Fuentes de consolidación: requerimiento original; `HV-007`, `HV-008`, `HV-013`, `HV-014`; aclaraciones 1, 3, 4, 6, 7, 8, 10 y 11.

## 4. Actors

| Actor | Type | Responsibility | Authorization | Source |
|---|---|---|---|---|
| `ADMIN` | Human | Crear eventos y definir tickets `COMPLIMENTARY` durante la creación; consultar el estado de aprovisionamiento de un Event; listar Events y consultar su disponibilidad. No inicia compras ni consulta Orders salvo que también pertenezca al grupo `CUSTOMER`. | Grupo Cognito `ADMIN`. | `HV-005`, `HV-014`; aclaraciones 6, 12 |
| `CUSTOMER` | Human | Consultar eventos y disponibilidad, iniciar compras y consultar exclusivamente sus propias Orders. | Grupo Cognito `CUSTOMER`. | `HV-014`, `HV-021` |
| Consumidor asíncrono de Orders | Autonomous process | Consumir mensajes, coordinar el Payment Mock y llevar Order y tickets a un resultado consistente. | Identidad técnica; detalle en arquitectura. | Requisito Funcional 3; `HV-003`, `HV-004`, `HV-008` |
| Proceso de expiración | Autonomous process | Identificar reservas vencidas y liberar conjuntamente sus tickets como máximo 15 s después de `expiresAt`. | Identidad técnica; detalle en arquitectura. | Requisitos Funcionales 2 y 6; `HV-012`; aclaración 14 |
| Proceso de aprovisionamiento de Events | Autonomous process | Escribir en segundo plano los Ticket de un Event en `PROVISIONING` y habilitarlo o marcarlo `FAILED`. | Identidad técnica; detalle en arquitectura. | Aclaración 6 |
| Proceso de reverso de pagos | Autonomous process | Solicitar al Payment Mock la cancelación de PaymentAttempts aprobados o de resultado desconocido de Orders cerradas sin confirmar, con reintento acotado. | Identidad técnica; detalle en arquitectura. | Aclaración 3 |
| Payment Mock | External system | Simular de forma determinista aprobación, rechazo o fallo de un pago, y ofrecer la cancelación idempotente de un PaymentAttempt. | Integración protegida; detalle en arquitectura. | `HV-008`; aclaración 3 |
| Amazon Cognito | External identity provider | Autenticar usuarios y emitir JWT con `cognito:groups`. | Proveedor de identidad. | `HV-014` |

El candidato, evaluador, entrevistador y desarrollador no son actores funcionales.

## 5. Domain model

| Concept | Consolidated definition | Cardinality / invariant | Source |
|---|---|---|---|
| `Event` | Evento con nombre, fecha y hora (`startsAt`), lugar, capacidad total, definición compacta de inventario (secciones, filas, asientos por fila y rangos de cortesía) y tickets individuales. Tiene un ciclo de aprovisionamiento `PROVISIONING` → `ENABLED` / `FAILED` (§7A). "Event habilitado" significa Event en `ENABLED`; es inmutable una vez habilitado. | `Event 1 -> N Ticket`; `Event 1 -> N Order`; capacidad máxima 50.000. | Requisito Funcional 1; `HV-006`, `HV-016`, `HV-017`; aclaraciones 2, 6, 11 |
| `Ticket` | Ubicación individual, numerada e identificable de forma única dentro de su Event; tiene exactamente un estado. Su identificador lo genera el sistema como `<sección>-<fila>-<asiento>` a partir de la definición del Event. | Pertenece a un solo Event y como máximo a una Reservation activa. | `HV-001`, `HV-006`, `HV-017`; aclaración 6 |
| `Inventory` | Vista de los estados de los tickets de un Event. La disponibilidad comercial se deriva de tickets `AVAILABLE`. | No es solo un contador independiente; debe ser coherente con los tickets. | Requisito Funcional 1; `HV-006` |
| `Order` | Solicitud de compra de entre 1 y 10 tickets individuales, sin repetidos, de un único Event. | `Order 1 -> 1..10 Ticket requested`; resultado all-or-nothing. | `HV-001`, `HV-017`; aclaración 1 |
| `Reservation` | Retención temporal y atómica de exactamente los tickets solicitados por una Order, con instante de expiración `expiresAt` igual a su creación más diez minutos. | `Order 1 -> 0..1 Reservation`; una Order tiene como máximo una activa. | `HV-012`, `HV-017`; aclaración 14 |
| `PaymentAttempt` | Intento identificable (`paymentAttemptId`) de pago asociado a una Order. Puede cancelarse ante el proveedor de forma idempotente. | Máximo un intento activo simultáneamente; un reintento autorizado es una operación diferente. | `HV-008`, `HV-011`; aclaración 3 |
| `Message` | Representación de trabajo asíncrono de una Order entregada al menos una vez. | Una entrega repetida no produce efectos adicionales. | `HV-010`, `HV-011` |
| `Customer identity` | Identidad autenticada propietaria de la Order. | Se deriva del JWT; no de un identificador libre enviado por el cliente. Como máximo una Order en `CREATED` por Event. | `HV-014`, `HV-021`; aclaración 8 |

### 5.1 Domain invariants

1. Un Event contiene múltiples Ticket individuales.
2. `Event.capacity = total de Ticket del Event`.
3. Una Order pertenece exactamente a un Event y solicita entre 1 y 10 Ticket de ese mismo Event, sin identificadores repetidos. *(mod — aclaración 1)*
4. Una Order no puede contener tickets de diferentes eventos.
5. Una Order se procesa all-or-nothing; no existe reserva, pago ni venta parcial.
6. Una Order tiene como máximo una Reservation activa.
7. Una Reservation contiene exactamente los tickets solicitados por su Order.
8. Un Ticket solo puede pertenecer a una Reservation activa a la vez.
9. Ticket y Order poseen máquinas de estado separadas.
10. El estado terminal de una Order debe ser coherente con el estado conjunto de sus tickets.
11. Un `CUSTOMER` tiene como máximo una Order en `CREATED` por Event. *(new — aclaración 8)*
12. Un Event solo es visible y vendible en `ENABLED`, y solo alcanza `ENABLED` tras comprobarse que existen exactamente `capacity` Ticket. *(new — aclaraciones 2, 6, 11)*
13. Un Event `ENABLED` es inmutable; no existe operación manual para habilitarlo ni deshabilitarlo. *(new — aclaración 11)*
14. El identificador de un Ticket es `<sección>-<fila>-<asiento>` y es único dentro de su Event. *(new — aclaración 6)*
15. Ninguna compra se confirma después de `expiresAt`, y una aprobación de pago tardía nunca reabre ni confirma una Order terminal. *(new — aclaraciones 3, 14)*
16. Solo puede iniciarse una compra sobre un Event `ENABLED` que no sea pasado en el instante de la solicitud. *(new — aclaración 5)*

## 6. Ticket state machine

### 6.1 Ticket states

| ID | State | Meaning | Terminal | Accounting effect | Inventory effect | Source |
|---|---|---|---|---|---|---|
| DS-001 | `AVAILABLE` | Ticket comercialmente disponible para ser reservado. | No | No es venta. | Cuenta como disponibilidad comercial. | Requerimiento; `HV-001`, `HV-006` |
| DS-002 | `RESERVED` | Ticket retenido por una Reservation activa antes de iniciar o mientras se prepara el pago. | No | No es venta. | No está disponible para otras Orders. | Requerimiento; `HV-001`, `HV-002`, `HV-012` |
| DS-003 | `PENDING_CONFIRMATION` | Ticket reservado cuyo pago está en curso o pendiente de respuesta definitiva. | No | No es venta. | Continúa bloqueado para otras Orders. | Requerimiento; `HV-004` |
| DS-004 | `SOLD` | Ticket de una Order cuyo pago terminó exitosamente. | Sí, irreversible | Venta confirmada. | No está disponible. | Requerimiento; `HV-003`, `HV-008` |
| DS-005 | `COMPLIMENTARY` | Ticket de cortesía creado en ese estado durante el aprovisionamiento del Event, a partir de los rangos de cortesía de su definición. *(mod — aclaración 6)* | Sí | No es venta y no es contable como venta. | Consume capacidad, pero nunca es disponibilidad comercial. | Requerimiento; `HV-005`, `HV-013`; aclaración 6 |

### 6.2 Ticket transitions

| ID | From | To | Trigger | Constraints | Source |
|---|---|---|---|---|---|
| ST-001 | `AVAILABLE` | `RESERVED` | Inicio formal de compra y creación exitosa de la Reservation. | Todos los tickets solicitados (1 a 10, sin repetidos) deben transicionar atómicamente; si uno no está disponible, ninguno cambia. Solo procede si la solicitud superó las comprobaciones previas de BR-031: Event `ENABLED` y no pasado, sistema de colas disponible y sin Order activa del `CUSTOMER` en el Event. *(mod — aclaraciones 1, 5, 8, 9)* | Requisito Funcional 2; `HV-002`, `HV-012`, `HV-017`; aclaraciones 1, 5, 8, 9 |
| ST-002 | `RESERVED` o `PENDING_CONFIRMATION` | `AVAILABLE` | Expiración de la Reservation: se alcanza `expiresAt` (diez minutos) sin compra confirmada. | Liberación conjunta de todos los tickets de la Reservation como máximo 15 s después de `expiresAt`; Order pasa a `EXPIRED`. *(mod — aclaración 14)* | Requisitos Funcionales 2 y 6; `HV-003`, `HV-004`, `HV-012`; aclaración 14 |
| ST-003 | `RESERVED` | `PENDING_CONFIRMATION` | Inicio del procesamiento de pago. | Todos los tickets de la Order cambian conjuntamente; no representa venta. No se inicia el pago si faltan menos de 15 s para `expiresAt`. *(mod — aclaración 14)* | `HV-004`, `HV-008`; aclaración 14 |
| ST-004 | `PENDING_CONFIRMATION` | `SOLD` | Payment Mock confirma pago exitoso. | Todos los tickets cambian conjuntamente y la Order pasa a `CONFIRMED`. Solo si `expiresAt` todavía no se alcanzó y la Order sigue en `CREATED`. *(mod — aclaraciones 3, 14)* | `HV-003`, `HV-004`, `HV-008`; aclaraciones 3, 14 |
| ST-005 | `RESERVED` o `PENDING_CONFIRMATION` | `AVAILABLE` | Rechazo definitivo del pago, fallo técnico definitivo o fallo definitivo de encolado, según corresponda. | Ningún ticket queda vendido; liberación conjunta e idempotente. | `HV-003`, `HV-004`, `HV-009`, `HV-018` |

`COMPLIMENTARY` y `AVAILABLE` se asignan al crear cada Ticket durante el aprovisionamiento del Event (§7A); `COMPLIMENTARY` no es una transición posterior desde otro estado. `SOLD` y `COMPLIMENTARY` no tienen transiciones de salida. *(mod — aclaración 6)*

## 7. Order state machine

### 7.1 Order states

| ID | State | Meaning | Terminal | Ticket consistency | Source |
|---|---|---|---|---|---|
| DS-006 | `CREATED` | Order persistida después de crear atómicamente su Reservation; puede estar encolándose o procesándose. Un `CUSTOMER` tiene como máximo una Order `CREATED` por Event. *(mod — aclaración 8)* | No | Sus tickets están `RESERVED` o `PENDING_CONFIRMATION`. | `HV-018`; aclaración 8 |
| DS-007 | `CONFIRMED` | Pago exitoso aplicado antes de `expiresAt` y compra completa confirmada. | Sí | Todos sus tickets están `SOLD`. | `HV-009`; aclaración 14 |
| DS-008 | `REJECTED` | Resultado funcional negativo posterior a la creación consistente de la Order, incluido el rechazo definitivo del pago. La indisponibilidad detectada antes de crear la Reservation no crea una Order. | Sí | Ningún ticket queda `SOLD`; la Reservation se cancela y todos sus tickets regresan conjuntamente a `AVAILABLE`. | `HV-009`; `HC-001` |
| DS-009 | `FAILED` | Fallo técnico definitivo posterior a la creación de la Order, incluido el fallo definitivo de encolado. Si el resultado del pago es desconocido al cerrarse, el sistema solicita el reverso del pago (BR-028); la marca de reverso no se expone. *(mod — aclaración 3)* | Sí | La Reservation se cancela y todos sus tickets regresan conjuntamente a `AVAILABLE`; no queda ningún ticket reservado ni vendido parcialmente. | `HV-009`, `HV-018`; aclaración 3 |
| DS-010 | `EXPIRED` | La Reservation alcanzó `expiresAt` (diez minutos) sin confirmación, incluida la Order en la que no se inició el pago por el margen de corte. Si el pago fue aprobado o su resultado es desconocido, el sistema solicita su reverso (BR-028); la Order no se reabre. *(mod — aclaraciones 3, 14)* | Sí | Todos los tickets reservados regresan a `AVAILABLE`. | `HV-009`, `HV-012`; aclaraciones 3, 14 |

`CONFIRMED`, `REJECTED`, `FAILED` y `EXPIRED` son terminales para el procesamiento funcional de la Order. No se introducen estados intermedios por conveniencia técnica. La marca de reverso de pago y la retención para revisión manual no son estados de negocio y no se exponen (aclaraciones 3 y 8).

### 7.2 Order transitions

| ID | From | To | Trigger | Constraints | Source |
|---|---|---|---|---|---|
| ST-006 | creación consistente | `CREATED` | Reservation completa creada y Order persistida. | Debe existir una única Reservation activa con todos los tickets solicitados (1 a 10, sin repetidos). El `CUSTOMER` no puede tener otra Order `CREATED` en el mismo Event. *(mod — aclaraciones 1, 8)* | `HV-002`, `HV-012`, `HV-018`; aclaraciones 1, 8 |
| ST-007 | `CREATED` | `CONFIRMED` | Pago exitoso. | Coincide atómicamente con todos los tickets en `SOLD`. Solo antes de `expiresAt`. *(mod — aclaraciones 3, 14)* | `HV-003`, `HV-009`; aclaraciones 3, 14 |
| ST-008 | `CREATED` | `REJECTED` | Rechazo funcional definitivo posterior a la creación de la Order, incluido el rechazo del pago. | Cancelar la Reservation y devolver conjuntamente todos sus tickets a `AVAILABLE`. | `HV-009`; `HC-001` |
| ST-009 | `CREATED` | `FAILED` | Fallo técnico definitivo o fallo definitivo de encolado. | Cancelar la Reservation y devolver conjuntamente todos sus tickets a `AVAILABLE`; no dejar reservas indefinidas. Si el resultado del pago es desconocido, marcar la Order como pendiente de reverso. *(mod — aclaración 3)* | `HV-009`, `HV-018`; aclaración 3 |
| ST-010 | `CREATED` | `EXPIRED` | Se alcanza `expiresAt` sin compra confirmada, o no se inició el pago porque faltaban menos de 15 s para `expiresAt`. | Liberación conjunta e idempotente, como máximo 15 s después de `expiresAt`. Si existe un PaymentAttempt aprobado o de resultado desconocido, marcar la Order como pendiente de reverso. *(mod — aclaraciones 3, 14)* | `HV-009`, `HV-012`; aclaraciones 3, 14 |

Cuando la Order activa de un `CUSTOMER` alcanza cualquier estado terminal, ese `CUSTOMER` puede iniciar una nueva compra para el mismo Event (BR-024; aclaración 8).

## 7A. Event provisioning state machine

Ciclo de aprovisionamiento del Event, visible para el `ADMIN` (aclaraciones 6 y 11). No es un estado de Ticket ni de Order. Para distinguirlo del `FAILED` de Order, en esta especificación se escribe `FAILED` (Event).

### 7A.1 Event provisioning states

| ID | Entity | State | Meaning | Terminal | Visibility | Source |
|---|---|---|---|---|---|---|
| DS-011 | Event | `PROVISIONING` | Solicitud de creación válida aceptada; el sistema escribe en segundo plano los Ticket. El Event no está habilitado. | No | Solo el `ADMIN`, mediante la consulta de estado de aprovisionamiento. No aparece en el listado; la disponibilidad y la compra lo tratan como inexistente. | Aclaraciones 5, 6, 11 |
| DS-012 | Event | `ENABLED` | Creación completada: existen exactamente `capacity` Ticket. Equivale a "Event habilitado". | Sí (sin transiciones de salida; inmutable) | Visible para `ADMIN` y `CUSTOMER` según BR-022. | Aclaraciones 2, 6, 11 |
| DS-013 | Event | `FAILED` (Event) | El aprovisionamiento falló definitivamente. | Sí | Nunca visible ni vendible para el `CUSTOMER`; el `ADMIN` lo observa en la consulta de estado. | Aclaraciones 5, 6, 11 |

### 7A.2 Event provisioning transitions

| ID | From | To | Trigger | Constraints | Source |
|---|---|---|---|---|---|
| ST-011 | solicitud de creación válida | `PROVISIONING` | El `ADMIN` solicita crear un Event y la solicitud supera todas las validaciones. | Una solicitud inválida, o con capacidad superior a 50.000, no crea el Event ni sus Ticket. La respuesta inmediata es HTTP 202 con `eventId` y estado de aprovisionamiento. | Aclaraciones 2, 6, 9 |
| ST-012 | `PROVISIONING` | `ENABLED` | El sistema termina de escribir los Ticket y comprueba que existen exactamente `capacity` Ticket. | Sin habilitación manual. Después de habilitado, el Event es inmutable. | Aclaraciones 2, 6, 11 |
| ST-013 | `PROVISIONING` | `FAILED` (Event) | El aprovisionamiento falla definitivamente. | El Event nunca es visible ni vendible. | Aclaración 6 |

No existe operación para deshabilitar un Event `ENABLED` ni para habilitar manualmente un Event (aclaración 11).

## 8. Functional capabilities

| ID | Capability | Consolidated scope |
|---|---|---|
| CAP-001 | Gestión de eventos | `ADMIN` solicita la creación de Events válidos, que se aprovisionan de forma asíncrona (`PROVISIONING` → `ENABLED` / `FAILED`) y cuyo estado puede consultar. `ADMIN` y `CUSTOMER` consultan Events futuros en `ENABLED`. *(mod — aclaraciones 6, 11, 12)* |
| CAP-002 | Consulta de disponibilidad | Lectura bajo demanda de la cantidad de tickets `AVAILABLE` y de una página por cursor de tickets `AVAILABLE`, filtrable por sección; informativa y sin streaming. *(mod — aclaración 10)* |
| CAP-003 | Reserva temporal | Reserva atómica de todos los tickets solicitados por máximo diez minutos. |
| CAP-004 | Recepción de compras | Inicio formal de compra con rechazos síncronos en el orden de BR-031, una Order activa por `CUSTOMER` y Event, creación consistente de Order/Reservation, encolado y retorno de Order ID. *(mod — aclaraciones 8, 9)* |
| CAP-005 | Procesamiento asíncrono | Consumo idempotente, invocación del Payment Mock respetando el margen de corte y cierre coherente de Order y tickets. *(mod — aclaraciones 3, 14)* |
| CAP-006 | Consulta de Order | Consulta de cualquier resultado de una Order propia sin revelar Orders de terceros. |
| CAP-007 | Expiración de reservas | Identificación y liberación conjunta de reservas vencidas. |
| CAP-008 | Concurrencia, atomicidad e idempotencia | Cero sobreventa y cero efectos duplicados bajo contención o entrega at-least-once. |
| CAP-009 | Reverso de pagos no aplicados | Cancelación idempotente ante el proveedor de PaymentAttempts aprobados o de resultado desconocido de Orders cerradas sin confirmar. *(new — aclaración 3)* |

## 9. Main flows

### MF-001 — Crear Event *(mod — aclaraciones 2, 6, 7, 9, 11)*

1. Un `ADMIN` autenticado envía una `Idempotency-Key` y suministra nombre, lugar, fecha y hora futuras, capacidad y una definición compacta del inventario: secciones con código único, filas con etiqueta única por sección y cantidad de asientos por fila (numerados desde 1), más rangos de asientos de cortesía.
2. Puede marcar asientos como `COMPLIMENTARY`, mediante los rangos de cortesía, exclusivamente durante esta creación.
3. El sistema valida la solicitud completa: nombre y lugar no vacíos, fecha/hora futura en UTC, capacidad entre 1 y 50.000, límites de la definición (VAL-013) y capacidad igual a la cantidad de asientos derivados, cortesías incluidas.
4. Si la solicitud es inválida, la rechaza completa sin crear el Event ni sus Ticket.
5. Si es válida, crea el Event en `PROVISIONING` y responde de inmediato (HTTP 202) con el `eventId` y el estado de aprovisionamiento.
6. En segundo plano, el sistema genera cada Ticket con identificador `<sección>-<fila>-<asiento>`, en `COMPLIMENTARY` si pertenece a un rango de cortesía y en `AVAILABLE` en otro caso.
7. Cuando comprueba que existen exactamente `capacity` Ticket, el Event pasa a `ENABLED`. Si el aprovisionamiento falla definitivamente, el Event pasa a `FAILED` y nunca es visible ni vendible.
8. El `ADMIN` puede consultar en cualquier momento el estado de aprovisionamiento (`PROVISIONING`, `ENABLED`, `FAILED`) con su progreso.
9. Una repetición con la misma `Idempotency-Key` y el mismo contenido devuelve el mismo `eventId` con su estado actual (HTTP 200 con indicación de repetición) sin crear otro Event.
10. Después de creado no se pueden agregar cortesías ni convertir otros estados a `COMPLIMENTARY`; un Event `ENABLED` es inmutable.

### MF-002 — Consultar Events y disponibilidad *(mod — aclaraciones 5, 10, 11, 12, 13)*

1. Un `CUSTOMER` o un `ADMIN` consulta eventos disponibles.
2. El sistema incluye eventos futuros en `ENABLED`, incluso cuando su disponibilidad comercial es cero, e informa por Event un indicador de agotado (`soldOut`), no la cantidad exacta de disponibles.
3. Los eventos pasados y los que no están en `ENABLED` no aparecen en la consulta normal.
4. Al consultar la disponibilidad de un Event, el sistema devuelve bajo demanda la cantidad de tickets `AVAILABLE` del Event y una página, paginada por cursor, de tickets `AVAILABLE` (identificador, sección, fila, asiento), con tamaño máximo de 100 y filtrable por sección. No lista tickets en otros estados.
5. La cantidad puede tener hasta 1 s de antigüedad y la respuesta indica el instante de cálculo (`generatedAt`).
6. La información es una fotografía informativa y no garantiza adquisición.
7. La disponibilidad por identificador de un Event pasado sigue respondiendo de forma informativa; la de un Event que no está en `ENABLED` lo trata como inexistente.

### MF-003 — Iniciar y confirmar compra *(mod — aclaraciones 1, 5, 8, 9, 14)*

1. El `CUSTOMER` consulta un Event y su disponibilidad vigente.
2. Selecciona entre 1 y 10 tickets individuales del mismo Event, sin repetidos.
3. La selección visual no crea Reservation ni inicia temporizador.
4. El `CUSTOMER` inicia formalmente la compra con una idempotency key.
5. El sistema evalúa los rechazos síncronos en el orden de BR-031: validación (1 a 10 tickets, sin repetidos); idempotencia (repetición o clave reutilizada); Event inexistente o no habilitado; Event pasado (`EVENT_NOT_ON_SALE`); Ticket inexistentes; sistema de colas indisponible (`SERVICE_UNAVAILABLE`); Order activa existente (`ACTIVE_ORDER_EXISTS`). Cualquier rechazo termina la solicitud sin crear Reservation, Order ni Order ID y sin modificar el inventario.
6. Si la solicitud es una repetición de una operación aceptada, retorna la Order existente sin repetir efectos.
7. El sistema valida otra vez todos los tickets solicitados.
8. El sistema intenta reservarlos de forma atómica.
9. Si cualquiera no existe o no está disponible, rechaza la solicitud completa sin crear Reservation, Order ni Order ID.
10. Si todos están disponibles, crea una Reservation con exactamente esos tickets.
11. En ese instante comienza el período máximo de diez minutos; `expiresAt` es la creación de la Reservation más diez minutos.
12. El sistema crea la Order en `CREATED`; sus tickets quedan `RESERVED`.
13. El sistema intenta encolar la solicitud en Amazon SQS Standard.
14. Al quedar un resultado persistido y consultable, retorna el identificador de Order sin esperar el pago.
15. El consumidor procesa el mensaje de forma idempotente. Si faltan menos de 15 s para `expiresAt`, no inicia el pago y la Order se cierra por expiración.
16. En otro caso invoca el Payment Mock; al iniciar el pago, todos los tickets pasan a `PENDING_CONFIRMATION`.
17. Si el pago es exitoso y `expiresAt` todavía no se alcanzó, todos pasan a `SOLD` y la Order a `CONFIRMED`.

No se permite cumplimiento parcial en ningún paso.

### MF-004 — Consultar Order *(mod — aclaración 3)*

1. Un `CUSTOMER` autenticado solicita una Order por identificador.
2. El sistema deriva el propietario desde el JWT.
3. Si la Order existe y pertenece al `CUSTOMER`, retorna su estado actual y, si no terminó exitosamente, una causa funcional comprensible sin detalles técnicos internos. La marca de reverso de pago no se expone.
4. Si no existe o pertenece a otro `CUSTOMER`, retorna funcionalmente “recurso no encontrado”.
5. `CONFIRMED`, `REJECTED`, `FAILED` y `EXPIRED` permanecen consultables.

### MF-005 — Reversar pago no aplicado *(new — aclaración 3)*

1. Una Order con un PaymentAttempt aprobado o de resultado desconocido se cierra sin confirmarse (`EXPIRED`, o `FAILED` con resultado desconocido). La transición de cierre marca la Order como pendiente de reverso.
2. La Order conserva su estado terminal y sus tickets quedan liberados.
3. El sistema solicita al proveedor la cancelación del PaymentAttempt identificado por `paymentAttemptId`, con reintento acotado.
4. Al confirmarse la cancelación, el sistema retira la marca y deja evidencia auditable.
5. Si se agotan los reintentos, el sistema alerta y el reverso queda pendiente para revisión manual.
6. Una aprobación que llega después del cierre nunca reabre ni confirma la Order.

## 10. Inputs and outputs

| Operation | Inputs | Outputs / observable result |
|---|---|---|
| Crear Event *(mod — aclaraciones 2, 6, 7, 9)* | Nombre, lugar, fecha/hora, capacidad, definición compacta de inventario (secciones, filas, asientos por fila y rangos de cortesía); `Idempotency-Key`; identidad `ADMIN`. | Solicitud válida: HTTP 202 con `eventId` y estado de aprovisionamiento `PROVISIONING`. Repetición con misma clave y contenido: HTTP 200 con indicación de repetición, mismo `eventId` y estado actual. Misma clave con contenido distinto: `IDEMPOTENCY_KEY_REUSED` (HTTP 422) sin efectos. Solicitud inválida o capacidad superior a 50.000: error de validación sin crear el Event ni sus Ticket. Rechazo de autorización. |
| Consultar estado de aprovisionamiento *(new — aclaraciones 6, 11, 12)* | `eventId`; identidad `ADMIN`. | Estado `PROVISIONING`, `ENABLED` o `FAILED` del Event con su progreso. |
| Consultar Events *(mod — aclaraciones 11, 12, 13)* | Identidad autenticada (`ADMIN` o `CUSTOMER`) y criterios de consulta que defina el contrato posterior. | Events futuros en `ENABLED`, cada uno con el indicador de agotado `soldOut`, sin la cantidad exacta de disponibles. |
| Consultar disponibilidad *(mod — aclaraciones 5, 6, 10, 12)* | Event identificado; filtro opcional por sección; cursor de página opcional; identidad `ADMIN` o `CUSTOMER`. | Cantidad de tickets `AVAILABLE` del Event, `generatedAt` (antigüedad de hasta 1 s) y una página de hasta 100 tickets `AVAILABLE` (identificador, sección, fila, asiento). Resultado informativo. Event pasado: responde de forma informativa. Event no `ENABLED`: tratado como inexistente. Cursor inválido o sección inexistente: error de validación. |
| Iniciar compra *(mod — aclaraciones 1, 4, 5, 8, 9)* | Entre 1 y 10 Ticket IDs, sin repetidos, del mismo Event; identidad `CUSTOMER`; idempotency key vinculada al `CUSTOMER`. | Order ID y estado persistido consultable cuando la reserva completa fue creada (Order `CREATED`, o `FAILED` si el encolado falló definitivamente con el sistema de colas disponible). Repetición de una operación aceptada: la Order existente en su estado actual. Rechazos síncronos sin Reservation, Order ni Order ID y sin cambio de inventario, en el orden de BR-031: error de validación; clave reutilizada (`IDEMPOTENCY_KEY_REUSED`); Event inexistente o no habilitado; `EVENT_NOT_ON_SALE` (HTTP 409); Ticket inexistentes; `SERVICE_UNAVAILABLE` (HTTP 503) con indicación de reintento; `ACTIVE_ORDER_EXISTS` (HTTP 409); Ticket inexistentes o no disponibles detectados en la reserva. |
| Procesar pago *(mod — aclaraciones 3, 14)* | Order y operación de pago identificables (`paymentAttemptId`). | Aprobación, rechazo o fallo determinista del mock; efectos idempotentes. No se inicia si faltan menos de 15 s para `expiresAt`. Una aprobación posterior a `expiresAt` o al cierre de la Order no confirma la compra. |
| Reversar pago *(new — aclaración 3)* | PaymentAttempt (`paymentAttemptId`) de una Order cerrada sin confirmar con resultado aprobado o desconocido. | Cancelación idempotente registrada por el proveedor, cualquiera que sea el estado del intento; evidencia auditable al confirmarse; alerta y revisión manual si se agotan los reintentos. |
| Consultar Order *(mod — aclaración 3)* | Order ID e identidad autenticada. | Estado y causa funcional autorizada, sin la marca de reverso; o recurso no encontrado. |

## 11. Functional requirements

### FR-001 — Crear Events *(mod — aclaraciones 2, 6, 11)*

El sistema debe permitir que un `ADMIN` solicite la creación de Events con nombre, fecha/hora, lugar, capacidad total y una definición compacta del inventario (secciones, filas, asientos por fila y rangos de cortesía). La creación es asíncrona: la solicitud se valida completa y, si es válida, el Event se crea en `PROVISIONING` y se responde de inmediato (HTTP 202) con el `eventId` y el estado de aprovisionamiento. El sistema genera los Ticket en segundo plano y habilita el Event (`ENABLED`) solo tras verificar que existen exactamente `capacity` Ticket; si el aprovisionamiento falla definitivamente, el Event queda en `FAILED`. La capacidad máxima es 50.000 Ticket; una solicitud que la supere se rechaza completa sin crear el Event ni sus Ticket. Debe poder definir tickets `COMPLIMENTARY` solamente durante esa creación. Fuente: Requisito Funcional 1; `HV-005`, `HV-014`, `HV-016`; aclaraciones 2, 6, 11. Status: DEFINED.

### FR-002 — Consultar Events disponibles *(mod — aclaraciones 5, 11, 12, 13)*

El sistema debe permitir a `ADMIN` y `CUSTOMER` consultar Events futuros en `ENABLED` (Event habilitado para consulta y venta). Un Event agotado continúa visible y se identifica mediante el indicador `soldOut`; el listado no informa la cantidad exacta de disponibles. Un Event pasado o no `ENABLED` no aparece en la consulta normal; la disponibilidad por identificador de un Event pasado sigue respondiendo de forma informativa. Fuente: Objetivo; Requisito Funcional 1; `HV-020`; aclaraciones 5, 11, 12, 13. Status: DEFINED.

### FR-003 — Mantener inventario por Event *(mod — aclaración 6)*

Cada Event debe mantener tickets individuales y disponibilidad coherente con sus estados. La capacidad total debe coincidir con el total de tickets derivados de su definición de inventario, cortesías incluidas. Fuente: Requisito Funcional 1; `HV-005`, `HV-006`, `HV-016`; aclaración 6. Status: DEFINED.

### FR-004 — Reservar tickets temporal y atómicamente *(mod — aclaraciones 1, 8)*

Al iniciar formalmente una compra de entre 1 y 10 tickets sin repetidos, el sistema debe revalidar y reservar todos los tickets solicitados durante máximo diez minutos; si uno no está disponible, no debe crear una Reservation parcial. Una solicitud con cero tickets, más de 10 o identificadores repetidos se rechaza con error de validación antes de reservar. La reserva no procede si el `CUSTOMER` ya tiene una Order `CREATED` en el mismo Event (FR-021). Fuente: Requisito Funcional 2; `HV-002`, `HV-012`, `HV-017`; aclaraciones 1, 8. Status: DEFINED.

### FR-005 — Encolar solicitudes de compra

Después de crear consistentemente la Reservation y la Order, el sistema debe intentar encolar la solicitud en Amazon SQS Standard para procesamiento asíncrono. Fuente: Requisito Funcional 3; `HV-002`, `HV-010`, `HV-018`. Status: DEFINED.

### FR-006 — Retornar identificador de Order *(mod — aclaración 9)*

El sistema debe retornar un Order ID sin esperar el procesamiento de pago únicamente cuando exista una Order persistida y consultable. Si la solicitud se rechaza síncronamente por cualquiera de las causas de BR-031 —incluida la indisponibilidad de algún ticket en la reserva inicial—, debe rechazarla sin crear Order ni retornar Order ID. Fuente: Requisito Funcional 3; `HV-018`; `HC-001`; aclaración 9. Status: DEFINED.

### FR-007 — Procesar Orders asíncronamente

Un consumidor debe procesar las Orders encoladas de forma asíncrona y segura ante entregas repetidas. Fuente: Objetivo; Requisito Funcional 3; `HV-010`, `HV-011`. Status: DEFINED.

### FR-008 — Actualizar Order e inventario consistentemente

El procesamiento debe mantener la coherencia entre el estado de Order, Reservation y todos sus tickets, sin resultados parciales. Fuente: Requisito Funcional 3; `HV-001`, `HV-003`, `HV-004`, `HV-009`. Status: DEFINED.

### FR-009 — Consultar Order propia *(mod — aclaración 3)*

El sistema debe permitir a un `CUSTOMER` consultar en cualquier momento una Order propia, incluida una terminal. Una Order inexistente o ajena debe observarse como recurso no encontrado. La consulta no expone la marca de reverso de pago. Fuente: Requisito Funcional 4; `HV-001`, `HV-009`, `HV-014`, `HV-021`; aclaración 3. Status: DEFINED.

### FR-010 — Prevenir sobreventa bajo concurrencia

El sistema debe garantizar que solicitudes concurrentes por un mismo Ticket solo permitan una Reservation ganadora y nunca produzcan sobreventa. Fuente: Contexto; Requisito Funcional 5; `HV-006`. Status: DEFINED.

### FR-011 — Liberar Reservations expiradas *(mod — aclaraciones 3, 14)*

Un proceso debe identificar Reservations que alcanzaron `expiresAt` (diez minutos) sin confirmación, llevar su Order a `EXPIRED` y devolver conjuntamente todos sus tickets a `AVAILABLE` como máximo 15 s después de `expiresAt`. Si la Order tiene un PaymentAttempt aprobado o de resultado desconocido, la marca como pendiente de reverso (FR-023). Excepción aceptada: §12.1. Fuente: Requisitos Funcionales 2 y 6; `HV-003`, `HV-004`, `HV-009`, `HV-012`; aclaraciones 3, 14. Status: DEFINED.

### FR-012 — Consultar disponibilidad bajo demanda *(mod — aclaraciones 2, 10)*

El sistema debe exponer una consulta reactiva que devuelva la cantidad de tickets `AVAILABLE` del Event y una página de tickets `AVAILABLE` (identificador, sección, fila, asiento), paginada por cursor, con tamaño máximo de 100 y filtrable por sección. No lista tickets en otros estados ni retorna todo el inventario en una respuesta. La cantidad puede tener hasta 1 s de antigüedad y la respuesta indica `generatedAt`. La respuesta no garantiza adquisición y debe revalidarse al iniciar compra. Fuente: Objetivo; Requisito Funcional 7; `HV-005`, `HV-006`, `HV-007`; aclaraciones 2, 10. Status: DEFINED.

### FR-013 — Mantener transiciones atómicas

Las transiciones de una Order y del conjunto de sus tickets deben completarse atómicamente desde la perspectiva funcional; no deben exponerse ventas parciales. Fuente: Notas generales; `HV-001`, `HV-017`. Status: DEFINED.

### FR-014 — Mantener transiciones auditables

El sistema debe conservar evidencia auditable de las transiciones de Ticket y Order. El contenido y mecanismo concreto se definen en arquitectura. Fuente: Notas generales. Status: DEFINED cualitativamente.

### FR-015 — Confirmar compra mediante Payment Mock *(mod — aclaraciones 3, 14)*

El sistema debe invocar un Payment Mock después de reservar todos los tickets y confirmar la compra únicamente ante un resultado exitoso aplicado antes de `expiresAt` con la Order en `CREATED`. No debe iniciar un pago si faltan menos de 15 s para `expiresAt`. Una aprobación posterior a `expiresAt` o al cierre de la Order no confirma la compra y se reversa conforme a FR-023. Fuente: `HV-003`, `HV-008`; aclaraciones 3, 14. Status: DEFINED.

### FR-016 — Rechazar y liberar all-or-nothing

Ante un ticket no disponible durante la reserva inicial, la solicitud debe rechazarse sin crear Order. Ante pago rechazado, expiración o fallo definitivo de una Order ya creada, ningún ticket puede quedar vendido parcialmente y su Reservation debe cancelarse o expirar devolviendo conjuntamente todos los tickets a `AVAILABLE`. Fuente: `HV-001`, `HV-002`, `HV-003`, `HV-009`, `HV-012`, `HV-017`, `HV-018`; `HC-001`. Status: DEFINED.

### FR-017 — Procesar operaciones idempotentemente *(mod — aclaraciones 3, 4, 7)*

Solicitudes repetidas, mensajes duplicados, procesamiento repetido, invocaciones repetidas de pago y Orders terminales no deben crear reservas, pagos, ventas, cambios de inventario ni transiciones adicionales. La creación de Events es idempotente por `Idempotency-Key` vinculada al `ADMIN` (BR-032). Una solicitud de compra rechazada síncronamente no deja resultado persistido, por lo que su repetición se procesa como solicitud nueva (BR-019). La cancelación de un PaymentAttempt es idempotente por `paymentAttemptId` (BR-034). Fuente: `HV-010`, `HV-011`; aclaraciones 3, 4, 7. Status: DEFINED.

### FR-018 — Autenticar mediante Cognito JWT

Las operaciones protegidas deben requerir JWT emitido por Amazon Cognito; el backend actúa como Resource Server y obtiene roles desde `cognito:groups`. Fuente: `HV-014`. Status: DEFINED.

### FR-019 — Autorizar por rol y propiedad *(mod — aclaración 12)*

`ADMIN` puede crear Events, definir cortesías, consultar el estado de aprovisionamiento, listar Events y consultar disponibilidad; no puede iniciar compras ni consultar Orders salvo que también pertenezca al grupo `CUSTOMER`. `CUSTOMER` puede consultar Events y disponibilidad, comprar y consultar solo sus Orders. La propiedad se deriva de la identidad autenticada. Fuente: `HV-005`, `HV-014`, `HV-021`; aclaración 12. Status: DEFINED.

### FR-020 — Consultar estado de aprovisionamiento de un Event *(new — aclaraciones 6, 11, 12)*

El sistema debe permitir a un `ADMIN` consultar el estado de aprovisionamiento de un Event (`PROVISIONING`, `ENABLED` o `FAILED`) con su progreso. El `CUSTOMER` no observa Events que no estén en `ENABLED`. Fuente: aclaraciones 6, 11, 12. Status: DEFINED.

### FR-021 — Una Order activa por CUSTOMER y Event *(new — aclaración 8)*

El sistema debe impedir que un `CUSTOMER` tenga más de una Order en `CREATED` para el mismo Event. Una segunda solicitud de compra para ese Event mientras la primera está activa se rechaza síncronamente con `ACTIVE_ORDER_EXISTS` (HTTP 409), sin crear Reservation, Order ni Order ID y sin modificar el inventario. Una repetición con la misma idempotency key se resuelve como repetición. Cuando la Order activa alcanza un estado terminal, el `CUSTOMER` puede iniciar una nueva compra para ese Event. Fuente: aclaración 8. Status: DEFINED.

### FR-022 — Rechazar compra sobre Event no vendible *(new — aclaraciones 5, 6)*

El sistema debe rechazar síncronamente la compra sobre un Event pasado con `EVENT_NOT_ON_SALE` (HTTP 409), distinto del rechazo por indisponibilidad, y tratar como inexistente un Event que no está en `ENABLED`. En ambos casos no crea Reservation, Order ni Order ID y no modifica el inventario. Fuente: aclaraciones 5, 6. Status: DEFINED.

### FR-023 — Reversar pagos aprobados o desconocidos sin compra confirmada *(new — aclaración 3)*

Cuando una Order con un PaymentAttempt aprobado o de resultado desconocido se cierra sin confirmarse, el sistema debe marcarla como pendiente de reverso en la transición de cierre, solicitar al proveedor la cancelación del PaymentAttempt con reintento acotado, retirar la marca y dejar evidencia auditable al confirmarse, y alertar y dejar el reverso pendiente de revisión manual si agota los reintentos. La Order conserva su estado terminal y la marca no se expone. Fuente: aclaración 3. Status: DEFINED.

### FR-024 — Rechazar compra con el sistema de colas indisponible *(new — aclaración 9)*

Cuando el sistema de colas está indisponible, el sistema debe rechazar síncronamente la compra con `SERVICE_UNAVAILABLE` (HTTP 503) e indicación de reintento, antes de reservar, sin crear Reservation, Order ni Order ID y sin modificar el inventario. Con el sistema de colas disponible, el fallo definitivo de encolado sigue produciendo una Order `FAILED` (ALT-006). Fuente: aclaración 9. Status: DEFINED.

## 12. Business rules

| ID | Rule | Source |
|---|---|---|
| BR-001 | Cada Ticket tiene exactamente un estado. | Notas generales; `HV-001` |
| BR-002 | Una Reservation dura como máximo diez minutos desde su creación completa y exitosa; `expiresAt` es ese instante límite y ninguna compra se confirma después de `expiresAt`, aunque sus tickets todavía no se hayan liberado. *(mod)* | Requisito Funcional 2; `HV-012`; aclaración 14 |
| BR-003 | Solo un pago exitoso aplicado antes de `expiresAt`, con la Order en `CREATED`, confirma la compra; si no se confirma antes del vencimiento, todos los tickets se liberan. Una aprobación tardía nunca reabre ni confirma una Order terminal. *(mod)* | Requisitos Funcionales 2 y 6; `HV-003`, `HV-008`; aclaraciones 3, 14 |
| BR-004 | `RESERVED` no representa venta. | Notas generales |
| BR-005 | `PENDING_CONFIRMATION` no representa venta. | Notas generales; `HV-004` |
| BR-006 | `SOLD` es final e irreversible. | Notas generales |
| BR-007 | `COMPLIMENTARY` es final, consume capacidad y no representa venta. | Notas generales; `HV-005`, `HV-013` |
| BR-008 | No se pueden vender más tickets que los disponibles ni vender el mismo Ticket a más de una persona. | Contexto; Requisito Funcional 5 |
| BR-009 | Toda transición debe ser atómica. | Notas generales |
| BR-010 | Toda transición debe ser auditable. | Notas generales |
| BR-011 | Expirar una Reservation devuelve conjuntamente todos sus tickets a `AVAILABLE`. | Requisitos Funcionales 2 y 6; `HV-012` |
| BR-012 | Solo `AVAILABLE` cuenta como disponibilidad comercial; `RESERVED`, `PENDING_CONFIRMATION`, `SOLD` y `COMPLIMENTARY` no. | Requisito Funcional 7; `HV-004`, `HV-005`, `HV-006` |
| BR-013 | Una Order se cumple completa o no se cumple; nunca hay reserva, confirmación o venta parcial. | `HV-001`, `HV-002`, `HV-003`, `HV-017` |
| BR-014 | Una Order pertenece a un único Event y solicita entre 1 y 10 tickets de ese Event, sin identificadores repetidos. *(mod)* | `HV-017`; aclaración 1 |
| BR-015 | Una Order tiene como máximo una Reservation activa, que contiene exactamente sus tickets solicitados. | `HV-012`, `HV-017` |
| BR-016 | `COMPLIMENTARY` solo se define por `ADMIN` durante la creación del Event, mediante los rangos de asientos de cortesía de la definición del inventario; no se agrega ni se alcanza por transición posterior. *(mod)* | `HV-005`, `HV-014`; aclaración 6 |
| BR-017 | La selección visual no reserva ni inicia el temporizador. | `HV-002`, `HV-012` |
| BR-018 | La disponibilidad mostrada es informativa y debe revalidarse al iniciar formalmente la compra. | `HV-007` |
| BR-019 | Una repetición de una operación aceptada conserva o retorna el resultado previo sin repetir efectos. Un rechazo síncrono (por indisponibilidad u otra causa) no persiste Reservation, Order ni registro de idempotencia; una repetición con la misma idempotency key se procesa como solicitud nueva y, si crea una Order, la clave queda registrada con esa Order. *(mod)* | `HV-011`; aclaración 4 |
| BR-020 | Una Order no puede tener más de un intento de pago activo simultáneamente. | `HV-011` |
| BR-021 | `Event.capacity` debe ser igual al número total de tickets derivados de la definición del inventario, incluidos los `COMPLIMENTARY`. *(mod)* | `HV-005`, `HV-016`; aclaración 6 |
| BR-022 | Un Event futuro en `ENABLED` sigue visible en el listado con disponibilidad comercial cero, identificado con `soldOut`; un Event pasado o no `ENABLED` no aparece en la consulta normal. La disponibilidad por identificador de un Event pasado sigue respondiendo de forma informativa. *(mod)* | `HV-020`; aclaraciones 5, 11, 13 |
| BR-023 | Para un `CUSTOMER`, una Order inexistente y una Order ajena son funcionalmente indistinguibles. | `HV-014`, `HV-021` |
| BR-024 | Un `CUSTOMER` no puede tener más de una Order en `CREATED` por Event; la segunda compra se rechaza con `ACTIVE_ORDER_EXISTS`. Al alcanzar la Order activa un estado terminal, puede iniciar otra compra para ese Event. Una repetición con la misma idempotency key se resuelve como repetición. *(new)* | Aclaración 8 |
| BR-025 | Un Event es pasado cuando su `startsAt` es menor o igual al instante actual del servidor, en UTC, al recibir la solicitud de compra (mismo criterio que lo excluye del listado). La compra sobre un Event pasado se rechaza con `EVENT_NOT_ON_SALE`. Una Reservation creada antes de `startsAt` puede confirmarse dentro de su vigencia aunque el Event haya comenzado. *(new)* | Aclaración 5 |
| BR-026 | "Event habilitado" significa Event en `ENABLED`. Un Event solo pasa a `ENABLED` tras comprobarse que la cantidad de Ticket creados coincide con `Event.capacity`; si el aprovisionamiento falla definitivamente queda en `FAILED` y nunca es visible ni vendible. No existe habilitación ni deshabilitación manual, y el Event es inmutable una vez habilitado. *(new)* | Aclaraciones 2, 6, 11 |
| BR-027 | Mientras un Event no está en `ENABLED`, no aparece en el listado, y la consulta de disponibilidad y el inicio de compra lo tratan como inexistente. Para el `CUSTOMER` un Event no `ENABLED` no existe; el `ADMIN` observa su ciclo mediante la consulta de estado de aprovisionamiento. *(new)* | Aclaraciones 5, 6, 11 |
| BR-028 | Una Order que termina sin confirmarse (`EXPIRED`, o `FAILED` con resultado de pago desconocido) teniendo un PaymentAttempt aprobado o de resultado desconocido conserva su estado terminal y sus tickets liberados, y su pago se reversa: la transición de cierre la marca como pendiente de reverso; el sistema solicita la cancelación con reintento acotado; al confirmarla retira la marca y lo audita; si agota los reintentos, alerta y queda pendiente de revisión manual. La marca no se expone en la consulta de la Order. *(new)* | Aclaración 3 |
| BR-029 | No se inicia un pago cuando faltan menos de 15 s para `expiresAt`; esa Order se cierra por expiración. *(new)* | Aclaración 14 |
| BR-030 | Los tickets de una Reservation vencida se liberan como máximo 15 s después de `expiresAt`. *(new)* | Aclaración 14 |
| BR-031 | Precedencia de rechazos en el inicio de compra: (1) validación; (2) idempotencia (repetición o clave reutilizada); (3) Event inexistente o no habilitado; (4) Event pasado; (5) Ticket inexistentes; (6) sistema de colas indisponible; (7) Order activa existente; (8) Ticket inexistentes o no disponibles detectados en la reserva. Ningún rechazo crea Reservation, Order ni Order ID ni modifica el inventario. *(new)* | Aclaración 9 |
| BR-032 | La creación de un Event es idempotente por `Idempotency-Key` vinculada al `ADMIN` autenticado, con vigencia mínima de 24 horas: misma clave y mismo contenido devuelven el mismo `eventId` con su estado de aprovisionamiento actual sin crear otro Event; misma clave con contenido distinto se rechaza con `IDEMPOTENCY_KEY_REUSED` sin efectos. *(new)* | Aclaración 7 |
| BR-033 | El sistema genera el identificador de cada Ticket como `<sección>-<fila>-<asiento>`; las secciones tienen código único, las filas etiqueta única dentro de su sección y los asientos se numeran desde 1. *(new)* | Aclaración 6 |
| BR-034 | La cancelación de un PaymentAttempt es idempotente por `paymentAttemptId` y válida cualquiera que sea el estado del intento ante el proveedor. Si la cancelación llega antes que el cobro, el proveedor la registra y rechaza cualquier cobro posterior con ese `paymentAttemptId`. *(new)* | Aclaración 3 |

### 12.1 Limitaciones funcionales aceptadas *(new — aclaración 8)*

Declaradas por la decisión humana que origina la aclaración 8; no son reglas nuevas:

1. La regla de una Order activa por `CUSTOMER` y Event (BR-024) no impide el acaparamiento mediante varias cuentas.
2. Una Order retenida para revisión manual por inconsistencia técnica (retención no expuesta, sin estado de negocio propio) mantiene el bloqueo de Order activa hasta su revisión. Según la respuesta humana a `ADR-005`, esa Order se retira del proceso de expiración; constituye, por tanto, una excepción aceptada a FR-011 limitada a ese escenario de inconsistencia.

## 13. Validations

| ID | Validation | Source |
|---|---|---|
| VAL-001 | La Reservation no puede permanecer vigente por más de diez minutos sin confirmación: ninguna compra se confirma después de `expiresAt`. *(mod)* | Requisito Funcional 2; `HV-012`; aclaración 14 |
| VAL-002 | Todos los tickets se revalidan inmediatamente antes de crear la Reservation. | Requisito Funcional 3; `HV-002`, `HV-007` |
| VAL-003 | Solo una Reservation vencida y no confirmada es elegible para liberación por expiración. | Requisitos Funcionales 2 y 6; `HV-012` |
| VAL-004 | Ninguna operación concurrente puede vender por encima de la disponibilidad o reservar dos veces el mismo Ticket. | Contexto; Requisito Funcional 5; `HV-006` |
| VAL-005 | Un Ticket no puede quedar simultáneamente en más de un estado. | Notas generales |
| VAL-006 | Nombre y lugar del Event son obligatorios y no vacíos. | `HV-016` |
| VAL-007 | Fecha y hora del Event son obligatorias, se interpretan respecto de UTC y deben ser futuras al crear. | `HV-016` |
| VAL-008 | La capacidad es un entero mayor que cero y menor o igual a 50.000. *(mod)* | `HV-016`; aclaración 2 |
| VAL-009 | La capacidad debe coincidir con la cantidad de asientos derivados de la definición del inventario, cortesías incluidas. *(mod)* | `HV-005`, `HV-016`; aclaración 6 |
| VAL-010 | Todos los tickets de una Order deben existir, pertenecer al mismo Event y estar `AVAILABLE` al crear la Reservation. | `HV-002`, `HV-017` |
| VAL-011 | La identidad propietaria de una Order se toma del JWT y debe coincidir al consultar. | `HV-014`, `HV-021` |
| VAL-012 | Una solicitud de compra contiene entre 1 y 10 Ticket IDs sin repetidos; se valida antes de reservar. Incumplirla produce error de validación sin Reservation, Order ni Order ID y sin cambio de inventario. *(new)* | Aclaración 1 |
| VAL-013 | La definición del inventario respeta los límites: como máximo 100 secciones, 2.000 filas en total, 1.000 asientos por fila y 500 rangos de cortesía; códigos de sección únicos y etiquetas de fila únicas dentro de su sección. *(new)* | Aclaración 6 |
| VAL-014 | Al iniciar una compra, el Event debe existir, estar en `ENABLED` y tener `startsAt` posterior al instante actual del servidor en UTC. *(new)* | Aclaraciones 5, 6 |
| VAL-015 | En la consulta de disponibilidad, el tamaño de página no supera 100; un cursor inválido o una sección inexistente producen error de validación. *(new)* | Aclaración 10 |
| VAL-016 | La creación de un Event exige `Idempotency-Key`; la misma clave con contenido distinto se rechaza. *(new)* | Aclaración 7 |

La ubicación técnica de estas validaciones pertenece a arquitectura.

### 13.1 Valores configurables desplegados *(new — aclaraciones 1, 2, 6, 7, 10, 14)*

| Value | Deployed value | Configurable | Source |
|---|---|---|---|
| Máximo de Ticket por Order | 10 | Sí | Aclaración 1 |
| Capacidad máxima por Event | 50.000 | Sí, con nuevas pruebas de capacidad | Aclaración 2 |
| Secciones por definición | 100 | Sí | Aclaración 6 |
| Filas totales por definición | 2.000 | Sí | Aclaración 6 |
| Asientos por fila | 1.000 | Sí | Aclaración 6 |
| Rangos de cortesía por definición | 500 | Sí | Aclaración 6 |
| Vigencia mínima de la `Idempotency-Key` de creación de Event | 24 horas | No indicado | Aclaración 7 |
| Tamaño máximo de página de disponibilidad | 100 | No indicado | Aclaración 10 |
| Antigüedad máxima de la cantidad disponible | 1 s | No indicado | Aclaración 10 |
| Demora máxima de liberación tras `expiresAt` | 15 s | Sí | Aclaración 14 |
| Margen de corte para iniciar un pago | 15 s | Sí | Aclaración 14 |

La duración de diez minutos de la Reservation proviene del requerimiento original y no se declara configurable.

## 14. Alternative flows

### ALT-001 — Reservation expirada *(mod — aclaraciones 3, 14)*

Al alcanzarse `expiresAt` (diez minutos desde la creación exitosa) sin pago confirmado, la Reservation expira, ninguna compra puede confirmarse, todos los tickets regresan conjuntamente a `AVAILABLE` como máximo 15 s después de `expiresAt` y la Order pasa a `EXPIRED`. Si existía un PaymentAttempt aprobado o de resultado desconocido, se aplica ALT-008. Fuente: Requisitos Funcionales 2 y 6; `HV-003`, `HV-012`; aclaraciones 3, 14.

### ALT-002 — Ticket no disponible

Si al iniciar formalmente la compra cualquier ticket ya no está `AVAILABLE`, la solicitud completa se rechaza sin crear Reservation, Order ni Order ID. `REJECTED` solo aplica a una Order creada consistentemente que recibe un rechazo funcional posterior. Fuente: `HV-002`, `HV-009`; `HC-001`.

### ALT-003 — Mensaje u operación duplicada *(mod — aclaraciones 4, 7)*

El sistema reconoce la misma operación aceptada y conserva su resultado sin repetir Reservation, PaymentAttempt, venta, inventario ni transición terminal. La repetición de una creación de Event con la misma clave y contenido devuelve el mismo `eventId` y su estado actual. La repetición de una solicitud de compra que fue rechazada síncronamente no encuentra resultado previo y se procesa como solicitud nueva: puede crear una Order si los tickets volvieron a estar disponibles. Fuente: `HV-010`, `HV-011`; aclaraciones 4, 7.

### ALT-004 — Fallo transitorio recuperable

Puede reintentarse cuando corresponda, sin crear un segundo intento de pago activo ni repetir efectos. La clasificación, límites y backoff concretos pertenecen a arquitectura. Fuente: Especificaciones Técnicas; `HV-011`.

### ALT-005 — Pago rechazado

El Payment Mock retorna rechazo definitivo; la Order termina en `REJECTED`, ningún ticket queda `SOLD` y todos los tickets reservados se liberan conjuntamente. Fuente: `HV-003`, `HV-008`, `HV-009`.

### ALT-006 — Fallo definitivo de encolado *(mod — aclaración 9)*

Con el sistema de colas disponible al iniciar la compra, si el encolado falla definitivamente la Order termina en `FAILED`, se cancela la Reservation y todos los tickets vuelven conjuntamente a `AVAILABLE`; no queda una reserva indefinida. Si el sistema de colas está indisponible antes de reservar, aplica ALT-010 y no se crea Order. Fuente: `HV-018`; aclaración 9.

### ALT-007 — Consulta no autorizada

Si un `CUSTOMER` solicita una Order de otro usuario, el resultado funcional es recurso no encontrado, igual que para un identificador inexistente. Fuente: `HV-014`, `HV-021`.

### ALT-008 — Reverso de pago no aplicado *(new — aclaración 3)*

Una Order con PaymentAttempt aprobado o de resultado desconocido se cierra como `EXPIRED`, o como `FAILED` con resultado desconocido. La Order conserva su estado terminal, sus tickets quedan liberados y queda marcada como pendiente de reverso. El sistema solicita la cancelación del PaymentAttempt con reintento acotado; al confirmarse retira la marca y lo audita. Una aprobación posterior nunca reabre ni confirma la Order. Fuente: aclaración 3.

### ALT-009 — Cancelación anterior al cobro *(new — aclaración 3)*

Si la cancelación de un PaymentAttempt llega al proveedor antes que el cobro, el proveedor la registra y rechaza cualquier cobro posterior con el mismo `paymentAttemptId`. Fuente: aclaración 3.

### ALT-010 — Sistema de colas indisponible al iniciar la compra *(new — aclaración 9)*

La compra se rechaza síncronamente con `SERVICE_UNAVAILABLE` (HTTP 503) e indicación de reintento, antes de reservar, sin crear Reservation, Order ni Order ID y sin modificar el inventario. El `CUSTOMER` puede repetir la solicitud. Fuente: aclaración 9.

### ALT-011 — Pago no iniciado por margen de corte *(new — aclaración 14)*

Si al procesar la Order faltan menos de 15 s para `expiresAt`, el sistema no inicia el pago; los tickets permanecen `RESERVED` y la Order se cierra por expiración (`EXPIRED`) con liberación conjunta de sus tickets. Fuente: aclaración 14.

### ALT-012 — Confirmación después del inicio del Event *(new — aclaración 5)*

Una Reservation creada antes de `startsAt` puede confirmarse dentro de su vigencia (antes de `expiresAt`) aunque el Event ya haya comenzado. Comportamiento aceptado. Fuente: aclaración 5.

### ALT-013 — Nueva compra tras finalizar la Order activa *(new — aclaración 8)*

Cuando la Order `CREATED` de un `CUSTOMER` en un Event alcanza cualquier estado terminal, ese `CUSTOMER` puede iniciar una nueva compra para el mismo Event. Fuente: aclaración 8.

## 15. Error scenarios

| ID | Scenario | Required outcome | Source |
|---|---|---|---|
| ERR-001 | Solicitudes concurrentes compiten por el mismo Ticket. | Solo una puede reservarlo; cero sobreventa. | Requisito Funcional 5; `HV-006` |
| ERR-002 | Algún Ticket no está disponible al reservar. | Rechazo síncrono completo; no se crea Reservation, Order ni Order ID. | `HV-002`, `HV-009`; `HC-001` |
| ERR-003 | Reservation alcanza `expiresAt` sin confirmación. *(mod)* | Ninguna confirmación posterior; Order `EXPIRED`; liberación conjunta a `AVAILABLE` como máximo 15 s después de `expiresAt`. | Requisitos Funcionales 2 y 6; `HV-012`; aclaración 14 |
| ERR-004 | Mensaje entregado más de una vez. | Reprocesamiento idempotente sin nuevos efectos. | `HV-010`, `HV-011` |
| ERR-005 | Error transitorio susceptible de retry. | Reintento seguro sin pago o transición duplicados; política concreta pendiente de arquitectura. | Especificaciones Técnicas; `HV-011` |
| ERR-006 | Order no existe. | Recurso no encontrado. | `HV-021` |
| ERR-007 | Encolado falla definitivamente con el sistema de colas disponible al iniciar la compra. *(mod)* | Order `FAILED`; cancelar Reservation y devolver conjuntamente todos sus tickets a `AVAILABLE`. | `HV-018`; aclaración 9 |
| ERR-008 | Pago rechazado o falla definitivamente después de crear la Order. *(mod)* | Order `REJECTED` por rechazo funcional o `FAILED` por fallo técnico; la Reservation se cancela y todos sus tickets regresan conjuntamente a `AVAILABLE`. Si el `FAILED` se produce con resultado de pago desconocido, la Order queda pendiente de reverso (BR-028). | `HV-003`, `HV-008`, `HV-009`; aclaración 3 |
| ERR-009 | `CUSTOMER` consulta Order ajena. | Recurso no encontrado sin revelar existencia. | `HV-014`, `HV-021` |
| ERR-010 | Solicitud de compra con 0 tickets, más de 10 o identificadores repetidos. *(new)* | Error de validación antes de reservar; sin Reservation, Order ni Order ID; inventario sin cambios. | Aclaración 1 |
| ERR-011 | Creación de Event con capacidad superior a 50.000 o definición fuera de límites o inválida. *(new)* | Rechazo completo con error de validación; no se crea el Event ni sus Ticket. | Aclaraciones 2, 6 |
| ERR-012 | Compra sobre un Event pasado. *(new)* | `EVENT_NOT_ON_SALE` (HTTP 409), distinto de la indisponibilidad; sin Reservation, Order ni Order ID; inventario sin cambios. | Aclaración 5 |
| ERR-013 | Compra o consulta de disponibilidad sobre un Event inexistente o que no está en `ENABLED` (`PROVISIONING` o `FAILED`). *(new)* | El Event se trata como inexistente; en la compra no se crea Reservation, Order ni Order ID. | Aclaraciones 5, 6 |
| ERR-014 | `CUSTOMER` con una Order `CREATED` en un Event inicia otra compra para el mismo Event con otra idempotency key. *(new)* | `ACTIVE_ORDER_EXISTS` (HTTP 409); sin Reservation, Order ni Order ID; inventario sin cambios. | Aclaración 8 |
| ERR-015 | Sistema de colas indisponible al iniciar la compra. *(new)* | `SERVICE_UNAVAILABLE` (HTTP 503) con indicación de reintento, antes de reservar; sin Reservation, Order ni Order ID; inventario sin cambios. | Aclaración 9 |
| ERR-016 | El proveedor aprueba un pago cuya Order ya terminó sin confirmarse, o cuya aprobación llega después de `expiresAt`. *(new)* | La Order no se confirma ni se reabre; conserva o alcanza su estado terminal; sus tickets quedan `AVAILABLE`; se solicita una única cancelación del PaymentAttempt. | Aclaraciones 3, 14 |
| ERR-017 | La cancelación de un pago agota sus reintentos. *(new)* | Alerta; el reverso queda pendiente de revisión manual; la Order conserva su estado terminal. | Aclaración 3 |
| ERR-018 | El aprovisionamiento de un Event falla definitivamente. *(new)* | Event `FAILED`; nunca visible ni vendible; el `ADMIN` lo observa en la consulta de estado. | Aclaración 6 |
| ERR-019 | Creación de Event con una `Idempotency-Key` ya usada por el mismo `ADMIN` con contenido distinto. *(new)* | `IDEMPOTENCY_KEY_REUSED` (HTTP 422); sin efectos. | Aclaración 7 |
| ERR-020 | Consulta de disponibilidad con cursor inválido o sección inexistente. *(new)* | Error de validación. | Aclaración 10 |
| ERR-021 | Solicitud de compra con una idempotency key ya usada por el mismo `CUSTOMER` con contenido distinto. *(new)* | Rechazo por clave reutilizada (`IDEMPOTENCY_KEY_REUSED`) sin efectos, con la precedencia de BR-031. | Aclaración 9; respuestas humanas `ADR-007` y `ADR-016` |

## 16. Acceptance criteria

### AC-001 — Crear Event válido *(mod — aclaraciones 2, 6, 7, 11)*

Given un `ADMIN` autenticado con `Idempotency-Key`, nombre y lugar no vacíos, instante futuro en UTC, capacidad entre 1 y 50.000 y una definición compacta dentro de los límites cuyo número de asientos derivados, cortesías incluidas, es igual a la capacidad
When solicita crear el Event
Then recibe de inmediato (HTTP 202) el `eventId` con estado `PROVISIONING`, y el Event pasa a `ENABLED` solo cuando existen exactamente `capacity` Ticket con identificador `<sección>-<fila>-<asiento>`, los de los rangos de cortesía en `COMPLIMENTARY` y el resto en `AVAILABLE`.

### AC-002 — Consultar Events *(mod — aclaraciones 5, 11, 13)*

Given Events futuros en `ENABLED` con y sin tickets comerciales, un Event pasado y un Event en `PROVISIONING`
When se realiza la consulta normal
Then devuelve ambos Events futuros `ENABLED`, informa `soldOut` verdadero para el agotado sin informar la cantidad exacta de disponibles, y excluye el pasado y el que está en `PROVISIONING`.

### AC-003 — Retornar identificador al iniciar compra *(mod — aclaración 9)*

Given el sistema de colas estaba disponible al iniciar la compra y una Reservation completa y su Order fueron creadas consistentemente
When la solicitud queda aceptada para continuar o existe un resultado persistido consultable por fallo definitivo de encolado
Then se retorna el Order ID sin esperar la respuesta del Payment Mock.

### AC-004 — Reservar tickets temporalmente *(mod — aclaraciones 1, 8)*

Given entre 1 y 10 tickets solicitados sin repetidos, todos `AVAILABLE` y del mismo Event, y el `CUSTOMER` no tiene otra Order `CREATED` en ese Event
When el `CUSTOMER` inicia formalmente la compra
Then se reservan todos atómicamente, se crea una Reservation exacta, comienza el plazo de diez minutos y se crea la Order `CREATED`.

### AC-005 — Procesar Order asíncronamente

Given una Order `CREATED` con todos sus tickets `RESERVED` y su mensaje disponible para consumo
When el consumidor inicia el procesamiento de pago
Then todos los tickets pasan conjuntamente a `PENDING_CONFIRMATION`, todavía no cuentan como venta y la respuesta síncrona original no permanece bloqueada.

### AC-006 — Consultar estado de Order

Given una Order propia existente en cualquier estado, incluido terminal
When su `CUSTOMER` la consulta
Then obtiene el estado actual y una causa funcional cuando no fue confirmada.

### AC-007 — Evitar sobreventa concurrente

Given dos o más solicitudes concurrentes incluyen el mismo Ticket `AVAILABLE`
When intentan reservarlo
Then como máximo una crea una Reservation que lo contiene y no se registra sobreventa.

### AC-008 — Liberar Reservation expirada *(mod — aclaraciones 3, 14)*

Given se alcanzó `expiresAt` (diez minutos desde la creación completa de la Reservation) sin confirmación
When el proceso de expiración actúa
Then la Order pasa a `EXPIRED`, todos sus tickets regresan conjuntamente a `AVAILABLE` como máximo 15 s después de `expiresAt` y, si existía un PaymentAttempt aprobado o de resultado desconocido, se solicita su cancelación.

### AC-009 — No liberar Reservation vigente *(mod — aclaración 14)*

Given una Reservation no confirmada cuyo `expiresAt` todavía no se alcanzó
When el proceso periódico evalúa reservas vencidas
Then no libera sus tickets ni cambia la Order a `EXPIRED`.

### AC-010 — Consultar disponibilidad actual *(mod — aclaraciones 2, 10)*

Given un Event `ENABLED` con tickets en distintos estados
When un `CUSTOMER` o `ADMIN` consulta su disponibilidad
Then recibe la cantidad de tickets `AVAILABLE` del Event, `generatedAt` y una página de como máximo 100 tickets que contiene solo tickets `AVAILABLE` (identificador, sección, fila, asiento), con cursor si hay más; no se listan tickets en otros estados ni todo el inventario en una respuesta, y la respuesta no garantiza adquisición.

### AC-011 — Mantener un único estado por Ticket *(mod — aclaración 10)*

Given cualquier Ticket existente
When se modifica su estado o se verifica el inventario
Then tiene exactamente uno de `AVAILABLE`, `RESERVED`, `PENDING_CONFIRMATION`, `SOLD` o `COMPLIMENTARY`. La consulta de disponibilidad no expone el estado de cada Ticket; solo lista los `AVAILABLE`.

### AC-012 — Mantener `SOLD` irreversible

Given un Ticket está `SOLD`
When se intenta otra transición
Then el Ticket permanece `SOLD`.

### AC-013 — Mantener `COMPLIMENTARY` final y no contable

Given un Ticket fue creado `COMPLIMENTARY`
When se intenta venderlo, reservarlo o cambiarlo
Then permanece `COMPLIMENTARY`, consume capacidad y no cuenta como venta.

### AC-014 — Mantener atomicidad de transición

Given una transición que afecta una Order y varios tickets
When la transición se completa o falla
Then no se observa un subconjunto confirmado, vendido, reservado o liberado de forma parcial.

### AC-015 — Auditar transiciones

Given una transición de Ticket u Order completada
When se consulta su evidencia de auditoría
Then existe evidencia de la transición sin exigir en esta especificación un mecanismo técnico concreto.

### AC-016 — Validar disponibilidad antes de crear Order

Given al menos uno de varios tickets solicitados ya no está `AVAILABLE`
When el `CUSTOMER` inicia formalmente la compra y el sistema revalida todos los tickets
Then rechaza la solicitud completa sin crear Reservation, Order ni Order ID, y ninguno de los demás tickets queda reservado.

### AC-017 — Rechazar Event inválido *(mod — aclaraciones 2, 6)*

Given datos con fecha no futura, capacidad no positiva, capacidad superior a 50.000, definición fuera de límites o capacidad diferente de la cantidad de asientos derivados
When un `ADMIN` intenta crear el Event
Then el sistema rechaza la creación completa con error de validación y no crea el Event ni sus Ticket.

### AC-018 — La selección no reserva

Given un `CUSTOMER` selecciona tickets sin iniciar formalmente la compra
When transcurre tiempo u otro usuario consulta
Then no existe Reservation, los tickets no cambian a `RESERVED` y el temporizador no comienza.

### AC-019 — Confirmar compra completa *(mod — aclaraciones 3, 14)*

Given el Payment Mock aprueba el pago y la aprobación se aplica antes de `expiresAt` con la Order en `CREATED`
When se procesa el resultado
Then todos los tickets pasan conjuntamente a `SOLD` y la Order a `CONFIRMED` una sola vez.

### AC-020 — Rechazar pago

Given el Payment Mock rechaza definitivamente el pago de una Order creada
When se procesa el resultado
Then la Order pasa a `REJECTED`, la Reservation se cancela, ningún ticket queda vendido y todos regresan conjuntamente a `AVAILABLE`.

### AC-021 — Fallar técnicamente sin parcialidad *(mod — aclaración 3)*

Given ocurre un fallo técnico definitivo durante el procesamiento de una Order creada
When la Order se cierra como `FAILED`
Then no existe venta parcial, la Reservation se cancela, todos sus tickets regresan conjuntamente a `AVAILABLE` y, si el resultado del pago es desconocido, se solicita la cancelación del PaymentAttempt sin exponer la marca de reverso.

### AC-022 — Fallo definitivo de encolado *(mod — aclaración 9)*

Given el sistema de colas estaba disponible al iniciar la compra, y una Order y Reservation fueron creadas pero su mensaje no puede encolarse definitivamente
When se determina el fallo
Then la Order queda `FAILED`, la Reservation se cancela y todos los tickets regresan conjuntamente a `AVAILABLE`.

### AC-023 — Reprocesar mensaje idempotentemente

Given el mismo mensaje de una Order se entrega varias veces
When los consumidores lo procesan
Then no se crea otra Reservation, no se inicia pago duplicado, no se vende dos veces, no se modifica otra vez el inventario y no se repite una transición terminal.

### AC-024 — Limitar intento de pago activo

Given una Order tiene un PaymentAttempt activo
When llega una repetición de la misma operación
Then no se inicia un segundo intento activo y se conserva el procesamiento existente.

### AC-025 — Reprocesar Order terminal *(mod — aclaración 3)*

Given una Order está `CONFIRMED`, `REJECTED`, `FAILED` o `EXPIRED`
When recibe nuevamente una operación ya procesada o una aprobación de pago tardía
Then conserva su estado, no se reabre ni se confirma y no ejecuta nuevos efectos de negocio distintos del reverso de BR-028.

### AC-026 — Impedir cortesías posteriores *(mod — aclaraciones 6, 11)*

Given un Event ya fue creado
When un actor intenta agregar una cortesía, convertir otro Ticket a `COMPLIMENTARY` o modificar el Event una vez `ENABLED`
Then la operación se rechaza o no existe; el Event habilitado permanece inmutable.

### AC-027 — No revelar Order ajena

Given un `CUSTOMER` consulta un identificador inexistente o perteneciente a otro usuario
When el sistema autoriza la consulta
Then en ambos casos responde funcionalmente recurso no encontrado.

### AC-028 — Autorizar roles *(mod — aclaraciones 6, 8, 12)*

Given JWT válidos con grupos `ADMIN`, `CUSTOMER` y ambos
When se solicitan operaciones protegidas
Then solo `ADMIN` crea Events, define cortesías y consulta el estado de aprovisionamiento; `ADMIN` y `CUSTOMER` listan Events y consultan disponibilidad; solo `CUSTOMER` (incluido un `ADMIN` que también pertenezca a `CUSTOMER`) inicia compras y consulta sus propias Orders; un `ADMIN` sin grupo `CUSTOMER` no puede comprar ni consultar Orders; y la regla de una Order activa por Event se aplica a la identidad `CUSTOMER` autenticada.

### AC-029 — Cumplir objetivo de carga

Given una prueba representativa de esta implementación con al menos 1.000 usuarios concurrentes y aproximadamente 200 solicitudes sostenidas por segundo
When se ejecuta la carga objetivo
Then no ocurren sobreventas ni ventas duplicadas.

### AC-030 — Latencia de disponibilidad *(mod — aclaración 10)*

Given la prueba de carga objetivo de esta implementación
When se miden consultas de disponibilidad paginadas (cantidad de `AVAILABLE` más una página)
Then su latencia observada cumple `p95 < 500 ms`.

### AC-031 — Latencia de inicio de compra

Given la prueba de carga objetivo de esta implementación
When se mide la operación síncrona de inicio de compra y reserva
Then su latencia observada cumple `p95 < 1 s`, sin incluir pago ni confirmación asíncrona.

### AC-032 — Rechazar tamaño de Order inválido *(new — aclaración 1)*

Given una solicitud con 0 tickets, más de 10 o identificadores repetidos
When el `CUSTOMER` inicia la compra
Then se rechaza con error de validación sin Reservation, Order ni Order ID y sin modificar el inventario.

### AC-033 — Rechazar capacidad superior al máximo *(new — aclaración 2)*

Given una capacidad superior a 50.000
When el `ADMIN` crea el Event
Then la solicitud se rechaza completa y no se crea el Event ni sus Ticket.

### AC-034 — Aprobación tardía sobre Order expirada *(new — aclaración 3)*

Given el proveedor aprueba un pago cuya Order ya expiró
When se procesa el resultado
Then la Order permanece `EXPIRED`, sus tickets `AVAILABLE` y se solicita una única cancelación del pago.

### AC-035 — Cancelación anterior al cobro *(new — aclaración 3)*

Given una cancelación de un PaymentAttempt recibida por el proveedor antes del cobro
When llega el cobro con el mismo `paymentAttemptId`
Then el cobro se rechaza.

### AC-036 — Repetir una solicitud rechazada *(new — aclaración 4)*

Given una solicitud de compra rechazada síncronamente por indisponibilidad
When se repite con la misma idempotency key y los tickets están disponibles
Then se crea una Order y la clave queda registrada con esa Order.

### AC-037 — Rechazar compra sobre Event pasado *(new — aclaración 5)*

Given un Event cuyo `startsAt` es menor o igual al instante actual del servidor en UTC
When un `CUSTOMER` inicia la compra
Then se rechaza con `EVENT_NOT_ON_SALE` (HTTP 409) sin crear Reservation, Order ni Order ID y sin modificar el inventario.

### AC-038 — Disponibilidad de Event pasado *(new — aclaración 5)*

Given un Event pasado en `ENABLED`
When se consulta su disponibilidad por identificador
Then responde de forma informativa, aunque el Event no aparezca en el listado.

### AC-039 — Event en aprovisionamiento no visible *(new — aclaraciones 6, 11)*

Given una solicitud de creación válida
When el `ADMIN` crea el Event
Then recibe el `eventId` en `PROVISIONING`, puede consultar su estado y progreso, y el Event no es visible para el `CUSTOMER` (no se lista; disponibilidad y compra lo tratan como inexistente) hasta `ENABLED`.

### AC-040 — Aprovisionamiento fallido *(new — aclaración 6)*

Given un aprovisionamiento de Event que falla definitivamente
When el sistema lo determina
Then el Event queda `FAILED` y no es visible ni vendible; el `ADMIN` observa `FAILED` en la consulta de estado.

### AC-041 — Repetir creación de Event *(new — aclaración 7)*

Given una creación de Event ya aceptada
When el `ADMIN` la repite con la misma `Idempotency-Key` y el mismo contenido
Then recibe el mismo `eventId` con su estado de aprovisionamiento actual (HTTP 200 con indicación de repetición) y no se crea otro Event.

### AC-042 — Reutilizar clave de creación con otro contenido *(new — aclaración 7)*

Given una `Idempotency-Key` ya usada por el mismo `ADMIN` para crear un Event
When la usa con un contenido distinto
Then se rechaza con `IDEMPOTENCY_KEY_REUSED` (HTTP 422) sin efectos.

### AC-043 — Rechazar segunda Order activa *(new — aclaración 8)*

Given un `CUSTOMER` con una Order `CREATED` en un Event
When inicia otra compra para el mismo Event con otra idempotency key
Then se rechaza con `ACTIVE_ORDER_EXISTS` (HTTP 409) sin crear Reservation, Order ni Order ID y sin modificar el inventario.

### AC-044 — Solicitudes simultáneas del mismo CUSTOMER *(new — aclaración 8)*

Given dos solicitudes simultáneas del mismo `CUSTOMER` para el mismo Event con claves distintas
When se procesan
Then como máximo una crea Order.

### AC-045 — Comprar tras finalizar la Order activa *(new — aclaración 8)*

Given la Order `CREATED` de un `CUSTOMER` en un Event alcanzó un estado terminal
When ese `CUSTOMER` inicia una nueva compra para el mismo Event
Then la solicitud no se rechaza por `ACTIVE_ORDER_EXISTS`.

### AC-046 — Sistema de colas indisponible *(new — aclaración 9)*

Given el sistema de colas indisponible
When un `CUSTOMER` inicia la compra
Then recibe `SERVICE_UNAVAILABLE` (HTTP 503) con indicación de reintento, sin Order ni cambios de inventario.

### AC-047 — Precedencia de rechazos *(new — aclaración 9)*

Given una solicitud de compra que incumple simultáneamente varias condiciones de BR-031
When el `CUSTOMER` la inicia
Then el rechazo informado corresponde a la condición de mayor precedencia (por ejemplo, un Event pasado se informa como `EVENT_NOT_ON_SALE` aunque el `CUSTOMER` tenga una Order activa en él) y no se crea nada.

### AC-048 — Parámetros de disponibilidad inválidos *(new — aclaración 10)*

Given un cursor inválido o una sección inexistente en el Event
When se consulta la disponibilidad
Then se responde con error de validación.

### AC-049 — No iniciar pago dentro del margen de corte *(new — aclaración 14)*

Given una Order `CREATED` cuyo procesamiento comienza cuando faltan menos de 15 s para `expiresAt`
When el consumidor la procesa
Then no se inicia el pago y la Order se cierra como `EXPIRED` con liberación conjunta de sus tickets.

### AC-050 — No confirmar después de `expiresAt` *(new — aclaraciones 3, 14)*

Given una Order `CREATED` cuyo `expiresAt` ya se alcanzó pero cuyos tickets todavía no se liberaron
When llega una aprobación del pago
Then la compra no se confirma, la Order termina `EXPIRED`, sus tickets se liberan como máximo 15 s después de `expiresAt` y se solicita la cancelación del pago.

### AC-051 — Confirmar tras el inicio del Event *(new — aclaración 5)*

Given una Reservation creada antes de `startsAt` y vigente
When el pago se aprueba antes de `expiresAt` pero después de `startsAt`
Then la compra se confirma.

Los objetivos de AC-029 a AC-031 son objetivos verificables de la prueba, no capacidad garantizada para producción. Los IDs `AC-001` a `AC-031` conservan su significado de la versión 4; los criterios introducidos por las aclaraciones funcionales usan IDs nuevos a partir de `AC-032`.

## 17. Concurrency, atomicity and consistency

- La unidad de competencia es el Ticket individual.
- Solo una solicitud concurrente puede obtener un mismo Ticket.
- La creación de Reservation es all-or-nothing sobre todos los tickets de la Order.
- Las transiciones conjuntas de tickets no pueden dejar subconjuntos vendidos, reservados o liberados.
- La coherencia observable debe preservar los invariantes entre Order, Reservation, Ticket e Inventory.
- Solicitudes simultáneas del mismo `CUSTOMER` para el mismo Event con claves distintas producen como máximo una Order `CREATED` (aclaración 8).
- La confirmación y la expiración de una misma Order son excluyentes: ninguna compra se confirma después de `expiresAt` (aclaración 14).
- Deben alcanzarse cero sobreventas y cero ventas duplicadas en las pruebas objetivo.
- El mecanismo concreto —optimistic locking o conditional writes, conforme a `TC-011`— lo define arquitectura.

Fuentes: FR-010, FR-021; BR-002, BR-008, BR-009, BR-013, BR-024; `HV-001`, `HV-002`, `HV-006`, `HV-017`; aclaraciones 8, 14.

## 18. Idempotency requirements

La idempotencia es obligatoria para:

- solicitudes repetidas de inicio de compra;
- procesamiento repetido de la misma Order;
- mensajes duplicados de SQS Standard;
- invocaciones repetidas del flujo de pago;
- procesamiento de Orders terminales;
- solicitudes repetidas de creación de Event, mediante `Idempotency-Key` vinculada al `ADMIN` y vigente al menos 24 horas (aclaración 7);
- cancelaciones repetidas de un PaymentAttempt, identificadas por `paymentAttemptId` (aclaración 3).

La entrega at-least-once no puede provocar reservas, PaymentAttempts, pagos, ventas, cambios de inventario ni transiciones terminales duplicadas. Una Order admite como máximo un PaymentAttempt activo. Un reintento autorizado tras un fallo recuperable debe poseer identidad propia y seguir asociado a la misma Order.

Repetición de solicitudes de compra (aclaraciones 4 y 9):

- La compra se identifica con una idempotency key vinculada al `CUSTOMER` autenticado.
- Una repetición con la misma clave y contenido de una operación aceptada retorna la Order existente en su estado actual sin repetir efectos.
- La misma clave con contenido distinto se rechaza (`IDEMPOTENCY_KEY_REUSED`) sin efectos.
- Un rechazo síncrono no persiste Reservation, Order ni registro de idempotencia; su repetición se procesa como solicitud nueva y, si crea una Order, la clave queda registrada con esa Order.

Repetición de creación de Event (aclaración 7): misma clave y contenido devuelven el mismo `eventId` con su estado actual (HTTP 200 con indicación de repetición) sin crear otro Event; misma clave con contenido distinto produce `IDEMPOTENCY_KEY_REUSED` (HTTP 422).

Los mecanismos concretos (registro de claves, conditional writes, registro de operaciones o equivalentes) son decisiones de arquitectura.

Fuente: `HV-010`, `HV-011`; aclaraciones 3, 4, 7, 9.

## 19. Asynchronous processing requirements

- La operación síncrona inicia formalmente la compra, aplica los rechazos de BR-031, revalida tickets y crea de forma consistente Reservation y Order.
- Si el sistema de colas está indisponible al iniciar la compra, se rechaza antes de reservar con `SERVICE_UNAVAILABLE` (HTTP 503) e indicación de reintento (aclaración 9).
- El trabajo posterior se envía a Amazon SQS Standard y se procesa de forma asíncrona.
- La respuesta síncrona no espera el pago; retorna un Order ID consultable.
- El consumidor no inicia el pago si faltan menos de 15 s para `expiresAt`; en otro caso invoca el Payment Mock y aplica el resultado a todos los tickets y a la Order, sin confirmar después de `expiresAt` (aclaración 14).
- Un pago aprobado o de resultado desconocido de una Order cerrada sin confirmar se reversa de forma asíncrona con reintento acotado (aclaración 3).
- SQS tiene semántica at-least-once; el consumidor debe ser idempotente.
- Ante fallo definitivo de encolado con el sistema de colas disponible, la Order queda `FAILED` y la Reservation se revierte.
- La creación de Events es asíncrona: la respuesta inmediata (HTTP 202) precede a la escritura de los Ticket y a la habilitación del Event (aclaración 6).
- Visibility timeout, retries, DLQ, polling, mecanismo de detección de indisponibilidad de colas y estrategia de publicación pertenecen a arquitectura.

## 20. Security requirements

### 20.1 Authentication

- Amazon Cognito autentica usuarios y emite JWT.
- El backend actúa como Resource Server.
- Los roles se obtienen del claim `cognito:groups`.

### 20.2 Authorization *(mod — aclaraciones 6, 7, 8, 12)*

- `ADMIN`: crear Events y definir tickets `COMPLIMENTARY` durante la creación; consultar el estado de aprovisionamiento de un Event (operación exclusiva de `ADMIN`); listar Events y consultar disponibilidad. No puede iniciar compras ni consultar Orders salvo que también pertenezca al grupo `CUSTOMER`.
- `CUSTOMER`: consultar Events y disponibilidad, iniciar compras y consultar solo sus Orders.
- La identidad propietaria de Order se deriva del JWT, nunca de un user ID libre del cliente.
- La `Idempotency-Key` de la creación de Events se vincula al `ADMIN` autenticado; la de la compra, al `CUSTOMER` autenticado.
- La regla de una Order activa por Event se aplica a la identidad `CUSTOMER` autenticada.
- Una consulta de Order ajena no revela la existencia del recurso.

OAuth2/OIDC, configuración de Cognito, mapeo técnico de autoridades y expiración de tokens pertenecen a arquitectura.

Fuente: `HV-014`, `HV-021`; aclaraciones 6, 7, 8, 12.

## 21. Non-functional requirements

| ID | Requirement | Basis | Status / target |
|---|---|---|---|
| NFR-001 | Alta concurrencia | explicit + `HV-015` | Objetivo de prueba: al menos 1.000 usuarios concurrentes y ~200 solicitudes sostenidas/s. |
| NFR-002 | Baja latencia bajo carga | explicit + `HV-015` | Disponibilidad `p95 < 500 ms`; inicio síncrono de compra/reserva `p95 < 1 s`. |
| NFR-003 | Procesamiento no bloqueante | explicit | API reactiva con Spring WebFlux; cualitativamente definido. |
| NFR-004 | Consistencia de inventario | explicit + Human Review | Cero sobreventas, cero ventas duplicadas y ausencia de resultados parciales. |
| NFR-005 | Auditabilidad | explicit | Todas las transiciones son auditables; formato posterior. |
| NFR-006 | Acceso rápido con NoSQL | explicit + `HV-019` | Amazon DynamoDB; métrica específica no definida. |
| NFR-007 | Escalabilidad | explicit context | El sistema debe escalar ante picos de demanda; los objetivos de prueba están en NFR-001 y NFR-002, sin garantía productiva. |
| NFR-010 | Seguridad | Human Review + evaluation-driven | JWT Cognito, roles y aislamiento de Orders definidos; controles técnicos posteriores. |
| NFR-012 | Mantenibilidad | explicit deliverable | Código en inglés, separación de capas, SOLID y patrones apropiados. |
| NFR-013 | Testabilidad y cobertura | explicit deliverable | Tests unitarios/reactivos/concurrencia y cobertura mínima de 90%. |
| NFR-015 | Idempotencia | `HV-010`, `HV-011` | Ninguna repetición produce efectos de negocio adicionales. |

Los umbrales de NFR-001 y NFR-002 son objetivos de prueba de esta implementación, no SLA ni garantía de capacidad productiva. Las aclaraciones 1 a 14 no modifican requisitos no funcionales: los límites que introducen (§13.1) son validaciones o reglas funcionales.

### 21.1 Retired NFR classifications

Los IDs siguientes se conservan como retirados para no reutilizarlos con otro significado. No representan requisitos no funcionales activos; sus contenidos permanecen exclusivamente como criterios de evaluación:

| Retired ID | Previous classification | Authoritative evaluation item |
|---|---|---|
| NFR-008 | Resiliencia | EVAL-005 |
| NFR-009 | Tolerancia a fallos | EVAL-005 |
| NFR-011 | Observabilidad | EVAL-013 |
| NFR-014 | Costos y gobernanza | EVAL-013 |

## 22. Technical constraints

| ID | Constraint | Consolidated interpretation | Source |
|---|---|---|---|
| TC-001 | Java 25 | Obligatorio; usar características modernas donde apliquen. Virtual Threads siguen siendo condicionales. | Stack obligatorio |
| TC-002 | Spring Boot 4.x | Framework base obligatorio. | Stack obligatorio |
| TC-003 | Spring WebFlux | API reactiva y no bloqueante. | Stack obligatorio |
| TC-004 | Amazon DynamoDB / DynamoDB Local | DynamoDB es persistencia objetivo; DynamoDB Local para desarrollo/pruebas locales. | `HV-019` |
| TC-005 | Amazon SQS Standard / LocalStack | Cola obligatoria SQS Standard; LocalStack la emula localmente. | `HV-010` |
| TC-006 | Docker | Contenerización de aplicación y servicios. | Stack obligatorio |
| TC-007 | Clean Architecture | Capas Domain, Use Cases e Infrastructure claramente separadas. | Stack obligatorio |
| TC-008 | `Mono` / `Flux` | La API retorna tipos reactivos donde corresponda. | Especificaciones Técnicas |
| TC-009 | Error handling reactivo | Retry cuando sea apropiado; política concreta no definida aquí. | Especificaciones Técnicas |
| TC-010 | At-least-once delivery | SQS debe procesarse con semántica al menos una vez e idempotencia de aplicación. | Especificaciones Técnicas; `HV-010`, `HV-011` |
| TC-011 | Optimistic locking o conditional writes | Se preserva la alternativa para actualizaciones de inventario; arquitectura selecciona y detalla. | Especificaciones Técnicas |
| TC-012 | Docker Compose | Debe levantar aplicación, DynamoDB Local y SQS mediante LocalStack, además de dependencias necesarias. | Entregable 5; `HV-010`, `HV-019` |
| TC-013 | Inglés en código | Variables, clases, métodos y comentarios en inglés. | Entregable 1 |
| TC-014 | Herramientas de pruebas | JUnit 5, Mockito y reactor-test se conservan conforme al lenguaje del requerimiento. | Entregable 3 |
| TC-015 | Calidad interna | Principios SOLID y patrones apropiados, sin seleccionarlos aquí. | Entregable 1 |
| TC-016 | Amazon Cognito y JWT | Cognito emite JWT y el backend actúa como Resource Server. | `HV-014` |
| TC-017 | Payment Mock | Servicio simulado independiente con respuestas exitosas y fallidas deterministas. | `HV-008` |

No se definen partition keys, sort keys, GSIs, tablas, `TransactWriteItems`, modo de consistencia, visibility timeout, retries, DLQ, polling, configuración AWS SDK, estructura Terraform, topología AWS, paquetes Java, clases Spring, adapters ni diseño interno del Payment Mock.

### 22.1 Restricción técnica condicionada — no aplicada *(new — aclaración 15)*

| Aspect | Content |
|---|---|
| Origen | Aclaración 15; `ADR-017` (sucesor `ADR-036`). |
| Situación | El entorno local fija LocalStack a un tag publicado antes del 23 de marzo de 2026. Si ese tag no ofrece SQS con redrive, contador de recepciones y cambio de visibilidad, el respaldo previsto es ElasticMQ. |
| Estado | **No activada.** Mientras no se active el respaldo, esta especificación no cambia. |
| Elementos que deberán revisarse antes de aplicarla | `HV-010`, `TC-005`, `TC-012`, `DEL-005`. Permanecen **sin cambios** en esta versión. |
| Requisito para activarla | Revisión funcional humana de `HV-010` y de los elementos listados antes de modificarlos. |

## 23. Deliverables

| ID | Deliverable | Consolidated content |
|---|---|---|
| DEL-001 | Repositorio de código fuente | Clean Architecture; nombres y comentarios en inglés; capas separadas; SOLID y patrones apropiados. |
| DEL-002 | README.md | Descripción, instalación, configuración, comandos Docker, decisiones arquitectónicas y ejemplos de endpoints. |
| DEL-003 | Tests unitarios | Cobertura mínima 90%; casos de uso, componentes reactivos y concurrencia; herramientas indicadas. |
| DEL-004 | Colección de solicitudes | Postman, Insomnia o curl con flujos principales. |
| DEL-005 | `docker-compose.yml` | Aplicación y dependencias, incluidas DynamoDB Local y SQS emulada con LocalStack. |

Diagramas y Terraform permanecen criterios diferenciales/de alto valor, no entregables obligatorios inferidos.

## 24. Evaluation criteria

| ID | Criterion | Classification |
|---|---|---|
| EVAL-001 | Cumplimiento funcional | MANDATORY |
| EVAL-002 | Calidad del diseño y decisiones de arquitectura | DIFFERENTIAL |
| EVAL-003 | Concurrencia y consistencia | DIFFERENTIAL |
| EVAL-004 | Asincronía y procesamiento basado en eventos | DIFFERENTIAL |
| EVAL-005 | Escalabilidad, resiliencia y tolerancia a fallos | HIGH_VALUE_ADDITION |
| EVAL-006 | Seguridad de la solución | MANDATORY como criterio de evaluación |
| EVAL-007 | Manejo seguro de secretos y credenciales | MANDATORY dentro de seguridad |
| EVAL-008 | Ataques comunes, idempotencia y abuso | MANDATORY dentro de seguridad |
| EVAL-009 | Comunicación clara y estructurada | HIGH_VALUE_ADDITION |
| EVAL-010 | Diagramas e interacciones | HIGH_VALUE_ADDITION |
| EVAL-011 | Infraestructura como Código con Terraform | DIFFERENTIAL |
| EVAL-012 | Experiencia Cloud-Native en AWS | HIGH_VALUE_ADDITION |
| EVAL-013 | Operación, costos, observabilidad y gobernanza | HIGH_VALUE_ADDITION |
| EVAL-014 | Limitaciones, mejoras y trade-offs productivos | MANDATORY como criterio de reflexión |

Estos elementos permanecen como criterios de evaluación. Solo seguridad e idempotencia adquirieron requisitos funcionales/no funcionales concretos por `HV-010`, `HV-011` y `HV-014`, ampliados por las aclaraciones 7, 8 y 12; Terraform, diagramas, despliegue AWS, costos, observabilidad y gobernanza no se transforman automáticamente en FR o entregables obligatorios. Los mecanismos de resiliencia que originan algunas aclaraciones solo se reflejan aquí en su comportamiento observable (por ejemplo, `SERVICE_UNAVAILABLE` en la aclaración 9).

## 25. Resolved ambiguities and Human Review decisions

No quedan ambigüedades funcionales pendientes. Los identificadores `HV-*` se preservan exclusivamente como trazabilidad de decisiones resueltas.

| HV | Decision | Consolidated outcome | Main propagated items |
|---|---|---|---|
| HV-001 | CONFIRMED_WITH_CHANGE | Estados originales pertenecen a Ticket; Order tiene máquina separada; procesamiento indivisible. | DS-001..DS-010, ST-001..ST-010, FR-008, FR-013, BR-013 |
| HV-002 | CONFIRMED_WITH_CHANGE | Selección visual no reserva; compra formal revalida y reserva antes de encolar. | MF-003, FR-004..FR-006, ST-001, AC-003, AC-004, AC-016, AC-018 |
| HV-003 | CONFIRMED_WITH_CHANGE | Solo pago exitoso confirma; fallo/rechazo/expiración libera todos. | ST-002, ST-004, ST-005, FR-011, FR-015, AC-008, AC-019, AC-020 |
| HV-004 | CONFIRMED_WITH_CHANGE | `PENDING_CONFIRMATION` significa pago en curso; entrada/salida conjunta. | DS-003, ST-003..ST-005, BR-005, AC-005 |
| HV-005 | CONFIRMED_WITH_CHANGE | Cortesías solo al crear Event, por ADMIN; consumen capacidad y no son venta. | DS-005, FR-001, BR-007, BR-016, AC-013, AC-026 |
| HV-006 | CONFIRMED_WITH_CHANGE | Inventario de tickets individuales y disponibilidad derivada. | Domain model, FR-003, FR-010, BR-012, VAL-004 |
| HV-007 | CONFIRMED_WITH_CHANGE | Disponibilidad bajo demanda, informativa, sin streaming; revalidación al comprar. | CAP-002, MF-002, FR-012, BR-018, AC-010, AC-016 |
| HV-008 | CONFIRMED_WITH_CHANGE | Payment Mock independiente; aprobación y fallos deterministas. | Actor Payment Mock, FR-015, TC-017, AC-005, AC-019, AC-020 |
| HV-009 | CONFIRMED_WITH_CHANGE | Order usa `CREATED`, `CONFIRMED`, `REJECTED`, `FAILED`, `EXPIRED`. | DS-006..DS-010, ST-006..ST-010, FR-009, ERR-002/008 |
| HV-010 | CONFIRMED_WITH_CHANGE | Amazon SQS Standard obligatorio; LocalStack local; aplicación maneja duplicados. | TC-005, TC-010, TC-012, FR-005, FR-017 |
| HV-011 | CONFIRMED_WITH_CHANGE | Idempotencia de solicitudes, órdenes, mensajes y pagos; un pago activo máximo. | FR-017, BR-019/020, NFR-015, AC-023..AC-025 |
| HV-012 | CONFIRMED_WITH_CHANGE | Temporizador inicia al crear Reservation completa; contiene tickets exactos; sin liberación parcial. | FR-004, FR-011, BR-002/015/017, VAL-001/003 |
| HV-013 | CONFIRMED_WITH_CHANGE | Reportes fuera de alcance; semántica contable de estados preservada. | Scope, DS-004/005, BR-004..BR-007 |
| HV-014 | CONFIRMED_WITH_CHANGE | Cognito JWT, roles ADMIN/CUSTOMER y propiedad de Order desde identidad autenticada. | Actors, FR-018/019, NFR-010, TC-016, AC-027/028 |
| HV-015 | CONFIRMED_WITH_CHANGE | Objetivos de prueba: 1.000 usuarios, ~200 req/s, p95 definidos y cero duplicidad/sobreventa. | NFR-001/002/004, AC-029..AC-031 |
| HV-016 | CONFIRMED_WITH_CHANGE | Campos obligatorios, instante futuro UTC, capacidad positiva e igual al total de tickets. | FR-001, BR-021, VAL-006..VAL-009, AC-001/017 |
| HV-017 | CONFIRMED_WITH_CHANGE | Order de un Event, 1..N tickets, 0..1 Reservation activa y all-or-nothing. | Domain invariants, FR-016, BR-013..BR-015 |
| HV-018 | CONFIRMED_WITH_CHANGE | Order inicia `CREATED`; fallo definitivo de encolado produce `FAILED` y revierte Reservation. | DS-006/009, ST-006/009, FR-005/006, ALT-006, AC-003, AC-022 |
| HV-019 | CONFIRMED_WITH_CHANGE | Amazon DynamoDB objetivo y DynamoDB Local para desarrollo/pruebas. | NFR-006, TC-004, TC-012, DEL-005 |
| HV-020 | CONFIRMED_WITH_CHANGE | Event futuro habilitado visible aunque agotado; Event pasado fuera de consulta normal. | MF-002, FR-002, BR-022, AC-002 |
| HV-021 | CONFIRMED_WITH_CHANGE | Order inexistente o ajena produce recurso no encontrado; terminales siguen consultables. | MF-004, FR-009/019, BR-023, ERR-006/009, AC-006, AC-027 |

Las decisiones `HV-*` conservan su resultado. Las aclaraciones de §25.2 precisan o amplían algunas de ellas (por ejemplo, `HV-017` con el rango 1..10, `HV-020` con `ENABLED` y `soldOut`, `HV-012` con los márgenes temporales) sin revertirlas; ver §28.

### 25.1 Manual clarification applied

| ID | Decision | Consolidated outcome | Main propagated items |
|---|---|---|---|
| HC-001 | CONFIRMED_WITH_CHANGE | Si falla la reserva inicial por indisponibilidad, se rechaza la solicitud sin crear Order ni retornar Order ID. `REJECTED` se utiliza únicamente para una Order creada que recibe un rechazo funcional posterior. | DS-008, ST-008, MF-003, FR-006, FR-016, ALT-002, ERR-002, AC-016 |

### 25.2 Functional clarifications incorporated *(new)*

Fuente: `architecture/ticketing.functional-clarifications.v1.md`, derivado de `human-review/ticketing.architecture-review.yaml` (`review.status: APPROVED`, 2026-10-04). Detalle de propagación en §0.

| # | Origin ID | Human decision in architecture review | Related HV / HC | Status in v5 |
|---|---|---|---|---|
| 1 | FG-001; ADR-003 | CONFIRMED; CONFIRMED | HV-017 | Incorporada |
| 2 | FG-002 | CONFIRMED_WITH_CHANGE | HV-016 | Incorporada |
| 3 | FG-003; ADR-008 | CONFIRMED_WITH_CHANGE; CONFIRMED | HV-003, HV-008, HV-011 | Incorporada |
| 4 | FG-004 | CONFIRMED | HV-011; HC-001 | Incorporada |
| 5 | FG-005 | CONFIRMED_WITH_CHANGE | HV-020 | Incorporada |
| 6 | ADR-004 → ADR-024 | CONFIRMED_WITH_CHANGE | HV-005, HV-016, HV-020 | Incorporada |
| 7 | ADR-007 → ADR-027 | CONFIRMED_WITH_CHANGE | HV-011 | Incorporada |
| 8 | ADR-013 → ADR-032 | CONFIRMED_WITH_CHANGE | HV-014, HV-017 | Incorporada |
| 9 | ADR-016 → ADR-035 | CONFIRMED_WITH_CHANGE | HV-018 | Incorporada |
| 10 | ADR-021 → ADR-040 | CONFIRMED_WITH_CHANGE | HV-007 | Incorporada |
| 11 | AV-002 | CONFIRMED_WITH_CHANGE | HV-020 | Incorporada |
| 12 | AV-005 | CONFIRMED | HV-014 | Incorporada |
| 13 | AV-001 | CONFIRMED | HV-020 | Incorporada |
| 14 | AV-003 | CONFIRMED | HV-012 | Incorporada |
| 15 | ADR-017 → ADR-036 | CONFIRMED_WITH_CHANGE (condicionada) | HV-010 | No aplicada; registrada en §22.1 |

## 26. Human validation status

- Human Functional Review status: `APPROVED`.
- Decisions incorporated: 21 of 21.
- Manual clarifications incorporated: 1 of 1 (`HC-001`).
- Human Architecture Review status: `APPROVED`.
- Functional clarifications incorporated: 14 of 14 aplicables; 1 condicionada registrada sin aplicar.
- New Human Validations generated in v5: 0.
- Pending Human Validations: 0.
- Pending HIGH blocking items: 0.
- Contradictions among approved decisions: none blocking; las tensiones literales detectadas fueron resueltas por decisiones humanas posteriores y explícitas (§28).
- Human validation required: false.

## 27. Traceability matrix

| Original source | Existing / new IDs | Human decision | Consolidated behavior | Acceptance evidence |
|---|---|---|---|---|
| Context: duplicates, double sale, concurrency | FR-010, FR-021, NFR-004, FR-017 | HV-006, HV-010, HV-011, HV-015; aclaraciones 4, 7, 8 | Ticket-level concurrency, all-or-nothing, idempotency and one active Order per customer and Event | AC-007, AC-014, AC-023..AC-025, AC-029, AC-036, AC-041..AC-044 |
| Functional Requirement 1: Events and inventory | CAP-001, FR-001..FR-003, FR-020, BR-016, BR-021, BR-026, BR-027, BR-032, BR-033, VAL-008, VAL-009, VAL-013, VAL-016, DS-011..DS-013, ST-011..ST-013 | HV-005, HV-006, HV-014, HV-016, HV-020; aclaraciones 2, 6, 7, 11, 12, 13 | ADMIN requests valid Events (max 50.000) from a compact definition; asynchronous provisioning to `ENABLED`/`FAILED`; idempotent creation; future `ENABLED` Events listed with `soldOut` | AC-001, AC-002, AC-010, AC-013, AC-017, AC-026, AC-028, AC-033, AC-039..AC-042 |
| Functional Requirement 2: temporary reservation | CAP-003, FR-004, BR-002/003, BR-014, BR-029, BR-030, VAL-001, VAL-012 | HV-002, HV-003, HV-012, HV-017; aclaraciones 1, 14 | Atomic Reservation of 1..10 tickets starts on formal purchase and expires at `expiresAt`; no confirmation afterwards; release within 15 s | AC-004, AC-008, AC-009, AC-016, AC-018, AC-032, AC-049, AC-050 |
| Functional Requirement 3: asynchronous purchase | CAP-004/005/009, FR-005..FR-008, FR-015, FR-022..FR-024, BR-019, BR-025, BR-028, BR-031, BR-034, VAL-014 | HV-001..HV-004, HV-008..HV-012, HV-018; HC-001; aclaraciones 3, 4, 5, 9, 14 | Ordered synchronous rejections, reserve, create Order, enqueue, pay asynchronously, close consistently and reverse unapplied payments | AC-003..AC-005, AC-014, AC-016, AC-019..AC-025, AC-034..AC-037, AC-043, AC-046, AC-047, AC-049..AC-051 |
| Functional Requirement 4: Order query | CAP-006, FR-009 | HV-001, HV-009, HV-014, HV-018, HV-021; aclaración 3 | Separate Order states, own-resource authorization, terminal results consultable, reversal mark not exposed | AC-006, AC-021, AC-027, AC-028 |
| Functional Requirement 5: concurrency | CAP-008, FR-010/013/021, BR-008/009/024 | HV-001, HV-006, HV-015, HV-017; aclaración 8 | No overbooking, no partial result, at most one active Order per customer and Event | AC-007, AC-014, AC-016, AC-029, AC-044 |
| Functional Requirement 6: release expired reservations | CAP-007, FR-011, ST-002/ST-010, BR-030, §12.1 | HV-003, HV-004, HV-009, HV-012; aclaraciones 3, 8, 14 | Order expires and all tickets return to AVAILABLE within 15 s after `expiresAt`; accepted exception for Orders held for manual review | AC-008, AC-009, AC-049, AC-050 |
| Functional Requirement 7: real-time availability | CAP-002, FR-012, BR-012/018/022, VAL-015 | HV-005, HV-006, HV-007; aclaraciones 5, 10 | On-demand count of `AVAILABLE` plus cursor page of `AVAILABLE` tickets; informative; past Events answer by identifier | AC-010, AC-011, AC-016, AC-030, AC-038, AC-048 |
| Notes: Ticket states | DS-001..DS-005, ST-001..ST-005 | HV-001, HV-003..HV-005, HV-013; aclaraciones 6, 14 | Coherent Ticket state machine and accounting semantics | AC-005, AC-008..AC-014, AC-019, AC-020, AC-026 |
| Stack and technical specifications | TC-001..TC-017; §22.1 | HV-008, HV-010, HV-014, HV-019; aclaración 15 (no aplicada) | Required stack preserved without physical design; conditional ElasticMQ fallback recorded only | DEL-005; AC-023, AC-028 |
| Deliverables | DEL-001..DEL-005, NFR-012/013 | HV-010, HV-019 | Local dependencies fixed to DynamoDB Local and LocalStack/SQS | Delivery verification |
| Evaluation criteria | EVAL-001..EVAL-014 | HV-010, HV-011, HV-014, HV-015; aclaraciones 7, 8, 9, 12 | Criteria retained; only approved decisions become requirements | AC-023..AC-025, AC-028..AC-031, AC-041..AC-046 |

### 27.1 Clarification-to-acceptance traceability *(new)*

| Aclaración | Human decision | Spec IDs | Acceptance evidence |
|---|---|---|---|
| 1 — FG-001 / ADR-003 | CONFIRMED | FR-004, BR-014, VAL-012, ERR-010 | AC-004, AC-032 |
| 2 — FG-002 | CONFIRMED_WITH_CHANGE | FR-001, FR-012, BR-026, VAL-008, ST-012, ERR-011 | AC-001, AC-010, AC-017, AC-033 |
| 3 — FG-003 / ADR-008 | CONFIRMED_WITH_CHANGE / CONFIRMED | FR-015, FR-017, FR-023, BR-003, BR-028, BR-034, CAP-009, MF-005, ALT-008, ALT-009, ERR-008, ERR-016, ERR-017 | AC-008, AC-019, AC-021, AC-025, AC-034, AC-035, AC-050 |
| 4 — FG-004 | CONFIRMED | FR-017, BR-019, ALT-003 | AC-036 |
| 5 — FG-005 | CONFIRMED_WITH_CHANGE | FR-002, FR-022, BR-022, BR-025, VAL-014, ALT-012, ERR-012, ERR-013 | AC-002, AC-037, AC-038, AC-051 |
| 6 — ADR-004 → ADR-024 | CONFIRMED_WITH_CHANGE | FR-001, FR-003, FR-020, BR-016, BR-021, BR-026, BR-027, BR-033, VAL-009, VAL-013, DS-011..DS-013, ST-011..ST-013, ERR-011, ERR-013, ERR-018 | AC-001, AC-017, AC-026, AC-028, AC-039, AC-040 |
| 7 — ADR-007 → ADR-027 | CONFIRMED_WITH_CHANGE | FR-017, BR-032, VAL-016, ALT-003, ERR-019 | AC-001, AC-041, AC-042 |
| 8 — ADR-013 → ADR-032 | CONFIRMED_WITH_CHANGE | FR-004, FR-021, BR-024, §12.1, ALT-013, ERR-014 | AC-004, AC-028, AC-043, AC-044, AC-045 |
| 9 — ADR-016 → ADR-035 | CONFIRMED_WITH_CHANGE | FR-006, FR-024, BR-031, ALT-006, ALT-010, ERR-007, ERR-015, ERR-021 | AC-003, AC-022, AC-046, AC-047 |
| 10 — ADR-021 → ADR-040 | CONFIRMED_WITH_CHANGE | CAP-002, FR-012, VAL-015, ERR-020 | AC-010, AC-011, AC-030, AC-048 |
| 11 — AV-002 | CONFIRMED_WITH_CHANGE | CAP-001, FR-002, BR-022, BR-026, BR-027, DS-011..DS-013 | AC-001, AC-002, AC-026, AC-039 |
| 12 — AV-005 | CONFIRMED | FR-019, FR-020, §20.2 | AC-028 |
| 13 — AV-001 | CONFIRMED | FR-002, BR-022 | AC-002 |
| 14 — AV-003 | CONFIRMED | FR-011, FR-015, BR-002, BR-029, BR-030, VAL-001, ALT-001, ALT-011, ERR-003 | AC-008, AC-009, AC-019, AC-049, AC-050 |

## 28. Analysis trail

| Observation | Classification | Authority | Consolidated decision |
|---|---|---|---|
| El requerimiento mezclaba estados de Ticket y Order. | CONTRADICTION resolved | Human Review `HV-001`, `HV-009`, `HV-018` | Dos máquinas separadas y coherentes. |
| Reserva y encolado no tenían orden inequívoco. | AMBIGUITY resolved | `HV-002`, `HV-012`, `HV-018` | Revalidar y reservar atómicamente; crear Order; luego encolar. |
| `HV-009` exigía `REJECTED` por indisponibilidad, mientras `HV-018` impedía crear la Order antes de reservar. | CONTRADICTION resolved | Aclaración humana `HC-001` | La indisponibilidad inicial rechaza la solicitud sin Order; `REJECTED` solo aplica después de crearla. |
| No existía trigger de venta. | AMBIGUITY resolved | `HV-003`, `HV-008` | Pago exitoso del mock produce venta confirmada. |
| `PENDING_CONFIRMATION` no tenía semántica. | AMBIGUITY resolved | `HV-004` | Pago en curso; tickets bloqueados y no vendidos. |
| Cortesías no tenían actor ni efecto de inventario. | AMBIGUITY resolved | `HV-005`, `HV-014`, `HV-016` | ADMIN las define al crear; consumen capacidad; no son venta. |
| Inventario individual o agregado no estaba resuelto. | AMBIGUITY resolved | `HV-006`, `HV-017` | Tickets individuales; agregado derivable. |
| “Tiempo real” era ambiguo. | AMBIGUITY resolved | `HV-007` | Lectura bajo demanda, sin canal continuo. |
| At-least-once no tenía regla funcional de duplicados. | AMBIGUITY resolved | `HV-010`, `HV-011` | Idempotencia obligatoria en todos los efectos de negocio. |
| Seguridad era solo criterio de evaluación. | AMBIGUITY resolved | `HV-014`, `HV-021` | Cognito JWT, roles y aislamiento de Orders. |
| Carga y latencia no eran verificables. | AMBIGUITY resolved | `HV-015` | Objetivos de prueba cuantitativos, no SLA productivo. |
| Persistencia y cola admitían opciones contradictorias. | CONTRADICTION resolved | `HV-010`, `HV-019` | DynamoDB/DynamoDB Local y SQS Standard/LocalStack. |
| v4 no fijaba máximo de tickets por Order ("uno o más"). | FUNCTIONAL GAP resolved | `FG-001`, `ADR-003` (aclaración 1) | 1 a 10 tickets sin repetidos; error de validación en otro caso. Precisa `HV-017` sin revertirlo. |
| v4 no fijaba capacidad máxima de un Event. | FUNCTIONAL GAP resolved | `FG-002` (aclaración 2) | Máximo 50.000; habilitación tras verificar capacidad; disponibilidad paginada. |
| No estaba definido el tratamiento de un pago aprobado o desconocido de una Order cerrada sin confirmar. | FUNCTIONAL GAP resolved | `FG-003`, `ADR-008` (aclaración 3) | La Order conserva su estado terminal; reverso idempotente; aprobación tardía nunca confirma. |
| No estaba definido el efecto de repetir una compra rechazada síncronamente. | FUNCTIONAL GAP resolved | `FG-004` (aclaración 4) | La repetición se reevalúa como solicitud nueva; coherente con `HC-001`. |
| No estaba definida la compra sobre un Event pasado. | FUNCTIONAL GAP resolved | `FG-005` (aclaración 5) | `EVENT_NOT_ON_SALE`; disponibilidad por identificador informativa; confirmación tras `startsAt` aceptada. |
| `HV-020` dejaba a arquitectura un posible estado de habilitación del Event. | AMBIGUITY resolved | `ADR-004`, `AV-002` (aclaraciones 6 y 11) | Ciclo `PROVISIONING` / `ENABLED` / `FAILED` visible para el `ADMIN`; "habilitado" = `ENABLED`. La autorización prevista por `HV-020` queda ejercida por decisión humana. |
| `HV-007` hablaba de retornar "la disponibilidad vigente de sus ubicaciones"; la aclaración 10 lista solo tickets `AVAILABLE` y admite hasta 1 s de antigüedad en la cantidad. | TENSION reconciled (no blocking) | `ADR-021` respuesta humana (aclaración 10), posterior y específica; `BR-018` | La consulta sigue siendo bajo demanda, informativa y revalidada al comprar, como exige `HV-007`; la forma de la respuesta la fija la decisión humana posterior. No requiere nueva decisión humana. |
| `HV-012` indica que los tickets quedan bloqueados "durante un máximo de diez minutos"; la aclaración 14 admite liberarlos hasta 15 s después de `expiresAt`. | TENSION reconciled (no blocking) | `AV-003` respuesta humana (aclaración 14), posterior y específica; su pregunta y recomendación declaran explícitamente esa demora | La vigencia confirmable de la Reservation sigue siendo de diez minutos (BR-002, VAL-001); la liberación física puede demorarse hasta 15 s (BR-030). El mismo revisor humano decidió ese punto de forma expresa; no se genera HV. |
| `HV-018` exige que el fallo definitivo de encolado produzca una Order `FAILED`; la aclaración 9 rechaza con 503 sin crear Order cuando el sistema de colas está indisponible. | TENSION reconciled (no blocking) | `ADR-016` respuesta humana (aclaración 9) | Los ámbitos no se solapan: el 503 ocurre antes de reservar y no existe Order; `HV-018` sigue rigiendo cuando la Order ya existe (ALT-006, ERR-007). |
| La aclaración 8 menciona Orders retenidas para revisión manual por inconsistencia técnica que mantienen el bloqueo; por la respuesta humana a `ADR-005` esas Orders se retiran de la expiración. | ACCEPTED LIMITATION recorded | Aclaración 8; respuesta humana `ADR-005` | Se registra en §12.1 como excepción aceptada a FR-011, limitada a ese escenario anómalo; no se introduce estado de negocio. Se informa al Architect para una posible aclaración del resultado observable. |
| La aclaración 15 condiciona un cambio de emulador local. | CONDITIONAL CONSTRAINT recorded | Aclaración 15; `ADR-017` | Solo nota en §22.1; `HV-010`, `TC-005`, `TC-012` y `DEL-005` sin cambios. |

## 29. Architecture readiness gate

| Condition | Result |
|---|---|
| No HIGH decision remains `PENDING` | PASS |
| No contradiction among approved decisions | PASS (tensiones reconciladas por decisiones humanas explícitas; §28) |
| All 21 approved functional decisions propagated | PASS |
| Manual clarification `HC-001` propagated | PASS |
| All 14 applicable functional clarifications propagated | PASS |
| Conditional clarification 15 recorded without being applied | PASS |
| Critical flows have Acceptance Criteria | PASS |
| Ticket, Order and Event provisioning state machines are coherent | PASS |
| Atomicity and all-or-nothing rules are explicit | PASS |
| Idempotency rules are explicit | PASS |
| Mandatory technical constraints are represented | PASS |
| No blocking functional ambiguity remains | PASS |

Result: `READY_FOR_ARCHITECTURE`. La arquitectura ya fue revisada y aprobada; este estado indica que la especificación no tiene decisiones funcionales pendientes y que es coherente con la arquitectura consolidada v2.

## 30. Mandatory self-validation

| # | Check | Result |
|---|---|---|
| 1 | Las 21 decisiones `HV-*` siguen procesadas y vigentes. | PASS |
| 2 | Ninguna respuesta humana aprobada fue ignorada. | PASS |
| 3 | Ninguna ambigüedad resuelta continúa pendiente. | PASS |
| 4 | No se introdujeron suposiciones funcionales fuera de las aclaraciones 1 a 14, la v4 y las respuestas humanas. | PASS |
| 5 | No se inventaron decisiones de arquitectura; solo se incorporó lo observable (códigos HTTP, códigos de error, estados visibles y campos funcionales citados en las aclaraciones). | PASS |
| 6 | Los IDs existentes conservaron su significado; ningún ID se eliminó ni se reutilizó. | PASS |
| 7 | Los elementos nuevos usan los siguientes IDs libres: FR-020..FR-024, BR-024..BR-034, VAL-012..VAL-016, DS-011..DS-013, ST-011..ST-013, ALT-008..ALT-013, ERR-010..ERR-021, AC-032..AC-051, CAP-009, MF-005. | PASS |
| 8 | Las máquinas de Ticket, Order y aprovisionamiento de Event son separadas y coherentes. | PASS |
| 9 | Los Acceptance Criteria reflejan las decisiones humanas y las aclaraciones. | PASS |
| 10 | La trazabilidad conecta fuente, IDs, Human Review, aclaración y aceptación (§27, §27.1). | PASS |
| 11 | Las decisiones de fuera de alcance están representadas. | PASS |
| 12 | Los estados de Ticket y Order no se mezclan; `FAILED` (Event) se distingue de `FAILED` de Order. | PASS |
| 13 | No existe cumplimiento parcial de una Order. | PASS |
| 14 | `COMPLIMENTARY` es consistente en dominio, reglas, flujos y aceptación. | PASS |
| 15 | La disponibilidad es consistentemente bajo demanda, informativa, paginada y revalidada. | PASS |
| 16 | El modelo de autorización es consistente con `ADMIN` y `CUSTOMER`, incluida la aclaración 12. | PASS |
| 17 | El gate fue recalculado. | PASS |
| 18 | La aclaración 15 no modificó `HV-010`, `TC-005`, `TC-012` ni `DEL-005`. | PASS |
| 19 | Cada una de las 14 aclaraciones aplicadas aparece en la tabla de §0. | PASS |

## 31. Architectural boundary

Esta especificación no selecciona ni define:

- modelo de partición, claves, índices, sharding, tablas o consistencia concreta de DynamoDB;
- operación transaccional concreta de DynamoDB;
- visibility timeout, retry count, backoff, DLQ o polling de SQS, ni colas adicionales;
- circuit breaker u otro mecanismo de detección de indisponibilidad; solo su resultado observable (`SERVICE_UNAVAILABLE`);
- caché, conteo o lectura concreta de la disponibilidad; solo su contrato observable;
- mecanismo de bloqueo de Order activa, de marca de reverso o de retención para revisión manual;
- estrategia de publicación, Outbox, Saga, CQRS o deduplicación técnica;
- OAuth2/OIDC, configuración o despliegue concreto de Cognito;
- protocolos o implementación interna del Payment Mock, salvo la regla funcional de cancelación;
- topología AWS, Terraform, networking o aislamiento de entornos;
- módulos, paquetes, clases, adapters, endpoints definitivos u OpenAPI.

Estas decisiones corresponden al Architect Agent y ya están documentadas en la arquitectura consolidada v2.
