---
artifact: feature-spec
schema_version: 1.0
feature: ticketing-event-processing
version: 2

agent:
  name: requirements-analyst
  version: 1.0

source:
  artifact: requirements/Prueba2026.md

status: READY_FOR_HUMAN_REVIEW
human_validation_required: true
open_questions: 21
blocking_questions: 13

generated_at: 2026-10-01T12:20:34-05:00
---

# Feature Specification — Ticketing Event Processing

## 1. Purpose

Especificar, sin diseñar la arquitectura, el comportamiento requerido para un backend reactivo de ticketing que gestione eventos, disponibilidad, reservas temporales, compras asíncronas y consultas de orden, preservando la consistencia del inventario ante alta concurrencia.

Resultado esperado: una base funcional, no funcional y trazable que permita al Architect Agent proponer una solución sin ocultar las decisiones de dominio que todavía requieren validación humana.

## 2. Problem statement

El sistema actual no escala adecuadamente durante los lanzamientos de eventos populares. Miles de usuarios intentan comprar simultáneamente y se presentan compras duplicadas, asientos vendidos a más de una persona y timeouts. El negocio necesita mantener tiempos de respuesta bajos, procesar compras mediante mensajería y evitar que se vendan más entradas de las disponibles.

- Problema actual: baja escalabilidad y fallos de consistencia (compras duplicadas, asientos vendidos a múltiples personas, timeouts) durante picos de demanda.
- Objetivo de negocio: vender entradas sin sobreventa y mantener información vigente sobre su disponibilidad y estado.
- Objetivo técnico explícito: backend reactivo, no bloqueante, con persistencia NoSQL y procesamiento asíncrono mediante cola de mensajes.
- Resultado esperado: solicitudes concurrentes procesadas de forma eficiente, inventario consistente, reservas expirables y estados consultables.

## 3. Actors

Regla aplicada: solo se registran como actores quienes interactúan directa o indirectamente con el sistema objetivo durante su operación (usuario humano, sistema externo o proceso autónomo). Los participantes del proceso de contratación, evaluación, entrevista, desarrollo o presentación no son actores funcionales.

| Actor | Type | Responsibility | Definition status |
|-------|------|----------------|-------------------|
| Usuario/cliente | Human | Consulta eventos y disponibilidad, inicia una compra y consulta el estado de su orden. | PARTIALLY_DEFINED: no se definen identidad, autenticación, permisos ni datos del cliente (`HV-014`). |
| Consumidor asíncrono de órdenes | Autonomous process | Procesa órdenes encoladas, valida disponibilidad real, actualiza inventario y cambia el estado de la orden. | EXPLICITLY_DEFINED |
| Proceso de liberación de reservas | Autonomous process | Identifica periódicamente reservas expiradas, las libera y devuelve entradas al inventario. | EXPLICITLY_DEFINED |

Observaciones:

- El documento exige "crear eventos", pero no nombra al actor que los crea ni lo distingue del usuario comprador. No se inventa un rol de administrador u organizador; la definición queda pendiente en `HV-014`.
- El documento no nombra al actor que genera entradas `COMPLIMENTARY` (`HV-005`).
- El candidato y el evaluador mencionados en "Forma de Evaluación" quedan excluidos de esta sección por la regla de identificación de actores. Su participación se conserva únicamente en las secciones 21 (Deliverables) y 22 (Evaluation criteria).

## 4. Domain concepts

| Concept | Definition | Source status |
|---------|------------|---------------|
| Event (Evento) | Evento de concierto, teatro o deporte con nombre, fecha, lugar, capacidad total e inventario de entradas disponibles. | explicitly_defined |
| Ticket (Entrada) | Unidad vendible que tiene exactamente un estado y cuya disponibilidad, relación con cliente e impacto en reportes depende de dicho estado. | partially_defined: no se aclara si es una unidad individual/asiento o una cantidad agregada (`HV-006`). |
| Inventory (Inventario) | Disponibilidad de entradas asociada a un evento, actualizada al procesar compras y liberar reservas. | partially_defined: no se define su estructura ni su fórmula completa. |
| Purchase request (Solicitud de compra) | Solicitud que inicia el proceso de compra y debe encolarse inmediatamente. | partially_defined: no se especifican sus datos ni su relación temporal con la reserva (`HV-002`). |
| Order (Orden de compra) | Resultado identificable de una solicitud de compra cuyo estado puede consultarse y es cambiado por el consumidor. | partially_defined: no se define su modelo ni su relación con los estados de las entradas (`HV-001`, `HV-017`). |
| Reservation (Reserva) | Retención temporal de entradas solicitadas por un máximo de diez minutos; si no se confirma, se libera. | partially_defined: no se define inicio exacto, identidad, cantidad ni estado posterior a la confirmación (`HV-012`). |
| Customer (Cliente) | Persona con la que una entrada se relaciona según su estado, y que inicia compras. | inferred: la redacción lo menciona, pero no hay atributos ni reglas de identidad explícitos. |
| Message/Queue (Mensaje/Cola) | Medio por el que las solicitudes de compra se delegan a consumidores asíncronos. | explicitly_defined como mecanismo; formato y semántica funcional del mensaje no definidos. |
| Report (Reporte operativo, contable, de inventario) | Tipos de reporte sobre los que impactan los estados de las entradas. | partially_defined: solo se menciona el impacto; no se solicita ninguna capacidad de reporte (`HV-013`). |
| Payment (Pago) | No aparece en el documento. | not_defined: se registra únicamente porque "confirmar la compra" podría depender de él (`HV-008`). |

## 5. Domain states

La introducción atribuye estos estados a las entradas, mientras el Requisito Funcional 4 presenta exactamente la misma lista como estados de una orden. Por ello la entidad queda sin resolver y todos los estados requieren `HV-001`.

| ID | Entity | State | Meaning | Final | Accounting effect | Inventory effect | Source | Confidence | Status |
|----|--------|-------|---------|-------|-------------------|------------------|--------|------------|--------|
| DS-001 | UNRESOLVED (Ticket, Order o ambos) | AVAILABLE | Disponible. | No indicado | No definido. | Representa disponibilidad; el cálculo exacto no está definido. | Lista de estados; Requisito Funcional 4 | MEDIUM | REQUIRES_HUMAN_VALIDATION |
| DS-002 | UNRESOLVED (Ticket, Order o ambos) | RESERVED | Reservada temporalmente. | No | Explícitamente no representa una venta. | Debe considerarse al consultar disponibilidad; el modelo de cómputo no se detalla. | Notas generales; Requisitos Funcionales 2, 4 y 7 | MEDIUM | REQUIRES_HUMAN_VALIDATION |
| DS-003 | UNRESOLVED (Ticket, Order o ambos) | PENDING_CONFIRMATION | Denominado "En Confirmación"; sin semántica operacional. | No indicado | Explícitamente no representa una venta. | No definido. | Lista de estados; Notas generales; Requisito Funcional 4 | LOW | REQUIRES_HUMAN_VALIDATION |
| DS-004 | UNRESOLVED (Ticket, Order o ambos) | SOLD | Vendida. | Sí, irreversible | Representa una venta, por contraste explícito con RESERVED y PENDING_CONFIRMATION; el detalle contable no está definido. | Debe considerarse al consultar disponibilidad. | Notas generales; Requisitos Funcionales 4 y 7 | MEDIUM | REQUIRES_HUMAN_VALIDATION |
| DS-005 | UNRESOLVED (Ticket, Order o ambos) | COMPLIMENTARY | Cortesía. | Sí | Final, pero no contable. | No definido. | Notas generales; Requisito Funcional 4 | LOW | REQUIRES_HUMAN_VALIDATION |

El documento no define ningún estado para una orden rechazada, fallida, cancelada o expirada (`HV-009`).

## 6. State transitions

Solo se registran transiciones sustentadas por el texto. La ausencia de otras transiciones no implica que estén prohibidas; implica que no fueron definidas.

