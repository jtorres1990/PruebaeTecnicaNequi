---
artifact: feature-spec
schema_version: 1.0
feature: ticketing-event-processing
version: 3

agent:
  name: requirements-analyst
  version: 1.0

source:
  requirement: requirements/Prueba2026.md
  previous_specification:
    artifact: feature-spec/ticketing.feature-spec.md
    version: 2
  human_review:
    artifact: human-review/ticketing.functional-review.yaml
    status: APPROVED
    decisions_incorporated: 21

status: READY_FOR_ARCHITECTURE
human_validation_required: false
open_questions: 0
blocking_questions: 0
architecture_can_start: true

generated_at: 2026-10-01T18:44:16-05:00
---

# Feature Specification — Ticketing Event Processing

## 1. Purpose

Especificar el comportamiento funcional y no funcional de un backend reactivo de ticketing que gestione eventos, tickets individuales, disponibilidad, reservas temporales, órdenes y compras asíncronas, preservando la consistencia del inventario ante alta concurrencia.

Esta versión consolida las 21 decisiones aprobadas en la Human Functional Review. No define arquitectura de solución, modelo físico de datos, topología de infraestructura, clases, paquetes, endpoints definitivos ni políticas operativas concretas.

## 2. Problem statement

El sistema actual presenta compras duplicadas, una misma ubicación vendida a más de una persona y timeouts durante picos de demanda. La solución debe mantener respuestas síncronas breves, delegar el procesamiento posterior a mensajería asíncrona y garantizar que ninguna ejecución concurrente o repetida produzca sobreventa, pagos duplicados ni ventas duplicadas.

- Objetivo de negocio: permitir compras de tickets sin sobreventa y con resultado consultable.
- Objetivo funcional: administrar tickets individuales, reservas con vencimiento, órdenes all-or-nothing y confirmación mediante un Payment Mock.
- Objetivo técnico explícito: backend reactivo y no bloqueante, persistencia NoSQL en DynamoDB y procesamiento asíncrono mediante Amazon SQS Standard.
- Resultado esperado: inventario consistente, operaciones idempotentes, autorización por roles y trazabilidad de estados.

## 3. Scope

### 3.1 In scope

- Creación y consulta de eventos.
- Tickets o ubicaciones individuales e inventario por evento.
- Consulta bajo demanda de disponibilidad.
- Reservas temporales de máximo diez minutos.
- Órdenes de compra all-or-nothing.
- Procesamiento asíncrono mediante Amazon SQS Standard.
- Payment Mock con resultados exitosos y fallidos deterministas.
- Control de concurrencia y prevención de sobreventa.
- Liberación de reservas expiradas.
- Idempotencia de solicitudes, mensajes, órdenes y pagos.
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
- Diseño físico de DynamoDB, parámetros operativos de SQS, topología AWS o estructura interna del Payment Mock.

Fuentes de consolidación: requerimiento original; `HV-007`, `HV-008`, `HV-013`, `HV-014`.

## 4. Actors

| Actor | Type | Responsibility | Authorization | Source |
|---|---|---|---|---|
| `ADMIN` | Human | Crear eventos y definir tickets `COMPLIMENTARY` durante la creación. | Grupo Cognito `ADMIN`. | `HV-005`, `HV-014` |
| `CUSTOMER` | Human | Consultar eventos y disponibilidad, iniciar compras y consultar exclusivamente sus propias Orders. | Grupo Cognito `CUSTOMER`. | `HV-014`, `HV-021` |
| Consumidor asíncrono de Orders | Autonomous process | Consumir mensajes, coordinar el Payment Mock y llevar Order y tickets a un resultado consistente. | Identidad técnica; detalle en arquitectura. | Requisito Funcional 3; `HV-003`, `HV-004`, `HV-008` |
| Proceso de expiración | Autonomous process | Identificar reservas vencidas y liberar conjuntamente sus tickets. | Identidad técnica; detalle en arquitectura. | Requisitos Funcionales 2 y 6; `HV-012` |
| Payment Mock | External system | Simular de forma determinista aprobación, rechazo o fallo de un pago. | Integración protegida; detalle en arquitectura. | `HV-008` |
| Amazon Cognito | External identity provider | Autenticar usuarios y emitir JWT con `cognito:groups`. | Proveedor de identidad. | `HV-014` |

El candidato, evaluador, entrevistador y desarrollador no son actores funcionales.

## 5. Domain model

| Concept | Consolidated definition | Cardinality / invariant | Source |
|---|---|---|---|
| `Event` | Evento con nombre, fecha y hora, lugar, capacidad total y tickets individuales. | `Event 1 -> N Ticket`; `Event 1 -> N Order`. | Requisito Funcional 1; `HV-006`, `HV-016`, `HV-017` |
| `Ticket` | Ubicación individual, numerada e identificable de forma única dentro de su Event; tiene exactamente un estado. | Pertenece a un solo Event y como máximo a una Reservation activa. | `HV-001`, `HV-006`, `HV-017` |
| `Inventory` | Vista de los estados de los tickets de un Event. La disponibilidad comercial se deriva de tickets `AVAILABLE`. | No es solo un contador independiente; debe ser coherente con los tickets. | Requisito Funcional 1; `HV-006` |
| `Order` | Solicitud de compra de uno o más tickets individuales de un único Event. | `Order 1 -> N Ticket requested`; resultado all-or-nothing. | `HV-001`, `HV-017` |
| `Reservation` | Retención temporal y atómica de exactamente los tickets solicitados por una Order. | `Order 1 -> 0..1 Reservation`; una Order tiene como máximo una activa. | `HV-012`, `HV-017` |
| `PaymentAttempt` | Intento identificable de pago asociado a una Order. | Máximo un intento activo simultáneamente; un reintento autorizado es una operación diferente. | `HV-008`, `HV-011` |
| `Message` | Representación de trabajo asíncrono de una Order entregada al menos una vez. | Una entrega repetida no produce efectos adicionales. | `HV-010`, `HV-011` |
| `Customer identity` | Identidad autenticada propietaria de la Order. | Se deriva del JWT; no de un identificador libre enviado por el cliente. | `HV-014`, `HV-021` |

### 5.1 Domain invariants

1. Un Event contiene múltiples Ticket individuales.
2. `Event.capacity = total de Ticket del Event`.
3. Una Order pertenece exactamente a un Event y solicita uno o más Ticket de ese mismo Event.
4. Una Order no puede contener tickets de diferentes eventos.
5. Una Order se procesa all-or-nothing; no existe reserva, pago ni venta parcial.
6. Una Order tiene como máximo una Reservation activa.
7. Una Reservation contiene exactamente los tickets solicitados por su Order.
8. Un Ticket solo puede pertenecer a una Reservation activa a la vez.
9. Ticket y Order poseen máquinas de estado separadas.
10. El estado terminal de una Order debe ser coherente con el estado conjunto de sus tickets.

## 6. Ticket state machine

### 6.1 Ticket states

