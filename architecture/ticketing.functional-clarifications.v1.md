---
artifact: functional-clarifications
schema_version: 1.0
feature: ticketing-event-processing
version: 1
source:
  human_review:
    artifact: human-review/ticketing.architecture-review.yaml
    status: APPROVED
  feature_spec:
    artifact: feature-spec/ticketing.feature-spec.v4.md
    version: 4
  architecture: architecture/ticketing.architecture.v2.md
addressed_to: requirements-analyst
status: PENDING_INCORPORATION
generated_at: 2026-10-04
---

# Functional Clarifications v1 — Ticketing Event Processing

Este documento registra, para el Requirements Analyst, las decisiones humanas de la Human Architecture Review que cambian o precisan comportamiento observable respecto de la Feature Specification v4. El Architect **no** modifica la especificación; estas aclaraciones ya están aplicadas en el diseño consolidado (`ticketing.architecture.v2.md`) y deben incorporarse en una versión posterior de la especificación.

Cada entrada indica: ID de origen, regla funcional, criterios de aceptación afectados (existentes y sugeridos) y sección de la especificación a actualizar. Los criterios sugeridos no tienen ID: los asigna el Requirements Analyst a partir del siguiente libre (`AC-032` en adelante) conforme a su contrato.

## 1. FG-001 — Ticket por Order (también ADR-003)

- **Regla funcional**: una Order solicita entre 1 y 10 Ticket individuales, todos del mismo Event y sin identificadores repetidos. Una solicitud con cero Ticket, más de 10 o repetidos se rechaza con error de validación antes de reservar, sin crear Reservation, Order ni Order ID y sin modificar el inventario. El máximo es configurable; el valor desplegado es 10.
- **AC afectados**: `AC-004`, `AC-016`. Sugerido: "Given una solicitud con 0, más de 10 o identificadores repetidos, When el `CUSTOMER` inicia la compra, Then se rechaza con error de validación sin Reservation, Order ni Order ID".
- **Secciones a actualizar**: §5 (Order: cardinalidad 1..10), §11 `FR-004`, §12 `BR-014`, §13 `VAL-010` (o validación nueva), §16.

## 2. FG-002 — Capacidad máxima de un Event

- **Regla funcional**: la capacidad máxima soportada es 50.000 Ticket por Event. Una solicitud que la supere se rechaza completa con error de validación, sin crear el Event ni sus Ticket. El Event solo queda habilitado para consulta y venta tras comprobar que la cantidad de Ticket creados coincide con `Event.capacity`. La consulta de disponibilidad es paginada y no retorna los 50.000 Ticket en una respuesta. El límite puede elevarse por configuración con nuevas pruebas de capacidad.
- **AC afectados**: `AC-001`, `AC-017`, `AC-010`. Sugerido: "Given una capacidad superior a 50.000, When el `ADMIN` crea el Event, Then se rechaza completa sin escribir nada".
- **Secciones a actualizar**: §13 `VAL-008`, §11 `FR-001`, `FR-012`, §16.

## 3. FG-003 — Pago aprobado o de resultado desconocido sin compra confirmada (también ADR-008)

- **Regla funcional**: una Order que termina sin confirmarse (`EXPIRED`, o `FAILED` con resultado de pago desconocido) conserva su estado terminal y sus Ticket quedan liberados; una aprobación tardía nunca la reabre ni la confirma. El sistema reversa el pago: la transición de cierre marca la Order como pendiente de reverso; el sistema solicita al proveedor la cancelación del PaymentAttempt con reintento acotado; al confirmarla retira la marca y lo audita; si agota los reintentos, se alerta y queda pendiente de revisión manual. La cancelación es idempotente por `paymentAttemptId` y válida en cualquier estado del intento; si llega antes que el cobro, el proveedor la registra y rechaza cualquier cobro posterior con ese `paymentAttemptId`. La marca de reverso no se expone en la consulta de la Order.
- **AC afectados**: `AC-019`, `AC-008`, `AC-021`, `AC-025`. Sugeridos: "Given el proveedor aprueba un pago cuya Order ya expiró, When se procesa el resultado, Then la Order permanece `EXPIRED`, sus Ticket `AVAILABLE` y se solicita una única cancelación del pago"; "Given una cancelación recibida antes del cobro, When llega el cobro, Then se rechaza".
- **Secciones a actualizar**: §11 `FR-015`, `FR-017`; §12 `BR-003` (o regla nueva); §14 (flujo alternativo nuevo de reverso); §15 (error nuevo); §16; §4 (Payment Mock: operación de cancelación).

