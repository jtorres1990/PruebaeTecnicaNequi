---
id: ADR-004
title: Event creation at capacity scale
status: PROPOSED
priority: MEDIUM
source_ids: [FR-001, FR-002, FR-003, BR-016, BR-021, BR-022, VAL-006, VAL-007, VAL-008, VAL-009, AC-001, AC-017, AC-026, MF-001]
related_adrs: [ADR-001, ADR-002, ADR-012, ADR-021]
---

# ADR-004 — Event creation at capacity scale

## Context

Un `ADMIN` crea un Event con todos sus Ticket individuales (`FR-001`, `MF-001`). La capacidad debe coincidir con la cantidad de Ticket (`BR-021`, `VAL-009`) y el resultado es un Event con inventario consistente o un rechazo completo (`AC-001`, `AC-017`).

Un Event puede tener más Ticket de los que caben en una sola operación atómica de DynamoDB. La creación ocurre por tanto en varias escrituras, y hay que decidir cómo evitar que se observe un inventario parcial y qué ocurre si falla a mitad de camino.

`HV-020` dejó a arquitectura la posibilidad de un atributo de habilitación; la especificación habla de "Events futuros habilitados" (`FR-002`).

## Options considered

### Option A — Creación síncrona en dos fases con atributo técnico de habilitación

1. Validar toda la solicitud. 2. Crear el Event con `enabled = false` y un `eventId` generado por el servidor. 3. Escribir los Ticket por lotes. 4. Habilitar el Event con una escritura condicional. Solo entonces se responde con el `eventId`.

- A favor: el inventario parcial nunca es observable ni comprable; la respuesta síncrona coincide con §10 de la especificación; no requiere componentes nuevos.
- En contra: la duración de la solicitud crece con la capacidad; un fallo deja items huérfanos que hay que limpiar.

### Option B — Creación asíncrona con aceptación inmediata

Responder de inmediato y construir el inventario en segundo plano, con una operación de consulta del progreso.

- A favor: tiempo de respuesta constante; admite capacidades muy grandes.
- En contra: exige una operación de consulta de progreso y un resultado "en construcción" que la especificación no sustenta; añade una cola o tarea más.

### Option C — Limitar la capacidad a lo que cabe en una transacción

- A favor: creación atómica real.
- En contra: capacidad máxima de unas decenas de Ticket, incompatible con el dominio (conciertos, deportes).

## Decision

Se adopta la **Option A**.

1. **Validación completa antes de escribir** (`VAL-006` a `VAL-009`): nombre y lugar no vacíos; instante futuro en UTC; capacidad entera positiva y no superior al máximo (`FG-002`); cantidad de Ticket igual a la capacidad; `ticketId` únicos dentro de la solicitud y con formato válido. Cualquier incumplimiento rechaza la creación completa sin escribir nada (`AC-017`).
2. **Identidad del Ticket**: el `ADMIN` suministra para cada Ticket un `ticketId` (etiqueta de la ubicación, única dentro del Event) y la marca de cortesía. La identidad global es `(eventId, ticketId)`.
3. **Fase 1** (`AP-001`): crear el Event con `enabled = false`, indexado bajo `EVENTS#PROVISIONING`.
4. **Fase 2** (`AP-002`): escribir los Ticket por lotes con concurrencia acotada, reintentando los items no procesados. Cada Ticket nace en `AVAILABLE` o en `COMPLIMENTARY` (`BR-016`).
5. **Fase 3** (`AP-003`): habilitar el Event (`enabled = true`, reindexado bajo `EVENTS#ENABLED`) junto con su registro de auditoría de creación, condicionado a `enabled = false`.
6. **No exposición de inventario parcial**: mientras `enabled = false`, el Event no aparece en el listado, la consulta de disponibilidad y el inicio de compra lo tratan como inexistente, y su `eventId` no ha sido devuelto a nadie.
7. **Fallo a mitad de camino**: la solicitud responde error sin `eventId`. Se intenta una compensación inmediata (borrado de lo escrito). Si la compensación no se completa, un proceso de limpieza (`CMP-015`) localiza los Events en provisioning con antigüedad superior a 15 minutos (`AP-018`) y elimina sus Ticket y el Event (`AP-019`).
8. **`enabled` es un atributo técnico**, no un estado de negocio. No existe operación para deshabilitar o rehabilitar un Event. Después de habilitado, el Event y la composición de su inventario son inmutables: no existe operación que añada Ticket ni que lleve un Ticket a `COMPLIMENTARY` (`AC-026`).

## Rationale

- Separar "escrito" de "habilitado" convierte una creación de muchas escrituras en un cambio de visibilidad atómico de un solo item, que es lo que observa cualquier otro actor (`AC-001`).
- El `eventId` aleatorio generado en servidor hace que los Ticket de un Event en construcción no sean direccionables, por lo que la compra no necesita incluir el Event en su transacción (`ADR-002`).
- La opción B obligaría a añadir una operación y un resultado intermedio no sustentados por la especificación. La opción C no sirve al dominio.

## Consequences

- La solicitud de creación dura en proporción a la capacidad; por eso se requiere un máximo (`FG-002`) y un límite de tamaño de cuerpo acorde.
- Un reintento del `ADMIN` tras un fallo crea un Event nuevo; el intento fallido nunca fue visible. La creación de Events no es idempotente ante una respuesta perdida tras un éxito: puede producir un Event duplicado. Es una limitación declarada; `FR-017` y §18 de la especificación no incluyen esta operación entre las idempotentes.
- El proceso de limpieza es un componente adicional de baja frecuencia en el worker.
- "Habilitado" se interpreta como "creación completada" (`AV-002`).

## Risks and mitigations

| Risk | Mitigation |
|---|---|
| `RISK-011` items huérfanos tras un fallo | Invisibles por diseño; compensación inmediata más limpieza periódica |
| Solicitud demasiado larga para capacidades altas | Máximo de capacidad (`FG-002`); concurrencia acotada en la escritura por lotes |
| La limpieza elimina un Event que aún se está creando | Umbral de antigüedad muy superior a la duración máxima de una creación; borrado condicionado a `enabled = false`; la habilitación falla si el Event ya no existe |

## Items to verify

| Item | Qué comprobar | Impacto si difiere |
|---|---|---|
| Tamaño máximo de la escritura por lotes | Items por solicitud (se asume 25) | Cambia el tamaño de bloque, no el diseño |
| Límite de tamaño de cuerpo en memoria del servidor reactivo | Valor por defecto y propiedad que lo configura en Spring Boot 4.x | Debe elevarse para admitir la capacidad máxima de `FG-002` |
| Duración de la creación con la capacidad máxima en DynamoDB Local | Medición real | Si es excesiva, reducir el máximo de `FG-002` o reconsiderar la Option B |

## Depends on

- `FG-002`: capacidad máxima por Event.
- `AV-002`: interpretación de "habilitado".
