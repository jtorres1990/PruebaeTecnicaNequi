---
id: ADR-024
title: Asynchronous Event provisioning at capacity scale
status: ACCEPTED
priority: MEDIUM
supersedes: ADR-004
human_decision: CONFIRMED_WITH_CHANGE
source_ids: [FR-001, FR-002, FR-003, FR-014, BR-016, BR-021, BR-022, VAL-006, VAL-007, VAL-008, VAL-009, AC-001, AC-017, AC-026, MF-001, CAP-001]
related_adrs: [ADR-022, ADR-023, ADR-027, ADR-028, ADR-029, ADR-031, ADR-032, ADR-035, ADR-039, ADR-040]
---

# ADR-024 — Asynchronous Event provisioning at capacity scale

Reemplaza a ADR-004.

## Human decision applied

Respuesta humana vinculante a ADR-004 (CONFIRMED_WITH_CHANGE), incorporada íntegramente:

> Se rechaza la creación síncrona y se adopta la creación asíncrona con SQS y el rol worker. Con el máximo de 50.000 Ticket por Event aprobado en FG-002, la escritura del inventario no debe ejecutarse dentro de la solicitud HTTP.
>
> La solicitud de creación valida completamente los datos, crea el Event en una fase técnica `PROVISIONING` no habilitada, publica un mensaje de aprovisionamiento y responde de inmediato con el `eventId` y el estado de aprovisionamiento. Una solicitud inválida o que supere la capacidad máxima se rechaza completa sin escribir nada.
>
> El inventario se describe mediante una definición compacta (secciones, filas y asientos, más las cortesías) y no como una lista de Ticket individuales en el cuerpo. El servidor genera los `ticketId` de forma determinista a partir de esa definición, que se conserva en el Event. El mensaje solo transporta el `eventId`.
>
> El worker consume el mensaje y escribe los Ticket por lotes con concurrencia acotada. La escritura debe ser idempotente por clave y reanudable ante una reentrega. Para que una reentrega duplicada no sobrescriba Ticket de un Event ya habilitado, el worker debe tomar un lease condicional sobre el Event y comprobar que continúa en `PROVISIONING` antes de cada lote. El Event se habilita mediante una escritura condicional, junto con su registro de auditoría, solo después de comprobar que la cantidad de Ticket creados coincide con `Event.capacity`.
>
> Mientras el Event no esté habilitado no aparece en el listado, y la consulta de disponibilidad y el inicio de compra lo tratan como inexistente. El `ADMIN` puede consultar el estado de aprovisionamiento del Event (`PROVISIONING`, `ENABLED` o `FAILED`).
>
> El aprovisionamiento usa una cola propia con DLQ, separada de la cola de Orders, con entrega at-least-once. Agotados los reintentos, el Event se marca `FAILED` y el proceso de limpieza elimina sus Ticket. Un Event que permanezca en `PROVISIONING` sin progreso más allá del umbral debe ser detectado por el mismo proceso y republicado o marcado `FAILED`.
>
> ADR-004, el modelo de datos, el contrato de mensajería, OpenAPI y la especificación funcional deben propagarse de forma consistente con esta decisión. AV-002 debe responderse teniendo en cuenta que el estado de aprovisionamiento pasa a ser visible para el `ADMIN`.

Decisiones aprobadas incorporadas: `FG-002` (máximo 50.000 Ticket, rechazo completo si se supera, verificación de cantidad antes de habilitar); `AV-002` ("habilitado" = `ENABLED`, visible para el `ADMIN`, inexistente para el `CUSTOMER`, inmutable, sin operación manual); idempotencia de la creación con `Idempotency-Key` del `ADMIN` (ADR-027); política de la cola de aprovisionamiento (ADR-029); claves deterministas por Ticket (ADR-022); planificación independiente de la limpieza (ADR-028); 202 con ubicación del estado (ADR-035); consulta de estado solo `ADMIN` (ADR-032); escritura por lotes con reintento de no procesados (ADR-039).

## Context

`FR-001` y `MF-001` piden que un `ADMIN` cree Events con sus Ticket individuales; la capacidad coincide con la cantidad de Ticket (`BR-021`, `VAL-009`) y la creación es completa o se rechaza (`AC-001`, `AC-017`). Con 50.000 Ticket la escritura del inventario excede lo razonable dentro de una solicitud HTTP. El cuerpo con 50.000 Ticket individuales excede además un límite de cuerpo bajo (ADR-032).

## Options considered

### Option A — Creación síncrona en dos fases (diseño de ADR-004)

- A favor: respuesta final en la misma solicitud.
- En contra: duración proporcional a la capacidad; cuerpo de varios MB. Rechazada por la decisión humana.

### Option B — Creación asíncrona con cola propia, worker con lease y habilitación verificada

- A favor: respuesta inmediata; aprovisionamiento reanudable y aislado de la ruta de compra; cuerpo compacto.
- En contra: estado intermedio visible para el `ADMIN`; una cola, un consumidor y un proceso de limpieza adicionales.