| ID | State | Meaning | Terminal | Accounting effect | Inventory effect | Source |
|---|---|---|---|---|---|---|
| DS-001 | `AVAILABLE` | Ticket comercialmente disponible para ser reservado. | No | No es venta. | Cuenta como disponibilidad comercial. | Requerimiento; `HV-001`, `HV-006` |
| DS-002 | `RESERVED` | Ticket retenido por una Reservation activa antes de iniciar o mientras se prepara el pago. | No | No es venta. | No está disponible para otras Orders. | Requerimiento; `HV-001`, `HV-002`, `HV-012` |
| DS-003 | `PENDING_CONFIRMATION` | Ticket reservado cuyo pago está en curso o pendiente de respuesta definitiva. | No | No es venta. | Continúa bloqueado para otras Orders. | Requerimiento; `HV-004` |
| DS-004 | `SOLD` | Ticket de una Order cuyo pago terminó exitosamente. | Sí, irreversible | Venta confirmada. | No está disponible. | Requerimiento; `HV-003`, `HV-008` |
| DS-005 | `COMPLIMENTARY` | Ticket de cortesía creado en ese estado durante la creación del Event. | Sí | No es venta y no es contable como venta. | Consume capacidad, pero nunca es disponibilidad comercial. | Requerimiento; `HV-005`, `HV-013` |

### 6.2 Ticket transitions

| ID | From | To | Trigger | Constraints | Source |
|---|---|---|---|---|---|
| ST-001 | `AVAILABLE` | `RESERVED` | Inicio formal de compra y creación exitosa de la Reservation. | Todos los tickets solicitados deben transicionar atómicamente; si uno no está disponible, ninguno cambia. | Requisito Funcional 2; `HV-002`, `HV-012`, `HV-017` |
| ST-002 | `RESERVED` o `PENDING_CONFIRMATION` | `AVAILABLE` | Expiración a los diez minutos sin confirmación. | Liberación conjunta de todos los tickets de la Reservation; Order pasa a `EXPIRED`. | Requisitos Funcionales 2 y 6; `HV-003`, `HV-004`, `HV-012` |
| ST-003 | `RESERVED` | `PENDING_CONFIRMATION` | Inicio del procesamiento de pago. | Todos los tickets de la Order cambian conjuntamente; no representa venta. | `HV-004`, `HV-008` |
| ST-004 | `PENDING_CONFIRMATION` | `SOLD` | Payment Mock confirma pago exitoso. | Todos los tickets cambian conjuntamente y la Order pasa a `CONFIRMED`. | `HV-003`, `HV-004`, `HV-008` |
| ST-005 | `RESERVED` o `PENDING_CONFIRMATION` | `AVAILABLE` | Rechazo definitivo del pago, fallo técnico definitivo o fallo definitivo de encolado, según corresponda. | Ningún ticket queda vendido; liberación conjunta e idempotente. | `HV-003`, `HV-004`, `HV-009`, `HV-018` |

`COMPLIMENTARY` se asigna al crear el Ticket dentro de la creación del Event; no es una transición posterior desde otro estado. `SOLD` y `COMPLIMENTARY` no tienen transiciones de salida.

## 7. Order state machine

### 7.1 Order states

| ID | State | Meaning | Terminal | Ticket consistency | Source |
|---|---|---|---|---|---|
| DS-006 | `CREATED` | Order persistida después de crear atómicamente su Reservation; puede estar encolándose o procesándose. | No | Sus tickets están `RESERVED` o `PENDING_CONFIRMATION`. | `HV-018` |
| DS-007 | `CONFIRMED` | Pago exitoso y compra completa confirmada. | Sí | Todos sus tickets están `SOLD`. | `HV-009` |
| DS-008 | `REJECTED` | Resultado funcional negativo, incluido ticket no disponible al iniciar la compra o rechazo definitivo de pago cuando existe Order persistida. | Sí | Ningún ticket queda `SOLD`; los reservados se liberan. | `HV-009` |
| DS-009 | `FAILED` | Fallo técnico definitivo, incluido fallo definitivo de encolado. | Sí | Ningún ticket queda parcialmente vendido; los recursos reservados quedan liberados o reconciliados a un estado consistente. | `HV-009`, `HV-018` |
| DS-010 | `EXPIRED` | La Reservation alcanzó diez minutos sin confirmación. | Sí | Todos los tickets reservados regresan a `AVAILABLE`. | `HV-009`, `HV-012` |

`CONFIRMED`, `REJECTED`, `FAILED` y `EXPIRED` son terminales para el procesamiento funcional de la Order. No se introducen estados intermedios por conveniencia técnica.

### 7.2 Order transitions

| ID | From | To | Trigger | Constraints | Source |
|---|---|---|---|---|---|
| ST-006 | creación consistente | `CREATED` | Reservation completa creada y Order persistida. | Debe existir una única Reservation activa con todos los tickets solicitados. | `HV-002`, `HV-012`, `HV-018` |
| ST-007 | `CREATED` | `CONFIRMED` | Pago exitoso. | Coincide atómicamente con todos los tickets en `SOLD`. | `HV-003`, `HV-009` |
| ST-008 | `CREATED` | `REJECTED` | Rechazo funcional definitivo del procesamiento o pago. | Ningún ticket vendido; liberar en conjunto lo reservado. | `HV-009` |
| ST-009 | `CREATED` | `FAILED` | Fallo técnico definitivo o fallo definitivo de encolado. | Cancelar Reservation y liberar tickets cuando corresponda; no dejar reserva indefinida. | `HV-009`, `HV-018` |
| ST-010 | `CREATED` | `EXPIRED` | Vence la Reservation sin compra confirmada. | Liberación conjunta e idempotente. | `HV-009`, `HV-012` |

## 8. Functional capabilities

| ID | Capability | Consolidated scope |
|---|---|---|
| CAP-001 | Gestión de eventos | `ADMIN` crea eventos válidos y los usuarios autorizados consultan eventos futuros habilitados. |
| CAP-002 | Consulta de disponibilidad | Lectura bajo demanda de los estados actuales de tickets; no streaming. |
| CAP-003 | Reserva temporal | Reserva atómica de todos los tickets solicitados por máximo diez minutos. |
| CAP-004 | Recepción de compras | Inicio formal de compra, creación consistente de Order/Reservation, encolado y retorno de Order ID. |
| CAP-005 | Procesamiento asíncrono | Consumo idempotente, invocación del Payment Mock y cierre coherente de Order y tickets. |
| CAP-006 | Consulta de Order | Consulta de cualquier resultado de una Order propia sin revelar Orders de terceros. |
| CAP-007 | Expiración de reservas | Identificación y liberación conjunta de reservas vencidas. |
| CAP-008 | Concurrencia, atomicidad e idempotencia | Cero sobreventa y cero efectos duplicados bajo contención o entrega at-least-once. |

## 9. Main flows

### MF-001 — Crear Event

1. Un `ADMIN` autenticado suministra nombre, lugar, fecha y hora futuras, capacidad positiva y el conjunto de tickets individuales.
2. Puede marcar ubicaciones como `COMPLIMENTARY` exclusivamente durante esta creación.
3. El sistema valida que nombre y lugar no estén vacíos, que la fecha/hora sea futura en UTC y que la capacidad coincida con la cantidad de tickets.
4. El sistema crea el Event con tickets `AVAILABLE` y/o `COMPLIMENTARY`.
5. Después de creado no se pueden agregar cortesías ni convertir otros estados a `COMPLIMENTARY`.

