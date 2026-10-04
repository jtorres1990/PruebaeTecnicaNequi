---
id: ADR-027
title: Idempotency for purchases, Event creation, messages and payments
status: ACCEPTED
priority: HIGH
supersedes: ADR-007
human_decision: CONFIRMED_WITH_CHANGE
source_ids: [FR-017, BR-019, BR-020, NFR-015, AC-023, AC-024, AC-025, ALT-003, ALT-004, ERR-004, TC-010, FR-001, EVAL-008]
related_adrs: [ADR-008, ADR-022, ADR-023, ADR-024, ADR-025, ADR-026, ADR-029, ADR-030, ADR-032, ADR-035]
---

# ADR-027 — Idempotency for purchases, Event creation, messages and payments

Reemplaza a ADR-007.

## Human decision applied

Respuesta humana vinculante a ADR-007 (CONFIRMED_WITH_CHANGE), incorporada íntegramente:

> Se confirma la idempotencia por identidad explícita de cada operación, persistida junto con su efecto. La solicitud de compra exige la cabecera `Idempotency-Key`, vinculada al `customerId` derivado del JWT y registrada en la misma transacción que la Order, con un hash del contenido y una vigencia mínima de 24 horas. El consumidor usa el `orderId` y el estado de la Order como identidad del mensaje. Una Order admite un único PaymentAttempt activo, protegido por lease y enviado al proveedor como clave de idempotencia. Toda transición exige la Order en `CREATED`. Se descartan la tabla de mensajes procesados y la cola FIFO.
>
> Una repetición con la misma clave y el mismo contenido retorna la Order ya creada en su estado actual, sin repetir Reservation ni efectos. La misma clave con contenido distinto se rechaza sin efectos. La única acción adicional permitida en una repetición es republicar el mensaje de una Order en `CREATED` sin `enqueuedAt`, conforme a ADR-006. Se añaden dos cambios.
>
> Carrera con la misma clave. Cuando dos solicitudes simultáneas usan la misma `Idempotency-Key`, solo una transacción se confirma. La solicitud que pierde no debe responder indisponibilidad aunque su transacción se haya cancelado también por el estado de los Ticket: debe releer el registro de idempotencia con lectura fuertemente consistente y responder como repetición, con la misma Order, o con el rechazo por clave reutilizada si el contenido difiere. La existencia del registro de idempotencia tiene prioridad sobre cualquier otra causa de cancelación.
>
> Creación de Events. La creación de un Event también debe ser idempotente. La solicitud exige `Idempotency-Key`, vinculada a la identidad del `ADMIN` y registrada en la misma escritura transaccional que crea el Event en `PROVISIONING`, con hash del contenido. Una repetición retorna el mismo `eventId` con su estado de aprovisionamiento actual y no crea un segundo Event ni publica un segundo aprovisionamiento. Esto es necesario porque no existe operación para eliminar un Event y un duplicado quedaría listado y vendible.

Decisiones aprobadas incorporadas: `FG-004` (la repetición de una solicitud rechazada se reevalúa; los rechazos no se persisten); republicación conforme a ADR-026 (sucesor de ADR-006); 200 con cabecera de repetición y `IDEMPOTENCY_KEY_REUSED` también en la creación de Events (ADR-035); vinculación de la clave de creación al sujeto del `ADMIN` (ADR-032); cancelación idempotente por `paymentAttemptId` (`FG-003`, ADR-030); una repetición no es una segunda compra a efectos del bloqueo de Order activa (ADR-032).

## Context

La idempotencia es obligatoria para solicitudes repetidas de compra, mensajes duplicados, invocaciones repetidas del pago y Orders terminales (`FR-017`, §18). Una Order admite un único PaymentAttempt activo (`BR-020`). La creación de Events se suma por decisión humana.

## Options considered

### Option A — Identidad explícita por operación, persistida junto con el efecto, más transiciones condicionales

- A favor: registro y efecto atómicos; cada frente usa su identidad natural.
- En contra: obliga al cliente a enviar una cabecera; la idempotencia del pago depende del proveedor.

### Option B — Tabla de mensajes y solicitudes procesadas

- En contra: el identificador de mensaje de SQS cambia en cada republicación; comprobación y efecto no atómicos. Descartada.

### Option C — Cola FIFO con deduplicación del servicio

- En contra: `TC-005` fija SQS Standard. Descartada.

## Decision

Se adopta la **Option A**.

### 1. Inicio de compra (`API-004`)

| Aspecto | Decisión |
|---|---|
| Identidad | (`customerId` = sujeto del JWT, `Idempotency-Key` obligatoria de 16 a 64 caracteres `[A-Za-z0-9_-]`) |
| Almacenamiento | Item `IDEM#<customerId>#<key>` creado en la transacción de reserva (ADR-023), con `orderId` y hash del contenido (`eventId` y `ticketIds` ordenados) |
| Vigencia | Mínimo 24 h mediante TTL; un registro presente se respeta aunque su vigencia haya pasado |
| Detección | Lectura fuertemente consistente antes de cualquier otra comprobación de negocio (`AP-009`) y condición "no existe" dentro de la transacción |
| Repetición, mismo contenido | 200 con cabecera `Idempotency-Replayed: true` y la Order en su estado actual. Única acción adicional: si la Order está en `CREATED`, sin `enqueuedAt` y sin cuarentena, republicar `MSG-001` y marcar `enqueuedAt` (ADR-026); si el circuito de publicación está abierto, se omite y el barrido la recogerá |
| Misma clave, contenido distinto | 422 `IDEMPOTENCY_KEY_REUSED`, sin efectos |
| Carrera con la misma clave | Ante cualquier cancelación de la transacción, relectura fuertemente consistente del registro; si existe, se responde como repetición o como clave reutilizada. Prevalece sobre `TICKETS_UNAVAILABLE`, `UNKNOWN_TICKETS`, `ACTIVE_ORDER_EXISTS` y el conflicto transaccional |
| Repetición de una solicitud rechazada | Ningún rechazo síncrono (`TICKETS_UNAVAILABLE`, `UNKNOWN_TICKETS`, `ACTIVE_ORDER_EXISTS`, `EVENT_NOT_ON_SALE`, `EVENT_NOT_FOUND`, `VALIDATION_ERROR`, `SERVICE_UNAVAILABLE`) persiste nada. La repetición se procesa como solicitud nueva y puede crear una Order; en ese caso la clave queda registrada con esa Order (`FG-004`) |
| Relación con el bloqueo de Order activa | Una repetición con la misma clave se resuelve en el paso de idempotencia, antes de la transacción, y nunca produce `ACTIVE_ORDER_EXISTS` (ADR-032) |