### Option C — Creación asíncrona mediante un proceso periódico que sondea Events en `PROVISIONING`, sin cola

- A favor: sin cola adicional.
- En contra: latencia de inicio igual a la periodicidad; sin reintento con backoff ni DLQ; la decisión humana exige cola propia con DLQ.

## Decision

Se adopta la **Option B**.

### 1. Solicitud de creación (`API-001`, rol `api`, `CMP-003`)

1. Autoridad `ADMIN`; `Idempotency-Key` obligatoria vinculada al sujeto del `ADMIN` (ADR-027).
2. Validación completa antes de escribir: nombre y lugar no vacíos (`VAL-006`); `startsAt` futuro en UTC (`VAL-007`); capacidad entera entre 1 y 50.000 (`VAL-008`, `FG-002`); definición compacta válida; capacidad igual a la cantidad de ubicaciones derivadas de la definición, incluidas las cortesías (`VAL-009`, `BR-021`). Cualquier incumplimiento rechaza todo sin escribir nada (`AC-017`).
3. Transacción `AP-001` (3 items): Event en `PROVISIONING` (con la definición, `availabilityShards`, `totalBatches`, indexado en `GSI1` bajo `EVENTS#PROVISIONING`), registro de idempotencia de creación y auditoría `EVENT_PROVISIONING_REQUESTED` con el `ADMIN` como actor.
4. Publicación de `MSG-002` en la cola de aprovisionamiento con el mismo presupuesto que la publicación de Orders (3 intentos, 500 ms por intento, 2 s en total, ADR-026). Si la publicación falla, el Event queda en `PROVISIONING` sin mensaje y lo recupera la detección de estancados (punto 5); la respuesta no cambia.
5. Respuesta 202 con `eventId`, `provisioningStatus = PROVISIONING` y la ubicación de `API-006`.

### 2. Definición compacta del inventario

- Secciones con código único; cada sección con filas de etiqueta única; cada fila con una cantidad de asientos numerados desde 1.
- Cortesías como rangos de asientos `(sección, fila, desde, hasta)` que deben existir en la definición y no solaparse.
- `ticketId = <sección>-<fila>-<asiento>`, generado por el servidor; único dentro del Event.
- Límites de tamaño (configurables, para mantener el cuerpo y el item Event acotados): hasta 100 secciones, hasta 2.000 filas en total, hasta 1.000 asientos por fila, hasta 500 rangos de cortesía.
- La definición se conserva en el Event y es inmutable.

### 3. Aprovisionamiento (rol `worker`, `CMP-022`, consumidor `CMP-025`)

1. Recibe `MSG-002` (solo `eventId`).
2. Lee el Event. Si está en `ENABLED` o `FAILED`, elimina el mensaje sin efectos.
3. Toma el lease condicional (`AP-024`): condición `provisioningStatus = PROVISIONING` y (sin lease, lease vencido o propio). Lease de 60 s. Si otro worker lo posee, pospone la visibilidad del mensaje hasta el fin del lease.
4. Genera las claves de Ticket desde la definición en orden determinista, divididas en **lotes de aprovisionamiento** de 100 Ticket (configurable), numerados.
5. Antes de cada lote, actualización condicional del Event (`AP-024`): exige `provisioningStatus = PROVISIONING` y lease propio; renueva el lease, registra `provisionedBatches` (lotes confirmados) y `lastProgressAtMs`. Si la condición falla, se detiene sin escribir.
6. Escribe el lote con escritura por lotes del almacén, hasta 4 solicitudes en paralelo, reintentando los items no procesados (`AP-002`). Cada Ticket nace en `AVAILABLE` (indexado en `GSI2`) o en `COMPLIMENTARY` (`BR-016`). La escritura es idempotente por clave.
7. Una reentrega reanuda desde `provisionedBatches`; reescribir un lote ya escrito es inocuo mientras el Event está en `PROVISIONING`.
8. Mientras progresa, el consumidor extiende la visibilidad del mensaje (heartbeat, ADR-029).
9. Verificación (`AP-025`): lectura fuertemente consistente por lotes de todas las claves generadas; exige que existan exactamente `capacity` Ticket y que cada uno tenga el estado inicial que le asigna la definición. Si faltan, reescribe los faltantes y verifica de nuevo (hasta 3 veces); si persiste, es un fallo transitorio del mensaje.
10. Habilitación (`AP-003`, 2 items): Event → `ENABLED`, `enabledAt`, reindexado bajo `EVENTS#ENABLED`, sin lease; condición `provisioningStatus = PROVISIONING` y lease propio; auditoría `EVENT_ENABLED` con las cantidades en `AVAILABLE` y `COMPLIMENTARY`.
11. Elimina el mensaje.

### 4. Fallo del aprovisionamiento