### MF-002 — Consultar Events y disponibilidad

1. Un `CUSTOMER` consulta eventos disponibles.
2. El sistema incluye eventos futuros habilitados, incluso cuando su disponibilidad comercial es cero.
3. Los eventos pasados no aparecen en la consulta normal.
4. Al consultar un Event, el sistema devuelve la disponibilidad vigente bajo demanda.
5. La información es una fotografía informativa y no garantiza adquisición.

### MF-003 — Iniciar y confirmar compra

1. El `CUSTOMER` consulta un Event y su disponibilidad vigente.
2. Selecciona uno o más tickets individuales del mismo Event.
3. La selección visual no crea Reservation ni inicia temporizador.
4. El `CUSTOMER` inicia formalmente la compra.
5. El sistema valida otra vez todos los tickets solicitados.
6. El sistema intenta reservarlos de forma atómica.
7. Si cualquiera no está disponible, rechaza la operación completa sin Reservation parcial.
8. Si todos están disponibles, crea una Reservation con exactamente esos tickets.
9. En ese instante comienza el período máximo de diez minutos.
10. El sistema crea la Order en `CREATED`; sus tickets quedan `RESERVED`.
11. El sistema intenta encolar la solicitud en Amazon SQS Standard.
12. Al quedar un resultado persistido y consultable, retorna el identificador de Order sin esperar el pago.
13. El consumidor procesa el mensaje de forma idempotente e invoca el Payment Mock.
14. Al iniciar el pago, todos los tickets pasan a `PENDING_CONFIRMATION`.
15. Si el pago es exitoso, todos pasan a `SOLD` y la Order a `CONFIRMED`.

No se permite cumplimiento parcial en ningún paso.

### MF-004 — Consultar Order

1. Un `CUSTOMER` autenticado solicita una Order por identificador.
2. El sistema deriva el propietario desde el JWT.
3. Si la Order existe y pertenece al `CUSTOMER`, retorna su estado actual y, si no terminó exitosamente, una causa funcional comprensible sin detalles técnicos internos.
4. Si no existe o pertenece a otro `CUSTOMER`, retorna funcionalmente “recurso no encontrado”.
5. `CONFIRMED`, `REJECTED`, `FAILED` y `EXPIRED` permanecen consultables.

## 10. Inputs and outputs

| Operation | Inputs | Outputs / observable result |
|---|---|---|
| Crear Event | Nombre, lugar, fecha/hora, capacidad, tickets individuales y marcas de cortesía; identidad `ADMIN`. | Event creado con inventario consistente o rechazo de validación/autorización. |
| Consultar Events | Identidad autenticada y criterios de consulta que defina el contrato posterior. | Events futuros habilitados; los agotados se muestran como tales. |
| Consultar disponibilidad | Event identificado. | Estado vigente bajo demanda de tickets; resultado informativo. |
| Iniciar compra | Uno o más Ticket IDs del mismo Event; identidad `CUSTOMER`; identidad inequívoca de operación repetible. | Order ID y estado persistido consultable, o rechazo completo sin reserva parcial. |
| Procesar pago | Order y operación de pago identificables. | Aprobación, rechazo o fallo determinista del mock; efectos idempotentes. |
| Consultar Order | Order ID e identidad autenticada. | Estado y causa funcional autorizada, o recurso no encontrado. |

## 11. Functional requirements

### FR-001 — Crear Events

El sistema debe permitir que un `ADMIN` cree Events con nombre, fecha/hora, lugar, capacidad total y tickets individuales. Debe poder definir tickets `COMPLIMENTARY` solamente durante esa creación. Fuente: Requisito Funcional 1; `HV-005`, `HV-014`, `HV-016`. Status: DEFINED.

### FR-002 — Consultar Events disponibles

El sistema debe permitir consultar Events futuros habilitados para consulta y venta. Un Event agotado continúa visible; uno pasado no aparece en la consulta normal. Fuente: Objetivo; Requisito Funcional 1; `HV-020`. Status: DEFINED.

### FR-003 — Mantener inventario por Event

Cada Event debe mantener tickets individuales y disponibilidad coherente con sus estados. La capacidad total debe coincidir con el total de tickets. Fuente: Requisito Funcional 1; `HV-005`, `HV-006`, `HV-016`. Status: DEFINED.

### FR-004 — Reservar tickets temporal y atómicamente

Al iniciar formalmente una compra, el sistema debe revalidar y reservar todos los tickets solicitados durante máximo diez minutos; si uno no está disponible, no debe crear una Reservation parcial. Fuente: Requisito Funcional 2; `HV-002`, `HV-012`, `HV-017`. Status: DEFINED.

### FR-005 — Encolar solicitudes de compra

Después de crear consistentemente la Reservation y la Order, el sistema debe intentar encolar la solicitud en Amazon SQS Standard para procesamiento asíncrono. Fuente: Requisito Funcional 3; `HV-002`, `HV-010`, `HV-018`. Status: DEFINED.

### FR-006 — Retornar identificador de Order

El sistema debe retornar un Order ID sin esperar el procesamiento de pago, solo cuando exista una Order persistida y consultable que refleje si la solicitud continúa o terminó por un fallo de aceptación. Fuente: Requisito Funcional 3; `HV-018`. Status: DEFINED.

### FR-007 — Procesar Orders asíncronamente

Un consumidor debe procesar las Orders encoladas de forma asíncrona y segura ante entregas repetidas. Fuente: Objetivo; Requisito Funcional 3; `HV-010`, `HV-011`. Status: DEFINED.

### FR-008 — Actualizar Order e inventario consistentemente

El procesamiento debe mantener la coherencia entre el estado de Order, Reservation y todos sus tickets, sin resultados parciales. Fuente: Requisito Funcional 3; `HV-001`, `HV-003`, `HV-004`, `HV-009`. Status: DEFINED.

### FR-009 — Consultar Order propia

El sistema debe permitir a un `CUSTOMER` consultar en cualquier momento una Order propia, incluida una terminal. Una Order inexistente o ajena debe observarse como recurso no encontrado. Fuente: Requisito Funcional 4; `HV-001`, `HV-009`, `HV-014`, `HV-021`. Status: DEFINED.

### FR-010 — Prevenir sobreventa bajo concurrencia

El sistema debe garantizar que solicitudes concurrentes por un mismo Ticket solo permitan una Reservation ganadora y nunca produzcan sobreventa. Fuente: Contexto; Requisito Funcional 5; `HV-006`. Status: DEFINED.

### FR-011 — Liberar Reservations expiradas

Un proceso debe identificar Reservations que cumplieron diez minutos sin confirmación, llevar su Order a `EXPIRED` y devolver conjuntamente todos sus tickets a `AVAILABLE`. Fuente: Requisitos Funcionales 2 y 6; `HV-003`, `HV-004`, `HV-009`, `HV-012`. Status: DEFINED.