## 4. FG-004 — Repetición de una solicitud rechazada

- **Regla funcional**: un rechazo síncrono (por indisponibilidad u otra causa) no persiste Reservation, Order ni registro de idempotencia. Una repetición con la misma idempotency key se procesa como solicitud nueva y puede crear una Order si los Ticket volvieron a estar disponibles; en ese caso la clave queda registrada con esa Order.
- **AC afectados**: `AC-016`, `AC-023` (contexto). Sugerido: "Given una solicitud rechazada por indisponibilidad, When se repite con la misma clave y los Ticket están disponibles, Then se crea una Order".
- **Secciones a actualizar**: §12 `BR-019`, §14 `ALT-003`, §18.

## 5. FG-005 — Compra sobre un Event pasado

- **Regla funcional**: un Event es pasado cuando su `startsAt` es menor o igual al instante actual del servidor en UTC al recibir la solicitud de compra (mismo criterio que lo excluye del listado). La compra sobre un Event pasado se rechaza síncronamente con un código propio (`EVENT_NOT_ON_SALE`, HTTP 409), distinto de la indisponibilidad, sin crear Reservation, Order ni Order ID y sin modificar el inventario. La consulta de disponibilidad por identificador de un Event pasado sigue respondiendo de forma informativa. Una Reservation creada antes de `startsAt` puede confirmarse dentro de su vigencia aunque el Event haya comenzado (comportamiento aceptado). Un Event en `PROVISIONING` o `FAILED` se trata como inexistente.
- **AC afectados**: `AC-002`, `AC-016`. Sugeridos: "Given un Event pasado, When un `CUSTOMER` inicia la compra, Then se rechaza con `EVENT_NOT_ON_SALE` sin crear nada"; "Given un Event pasado, When se consulta su disponibilidad por identificador, Then responde de forma informativa".
- **Secciones a actualizar**: §9 `MF-002`, `MF-003`; §11 `FR-002`; §12 `BR-022`; §13 `VAL-010` (o validación nueva); §15 (error nuevo); §16.

## 6. ADR-004 (sucesor ADR-024) — Creación asíncrona de Events

- **Regla funcional**: la creación de un Event es asíncrona. La solicitud se valida completa; si es válida, el Event se crea en la fase `PROVISIONING` (no habilitado) y la respuesta inmediata (HTTP 202) devuelve el `eventId` y el estado de aprovisionamiento. El inventario se describe con una definición compacta: secciones con código único, filas con etiqueta única por sección y cantidad de asientos por fila (numerados desde 1), más rangos de asientos de cortesía; el sistema genera cada `ticketId` como `<sección>-<fila>-<asiento>`. La capacidad debe coincidir con la cantidad de asientos derivados, cortesías incluidas. Límites de la definición (configurables): 100 secciones, 2.000 filas en total, 1.000 asientos por fila, 500 rangos de cortesía. El sistema escribe los Ticket en segundo plano y habilita el Event (`ENABLED`) solo tras verificar que existen exactamente `capacity` Ticket; si el aprovisionamiento falla definitivamente, el Event queda en `FAILED` y nunca es visible ni vendible. Mientras no está `ENABLED`, el Event no aparece en el listado y la disponibilidad y la compra lo tratan como inexistente. El `ADMIN` puede consultar el estado de aprovisionamiento (`PROVISIONING`, `ENABLED`, `FAILED`) con su progreso.
- **AC afectados**: `AC-001` (el Event "se crea" en dos pasos observables), `AC-017`, `AC-026`, `AC-028`. Sugeridos: "Given una solicitud válida, When el `ADMIN` crea el Event, Then recibe el `eventId` en `PROVISIONING` y el Event no es visible para el `CUSTOMER` hasta `ENABLED`"; "Given un aprovisionamiento que falla definitivamente, Then el Event queda `FAILED` y no es visible ni vendible".
- **Secciones a actualizar**: §5 (Event: ciclo de aprovisionamiento; Ticket: identidad generada), §8 `CAP-001`, §9 `MF-001`, §10 (Crear Event: entradas y resultado; nueva operación de consulta de aprovisionamiento), §11 `FR-001`, §13 `VAL-009`, §16.