- Errores transitorios: el mensaje no se elimina y se reintenta con backoff (ADR-029).
- En la última recepción permitida, si vuelve a fallar: transición `AP-026` (2 items) Event → `FAILED` con causa `PROVISIONING_FAILED`, reindexado bajo `EVENTS#FAILED`, auditoría `EVENT_PROVISIONING_FAILED`; el mensaje pasa a la DLQ.

### 5. Limpieza y detección de estancados (rol `worker`, `CMP-015`, planificación propia, ADR-028)

- Cada 60 s (configurable) consulta `GSI1` bajo `EVENTS#PROVISIONING` (`AP-018`) y selecciona los Events sin progreso desde hace más de 3 minutos (`lastProgressAtMs`).
  - Si se han republicado menos de 3 veces: incrementa el contador de forma condicional (`AP-033`) y republica `MSG-002`.
  - En otro caso: `AP-026` (→ `FAILED`).
- Consulta `GSI1` bajo `EVENTS#FAILED` y purga los Ticket del Event (`AP-027`): genera las claves desde la definición, las elimina por lotes y marca `ticketsPurgedAt`, que retira el Event del índice. El item Event permanece en `FAILED` y su estado sigue consultable por el `ADMIN`.

### 6. Visibilidad y ciclo de vida

| `provisioningStatus` | Listado (`API-002`) | Disponibilidad (`API-003`) | Compra (`API-004`) | Estado de aprovisionamiento (`API-006`, solo `ADMIN`) |
|---|---|---|---|---|
| `PROVISIONING` | No aparece | `EVENT_NOT_FOUND` | `EVENT_NOT_FOUND` | Visible, con progreso |
| `ENABLED` | Aparece si es futuro | Responde (también si es pasado, `FG-005`) | Permitida si es futuro | Visible |
| `FAILED` | No aparece | `EVENT_NOT_FOUND` | `EVENT_NOT_FOUND` | Visible, con causa |

- Transiciones: `PROVISIONING` → `ENABLED` o `PROVISIONING` → `FAILED`. Ambas finales. No hay operación manual para habilitar ni deshabilitar (`AV-002`).
- `provisioningStatus` es el ciclo de vida técnico del Event aprobado en `AV-002`; no es un estado de Ticket ni de Order.
- Después de `ENABLED`, el Event, su definición y la composición del inventario son inmutables; no existe operación que añada Ticket ni que lleve un Ticket a `COMPLIMENTARY` (`AC-026`).

## Rationale

- El cambio de visibilidad sigue siendo una escritura condicional de un solo item (más su auditoría), lo que observa cualquier otro actor (`AC-001`).
- Las claves deterministas hacen la escritura idempotente y reanudable sin estado adicional, y permiten verificar y purgar sin índice por Event.
- El lease con comprobación antes de cada lote impide que una reentrega sobrescriba Ticket de un Event ya habilitado: la reentrega no obtiene el lease porque el Event ya no está en `PROVISIONING`.
- La verificación por lectura consistente de todas las claves comprueba la cantidad real de Ticket creados, no un contador.

## Consequences

- `API-001` responde 202 y la creación es observable en dos pasos; la especificación debe actualizarse (`ticketing.functional-clarifications.v1.md`).
- La verificación lee cada Ticket una vez con lectura fuerte: coste único por Event.
- Un Event `FAILED` no se elimina: permanece consultable para el `ADMIN`; el `ADMIN` crea uno nuevo con otra `Idempotency-Key`.
- Si la publicación inicial falla, el aprovisionamiento comienza tras el umbral de estancado (3 minutos).

## Risks and mitigations

| Risk | Mitigation |
|---|---|
| `RISK-011` aprovisionamiento interrumpido | Invisible por diseño; reanudación por reentrega; detección de estancados; `FAILED` y purga |
| `RISK-016` un worker pausado más allá del lease escribe un lote después de la habilitación | Lease de 60 s frente a lotes de milisegundos; comprobación antes de cada lote; la habilitación exige el lease propio; prueba determinista de reentrega (ADR-038); riesgo residual declarado |
| Aprovisionamiento que compite con las compras por capacidad | Concurrencia baja (1 mensaje por instancia, 4 escrituras por lotes en paralelo) |

## Items to verify

| Item | Qué comprobar | Impacto si difiere |
|---|---|---|
| Escritura por lotes | Se asumen 25 items por solicitud y devolución de items no procesados | Cambia el número de solicitudes por lote, no el diseño |
| Lectura por lotes con consistencia fuerte | Se asumen 100 claves por solicitud y soporte de lectura consistente, también en DynamoDB Local | Si DynamoDB Local no la soporta, la verificación en local usa lecturas individuales consistentes |
| Tamaño del item Event con la definición máxima | Inferior al límite de item | Reducir los límites de la definición |
| Duración del aprovisionamiento de 50.000 Ticket en DynamoDB Local y en AWS | Medición | Ajustar paralelismo y tamaño de lote |

## Depends on

- `FG-002`, `AV-002` (resueltos).