| ID | From | To | Trigger | Constraints | Expiration | Atomicity | Source | Status |
|----|------|----|---------|-------------|------------|-----------|--------|--------|
| ST-001 | AVAILABLE (interpretación probable, no inequívoca) | RESERVED | Inicio de una compra por el usuario | Debe evitar sobreventa; la secuencia frente al encolado no está definida (`HV-002`). | La reserva dura como máximo 10 minutos. | Requerida | Requisito Funcional 2; Notas generales | REQUIRES_HUMAN_VALIDATION |
| ST-002 | RESERVED | AVAILABLE | Proceso periódico detecta que se superó el tiempo límite sin confirmación | Las entradas vuelven al inventario disponible. | Máximo 10 minutos; instante inicial no definido (`HV-012`). | Requerida | Requisitos Funcionales 2 y 6; Notas generales | PARTIALLY_DEFINED |
| ST-003 | UNDEFINED | UNDEFINED | Consumidor valida disponibilidad real, actualiza inventario y cambia el estado de la orden | Debe mantener consistencia y no producir sobreventa. | No definida | Requerida | Requisito Funcional 3; Notas generales | REQUIRES_HUMAN_VALIDATION |

Transiciones no definidas por el documento:

- Entrada y salida de `PENDING_CONFIRMATION` — trigger: UNDEFINED (`HV-004`).
- Transición que produce `SOLD` — trigger: UNDEFINED (`HV-003`, `HV-008`).
- Creación de `COMPLIMENTARY` — trigger: UNDEFINED (`HV-005`).
- Resultado de una orden que no puede completarse — estado destino: UNDEFINED (`HV-009`).

## 7. Functional capabilities

### CAP-001 — Gestión de eventos

Crear y consultar eventos con su información básica e inventario.

### CAP-002 — Consulta de disponibilidad

Consultar la disponibilidad actual de entradas para eventos específicos considerando ventas y reservas temporales.

### CAP-003 — Reserva temporal de entradas

Reservar temporalmente las entradas solicitadas al iniciarse una compra, por un máximo de diez minutos.

### CAP-004 — Recepción de compras

Recibir solicitudes de compra, encolarlas inmediatamente y retornar un identificador de orden.

### CAP-005 — Procesamiento asíncrono de órdenes

Consumir órdenes, validar disponibilidad real, actualizar inventario y cambiar su estado.

### CAP-006 — Consulta de estado de orden

Consultar en cualquier momento el estado actual de una orden mediante su identificador.

### CAP-007 — Liberación automática de reservas expiradas

Detectar periódicamente reservas expiradas no confirmadas, liberarlas y restituir sus entradas al inventario.

### CAP-008 — Control de concurrencia y trazabilidad de estados

Impedir sobreventa y condiciones de carrera, y mantener transiciones atómicas y auditables.

## 8. Main flows

### MF-001 — Crear y consultar eventos

1. Un actor no definido por el documento suministra nombre, fecha, lugar y capacidad total.
2. El sistema crea el evento y su inventario asociado.
3. Un usuario consulta los eventos disponibles.
4. El sistema devuelve la información de los eventos consultados.

La autorización, las validaciones de campos, el significado de "evento disponible" y la inicialización exacta del inventario no están definidos (`HV-014`, `HV-016`, `HV-020`).

### MF-002 — Iniciar y procesar una compra

1. El usuario inicia una compra solicitando entradas.
2. El sistema debe reservar temporalmente las entradas y debe encolar inmediatamente la solicitud; el orden entre ambas operaciones no está definido (`HV-002`).
3. El sistema retorna un identificador de orden.
4. Un consumidor procesa la orden de forma asíncrona.
5. El consumidor valida disponibilidad real, actualiza inventario y cambia el estado de la orden.
6. El usuario consulta el estado de la orden mediante su identificador.

Los estados intermedios, el resultado ante disponibilidad insuficiente y el disparador que confirma la compra no están definidos (`HV-003`, `HV-004`, `HV-009`).

### MF-003 — Liberar una reserva expirada

1. Un proceso periódico identifica una reserva que superó el tiempo límite sin confirmarse.
2. El sistema libera la reserva.
3. Las entradas vuelven al inventario disponible.

### MF-004 — Consultar disponibilidad

1. Un usuario solicita la disponibilidad de entradas de un evento específico.
2. El sistema retorna la disponibilidad actual considerando entradas vendidas y reservadas temporalmente.

La semántica de "tiempo real" no está definida (`HV-007`).

## 9. Inputs

| Input | Explicit data | Missing or ambiguous data |
|-------|---------------|---------------------------|
| Creación de evento | Nombre, fecha, lugar, capacidad total. | Formato, obligatoriedad, zona horaria, restricciones de fecha y capacidad (`HV-016`). |
| Consulta de eventos | No especificado. | Filtros, paginación, definición de "disponibles" (`HV-020`). |
| Solicitud de compra | Entradas solicitadas; evento implícito. | Cantidad o identificadores, cliente, datos de pago, límites (`HV-006`, `HV-008`, `HV-014`, `HV-017`). |
| Consulta de orden | Identificador de orden. | Formato y comportamiento cuando no existe (`HV-021`). |
| Consulta de disponibilidad | Evento(s) específico(s). | Identificador, formato y semántica de "tiempo real" (`HV-007`). |

## 10. Outputs

| Output | Explicit content | Missing or ambiguous data |
|--------|------------------|---------------------------|
| Evento creado/consultado | Información básica e inventario disponible. | Representación y resultados de error. |
| Aceptación de compra | Identificador de orden retornado inmediatamente. | Estado inicial y garantías ante fallo de encolado (`HV-018`). |
| Estado de orden | Uno de AVAILABLE, RESERVED, PENDING_CONFIRMATION, SOLD o COMPLIMENTARY. | Propiedad correcta de esos estados y representación de rechazos/fallos (`HV-001`, `HV-009`). |
| Disponibilidad de evento | Disponibilidad actual considerando entradas vendidas y reservadas. | Fórmula, granularidad y mecanismo de actualización (`HV-006`, `HV-007`). |
| Resultado asíncrono | Inventario actualizado y estado de orden cambiado. | Estado final exacto y causa de rechazo (`HV-009`). |

## 11. Functional requirements

### FR-001 — Crear eventos

- Description: El sistema debe permitir crear eventos con nombre, fecha, lugar y capacidad total.
- Source: Requisitos Funcionales, punto 1.
- Dependencies: CAP-001; HV-014; HV-016.
- Status: PARTIALLY_DEFINED.

### FR-002 — Consultar eventos

- Description: El sistema debe permitir consultar eventos disponibles y su información básica.
- Source: Objetivo; Requisitos Funcionales, punto 1.
- Dependencies: CAP-001; HV-020.
- Status: PARTIALLY_DEFINED.

### FR-003 — Mantener inventario por evento

- Description: Cada evento debe tener un inventario de entradas disponibles que se actualice conforme se procesan compras.
- Source: Requisitos Funcionales, punto 1.
- Dependencies: CAP-001; CAP-002; FR-008; BR-008; HV-006.
- Status: PARTIALLY_DEFINED.

### FR-004 — Reservar temporalmente entradas

- Description: Al iniciar una compra, el sistema debe reservar temporalmente las entradas solicitadas durante un máximo de diez minutos para evitar sobreventa.
- Source: Requisitos Funcionales, punto 2.
- Dependencies: CAP-003; ST-001; VAL-001; HV-002; HV-006; HV-012.
- Status: REQUIRES_HUMAN_VALIDATION.

### FR-005 — Encolar solicitudes de compra

- Description: El sistema debe encolar inmediatamente cada solicitud de compra para su procesamiento asíncrono.
- Source: Requisitos Funcionales, punto 3.
- Dependencies: CAP-004; TC-005; TC-010; HV-002; HV-010; HV-018.
- Status: REQUIRES_HUMAN_VALIDATION.

### FR-006 — Retornar identificador de orden

- Description: Tras recibir la solicitud de compra, el sistema debe retornar inmediatamente un identificador de orden.
- Source: Requisitos Funcionales, punto 3.
- Dependencies: CAP-004; FR-005; HV-018.
- Status: PARTIALLY_DEFINED.

### FR-007 — Procesar órdenes asíncronamente

- Description: Un consumidor debe procesar las órdenes encoladas de forma asíncrona.
- Source: Objetivo; Requisitos Funcionales, punto 3.
- Dependencies: CAP-005; FR-005; TC-005; TC-010; HV-011.
- Status: DEFINED.