### FR-012 — Consultar disponibilidad bajo demanda

El sistema debe exponer una consulta reactiva que obtenga la disponibilidad vigente de tickets. La respuesta no garantiza adquisición y debe revalidarse al iniciar compra. Fuente: Objetivo; Requisito Funcional 7; `HV-005`, `HV-006`, `HV-007`. Status: DEFINED.

### FR-013 — Mantener transiciones atómicas

Las transiciones de una Order y del conjunto de sus tickets deben completarse atómicamente desde la perspectiva funcional; no deben exponerse ventas parciales. Fuente: Notas generales; `HV-001`, `HV-017`. Status: DEFINED.

### FR-014 — Mantener transiciones auditables

El sistema debe conservar evidencia auditable de las transiciones de Ticket y Order. El contenido y mecanismo concreto se definen en arquitectura. Fuente: Notas generales. Status: DEFINED cualitativamente.

### FR-015 — Confirmar compra mediante Payment Mock

El sistema debe invocar un Payment Mock después de reservar todos los tickets y confirmar la compra únicamente ante un resultado exitoso. Fuente: `HV-003`, `HV-008`. Status: DEFINED.

### FR-016 — Rechazar y liberar all-or-nothing

Ante ticket no disponible, pago rechazado, expiración o fallo definitivo, ningún ticket puede quedar vendido parcialmente y los recursos reservados deben liberarse conjuntamente cuando corresponda. Fuente: `HV-001`, `HV-002`, `HV-003`, `HV-009`, `HV-012`, `HV-017`, `HV-018`. Status: DEFINED.

### FR-017 — Procesar operaciones idempotentemente

Solicitudes repetidas, mensajes duplicados, procesamiento repetido, invocaciones repetidas de pago y Orders terminales no deben crear reservas, pagos, ventas, cambios de inventario ni transiciones adicionales. Fuente: `HV-010`, `HV-011`. Status: DEFINED.

### FR-018 — Autenticar mediante Cognito JWT

Las operaciones protegidas deben requerir JWT emitido por Amazon Cognito; el backend actúa como Resource Server y obtiene roles desde `cognito:groups`. Fuente: `HV-014`. Status: DEFINED.

### FR-019 — Autorizar por rol y propiedad

`ADMIN` puede crear Events y definir cortesías; `CUSTOMER` puede consultar, comprar y consultar solo sus Orders. La propiedad se deriva de la identidad autenticada. Fuente: `HV-005`, `HV-014`, `HV-021`. Status: DEFINED.

## 12. Business rules

| ID | Rule | Source |
|---|---|---|
| BR-001 | Cada Ticket tiene exactamente un estado. | Notas generales; `HV-001` |
| BR-002 | Una Reservation dura como máximo diez minutos desde su creación completa y exitosa. | Requisito Funcional 2; `HV-012` |
| BR-003 | Solo un pago exitoso confirma la compra; si no se confirma antes del vencimiento, todos los tickets se liberan. | Requisitos Funcionales 2 y 6; `HV-003`, `HV-008` |
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
| BR-014 | Una Order pertenece a un único Event y solicita uno o más tickets de ese Event. | `HV-017` |
| BR-015 | Una Order tiene como máximo una Reservation activa, que contiene exactamente sus tickets solicitados. | `HV-012`, `HV-017` |
| BR-016 | `COMPLIMENTARY` solo se define por `ADMIN` durante la creación del Event; no se agrega ni se alcanza por transición posterior. | `HV-005`, `HV-014` |
| BR-017 | La selección visual no reserva ni inicia el temporizador. | `HV-002`, `HV-012` |
| BR-018 | La disponibilidad mostrada es informativa y debe revalidarse al iniciar formalmente la compra. | `HV-007` |
| BR-019 | Una repetición de la misma operación conserva o retorna el resultado previo sin repetir efectos. | `HV-011` |
| BR-020 | Una Order no puede tener más de un intento de pago activo simultáneamente. | `HV-011` |
| BR-021 | `Event.capacity` debe ser igual al número total de tickets, incluidos los `COMPLIMENTARY`. | `HV-005`, `HV-016` |
| BR-022 | Un Event futuro habilitado sigue visible con disponibilidad comercial cero; un Event pasado no aparece en la consulta normal. | `HV-020` |
| BR-023 | Para un `CUSTOMER`, una Order inexistente y una Order ajena son funcionalmente indistinguibles. | `HV-014`, `HV-021` |

## 13. Validations

| ID | Validation | Source |
|---|---|---|
| VAL-001 | La Reservation no puede permanecer vigente por más de diez minutos sin confirmación. | Requisito Funcional 2; `HV-012` |
| VAL-002 | Todos los tickets se revalidan inmediatamente antes de crear la Reservation. | Requisito Funcional 3; `HV-002`, `HV-007` |
| VAL-003 | Solo una Reservation vencida y no confirmada es elegible para liberación por expiración. | Requisitos Funcionales 2 y 6; `HV-012` |
| VAL-004 | Ninguna operación concurrente puede vender por encima de la disponibilidad o reservar dos veces el mismo Ticket. | Contexto; Requisito Funcional 5; `HV-006` |
| VAL-005 | Un Ticket no puede quedar simultáneamente en más de un estado. | Notas generales |
| VAL-006 | Nombre y lugar del Event son obligatorios y no vacíos. | `HV-016` |
| VAL-007 | Fecha y hora del Event son obligatorias, se interpretan respecto de UTC y deben ser futuras al crear. | `HV-016` |
| VAL-008 | La capacidad es un entero mayor que cero. | `HV-016` |
| VAL-009 | La capacidad debe coincidir con la cantidad total de tickets creados, incluidas cortesías. | `HV-005`, `HV-016` |
| VAL-010 | Todos los tickets de una Order deben existir, pertenecer al mismo Event y estar `AVAILABLE` al crear la Reservation. | `HV-002`, `HV-017` |
| VAL-011 | La identidad propietaria de una Order se toma del JWT y debe coincidir al consultar. | `HV-014`, `HV-021` |

La ubicación técnica de estas validaciones pertenece a arquitectura.

## 14. Alternative flows

### ALT-001 — Reservation expirada

Al cumplirse diez minutos desde la creación exitosa sin pago confirmado, la Reservation expira, todos los tickets regresan conjuntamente a `AVAILABLE` y la Order pasa a `EXPIRED`. Fuente: Requisitos Funcionales 2 y 6; `HV-003`, `HV-012`.

### ALT-002 — Ticket no disponible

Si al iniciar formalmente la compra cualquier ticket ya no está `AVAILABLE`, no se crea Reservation parcial y la operación completa se rechaza. Si existe una Order persistida para reflejar el resultado, termina en `REJECTED`. Fuente: `HV-002`, `HV-009`.

### ALT-003 — Mensaje u operación duplicada

El sistema reconoce la misma operación y conserva su resultado sin repetir Reservation, PaymentAttempt, venta, inventario ni transición terminal. Fuente: `HV-010`, `HV-011`.

### ALT-004 — Fallo transitorio recuperable