## 7. ADR-007 (sucesor ADR-027) — Idempotencia de la creación de Events

- **Regla funcional**: la creación de un Event exige una idempotency key (`Idempotency-Key`) vinculada al `ADMIN` autenticado, con vigencia mínima de 24 horas. Una repetición con la misma clave y el mismo contenido devuelve el mismo `eventId` con su estado de aprovisionamiento actual (HTTP 200 con indicación de repetición) y no crea un segundo Event. La misma clave con contenido distinto se rechaza (`IDEMPOTENCY_KEY_REUSED`, HTTP 422) sin efectos.
- **AC afectados**: `AC-001`. Sugerido: "Given una creación ya aceptada, When el `ADMIN` la repite con la misma clave y contenido, Then recibe el mismo `eventId` y no se crea otro Event".
- **Secciones a actualizar**: §10 (Crear Event: identidad de operación repetible), §11 `FR-017`, §18 (añadir la creación de Events a las operaciones idempotentes).

## 8. ADR-013 (sucesor ADR-032) — Una Order activa por cliente y por Event

- **Regla funcional**: un `CUSTOMER` no puede tener más de una Order en `CREATED` para el mismo Event. Una segunda solicitud de compra para ese Event mientras la primera está activa se rechaza síncronamente (`ACTIVE_ORDER_EXISTS`, HTTP 409), sin crear Reservation, Order ni Order ID y sin modificar el inventario. Una repetición con la misma idempotency key se resuelve como repetición y no como segunda compra. Al alcanzar la Order activa un estado terminal, el cliente puede iniciar una nueva compra para ese Event. Limitaciones aceptadas: la regla no impide el acaparamiento con varias cuentas; una Order retenida para revisión manual por inconsistencia técnica (cuarentena, no expuesta) mantiene el bloqueo hasta su revisión.
- **AC afectados**: `AC-004`, `AC-016`, `AC-028`. Sugeridos: "Given un `CUSTOMER` con una Order `CREATED` en un Event, When inicia otra compra para el mismo Event, Then se rechaza con `ACTIVE_ORDER_EXISTS` sin crear nada"; "Given dos solicitudes simultáneas del mismo `CUSTOMER` para el mismo Event con claves distintas, Then como máximo una crea Order".
- **Secciones a actualizar**: §5 (invariante nuevo), §11 `FR-004` o requisito nuevo, §12 (regla nueva), §15 (error nuevo), §16, §20.2.

## 9. ADR-016 (sucesor ADR-035) — Códigos de rechazo, 503 con circuito abierto y precedencia

- **Regla funcional**:
  - `ACTIVE_ORDER_EXISTS` (409) según la entrada 8.
  - `EVENT_NOT_ON_SALE` (409) según la entrada 5.
  - Cuando el sistema de colas está indisponible (circuito de publicación abierto), la compra se rechaza síncronamente con `SERVICE_UNAVAILABLE` (503) e indicación de reintento, **antes de reservar**, sin crear Reservation, Order ni Order ID y sin modificar el inventario. Con el sistema de colas disponible, el fallo definitivo de encolado sigue produciendo una Order `FAILED` (`ALT-006`).
  - La creación de Events responde 202 y su repetición 200 (entrada 6 y 7).
  - Precedencia de rechazos en la compra: validación; idempotencia (repetición o clave reutilizada); Event inexistente o no habilitado; Event pasado; Ticket inexistentes; colas indisponibles; Order activa existente; Ticket inexistentes o no disponibles detectados en la reserva.
- **AC afectados**: `AC-003`, `AC-016`, `AC-022`. Sugerido: "Given el sistema de colas indisponible, When un `CUSTOMER` inicia la compra, Then recibe indisponibilidad temporal sin Order ni cambios de inventario".
- **Secciones a actualizar**: §10 (Iniciar compra: resultados), §14 `ALT-006` (precisión), §15 (escenarios nuevos), §16.