### FR-008 — Validar y actualizar durante el procesamiento

- Description: Al procesar una orden, el consumidor debe validar la disponibilidad real, actualizar el inventario y cambiar el estado de la orden.
- Source: Requisitos Funcionales, punto 3.
- Dependencies: CAP-005; FR-003; VAL-002; ST-003; HV-001; HV-009.
- Status: REQUIRES_HUMAN_VALIDATION.

### FR-009 — Consultar estado de orden

- Description: El sistema debe permitir consultar en cualquier momento el estado actual de una orden mediante su identificador.
- Source: Requisitos Funcionales, punto 4.
- Dependencies: CAP-006; DS-001 a DS-005; HV-001; HV-021.
- Status: REQUIRES_HUMAN_VALIDATION.

### FR-010 — Prevenir sobreventa bajo concurrencia

- Description: El sistema debe manejar condiciones de carrera y garantizar que solicitudes concurrentes para el mismo evento no resulten en sobreventa de entradas.
- Source: Descripción del Contexto; Requisitos Funcionales, punto 5.
- Dependencies: CAP-008; BR-008; NFR-001; NFR-004; TC-011.
- Status: DEFINED.

### FR-011 — Liberar reservas expiradas automáticamente

- Description: Un proceso periódico debe identificar y liberar las reservas que superaron el tiempo límite sin confirmarse, devolviendo las entradas al inventario disponible.
- Source: Requisitos Funcionales, puntos 2 y 6.
- Dependencies: CAP-007; ST-002; VAL-003; HV-003; HV-012.
- Status: PARTIALLY_DEFINED.

### FR-012 — Consultar disponibilidad actual

- Description: El sistema debe exponer una consulta reactiva que retorne en tiempo real la disponibilidad actual de entradas para eventos específicos, considerando entradas vendidas y reservadas temporalmente.
- Source: Objetivo; Requisitos Funcionales, punto 7.
- Dependencies: CAP-002; FR-003; BR-012; TC-003; TC-008; HV-006; HV-007.
- Status: REQUIRES_HUMAN_VALIDATION.

### FR-013 — Mantener transiciones atómicas

- Description: El sistema debe ejecutar atómicamente las transiciones de estado.
- Source: Notas generales.
- Dependencies: CAP-008; BR-009; TC-011.
- Status: DEFINED.

### FR-014 — Mantener transiciones auditables

- Description: El sistema debe permitir auditar las transiciones de estado.
- Source: Notas generales.
- Dependencies: CAP-008; BR-010; NFR-005.
- Status: PARTIALLY_DEFINED: no se define la información mínima de auditoría.

## 12. Business rules

### BR-001 — Estado único

Cada entrada solo puede tener un estado a la vez. Fuente: Notas generales. Status: DEFINED.

### BR-002 — Reserva con duración limitada

Una reserva temporal puede durar como máximo diez minutos. Fuente: Requisito Funcional 2. Status: PARTIALLY_DEFINED porque no se especifica cuándo comienza el plazo (`HV-012`).

### BR-003 — Reserva no confirmada expira

Si la compra no se confirma dentro del tiempo máximo, las entradas vuelven al inventario disponible. Fuente: Requisitos Funcionales 2 y 6. Status: PARTIALLY_DEFINED porque "confirmar" carece de trigger (`HV-003`).

### BR-004 — RESERVED no es venta

El estado RESERVED no representa una venta. Fuente: Notas generales. Status: DEFINED.

### BR-005 — PENDING_CONFIRMATION no es venta

El estado PENDING_CONFIRMATION no representa una venta. Fuente: Notas generales. Status: DEFINED.

### BR-006 — SOLD es final

SOLD es un estado final e irreversible. Fuente: Notas generales. Status: DEFINED.

### BR-007 — COMPLIMENTARY es final y no contable

COMPLIMENTARY es final, pero no contable. Fuente: Notas generales. Status: PARTIALLY_DEFINED porque su efecto sobre el inventario no se indica (`HV-005`).

### BR-008 — No sobreventa

No pueden venderse más entradas de las disponibles, incluso con solicitudes concurrentes. Fuente: Descripción del Contexto; Requisito Funcional 5. Status: DEFINED.

### BR-009 — Atomicidad de transición

Cada transición de estado debe ser atómica. Fuente: Notas generales. Status: DEFINED.

### BR-010 — Auditabilidad de transición

Cada transición de estado debe ser auditable. Fuente: Notas generales. Status: PARTIALLY_DEFINED: no se define la información mínima de auditoría.

### BR-011 — Liberación restituye disponibilidad

Las entradas de una reserva expirada vuelven al inventario disponible. Fuente: Requisitos Funcionales 2 y 6. Status: DEFINED.

### BR-012 — Disponibilidad considera ventas y reservas

La disponibilidad debe considerar tanto las entradas vendidas como las reservadas temporalmente. Fuente: Requisito Funcional 7. Status: PARTIALLY_DEFINED por la granularidad no resuelta (`HV-006`) y por el tratamiento no definido de PENDING_CONFIRMATION y COMPLIMENTARY (`HV-004`, `HV-005`).

## 13. Validations

### VAL-001 — Límite temporal de reserva

- Rule: La reserva no debe permanecer vigente por más de diez minutos sin confirmación.
- Source: Requisito Funcional 2.
- Status: PARTIALLY_DEFINED.

### VAL-002 — Disponibilidad real al procesar

- Rule: Antes de actualizar inventario y estado, el consumidor debe validar la disponibilidad real de las entradas solicitadas.
- Source: Requisito Funcional 3.
- Status: DEFINED.

### VAL-003 — Elegibilidad para liberación

- Rule: Solo una reserva que haya superado el tiempo límite sin confirmarse debe ser identificada para liberación automática.
- Source: Requisitos Funcionales 2 y 6.
- Status: PARTIALLY_DEFINED.

### VAL-004 — Capacidad suficiente ante concurrencia

- Rule: Una actualización de inventario no debe dejar más entradas vendidas que las disponibles.
- Source: Descripción del Contexto; Requisito Funcional 5.
- Status: DEFINED.

### VAL-005 — Exclusividad de estado

- Rule: Una entrada no puede quedar simultáneamente en más de un estado.
- Source: Notas generales.
- Status: DEFINED.

No se determina en esta especificación dónde se aplica cada validación.

## 14. Alternative flows

### ALT-001 — Reserva no confirmada

Si la reserva supera el máximo de diez minutos sin confirmación, el proceso periódico la libera y devuelve sus entradas al inventario. Fuente: Requisitos Funcionales 2 y 6. Status: PARTIALLY_DEFINED.

### ALT-002 — Disponibilidad insuficiente durante el procesamiento

El consumidor debe validar la disponibilidad real. Si no es suficiente, no puede producirse sobreventa; el estado y la respuesta observable de la orden no están definidos. Fuente: Requisitos Funcionales 3 y 5. Status: REQUIRES_HUMAN_VALIDATION (`HV-009`).

### ALT-003 — Reprocesamiento de un mensaje

El procesamiento debe garantizarse al menos una vez, por lo que una orden podría recibirse más de una vez. El comportamiento de negocio frente al duplicado no está definido. Fuente: Especificaciones Técnicas; criterio de seguridad sobre idempotencia. Status: REQUIRES_HUMAN_VALIDATION (`HV-011`).

### ALT-004 — Error reactivo transitorio

Debe aplicarse retry cuando sea apropiado, pero el documento no define qué errores son reintentables ni el resultado funcional al agotar los intentos. Fuente: Especificaciones Técnicas. Status: PARTIALLY_DEFINED.

## 15. Error scenarios

### ERR-001 — Solicitudes concurrentes compiten por la misma disponibilidad

- Expected outcome: ninguna ejecución puede provocar sobreventa; las condiciones de carrera deben manejarse correctamente.
- Source: Descripción del Contexto; Requisito Funcional 5.
- Status: DEFINED.

### ERR-002 — Disponibilidad insuficiente al consumir la orden

- Expected outcome: no actualizar el inventario de forma que produzca sobreventa.
- Undefined: estado de la orden, liberación de la reserva y respuesta consultable (`HV-009`).
- Source: Requisito Funcional 3.
- Status: REQUIRES_HUMAN_VALIDATION.