Puede reintentarse cuando corresponda, sin crear un segundo intento de pago activo ni repetir efectos. La clasificación, límites y backoff concretos pertenecen a arquitectura. Fuente: Especificaciones Técnicas; `HV-011`.

### ALT-005 — Pago rechazado

El Payment Mock retorna rechazo definitivo; la Order termina en `REJECTED`, ningún ticket queda `SOLD` y todos los tickets reservados se liberan conjuntamente. Fuente: `HV-003`, `HV-008`, `HV-009`.

### ALT-006 — Fallo definitivo de encolado

La Order termina en `FAILED`, se cancela la Reservation y todos los tickets vuelven conjuntamente a `AVAILABLE`; no queda una reserva indefinida. Fuente: `HV-018`.

### ALT-007 — Consulta no autorizada

Si un `CUSTOMER` solicita una Order de otro usuario, el resultado funcional es recurso no encontrado, igual que para un identificador inexistente. Fuente: `HV-014`, `HV-021`.

## 15. Error scenarios

| ID | Scenario | Required outcome | Source |
|---|---|---|---|
| ERR-001 | Solicitudes concurrentes compiten por el mismo Ticket. | Solo una puede reservarlo; cero sobreventa. | Requisito Funcional 5; `HV-006` |
| ERR-002 | Algún Ticket no está disponible al reservar. | Rechazo completo; ninguna Reservation parcial. | `HV-002`, `HV-009` |
| ERR-003 | Reservation supera diez minutos. | Order `EXPIRED`; liberación conjunta a `AVAILABLE`. | Requisitos Funcionales 2 y 6; `HV-012` |
| ERR-004 | Mensaje entregado más de una vez. | Reprocesamiento idempotente sin nuevos efectos. | `HV-010`, `HV-011` |
| ERR-005 | Error transitorio susceptible de retry. | Reintento seguro sin pago o transición duplicados; política concreta pendiente de arquitectura. | Especificaciones Técnicas; `HV-011` |
| ERR-006 | Order no existe. | Recurso no encontrado. | `HV-021` |
| ERR-007 | Encolado falla definitivamente. | Order `FAILED`; cancelar Reservation y liberar tickets. | `HV-018` |
| ERR-008 | Pago rechazado o falla definitivamente. | Order `REJECTED` por rechazo funcional o `FAILED` por fallo técnico; ningún Ticket vendido parcialmente y liberación consistente. | `HV-003`, `HV-008`, `HV-009` |
| ERR-009 | `CUSTOMER` consulta Order ajena. | Recurso no encontrado sin revelar existencia. | `HV-014`, `HV-021` |

## 16. Acceptance criteria

### AC-001 — Crear Event válido

Given un `ADMIN` autenticado con nombre y lugar no vacíos, instante futuro en UTC, capacidad positiva y exactamente ese número de tickets
When solicita crear el Event
Then el Event se crea con inventario consistente y las cortesías indicadas nacen en `COMPLIMENTARY`.

### AC-002 — Rechazar Event inválido

Given datos con fecha no futura, capacidad no positiva o capacidad diferente de la cantidad de tickets
When un `ADMIN` intenta crear el Event
Then el sistema rechaza la creación completa.

### AC-003 — Consultar Events disponibles

Given Events futuros habilitados con y sin tickets comerciales y un Event pasado
When se realiza la consulta normal
Then devuelve ambos Events futuros, marca el agotado con disponibilidad cero y excluye el pasado.

### AC-004 — Consultar disponibilidad bajo demanda

Given un Event con tickets en distintos estados
When un `CUSTOMER` consulta disponibilidad
Then solo los tickets `AVAILABLE` se informan como comercialmente disponibles y la respuesta no garantiza adquisición.

### AC-005 — La selección no reserva

Given un `CUSTOMER` que selecciona tickets sin iniciar formalmente la compra
When transcurre tiempo o otro usuario consulta
Then no existe Reservation, los tickets no cambian a `RESERVED` y el temporizador no comienza.

### AC-006 — Crear Reservation y Order

Given todos los tickets solicitados están `AVAILABLE` y pertenecen al mismo Event
When el `CUSTOMER` inicia formalmente la compra
Then se reservan todos atómicamente, se crea una Reservation exacta, comienza el plazo de diez minutos y se crea la Order `CREATED`.

### AC-007 — Rechazo all-or-nothing por indisponibilidad

Given al menos uno de varios tickets solicitados ya no está `AVAILABLE`
When se intenta crear la Reservation
Then se rechaza la operación completa y ninguno de los demás tickets queda reservado.

### AC-008 — Retornar Order ID sin esperar pago

Given Order y Reservation fueron creadas consistentemente y existe un estado persistido consultable
When la solicitud es aceptada para continuar por el flujo asíncrono
Then se retorna el Order ID sin esperar la respuesta del Payment Mock.

### AC-009 — Iniciar pago

Given una Order `CREATED` con todos sus tickets `RESERVED`
When el consumidor inicia el pago
Then todos los tickets pasan conjuntamente a `PENDING_CONFIRMATION` y aún no cuentan como venta.

### AC-010 — Confirmar compra completa

Given el Payment Mock aprueba el pago dentro de la vigencia de la Reservation
When se procesa el resultado
Then todos los tickets pasan conjuntamente a `SOLD` y la Order a `CONFIRMED` una sola vez.

### AC-011 — Rechazar pago

Given el Payment Mock rechaza definitivamente el pago
When se procesa el resultado
Then la Order pasa a `REJECTED`, ningún ticket queda vendido y todos regresan conjuntamente a `AVAILABLE`.

### AC-012 — Expirar compra

Given pasaron diez minutos desde la creación completa de la Reservation sin confirmación
When el proceso de expiración actúa
Then la Order pasa a `EXPIRED` y todos sus tickets regresan conjuntamente a `AVAILABLE`.

### AC-013 — Fallar técnicamente sin parcialidad

Given ocurre un fallo técnico definitivo durante el procesamiento
When se cierra la Order
Then queda `FAILED`, no existe venta parcial y los recursos reservados quedan en estado consistente.

### AC-014 — Fallo definitivo de encolado

Given una Order y Reservation creadas cuyo mensaje no puede encolarse definitivamente
When se determina el fallo
Then la Order queda `FAILED`, la Reservation se cancela y todos los tickets regresan a `AVAILABLE`.

### AC-015 — Evitar sobreventa concurrente

Given dos o más solicitudes concurrentes incluyen el mismo Ticket `AVAILABLE`
When intentan reservarlo
Then como máximo una crea una Reservation que lo contiene y no se registra sobreventa.

### AC-016 — Reprocesar mensaje idempotentemente

Given el mismo mensaje de una Order se entrega varias veces
When los consumidores lo procesan
Then no se crea otra Reservation, no se inicia pago duplicado, no se vende dos veces, no se modifica otra vez el inventario y no se repite una transición terminal.

### AC-017 — Limitar intento de pago activo

Given una Order tiene un PaymentAttempt activo
When llega una repetición de la misma operación
Then no se inicia un segundo intento activo y se conserva el procesamiento existente.

### AC-018 — Reprocesar Order terminal