## 10. ADR-021 (sucesor ADR-040) — Contrato de disponibilidad

- **Regla funcional**: la consulta de disponibilidad de un Event devuelve la cantidad de Ticket en `AVAILABLE` del Event y una página de Ticket en `AVAILABLE` (identificador, sección, fila, asiento), paginada por cursor, con tamaño máximo de 100 y filtrable por sección. Ya no lista los Ticket en otros estados. La cantidad puede tener hasta 1 segundo de antigüedad y la respuesta indica el instante de cálculo (`generatedAt`); sigue siendo informativa y la compra revalida. Un cursor inválido o una sección inexistente producen error de validación.
- **AC afectados**: `AC-010` (solo `AVAILABLE` se informa), `AC-011` (la consulta deja de mostrar el estado de cada Ticket), `AC-030`.
- **Secciones a actualizar**: §8 `CAP-002`, §9 `MF-002` paso 4, §10 (Consultar disponibilidad: entradas y salida), §11 `FR-012`, §16 `AC-010`.

## 11. AV-002 — Significado de "Event habilitado"

- **Regla funcional**: "Event habilitado" significa que su creación se completó: el Event está en `ENABLED`. El ciclo `PROVISIONING` → `ENABLED` o `FAILED` es visible para el `ADMIN`. Para el `CUSTOMER`, un Event que no está en `ENABLED` no existe. No existe operación manual para habilitar ni deshabilitar un Event, y el Event es inmutable una vez habilitado.
- **AC afectados**: `AC-002`, `AC-001`.
- **Secciones a actualizar**: §8 `CAP-001`, §11 `FR-002`, §12 `BR-022`, §5 (Event).

## 12. AV-005 — Acceso del ADMIN a consultas

- **Regla funcional**: un `ADMIN` puede listar Events, consultar su disponibilidad y consultar el estado de aprovisionamiento. No puede iniciar compras ni consultar Orders salvo que pertenezca también al grupo `CUSTOMER`.
- **AC afectados**: `AC-028`.
- **Secciones a actualizar**: §4 (Actors), §11 `FR-019`, §20.2.

## 13. AV-001 — Indicador de agotado en el listado

- **Regla funcional**: el listado de Events informa por Event un indicador de agotado (`soldOut`) y no la cantidad exacta de disponibles; la cantidad se obtiene en la consulta de disponibilidad.
- **AC afectados**: `AC-002` ("marca el agotado con disponibilidad cero").
- **Secciones a actualizar**: §10 (Consultar Events), §16 `AC-002`.

## 14. AV-003 — Márgenes temporales de la Reservation

- **Regla funcional**: ninguna compra se confirma después de `expiresAt` (diez minutos desde la creación de la Reservation), aunque sus Ticket aún no hayan sido liberados; los Ticket de una Reservation vencida se liberan como máximo 15 segundos después de `expiresAt`; no se inicia un pago cuando faltan menos de 15 segundos para `expiresAt` y esa Order se cierra por expiración. Valores configurables; los indicados son los desplegados.
- **AC afectados**: `AC-008`, `AC-009`, `AC-019`.
- **Secciones a actualizar**: §12 `BR-002`, §13 `VAL-001`, §16 `AC-008`, `AC-019`.

## 15. Cambio de restricción técnica condicionado (no funcional)

- **Origen**: ADR-017 (sucesor ADR-036).
- **Contenido**: el entorno local fija LocalStack a un tag publicado antes del 23 de marzo de 2026 (fecha desde la cual la imagen `latest` exige token, dato aportado por la revisión humana). Si ese tag no ofrece SQS con redrive, contador de recepciones y cambio de visibilidad, el respaldo es ElasticMQ, lo que **requiere revisar** `HV-010`, `TC-005`, `TC-012` y `DEL-005` (que nombran LocalStack) antes de aplicarse. Mientras no se active el respaldo, la especificación no cambia.
- **Secciones a revisar si se activa**: §22 `TC-005`, `TC-012`; §23 `DEL-005`.