### ERR-003 — Reserva supera el límite sin confirmación

- Expected outcome: liberar la reserva y devolver las entradas al inventario.
- Source: Requisitos Funcionales 2 y 6.
- Status: PARTIALLY_DEFINED.

### ERR-004 — Mensaje de orden entregado más de una vez

- Expected outcome: UNDEFINED; la entrega at-least-once exige contemplar duplicados, pero la regla funcional no se especifica (`HV-011`).
- Source: Especificaciones Técnicas; Descripción del Contexto (compras duplicadas); criterio de seguridad sobre idempotencia.
- Status: REQUIRES_HUMAN_VALIDATION.

### ERR-005 — Error susceptible de retry en flujo reactivo

- Expected outcome: aplicar una estrategia de retry cuando sea apropiado.
- Undefined: clasificación de errores, límites y resultado al agotar intentos.
- Source: Especificaciones Técnicas.
- Status: PARTIALLY_DEFINED.

### ERR-006 — Consulta de una orden inexistente

- Expected outcome: UNDEFINED (`HV-021`).
- Source: ausencia detectada en Requisito Funcional 4.
- Status: REQUIRES_HUMAN_VALIDATION.

## 16. Acceptance criteria

### AC-001 — Crear evento

Given datos de evento con nombre, fecha, lugar y capacidad total
When se solicita crear el evento
Then el sistema crea un evento con esos datos y un inventario de entradas disponibles asociado.

### AC-002 — Consultar eventos

Given que existen eventos registrados
When un usuario consulta los eventos disponibles
Then el sistema devuelve su información básica.

### AC-003 — Retornar identificador al iniciar compra

Given una solicitud de compra recibida
When el sistema la acepta para procesamiento asíncrono
Then la solicitud se encola inmediatamente y se retorna un identificador de orden sin esperar el procesamiento del consumidor.

### AC-004 — Reservar entradas temporalmente

Given una compra iniciada para entradas solicitadas
When el sistema establece la reserva
Then las entradas quedan temporalmente reservadas por un máximo de diez minutos y se consideran reservadas al consultar la disponibilidad.

### AC-005 — Procesar orden asíncronamente

Given una orden encolada
When el consumidor la procesa
Then valida la disponibilidad real, actualiza el inventario y cambia el estado de la orden.

### AC-006 — Consultar estado de orden

Given un identificador de orden existente
When se consulta su estado
Then el sistema devuelve el estado actual de la orden.

### AC-007 — Evitar sobreventa concurrente

Given solicitudes concurrentes para el mismo evento
When en conjunto solicitan más entradas de las disponibles
Then el sistema no registra ventas por encima de la disponibilidad.

### AC-008 — Liberar reserva expirada

Given una reserva que superó el tiempo límite sin confirmarse
When el proceso periódico la identifica
Then la libera y sus entradas vuelven al inventario disponible.

### AC-009 — No liberar reserva vigente

Given una reserva que no ha superado el tiempo límite
When se ejecuta el proceso periódico
Then la reserva no se libera por expiración.

### AC-010 — Consultar disponibilidad actual

Given un evento específico con entradas vendidas y entradas reservadas temporalmente
When se consulta su disponibilidad
Then la respuesta considera ambos grupos al informar las entradas disponibles.

### AC-011 — Mantener un único estado por entrada

Given una entrada con un estado vigente
When ocurre una transición de estado
Then la entrada queda en un solo estado.

### AC-012 — Mantener SOLD irreversible

Given una entrada u orden en estado SOLD, sujeto a resolver `HV-001`
When se intenta cualquier otra transición
Then el estado SOLD se conserva.

### AC-013 — Mantener COMPLIMENTARY final y no contable

Given una entrada u orden en estado COMPLIMENTARY, sujeto a resolver `HV-001`
When se intenta cualquier otra transición
Then el estado COMPLIMENTARY se conserva y no se contabiliza como venta.

### AC-014 — Atomicidad de transición

Given una transición de estado
When se ejecuta
Then el cambio se completa de forma atómica, sin exponer un estado parcial.

### AC-015 — Auditabilidad de transición

Given una transición de estado completada
When se revisa la información de auditoría
Then existe evidencia auditable de la transición; su contenido mínimo no está definido por el documento.

### AC-016 — Validar disponibilidad en consumo

Given una orden que llega al consumidor
When el consumidor la procesa
Then valida la disponibilidad real antes de actualizar el inventario.

## 17. Concurrency requirements

| Item | Classification | Requirement | Source | Status |
|------|----------------|-------------|--------|--------|
| FR-010 | FUNCTIONAL_REQUIREMENT | Evitar sobreventa y manejar condiciones de carrera entre solicitudes del mismo evento. | Requisito Funcional 5 | DEFINED |
| BR-008 | BUSINESS_RULE | Nunca vender por encima de la disponibilidad. | Contexto; Requisito Funcional 5 | DEFINED |
| FR-013 / BR-009 | FUNCTIONAL_REQUIREMENT / BUSINESS_RULE | Ejecutar atómicamente las transiciones de estado. | Notas generales | DEFINED |
| TC-011 | TECHNICAL_CONSTRAINT | Usar optimistic locking o conditional writes para actualizar inventario. | Especificaciones Técnicas | DEFINED con alternativa preservada |
| NFR-001 | NON_FUNCTIONAL_REQUIREMENT | Manejar miles de solicitudes concurrentes eficientemente. | Contexto; Objetivo | PARTIALLY_DEFINED |
| NFR-003 | NON_FUNCTIONAL_REQUIREMENT | Manejar operaciones concurrentes sin bloqueos. | Objetivo; Requisitos Técnicos | DEFINED cualitativamente |
| NFR-004 | NON_FUNCTIONAL_REQUIREMENT | Consistencia de inventario. | Contexto; Objetivo | DEFINED cualitativamente |
| ERR-004 / HV-011 | AMBIGUITY | Procesamiento duplicado bajo entrega at-least-once. | Especificaciones Técnicas | REQUIRES_HUMAN_VALIDATION |

El documento no fija volumen exacto, distribución de carga, tasa de contención ni criterio cuantitativo de éxito (`HV-015`). No se selecciona algoritmo de concurrencia.

## 18. Asynchronous processing requirements

- Operación iniciadora: solicitud de compra del usuario (`FR-004`, `FR-005`).
- Respuesta inmediata: identificador de orden (`FR-006`).
- Trabajo asíncrono explícito: consumir la orden, validar disponibilidad real, actualizar inventario y cambiar el estado de la orden (`FR-007`, `FR-008`).
- Resultado consultable: estado actual de la orden por identificador (`FR-009`).
- Garantía de entrega: procesamiento al menos una vez (`TC-010`), con tensión de obligatoriedad en `HV-010` y semántica de duplicados pendiente en `HV-011`.
- Orden no resuelto: no se define si la reserva ocurre antes de encolar o dentro del consumidor (`HV-002`).
- Confirmación no resuelta: no se define qué convierte una reserva en venta (`HV-003`, `HV-008`).
- Estado inicial no resuelto: no se define el estado de la orden al retornar su identificador (`HV-018`).

## 19. Non-functional requirements

### NFR-001 — Alta concurrencia

- Requirement: Manejar eficientemente miles de solicitudes concurrentes.
- Basis: explicit.
- Source: Descripción del Contexto; Objetivo.
- Status: PARTIALLY_DEFINED: sin carga verificable exacta (`HV-015`).

### NFR-002 — Baja latencia bajo carga

- Requirement: Mantener tiempos de respuesta bajos incluso bajo alta carga.
- Basis: explicit.
- Source: Descripción del Contexto.
- Status: PARTIALLY_DEFINED: sin umbral ni percentil (`HV-015`).

### NFR-003 — Procesamiento no bloqueante

- Requirement: Manejar operaciones concurrentes sin bloqueos mediante un modelo reactivo.
- Basis: explicit.
- Source: Objetivo; Stack Tecnológico; Especificaciones Técnicas.
- Status: DEFINED cualitativamente.

### NFR-004 — Consistencia de inventario