Given una Order está `CONFIRMED`, `REJECTED`, `FAILED` o `EXPIRED`
When recibe nuevamente una operación ya procesada
Then conserva su estado y no ejecuta nuevos efectos de negocio.

### AC-019 — Mantener `SOLD` irreversible

Given un Ticket está `SOLD`
When se intenta otra transición
Then el Ticket permanece `SOLD`.

### AC-020 — Mantener `COMPLIMENTARY` final

Given un Ticket fue creado `COMPLIMENTARY`
When se intenta venderlo, reservarlo o cambiarlo
Then permanece `COMPLIMENTARY`, consume capacidad y no cuenta como venta.

### AC-021 — Impedir cortesías posteriores

Given un Event ya fue creado
When un actor intenta agregar una cortesía o convertir otro Ticket a `COMPLIMENTARY`
Then la operación se rechaza.

### AC-022 — Consultar Order propia terminal

Given una Order propia existente en cualquier estado, incluido terminal
When su `CUSTOMER` la consulta
Then obtiene el estado actual y una causa funcional cuando no fue confirmada.

### AC-023 — No revelar Order ajena

Given un `CUSTOMER` consulta un identificador inexistente o perteneciente a otro usuario
When el sistema autoriza la consulta
Then en ambos casos responde funcionalmente recurso no encontrado.

### AC-024 — Autorizar roles

Given JWT válidos con grupos `ADMIN` y `CUSTOMER`
When se solicitan operaciones protegidas
Then solo `ADMIN` crea Events y define cortesías, mientras `CUSTOMER` consulta, compra y consulta sus propias Orders.

### AC-025 — Auditar transiciones

Given una transición de Ticket u Order completada
When se consulta su evidencia de auditoría
Then existe evidencia de la transición sin exigir en esta especificación un mecanismo técnico concreto.

### AC-026 — Cumplir objetivo de carga

Given una prueba representativa de esta implementación con al menos 1.000 usuarios concurrentes y aproximadamente 200 solicitudes sostenidas por segundo
When se ejecuta la carga objetivo
Then no ocurren sobreventas ni ventas duplicadas.

### AC-027 — Latencia de disponibilidad

Given la prueba de carga objetivo de esta implementación
When se miden consultas de disponibilidad
Then su latencia observada cumple `p95 < 500 ms`.

### AC-028 — Latencia de inicio de compra

Given la prueba de carga objetivo de esta implementación
When se mide la operación síncrona de inicio de compra y reserva
Then su latencia observada cumple `p95 < 1 s`, sin incluir pago ni confirmación asíncrona.

Los objetivos de AC-026 a AC-028 son objetivos verificables de la prueba, no capacidad garantizada para producción.

## 17. Concurrency, atomicity and consistency

- La unidad de competencia es el Ticket individual.
- Solo una solicitud concurrente puede obtener un mismo Ticket.
- La creación de Reservation es all-or-nothing sobre todos los tickets de la Order.
- Las transiciones conjuntas de tickets no pueden dejar subconjuntos vendidos, reservados o liberados.
- La coherencia observable debe preservar los invariantes entre Order, Reservation, Ticket e Inventory.
- Deben alcanzarse cero sobreventas y cero ventas duplicadas en las pruebas objetivo.
- El mecanismo concreto —optimistic locking o conditional writes, conforme a `TC-011`— lo define arquitectura.

Fuentes: FR-010; BR-008, BR-009, BR-013; `HV-001`, `HV-002`, `HV-006`, `HV-017`.

## 18. Idempotency requirements

La idempotencia es obligatoria para:

- solicitudes repetidas de inicio de compra;
- procesamiento repetido de la misma Order;
- mensajes duplicados de SQS Standard;
- invocaciones repetidas del flujo de pago;
- procesamiento de Orders terminales.

La entrega at-least-once no puede provocar reservas, PaymentAttempts, pagos, ventas, cambios de inventario ni transiciones terminales duplicadas. Una Order admite como máximo un PaymentAttempt activo. Un reintento autorizado tras un fallo recuperable debe poseer identidad propia y seguir asociado a la misma Order.

Idempotency keys, conditional writes, registro de operaciones y mecanismos equivalentes son decisiones de arquitectura.

Fuente: `HV-010`, `HV-011`.

## 19. Asynchronous processing requirements

- La operación síncrona inicia formalmente la compra, revalida tickets y crea de forma consistente Reservation y Order.
- El trabajo posterior se envía a Amazon SQS Standard y se procesa de forma asíncrona.
- La respuesta síncrona no espera el pago; retorna un Order ID consultable.
- El consumidor invoca el Payment Mock y aplica el resultado a todos los tickets y a la Order.
- SQS tiene semántica at-least-once; el consumidor debe ser idempotente.
- Ante fallo definitivo de encolado, la Order queda `FAILED` y la Reservation se revierte.
- Visibility timeout, retries, DLQ, polling y estrategia de publicación pertenecen a arquitectura.

## 20. Security requirements

### 20.1 Authentication

- Amazon Cognito autentica usuarios y emite JWT.
- El backend actúa como Resource Server.
- Los roles se obtienen del claim `cognito:groups`.

### 20.2 Authorization

- `ADMIN`: crear Events y definir tickets `COMPLIMENTARY` durante la creación.
- `CUSTOMER`: consultar Events y disponibilidad, iniciar compras y consultar solo sus Orders.
- La identidad propietaria de Order se deriva del JWT, nunca de un user ID libre del cliente.
- Una consulta de Order ajena no revela la existencia del recurso.

OAuth2/OIDC, configuración de Cognito, mapeo técnico de autoridades y expiración de tokens pertenecen a arquitectura.

Fuente: `HV-014`, `HV-021`.

## 21. Non-functional requirements

| ID | Requirement | Basis | Status / target |
|---|---|---|---|
| NFR-001 | Alta concurrencia | explicit + `HV-015` | Objetivo de prueba: al menos 1.000 usuarios concurrentes y ~200 solicitudes sostenidas/s. |
| NFR-002 | Baja latencia bajo carga | explicit + `HV-015` | Disponibilidad `p95 < 500 ms`; inicio síncrono de compra/reserva `p95 < 1 s`. |
| NFR-003 | Procesamiento no bloqueante | explicit | API reactiva con Spring WebFlux; cualitativamente definido. |
| NFR-004 | Consistencia de inventario | explicit + Human Review | Cero sobreventas, cero ventas duplicadas y ausencia de resultados parciales. |
| NFR-005 | Auditabilidad | explicit | Todas las transiciones son auditables; formato posterior. |
| NFR-006 | Acceso rápido con NoSQL | explicit + `HV-019` | Amazon DynamoDB; métrica específica no definida. |
| NFR-007 | Escalabilidad | context + evaluation-driven | Consideración explícita; sin garantía productiva. |
| NFR-008 | Resiliencia | evaluation-driven | Debe considerarse; objetivos concretos posteriores. |
| NFR-009 | Tolerancia a fallos | evaluation-driven | Debe considerarse; objetivos concretos posteriores. |
| NFR-010 | Seguridad | Human Review + evaluation-driven | JWT Cognito, roles y aislamiento de Orders definidos; controles técnicos posteriores. |
| NFR-011 | Observabilidad | evaluation-driven | Criterio de evaluación; no se inventan métricas obligatorias. |
| NFR-012 | Mantenibilidad | explicit deliverable | Código en inglés, separación de capas, SOLID y patrones apropiados. |
| NFR-013 | Testabilidad y cobertura | explicit deliverable | Tests unitarios/reactivos/concurrencia y cobertura mínima de 90%. |
| NFR-014 | Costos y gobernanza | evaluation-driven | Criterio de evaluación cloud-native; no se convierte en comportamiento funcional. |
| NFR-015 | Idempotencia | `HV-010`, `HV-011` | Ninguna repetición produce efectos de negocio adicionales. |