### 2. Creación de Event (`API-001`)

| Aspecto | Decisión |
|---|---|
| Identidad | (`adminSubject` = sujeto del JWT del `ADMIN`, `Idempotency-Key` con el mismo formato) |
| Almacenamiento | Item `IDEMEVT#<adminSubject>#<key>` creado en la transacción `AP-001` junto con el Event en `PROVISIONING` y su auditoría, con `eventId` y hash del contenido normalizado (nombre, lugar, `startsAt`, capacidad, definición) |
| Vigencia | Mínimo 24 h mediante TTL |
| Detección | Lectura fuertemente consistente (`AP-022`) antes de validar el negocio y condición "no existe" en la transacción |
| Repetición, mismo contenido | 200 con `Idempotency-Replayed: true`, mismo `eventId` y su `provisioningStatus` actual. No crea un segundo Event ni publica un segundo `MSG-002`. Si el mensaje original se perdió, lo recupera la detección de estancados (ADR-024) |
| Misma clave, contenido distinto | 422 `IDEMPOTENCY_KEY_REUSED` |
| Carrera con la misma clave | Igual que en la compra: relectura del registro ante cancelación |

### 3. Mensaje SQS duplicado (`MSG-001`)

| Aspecto | Decisión |
|---|---|
| Identidad | `orderId`; el identificador de mensaje de SQS no se usa |
| Almacenamiento | Item Order: `status`, `paymentAttemptId`, `paymentLeaseOwner`, `paymentLeaseUntilMs`, `quarantinedAt` |
| Respuesta | Order terminal o en cuarentena: eliminar sin efectos. PaymentAttempt con lease vigente de otro consumidor: no eliminar, posponer visibilidad. Lease vencido: reclamar y reanudar el mismo intento (ADR-029) |

### 4. Mensaje de aprovisionamiento duplicado (`MSG-002`)

Identidad `eventId`; almacenamiento en el Event (`provisioningStatus`, lease, `provisionedBatches`). Event `ENABLED` o `FAILED`: eliminar sin efectos. Lease ajeno vigente: posponer. Reentrega: reanudar desde `provisionedBatches` (ADR-024).

### 5. PaymentAttempt único activo

| Aspecto | Decisión |
|---|---|
| Identidad | `paymentAttemptId = <orderId>-<paymentAttemptNo>`; en esta versión `paymentAttemptNo = 1` |
| Apertura | El inicio de pago exige que no exista `paymentAttemptId` (ADR-025) |
| Ante el proveedor | `paymentAttemptId` como clave de idempotencia de la autorización y de la cancelación (ADR-030) |
| Reintentos transitorios | Reutilizan el mismo `paymentAttemptId` (`ALT-004`) |
| Repetición | Se conserva el intento existente (`AC-024`) |

### 6. Order terminal

Toda transición exige `status = CREATED` (ADR-025). Una repetición conserva el estado terminal sin pago, venta, inventario ni auditoría adicionales (`AC-025`). La única escritura posible sobre una Order terminal es técnica: la gestión de su reverso de pago (ADR-025), que no cambia su estado.

## Rationale

- Registrar la clave en la misma transacción que el efecto elimina la ventana entre comprobar y ejecutar.
- Releer el registro ante cualquier cancelación hace determinista la carrera con la misma clave.
- La idempotencia de creación evita Events duplicados, que no podrían eliminarse y quedarían listados y vendibles.

## Consequences

- `API-001` y `API-004` exigen `Idempotency-Key`.
- Cada compra y cada creación añaden una lectura consistente y un item con TTL.
- Pasadas 24 h y borrado el registro, una repetición de creación crearía un Event nuevo (`RISK-022`).

## Risks and mitigations

| Risk | Mitigation |
|---|---|
| Crecimiento del almacén de idempotencia | TTL; el registro solo se crea si la operación tiene éxito; limitación por tasa (ADR-032) |
| `RISK-022` duplicado de Event tras la vigencia de la clave | Vigencia de 24 h documentada; el `ADMIN` consulta el estado antes de repetir |
| Dos consumidores invocan el pago para el mismo intento tras vencer un lease | Proveedor idempotente por `paymentAttemptId`; aplicación condicional del resultado |

## Items to verify

| Item | Qué comprobar | Impacto si difiere |
|---|---|---|
| Demora del borrado por TTL y soporte en DynamoDB Local | Documentación y prueba | Ninguno funcional |

## Depends on

- `FG-003`, `FG-004` (resueltos).