- Requirement: Garantizar la consistencia del inventario, sin sobreventa ni venta múltiple de la misma disponibilidad.
- Basis: explicit.
- Source: Descripción del Contexto; Objetivo; Requisito Funcional 5.
- Status: DEFINED cualitativamente.

### NFR-005 — Auditabilidad

- Requirement: Las transiciones de estado deben ser auditables.
- Basis: explicit.
- Source: Notas generales.
- Status: PARTIALLY_DEFINED.

### NFR-006 — Acceso rápido a datos

- Requirement: Utilizar persistencia NoSQL para acceso rápido a datos.
- Basis: explicit.
- Source: Objetivo.
- Status: PARTIALLY_DEFINED: sin métrica.

### NFR-007 — Escalabilidad

- Requirement: Considerar la escalabilidad de la solución.
- Basis: evaluation-driven ("Adicional"), además del problema explícito de escala del contexto.
- Source: Descripción del Contexto; Criterios de Evaluación.
- Status: PARTIALLY_DEFINED.

### NFR-008 — Resiliencia

- Requirement: Considerar la resiliencia de la solución.
- Basis: evaluation-driven ("Adicional").
- Source: Criterios de Evaluación.
- Status: PARTIALLY_DEFINED.

### NFR-009 — Tolerancia a fallos

- Requirement: Considerar la tolerancia a fallos.
- Basis: evaluation-driven ("Adicional").
- Source: Criterios de Evaluación.
- Status: PARTIALLY_DEFINED.

### NFR-010 — Seguridad

- Requirement: Aplicar un enfoque de seguridad que contemple manejo seguro de secretos y credenciales, y ataques comunes (reintentos maliciosos, idempotencia, abuso de recursos).
- Basis: evaluation-driven.
- Source: Criterios de Evaluación.
- Status: PARTIALLY_DEFINED: no se fijan controles concretos.

### NFR-011 — Observabilidad

- Requirement: Considerar la observabilidad.
- Basis: evaluation-driven ("alto valor agregado").
- Source: Criterios de Evaluación.
- Status: PARTIALLY_DEFINED.

### NFR-012 — Mantenibilidad

- Requirement: Código en inglés, capas claramente separadas, principios SOLID y patrones de diseño apropiados.
- Basis: explicit (entregable).
- Source: Entregable 1.
- Status: DEFINED cualitativamente.

### NFR-013 — Testabilidad y cobertura

- Requirement: Tests unitarios de casos de uso, componentes reactivos y concurrencia, con cobertura mínima del 90%.
- Basis: explicit (entregable).
- Source: Entregable 3.
- Status: DEFINED.

### NFR-014 — Costos y gobernanza

- Requirement: Considerar costos y gobernanza en el enfoque cloud-native.
- Basis: evaluation-driven ("alto valor agregado").
- Source: Criterios de Evaluación.
- Status: PARTIALLY_DEFINED.

No se inventan SLA, throughput, percentiles de latencia, objetivos de disponibilidad ni tiempos de recuperación. El documento no declara requisitos explícitos de disponibilidad del servicio.

## 20. Technical constraints

### TC-001 — Java 25

Usar Java 25 y características modernas del lenguaje (Records, Pattern Matching, Virtual Threads si aplica). `Virtual Threads` es condicional, no obligatorio. Fuente: Stack Tecnológico Obligatorio.

### TC-002 — Spring Boot 4.x

Usar Spring Boot 4.x como framework base. Fuente: Stack Tecnológico Obligatorio.

### TC-003 — Spring WebFlux

Usar Spring WebFlux para implementar la API reactiva con programación no bloqueante. Fuente: Stack Tecnológico Obligatorio.

### TC-004 — Persistencia de eventos, órdenes e inventario

Usar una base de datos para persistir eventos, órdenes e inventario. El Objetivo exige persistencia NoSQL. DynamoDB Local aparece como opción ("puede usar DynamoDB Local u otra solución de persistencia"), aunque el Entregable 5 lo enumera entre las dependencias; se preserva la opcionalidad y se registra la tensión en `HV-019`. Fuente: Objetivo; Stack Tecnológico Obligatorio; Entregable 5.

### TC-005 — Cola de mensajes

Usar una cola de mensajes para el procesamiento asíncrono de órdenes. LocalStack u otra solución que cumpla la función aparece como opción, sujeto a la tensión con `TC-010` (`HV-010`). Fuente: Stack Tecnológico Obligatorio.

### TC-006 — Docker

Contenerizar la aplicación y los servicios de infraestructura. Fuente: Stack Tecnológico Obligatorio.

### TC-007 — Clean Architecture

Organizar el código en capas claramente separadas (Domain, Use Cases, Infrastructure). No se diseñan paquetes ni módulos en esta especificación. Fuente: Stack Tecnológico Obligatorio; Entregable 1.

### TC-008 — Tipos reactivos

La API debe ser completamente reactiva y retornar `Mono` y `Flux` donde corresponda. Fuente: Especificaciones Técnicas.

### TC-009 — Manejo reactivo de errores

Implementar manejo de errores reactivo con estrategias de retry cuando sea apropiado. No se prescribe una política concreta. Fuente: Especificaciones Técnicas.

### TC-010 — SQS y entrega al menos una vez

Configurar SQS para garantizar procesamiento al menos una vez (at-least-once delivery). La aparente obligatoriedad de SQS frente a la libertad previa de escoger cola requiere `HV-010`. Fuente: Especificaciones Técnicas.

### TC-011 — Control de concurrencia en inventario

Las operaciones de actualización de inventario deben usar optimistic locking o conditional writes. Se conserva la alternativa; no se selecciona una. Fuente: Especificaciones Técnicas.

### TC-012 — Docker Compose

Proveer un `docker-compose.yml` que levante todos los servicios necesarios. Fuente: Especificaciones Técnicas; Entregable 5.

### TC-013 — Convención de idioma

Nombres de variables, clases, métodos y comentarios en inglés. Fuente: Entregable 1.

### TC-014 — Herramientas de pruebas

Usar frameworks como JUnit 5, Mockito y reactor-test. La expresión "como" se conserva: son ejemplos y no se convierte cada herramienta en mandato inequívoco. Fuente: Entregable 3.

### TC-015 — Calidad interna

Seguir principios SOLID y patrones de diseño apropiados, sin seleccionar patrones desde esta especificación. Fuente: Entregable 1.

## 21. Deliverables

### DEL-001 — Repositorio de código fuente

- Estructura de Clean Architecture.
- Nombres de variables, clases, métodos y comentarios en inglés.
- Capas de dominio, casos de uso e infraestructura claramente separadas.
- Principios SOLID y patrones de diseño apropiados.
- Source: Entregables, punto 1.

### DEL-002 — README.md

- Descripción breve de la solución implementada.
- Instrucciones detalladas de instalación y configuración.
- Comandos para levantar la aplicación con Docker.
- Descripción de decisiones arquitectónicas relevantes.
- Ejemplos de uso de los endpoints principales.
- Source: Entregables, punto 2.

### DEL-003 — Tests unitarios

- Cobertura mínima del 90%.
- Tests de casos de uso con mocks de repositorios.
- Tests de componentes reactivos (WebFlux controllers y services).
- Tests que verifiquen el manejo correcto de concurrencia.
- Frameworks como JUnit 5, Mockito y reactor-test.
- Source: Entregables, punto 3.

### DEL-004 — Colección de solicitudes

Colección Postman, Insomnia o archivo curl que demuestre los flujos principales del sistema. Se mantiene abierta la elección de formato. Source: Entregables, punto 4.

### DEL-005 — docker-compose.yml

Archivo configurado con la aplicación y todas las dependencias. El documento enumera entre paréntesis DynamoDB Local y SQS/LocalStack; su carácter obligatorio depende de `HV-010` y `HV-019`. Source: Entregables, punto 5; Especificaciones Técnicas.

Elementos vinculados a la evaluación que no figuran en la lista de entregables: diagramas (`EVAL-010`) y Terraform (`EVAL-011`). No se convierten en entregables obligatorios.

## 22. Evaluation criteria

Los criterios de evaluación se conservan como elementos trazables y no se convierten en requisitos funcionales. La clasificación sigue el lenguaje literal del documento.