Los umbrales de NFR-001 y NFR-002 son objetivos de prueba de esta implementación, no SLA ni garantía de capacidad productiva.

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

Estos elementos permanecen como criterios de evaluación. Solo seguridad e idempotencia adquirieron requisitos funcionales/no funcionales concretos por `HV-010`, `HV-011` y `HV-014`; Terraform, diagramas, despliegue AWS, costos, observabilidad y gobernanza no se transforman automáticamente en FR o entregables obligatorios.

## 25. Resolved ambiguities and Human Review decisions

No quedan ambigüedades funcionales pendientes. Los identificadores `HV-*` se preservan exclusivamente como trazabilidad de decisiones resueltas.

| HV | Decision | Consolidated outcome | Main propagated items |
|---|---|---|---|
| HV-001 | CONFIRMED_WITH_CHANGE | Estados originales pertenecen a Ticket; Order tiene máquina separada; procesamiento indivisible. | DS-001..DS-010, ST-001..ST-010, FR-008, FR-013, BR-013 |
| HV-002 | CONFIRMED_WITH_CHANGE | Selección visual no reserva; compra formal revalida y reserva antes de encolar. | MF-003, FR-004..FR-006, ST-001, AC-005..AC-008 |
| HV-003 | CONFIRMED_WITH_CHANGE | Solo pago exitoso confirma; fallo/rechazo/expiración libera todos. | ST-002, ST-004, ST-005, FR-011, FR-015, AC-010..AC-012 |
| HV-004 | CONFIRMED_WITH_CHANGE | `PENDING_CONFIRMATION` significa pago en curso; entrada/salida conjunta. | DS-003, ST-003..ST-005, BR-005, AC-009 |
| HV-005 | CONFIRMED_WITH_CHANGE | Cortesías solo al crear Event, por ADMIN; consumen capacidad y no son venta. | DS-005, FR-001, BR-007, BR-016, AC-020..AC-021 |
| HV-006 | CONFIRMED_WITH_CHANGE | Inventario de tickets individuales y disponibilidad derivada. | Domain model, FR-003, FR-010, BR-012, VAL-004 |
| HV-007 | CONFIRMED_WITH_CHANGE | Disponibilidad bajo demanda, informativa, sin streaming; revalidación al comprar. | CAP-002, MF-002, FR-012, BR-018, AC-004 |
| HV-008 | CONFIRMED_WITH_CHANGE | Payment Mock independiente; aprobación y fallos deterministas. | Actor Payment Mock, FR-015, TC-017, AC-009..AC-011 |
| HV-009 | CONFIRMED_WITH_CHANGE | Order usa `CREATED`, `CONFIRMED`, `REJECTED`, `FAILED`, `EXPIRED`. | DS-006..DS-010, ST-006..ST-010, FR-009, ERR-002/008 |
| HV-010 | CONFIRMED_WITH_CHANGE | Amazon SQS Standard obligatorio; LocalStack local; aplicación maneja duplicados. | TC-005, TC-010, TC-012, FR-005, FR-017 |
| HV-011 | CONFIRMED_WITH_CHANGE | Idempotencia de solicitudes, órdenes, mensajes y pagos; un pago activo máximo. | FR-017, BR-019/020, NFR-015, AC-016..AC-018 |
| HV-012 | CONFIRMED_WITH_CHANGE | Temporizador inicia al crear Reservation completa; contiene tickets exactos; sin liberación parcial. | FR-004, FR-011, BR-002/015/017, VAL-001/003 |
| HV-013 | CONFIRMED_WITH_CHANGE | Reportes fuera de alcance; semántica contable de estados preservada. | Scope, DS-004/005, BR-004..BR-007 |
| HV-014 | CONFIRMED_WITH_CHANGE | Cognito JWT, roles ADMIN/CUSTOMER y propiedad de Order desde identidad autenticada. | Actors, FR-018/019, NFR-010, TC-016, AC-023/024 |
| HV-015 | CONFIRMED_WITH_CHANGE | Objetivos de prueba: 1.000 usuarios, ~200 req/s, p95 definidos y cero duplicidad/sobreventa. | NFR-001/002/004, AC-026..AC-028 |
| HV-016 | CONFIRMED_WITH_CHANGE | Campos obligatorios, instante futuro UTC, capacidad positiva e igual al total de tickets. | FR-001, BR-021, VAL-006..VAL-009, AC-001/002 |
| HV-017 | CONFIRMED_WITH_CHANGE | Order de un Event, 1..N tickets, 0..1 Reservation activa y all-or-nothing. | Domain invariants, FR-016, BR-013..BR-015 |
| HV-018 | CONFIRMED_WITH_CHANGE | Order inicia `CREATED`; fallo definitivo de encolado produce `FAILED` y revierte Reservation. | DS-006/009, ST-006/009, FR-005/006, ALT-006, AC-014 |
| HV-019 | CONFIRMED_WITH_CHANGE | Amazon DynamoDB objetivo y DynamoDB Local para desarrollo/pruebas. | NFR-006, TC-004, TC-012, DEL-005 |
| HV-020 | CONFIRMED_WITH_CHANGE | Event futuro habilitado visible aunque agotado; Event pasado fuera de consulta normal. | MF-002, FR-002, BR-022, AC-003 |
| HV-021 | CONFIRMED_WITH_CHANGE | Order inexistente o ajena produce recurso no encontrado; terminales siguen consultables. | MF-004, FR-009/019, BR-023, ERR-006/009, AC-022/023 |

## 26. Human validation status

- Human Review status: `APPROVED`.
- Decisions incorporated: 21 of 21.
- Pending Human Validations: 0.
- Pending HIGH blocking items: 0.
- Contradictions among approved decisions: none detected.
- Human validation required: false.

## 27. Traceability matrix