Forma de evaluación (contexto): reunión técnica en la que el candidato presenta la solución y explica sus decisiones de diseño, arquitectura y tecnología; se contrasta lo requerido con lo implementado, con énfasis en el razonamiento técnico, los trade-offs y la experiencia en sistemas distribuidos, cloud-native y seguros.

### EVAL-001 — Cumplimiento funcional

- Classification: MANDATORY.
- Criterion: los puntos de la prueba están correctamente abordados; se evalúa la implementación y la coherencia de la solución.
- Source wording: "Cumplimiento funcional de los requerimientos".

### EVAL-002 — Calidad del diseño y decisiones de arquitectura

- Classification: DIFFERENTIAL.
- Criterion: justificar por qué la solución se diseñó así, incluida la elección de patrones arquitectónicos.
- Source wording: "valor diferencial".

### EVAL-003 — Concurrencia y consistencia

- Classification: DIFFERENTIAL (dentro de EVAL-002).
- Criterion: justificar el manejo de concurrencia y consistencia.

### EVAL-004 — Asincronía y procesamiento basado en eventos

- Classification: DIFFERENTIAL (dentro de EVAL-002).
- Criterion: justificar el uso adecuado de asincronía y procesamiento basado en eventos.

### EVAL-005 — Escalabilidad, resiliencia y tolerancia a fallos

- Classification: HIGH_VALUE_ADDITION.
- Criterion: presentar consideraciones de escalabilidad, resiliencia y tolerancia a fallos.
- Source wording: "(Adicional)", dentro del bloque de valor diferencial.

### EVAL-006 — Seguridad de la solución

- Classification: MANDATORY como criterio de evaluación; no prescribe controles funcionales.
- Criterion: explicar el enfoque de seguridad aplicado al sistema.

### EVAL-007 — Secretos y credenciales

- Classification: MANDATORY (dentro de EVAL-006).
- Criterion: manejo seguro de secretos y credenciales.

### EVAL-008 — Ataques comunes y abuso

- Classification: MANDATORY (dentro de EVAL-006).
- Criterion: consideraciones frente a reintentos maliciosos, idempotencia y abuso de recursos.

### EVAL-009 — Comunicación clara y estructurada

- Classification: HIGH_VALUE_ADDITION.
- Criterion: comunicar la solución de forma clara y ordenada.
- Source wording: "Se otorgará valor adicional".

### EVAL-010 — Diagramas e interacciones

- Classification: HIGH_VALUE_ADDITION (dentro de EVAL-009).
- Criterion: diagramas de arquitectura, diagramas de flujo o secuencia y explicación de las interacciones entre componentes.

### EVAL-011 — Infraestructura como Código

- Classification: DIFFERENTIAL.
- Criterion: infraestructura definida con Terraform y explicación del código y de las decisiones (networking, seguridad, escalabilidad, aislamiento de entornos).
- Source wording: "valor diferencial", "factor diferencial".

### EVAL-012 — Experiencia Cloud-Native en AWS

- Classification: HIGH_VALUE_ADDITION.
- Criterion: demostrar experiencia práctica en AWS, incluidas buenas prácticas de seguridad en la nube.
- Source wording: "alto valor agregado".

### EVAL-013 — Operación, costos, observabilidad y gobernanza

- Classification: HIGH_VALUE_ADDITION (dentro de EVAL-012).
- Criterion: enfoque cloud-native orientado a operación y producción; consideraciones de costos, observabilidad y gobernanza.

### EVAL-014 — Madurez técnica y experiencia práctica

- Classification: MANDATORY como criterio de reflexión.
- Criterion: reflexionar sobre limitaciones de la solución, posibles mejoras y decisiones que cambiarían en un entorno productivo real.

### Cobertura de aspectos evaluados

| Evaluated aspect | EVAL item | Classification |
|------------------|-----------|----------------|
| Functional compliance | EVAL-001 | MANDATORY |
| Architecture decisions | EVAL-002 | DIFFERENTIAL |
| Concurrency and consistency | EVAL-003 | DIFFERENTIAL |
| Event-driven processing | EVAL-004 | DIFFERENTIAL |
| Scalability | EVAL-005 | HIGH_VALUE_ADDITION |
| Resilience | EVAL-005 | HIGH_VALUE_ADDITION |
| Fault tolerance | EVAL-005 | HIGH_VALUE_ADDITION |
| Security | EVAL-006, EVAL-007, EVAL-008 | MANDATORY |
| Communication | EVAL-009 | HIGH_VALUE_ADDITION |
| Diagrams | EVAL-010 | HIGH_VALUE_ADDITION |
| Infrastructure as Code | EVAL-011 | DIFFERENTIAL |
| AWS cloud-native experience | EVAL-012 | HIGH_VALUE_ADDITION |
| Cost considerations | EVAL-013 | HIGH_VALUE_ADDITION |
| Observability | EVAL-013 | HIGH_VALUE_ADDITION |
| Governance | EVAL-013 | HIGH_VALUE_ADDITION |
| Limitations | EVAL-014 | MANDATORY |
| Future improvements | EVAL-014 | MANDATORY |
| Production trade-offs | EVAL-014 | MANDATORY |

## 23. Ambiguities and contradictions

### Comprobaciones obligatorias

| Check | Finding | Result |
|-------|---------|--------|
| A. Ticket State vs Order State | La lista se atribuye a las entradas en el contexto y a la orden en el Requisito Funcional 4. | Inconsistente → `HV-001` |
| B. Reservation sequence | "Al iniciar una compra" se reserva, pero la solicitud se encola "inmediatamente" y el consumidor valida disponibilidad real. | No inequívoco → `HV-002` |
| C. Purchase confirmation | Se exige confirmar la compra dentro del plazo, sin trigger explícito. | Sin trigger → `HV-003` |
| D. PENDING_CONFIRMATION | Solo se nombra y se indica que no representa venta. | Sin significado, entrada ni salida → `HV-004` |
| E. COMPLIMENTARY | Solo se define como final y no contable. | Sin creación, actor ni efecto de inventario → `HV-005` |
| F. Individual ticket vs quantity | Se habla de "asientos vendidos a múltiples personas" y de "cada entrada", pero el evento solo exige capacidad total. | No resuelto → `HV-006` |
| G. Real-time availability | "Retorne en tiempo real la disponibilidad actual". | Semántica no clara → `HV-007` |
| H. Payment | No existe flujo, proveedor ni confirmación de pago en el documento. | No definido → `HV-008` |

### Hallazgos

| Finding | Classification | Description | Related validation |
|---------|----------------|-------------|--------------------|
| AMB-001 | CONTRADICTION | Los estados se introducen como estados de una entrada, pero la consulta de orden usa exactamente la misma lista como estados de orden. | HV-001 |
| AMB-002 | AMBIGUITY | "Al iniciar una compra" se reserva, pero la solicitud se encola inmediatamente y el consumidor valida disponibilidad; no hay orden inequívoco. | HV-002 |
| AMB-003 | AMBIGUITY | No se define la acción o condición que confirma la compra y produce una venta. | HV-003, HV-008 |
| AMB-004 | AMBIGUITY | PENDING_CONFIRMATION no tiene semántica operacional ni transiciones definidas. | HV-004 |
| AMB-005 | AMBIGUITY | COMPLIMENTARY no tiene flujo de creación, actor autorizado ni efecto de inventario. | HV-005 |
| AMB-006 | AMBIGUITY | Se mencionan asientos, entradas y capacidad agregada sin fijar la granularidad del inventario. | HV-006, HV-017 |
| AMB-007 | AMBIGUITY | "En tiempo real" puede significar lectura actual al solicitarla o actualizaciones continuas. | HV-007 |
| AMB-008 | AMBIGUITY | La compra debe confirmarse, pero no existe flujo ni proveedor de pago definido. | HV-008 |
| AMB-009 | AMBIGUITY | No hay estado ni resultado definido para disponibilidad insuficiente o fallo asíncrono. | HV-009 |
| AMB-010 | CONTRADICTION | La cola puede ser cualquier solución que cumpla la función, pero luego se ordena configurar SQS. | HV-010 |
| AMB-011 | AMBIGUITY | At-least-once permite duplicados, pero no se define la semántica de idempotencia de negocio. | HV-011 |
| AMB-012 | AMBIGUITY | No se define cuándo inicia el plazo de la reserva ni qué unidad concreta representa una reserva. | HV-012 |
| AMB-013 | AMBIGUITY | Se menciona impacto en reportes operativos, contables y de inventario, pero no se solicitan capacidades de reporte. | HV-013 |
| AMB-014 | AMBIGUITY | No se definen autenticación, autorización ni roles, aunque la seguridad se evalúa. | HV-014 |
| AMB-015 | AMBIGUITY | "Miles", "eficiente" y "tiempos bajos" carecen de umbrales verificables. | HV-015 |
| AMB-016 | AMBIGUITY | Fecha y capacidad del evento no tienen reglas de validación ni zona horaria. | HV-016 |
| AMB-017 | AMBIGUITY | No se define la cardinalidad entre orden, reserva y entradas o cantidad. | HV-017 |
| AMB-018 | AMBIGUITY | No se define el estado inicial de la orden cuyo identificador se retorna inmediatamente. | HV-018 |
| AMB-019 | AMBIGUITY | La base de datos puede ser "otra solución de persistencia", el Objetivo exige NoSQL y el Entregable 5 enumera DynamoDB Local. | HV-019 |
| AMB-020 | AMBIGUITY | "Eventos disponibles" no define el criterio de disponibilidad de un evento. | HV-020 |
| AMB-021 | AMBIGUITY | No se define el resultado de consultar una orden inexistente. | HV-021 |

## 24. Human validation required

| ID | Question | Reason | Impact | Priority | Related items |
|----|----------|--------|--------|----------|---------------|
| HV-001 | ¿AVAILABLE, RESERVED, PENDING_CONFIRMATION, SOLD y COMPLIMENTARY pertenecen a Ticket, a Order o a ambos mediante máquinas de estado distintas? | El texto atribuye los estados primero a las entradas y después a las órdenes. | Define entidades, transiciones, consulta de orden y efecto contable. | HIGH | DS-001, DS-002, DS-003, DS-004, DS-005, ST-001, ST-002, ST-003, FR-008, FR-009, BR-001, AC-012, AC-013, Domain:Order, Domain:Ticket |
| HV-002 | ¿La secuencia es solicitar → reservar → encolar, o solicitar → encolar → reservar de forma asíncrona en el consumidor? | La reserva "al iniciar una compra" y el encolado "inmediato" no fijan un orden. | Afecta consistencia, respuesta inmediata y comportamiento ante fallos. | HIGH | FR-004, FR-005, FR-006, ST-001, MF-002, AC-003, AC-004 |
| HV-003 | ¿Qué acción o condición convierte una reserva en una venta confirmada? | El documento exige confirmar dentro de diez minutos, pero no define el trigger. | Impide completar la máquina de estados y determinar qué reservas expiran. | HIGH | BR-003, ST-002, ST-003, FR-011, VAL-003, ALT-001, DS-004 |
| HV-004 | ¿Qué significa PENDING_CONFIRMATION, cómo se alcanza y cómo se abandona? | Solo se nombra y se aclara que no representa una venta. | Impide definir el flujo, sus tiempos y el estado consultable. | HIGH | DS-003, BR-005, BR-012, FR-009, ST-003 |
| HV-005 | ¿Cómo se crea una entrada COMPLIMENTARY, quién puede generarla y consume inventario o capacidad? | Solo se define como final y no contable. | Afecta actores, inventario y disponibilidad. | HIGH | DS-005, BR-007, BR-012, AC-013, FR-003, FR-012 |
| HV-006 | ¿El dominio administra entradas o asientos individuales, o únicamente una cantidad disponible por evento? | El texto habla de asientos y de "cada entrada", pero el evento solo exige capacidad total e inventario. | Cambia el modelo de dominio, las reservas y el control de concurrencia. | HIGH | Domain:Ticket, Domain:Inventory, FR-003, FR-004, FR-012, BR-001, BR-012 |
| HV-007 | ¿"Disponibilidad en tiempo real" significa una lectura actual bajo demanda o actualizaciones continuas hacia el cliente? | La semántica no está definida y no debe asumirse una tecnología. | Afecta el contrato funcional de la consulta de disponibilidad. | MEDIUM | FR-012, CAP-002, AC-010 |
| HV-008 | ¿Existe un flujo o proveedor de pago, y la confirmación de la compra depende de un resultado de pago? | Se habla de compra confirmada, pero el documento no menciona pago. | Define el trigger de SOLD y los fallos de confirmación. | HIGH | Domain:Payment, DS-004, BR-003, ST-003 |
| HV-009 | ¿Qué estado y resultado consultable debe tener una orden rechazada por disponibilidad insuficiente o por fallo en el procesamiento asíncrono? | La lista de estados no incluye rechazo, fallo ni cancelación, y el consumidor puede no encontrar disponibilidad. | Impide definir flujos alternativos, errores y terminalidad de las órdenes. | HIGH | FR-008, FR-009, ALT-002, ERR-002, AC-005 |
| HV-010 | ¿SQS es obligatorio, o puede usarse cualquier cola que cumpla la función? | El stack permite cualquier solución, pero las Especificaciones Técnicas ordenan configurar SQS. | Cambia una restricción tecnológica y el contenido del docker-compose. | HIGH | TC-005, TC-010, DEL-005 |
| HV-011 | ¿Cuál es la regla funcional ante la entrega o el envío repetido de una misma solicitud u orden? | At-least-once admite duplicados y las compras duplicadas son un problema declarado. | Es crítica para no duplicar ventas ni descuentos de inventario. | HIGH | ALT-003, ERR-004, FR-007, TC-010, EVAL-008 |
| HV-012 | ¿Cuándo empieza exactamente el plazo de diez minutos, y la reserva se identifica por orden, cliente, entradas o evento y cantidad? | El máximo está definido, pero no su origen temporal ni su alcance. | Afecta expiración, liberación y concurrencia. | HIGH | FR-004, FR-011, BR-002, ST-002, VAL-001, VAL-003, Domain:Reservation |
| HV-013 | ¿Los reportes operativos, contables y de inventario están dentro del alcance funcional, o solo debe preservarse la semántica de los estados? | Se menciona su impacto, pero no se pide consultar ni generar reportes. | Evita agregar o excluir capacidades sin autorización. | MEDIUM | Domain:Report, DS-004, DS-005 |
| HV-014 | ¿Qué actores, roles y reglas de autenticación y autorización aplican a crear eventos, comprar, consultar órdenes y emitir cortesías? | La seguridad es criterio de evaluación, pero el acceso funcional no está definido. | Afecta actores, permisos y exposición de datos. | MEDIUM | Actors:Usuario/cliente, FR-001, FR-004, FR-009, FR-012, NFR-010, EVAL-006 |
| HV-015 | ¿Cuáles son los umbrales verificables de concurrencia, throughput y latencia bajo carga? | "Miles", "eficiente" y "bajos" no permiten una verificación objetiva. | Afecta la aceptación no funcional. | MEDIUM | NFR-001, NFR-002, AC-007, EVAL-005 |
| HV-016 | ¿Qué validaciones aplican a la fecha y la capacidad del evento (zona horaria, fechas pasadas, cero, valores negativos)? | Solo se enumeran los campos básicos. | Afecta las reglas de creación e integridad de datos. | MEDIUM | FR-001, AC-001, Domain:Event |
| HV-017 | ¿Cuál es la cardinalidad entre una orden, una reserva y las entradas o cantidades solicitadas, y puede una orden abarcar más de un evento? | La relación entre estos conceptos no está definida. | Afecta el modelo de dominio, la atomicidad y la consulta de orden. | HIGH | Domain:Order, Domain:Reservation, Domain:Ticket, FR-004, FR-009 |
| HV-018 | ¿Cuál es el estado inicial de la orden cuyo identificador se retorna, y qué debe ocurrir si el encolado falla? | Se exige retornar un identificador inmediatamente, sin definir estado inicial ni aceptación fallida. | Afecta el contrato de recepción y la consulta posterior. | HIGH | FR-005, FR-006, FR-009, MF-002, AC-003 |
| HV-019 | ¿DynamoDB Local es obligatorio, o puede usarse cualquier otra persistencia NoSQL? | El stack permite "otra solución de persistencia", el Objetivo exige NoSQL y el Entregable 5 enumera DynamoDB Local. | Cambia una restricción tecnológica y el contenido del docker-compose. | MEDIUM | TC-004, NFR-006, DEL-005 |
| HV-020 | ¿Qué significa "evento disponible" al consultar eventos? | El Objetivo habla de consultar eventos disponibles sin definir el criterio. | Afecta el resultado de la consulta de eventos. | LOW | FR-002, AC-002, MF-001 |
| HV-021 | ¿Qué resultado debe obtenerse al consultar una orden inexistente? | El Requisito Funcional 4 solo describe la consulta de una orden existente. | Afecta el escenario de error de la consulta de orden. | LOW | ERR-006, FR-009, AC-006 |