| Original source | Existing / new IDs | Human decision | Consolidated behavior | Acceptance evidence |
|---|---|---|---|---|
| Context: duplicates, double sale, concurrency | FR-010, NFR-004, FR-017 | HV-006, HV-010, HV-011, HV-015 | Ticket-level concurrency, all-or-nothing and idempotency | AC-015..AC-018, AC-026 |
| Functional Requirement 1: Events and inventory | CAP-001, FR-001..FR-003 | HV-005, HV-006, HV-014, HV-016, HV-020 | ADMIN creates valid Events with individual tickets; future Events remain visible when sold out | AC-001..AC-004, AC-020..AC-021, AC-024 |
| Functional Requirement 2: temporary reservation | CAP-003, FR-004, BR-002/003 | HV-002, HV-003, HV-012, HV-017 | Atomic Reservation starts on formal purchase and expires in ten minutes | AC-005..AC-007, AC-012 |
| Functional Requirement 3: asynchronous purchase | CAP-004/005, FR-005..FR-008 | HV-001..HV-004, HV-008..HV-012, HV-018 | Reserve, create Order, enqueue, pay asynchronously and close consistently | AC-008..AC-018 |
| Functional Requirement 4: Order query | CAP-006, FR-009 | HV-001, HV-009, HV-014, HV-018, HV-021 | Separate Order states, own-resource authorization, terminal results consultable | AC-022..AC-024 |
| Functional Requirement 5: concurrency | CAP-008, FR-010/013, BR-008/009 | HV-001, HV-006, HV-015, HV-017 | No overbooking and no partial result | AC-007, AC-015, AC-026 |
| Functional Requirement 6: release expired reservations | CAP-007, FR-011, ST-002/ST-010 | HV-003, HV-004, HV-009, HV-012 | Order expires and all tickets return to AVAILABLE | AC-012 |
| Functional Requirement 7: real-time availability | CAP-002, FR-012, BR-012/018 | HV-005, HV-006, HV-007 | Current read on demand, no streaming, revalidation at purchase | AC-004, AC-007 |
| Notes: Ticket states | DS-001..DS-005, ST-001..ST-005 | HV-001, HV-003..HV-005, HV-013 | Coherent Ticket state machine and accounting semantics | AC-009..AC-012, AC-019..AC-021 |
| Stack and technical specifications | TC-001..TC-017 | HV-008, HV-010, HV-014, HV-019 | Required stack preserved without physical design | DEL-005; AC-016, AC-024 |
| Deliverables | DEL-001..DEL-005, NFR-012/013 | HV-010, HV-019 | Local dependencies fixed to DynamoDB Local and LocalStack/SQS | Delivery verification |
| Evaluation criteria | EVAL-001..EVAL-014 | HV-010, HV-011, HV-014, HV-015 | Criteria retained; only approved decisions become requirements | AC-016..AC-018, AC-024, AC-026..AC-028 |

## 28. Analysis trail

| Observation | Classification | Authority | Consolidated decision |
|---|---|---|---|
| El requerimiento mezclaba estados de Ticket y Order. | CONTRADICTION resolved | Human Review `HV-001`, `HV-009`, `HV-018` | Dos máquinas separadas y coherentes. |
| Reserva y encolado no tenían orden inequívoco. | AMBIGUITY resolved | `HV-002`, `HV-012`, `HV-018` | Revalidar y reservar atómicamente; crear Order; luego encolar. |
| No existía trigger de venta. | AMBIGUITY resolved | `HV-003`, `HV-008` | Pago exitoso del mock produce venta confirmada. |
| `PENDING_CONFIRMATION` no tenía semántica. | AMBIGUITY resolved | `HV-004` | Pago en curso; tickets bloqueados y no vendidos. |
| Cortesías no tenían actor ni efecto de inventario. | AMBIGUITY resolved | `HV-005`, `HV-014`, `HV-016` | ADMIN las define al crear; consumen capacidad; no son venta. |
| Inventario individual o agregado no estaba resuelto. | AMBIGUITY resolved | `HV-006`, `HV-017` | Tickets individuales; agregado derivable. |
| “Tiempo real” era ambiguo. | AMBIGUITY resolved | `HV-007` | Lectura bajo demanda, sin canal continuo. |
| At-least-once no tenía regla funcional de duplicados. | AMBIGUITY resolved | `HV-010`, `HV-011` | Idempotencia obligatoria en todos los efectos de negocio. |
| Seguridad era solo criterio de evaluación. | AMBIGUITY resolved | `HV-014`, `HV-021` | Cognito JWT, roles y aislamiento de Orders. |
| Carga y latencia no eran verificables. | AMBIGUITY resolved | `HV-015` | Objetivos de prueba cuantitativos, no SLA productivo. |
| Persistencia y cola admitían opciones contradictorias. | CONTRADICTION resolved | `HV-010`, `HV-019` | DynamoDB/DynamoDB Local y SQS Standard/LocalStack. |

## 29. Architecture readiness gate

| Condition | Result |
|---|---|
| No HIGH decision remains `PENDING` | PASS |
| No contradiction among approved decisions | PASS |
| All 21 approved decisions propagated | PASS |
| Critical flows have Acceptance Criteria | PASS |
| Ticket and Order state machines are coherent | PASS |
| Atomicity and all-or-nothing rules are explicit | PASS |
| Idempotency rules are explicit | PASS |
| Mandatory technical constraints are represented | PASS |
| No blocking functional ambiguity remains | PASS |

Result: `READY_FOR_ARCHITECTURE`.

## 30. Mandatory self-validation

| # | Check | Result |
|---|---|---|
| 1 | Las 21 decisiones `HV-*` fueron procesadas. | PASS |
| 2 | Ninguna respuesta humana aprobada fue ignorada. | PASS |
| 3 | Ninguna ambigüedad resuelta continúa pendiente. | PASS |
| 4 | No se introdujeron nuevas suposiciones funcionales. | PASS |
| 5 | No se inventaron decisiones de arquitectura. | PASS |
| 6 | Los IDs existentes se conservaron cuando mantuvieron su significado. | PASS |
| 7 | Los elementos nuevos usan los siguientes IDs libres y no reutilizan significados. | PASS |
| 8 | Las máquinas de Ticket y Order son separadas y coherentes. | PASS |
| 9 | Los Acceptance Criteria reflejan las decisiones humanas. | PASS |
| 10 | La trazabilidad conecta fuente, IDs, Human Review y aceptación. | PASS |
| 11 | Las decisiones de fuera de alcance están representadas. | PASS |
| 12 | Los estados de Ticket y Order no se mezclan. | PASS |
| 13 | No existe cumplimiento parcial de una Order. | PASS |
| 14 | `COMPLIMENTARY` es consistente en dominio, reglas, flujos y aceptación. | PASS |
| 15 | La disponibilidad es consistentemente bajo demanda, informativa y revalidada. | PASS |
| 16 | El modelo de autorización es consistente con `ADMIN` y `CUSTOMER`. | PASS |
| 17 | El gate de arquitectura fue recalculado. | PASS |

## 31. Architectural boundary

Esta especificación no selecciona ni define:

- modelo de partición, claves, índices, tablas o consistencia concreta de DynamoDB;
- operación transaccional concreta de DynamoDB;
- visibility timeout, retry count, backoff, DLQ o polling de SQS;
- estrategia de publicación, Outbox, Saga, CQRS o deduplicación técnica;
- OAuth2/OIDC, configuración o despliegue concreto de Cognito;
- protocolos o implementación interna del Payment Mock;
- topología AWS, Terraform, networking o aislamiento de entornos;
- módulos, paquetes, clases, adapters, endpoints definitivos u OpenAPI.

Estas decisiones corresponden al Architect Agent y a etapas posteriores.