Las respuestas se registran en `human-review/ticketing.functional-review.yaml`. Esta especificación no responde ninguna de estas preguntas.

## 25. Open questions

Las 21 preguntas `HV-001` a `HV-021` permanecen abiertas.

- HIGH (13, bloqueantes): HV-001, HV-002, HV-003, HV-004, HV-005, HV-006, HV-008, HV-009, HV-010, HV-011, HV-012, HV-017, HV-018.
- MEDIUM (6, no bloqueantes): HV-007, HV-013, HV-014, HV-015, HV-016, HV-019.
- LOW (2, no bloqueantes): HV-020, HV-021.

Las preguntas HIGH corresponden a decisiones que la arquitectura no debería tomar silenciosamente. Mientras permanezcan pendientes, `architecture_can_start` es `false`.

## 26. Traceability

| Specification item | Requirement source |
|--------------------|--------------------|
| Purpose; Problem statement | Descripción del Contexto; Objetivo |
| Actors | Objetivo; Requisitos Funcionales 2, 3 y 6 |
| Domain concepts | Descripción del Contexto; Objetivo; Requisitos Funcionales 1–7 |
| DS-001..DS-005 | Lista de estados; Notas generales; Requisito Funcional 4 |
| ST-001 | Requisito Funcional 2; Notas generales |
| ST-002 | Requisitos Funcionales 2 y 6; Notas generales |
| ST-003 | Requisito Funcional 3; Notas generales |
| CAP-001; FR-001..FR-003 | Objetivo; Requisito Funcional 1 |
| CAP-002; FR-012; MF-004 | Objetivo; Requisito Funcional 7 |
| CAP-003; FR-004 | Requisito Funcional 2 |
| CAP-004; FR-005..FR-006 | Requisito Funcional 3 |
| CAP-005; FR-007..FR-008 | Objetivo; Requisito Funcional 3 |
| CAP-006; FR-009 | Requisito Funcional 4 |
| CAP-007; FR-011; MF-003 | Requisitos Funcionales 2 y 6 |
| CAP-008; FR-010, FR-013, FR-014 | Descripción del Contexto; Notas generales; Requisito Funcional 5 |
| BR-001, BR-004..BR-007, BR-009, BR-010 | Notas generales |
| BR-002, BR-003, BR-011 | Requisitos Funcionales 2 y 6 |
| BR-008 | Descripción del Contexto; Requisito Funcional 5 |
| BR-012 | Requisito Funcional 7 |
| VAL-001..VAL-005 | Notas generales; Requisitos Funcionales 2, 3, 5 y 6 |
| ALT-001..ALT-004; ERR-001..ERR-006 | Requisitos Funcionales 2–6; Especificaciones Técnicas |
| AC-001..AC-016 | Notas generales; Requisitos Funcionales 1–7 |
| NFR-001..NFR-006 | Descripción del Contexto; Objetivo; Notas generales |
| NFR-007..NFR-011, NFR-014 | Criterios de Evaluación |
| NFR-012, NFR-013 | Entregables 1 y 3 |
| TC-001..TC-007 | Stack Tecnológico Obligatorio |
| TC-008..TC-012 | Especificaciones Técnicas |
| TC-013..TC-015 | Entregables 1 y 3 |
| DEL-001..DEL-005 | Entregables 1–5 |
| EVAL-001..EVAL-014 | Forma de Evaluación; Criterios de Evaluación |
| HV-001..HV-021 | Ambigüedades y contradicciones de la sección 23 |

## 27. Analysis trail

| Observation | Classification | Source | Decision |
|-------------|----------------|--------|----------|
| El contexto reporta duplicados, doble venta y timeouts. | NON_FUNCTIONAL_REQUIREMENT / ERROR_SCENARIO | Descripción del Contexto | Registrar consistencia, concurrencia y latencia sin inventar SLA. |
| Los cinco estados se atribuyen a entradas y también a órdenes. | CONTRADICTION | Lista de estados; Requisito Funcional 4 | Mantener Entity como UNRESOLVED y crear HV-001. |
| La reserva ocurre al iniciar la compra, mientras la solicitud se encola inmediatamente. | AMBIGUITY | Requisitos Funcionales 2 y 3 | No seleccionar orden; crear HV-002. |
| La expiración máxima es diez minutos. | BUSINESS_RULE / VALIDATION | Requisitos Funcionales 2 y 6 | Conservar el valor exacto; preguntar por el inicio del cómputo en HV-012. |
| El consumidor valida disponibilidad, actualiza inventario y estado. | FUNCTIONAL_REQUIREMENT | Requisito Funcional 3 | Registrar FR-008; no diseñar transacción ni algoritmo. |
| Se exige no sobreventa ante condiciones de carrera. | FUNCTIONAL_REQUIREMENT / BUSINESS_RULE / NON_FUNCTIONAL_REQUIREMENT | Contexto; Requisito Funcional 5 | Separar comportamiento, regla y atributo sin seleccionar mecanismo. |
| Optimistic locking o conditional writes. | TECHNICAL_CONSTRAINT | Especificaciones Técnicas | Preservar la alternativa; no elegir. |
| La cola es abierta en el stack, pero SQS aparece obligatorio después. | CONTRADICTION | Stack; Especificaciones Técnicas | Crear HV-010; no elegir tecnología. |
| La persistencia es abierta en el stack, NoSQL en el Objetivo y DynamoDB Local en el Entregable 5. | AMBIGUITY | Objetivo; Stack; Entregable 5 | Preservar opcionalidad; crear HV-019. |
| At-least-once puede duplicar procesamiento. | TECHNICAL_CONSTRAINT / AMBIGUITY | Especificaciones Técnicas; Criterios de Evaluación | Crear HV-011; no definir política de deduplicación. |
| "Tiempo real" no define el modo de entrega. | AMBIGUITY | Requisito Funcional 7 | Crear HV-007; no seleccionar tecnología. |
| Los reportes solo se mencionan como impacto de los estados. | OUT_OF_SCOPE / AMBIGUITY | Párrafo previo a la lista de estados | No añadir capacidades de reporte; crear HV-013. |
| No hay pago definido. | AMBIGUITY | Requisitos Funcionales 2 y 3 | No inventar flujo de pago; crear HV-008. |
| Candidato y evaluador aparecen en "Forma de Evaluación". | EVALUATION_CRITERION | Forma de Evaluación | Excluirlos de Actors; conservarlos solo en Deliverables y Evaluation criteria. |
| Seguridad, IaC y AWS aparecen en evaluación con grados distintos. | EVALUATION_CRITERION | Criterios de Evaluación | Conservar MANDATORY, DIFFERENTIAL y HIGH_VALUE_ADDITION sin convertirlos en FR. |
| El documento exige entregables y cobertura mínima. | DELIVERABLE / NON_FUNCTIONAL_REQUIREMENT | Entregables | Registrar cada entregable y el 90% de forma trazable. |
| No se define quién crea eventos ni cortesías. | AMBIGUITY | Requisito Funcional 1; Notas generales | No inventar roles; remitir a HV-005 y HV-014. |

Esta especificación no selecciona base de datos, tipo de cola, patrón de consistencia, patrón de mensajería, módulos, clases, tablas, endpoints definitivos, política de retry, DLQ, FIFO/Standard, Outbox, Saga ni CQRS.
