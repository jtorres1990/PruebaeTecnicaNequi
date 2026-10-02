---
id: ADR-007
title: Idempotency
status: PROPOSED
priority: HIGH
source_ids: [FR-017, BR-019, BR-020, NFR-015, AC-023, AC-024, AC-025, ALT-003, ALT-004, ERR-004, TC-010, EVAL-008]
related_adrs: [ADR-002, ADR-005, ADR-006, ADR-008, ADR-010, ADR-011, ADR-013]
---

# ADR-007 — Idempotency

## Context

La idempotencia es obligatoria para solicitudes repetidas de inicio de compra, mensajes duplicados de SQS Standard, invocaciones repetidas del flujo de pago y Orders terminales (`FR-017`, §18 de la especificación). Una Order admite como máximo un PaymentAttempt activo (`BR-020`) y un reintento autorizado debe tener identidad propia.

Se requiere un mecanismo por frente, con identidad de la operación, almacenamiento, vigencia y respuesta ante una repetición.

## Options considered

### Option A — Identidad explícita por operación, persistida junto con el efecto, más transiciones condicionales

Idempotency key del cliente registrada en la misma transacción que la Order; el consumidor se apoya en el estado de la Order y en condiciones; el pago usa la identidad del PaymentAttempt como clave de idempotencia ante el proveedor.

- A favor: el registro de idempotencia y el efecto son atómicos; no hay tabla de mensajes procesados; cada frente usa la identidad natural de su operación.
- En contra: exige que el cliente envíe una cabecera; la protección ante el proveedor depende de que este respete la clave de idempotencia.

### Option B — Tabla de mensajes y solicitudes procesadas

Registrar cada identificador de mensaje SQS y cada solicitud en un almacén de deduplicación consultado antes de procesar.

- A favor: patrón genérico.
- En contra: el identificador de mensaje SQS cambia en cada republicación, por lo que no identifica la operación de negocio; la comprobación previa y el efecto no son atómicos; añade escrituras y un almacén con vigencia propia.

### Option C — Cola FIFO con deduplicación del servicio

- A favor: deduplicación sin código.
- En contra: `TC-005` y `HV-010` fijan SQS Standard y exigen idempotencia de aplicación. Se descarta por violar la especificación.

## Decision

Se adopta la **Option A**.

### 1. Solicitud repetida de inicio de compra

| Aspecto | Decisión |
|---|---|
| Identidad | Par (`customerId` derivado del JWT, cabecera `Idempotency-Key` obligatoria generada por el cliente, entre 16 y 64 caracteres de un alfabeto restringido) |
| Almacenamiento | Idempotency record en la tabla, creado en la misma transacción que la Order (`ADR-002`). Contiene `orderId` y un hash del contenido de la solicitud (`eventId` más `ticketId` ordenados) |
| Vigencia | Mínimo 24 horas mediante TTL. Un registro que todavía existe se respeta aunque su vigencia haya pasado |
| Detección | Lectura consistente del registro antes de cualquier otra comprobación de negocio (`AP-009`); además, la condición "no existe" dentro de la transacción resuelve la carrera entre dos solicitudes simultáneas con la misma clave |
| Respuesta ante repetición con el mismo contenido | Se retorna la Order ya creada en su estado actual, sin repetir Reservation, encolado normal ni efectos. La respuesta se marca como repetición |
| Respuesta ante la misma clave con contenido distinto | Rechazo sin efectos (`IDEMPOTENCY_KEY_REUSED`): no es la misma operación |
| Repetición de una solicitud que fue rechazada síncronamente | No existe registro, porque el rechazo no persiste nada (`HC-001`). La repetición se reevalúa. Vacío funcional `FG-004` |
| Cabecera ausente o inválida | Error de validación |

Única acción adicional permitida en una repetición: si la Order está en `CREATED` y no tiene `enqueuedAt`, se reintenta la publicación (`ADR-006`).

### 2. Mensaje SQS duplicado

| Aspecto | Decisión |
|---|---|
| Identidad | `orderId` contenido en el mensaje. El identificador de mensaje de SQS no se usa como identidad |
| Almacenamiento | El propio item Order: `status`, `paymentAttemptId`, `paymentLeaseOwner`, `paymentLeaseUntilMs` |
| Vigencia | La vida de la Order |
| Respuesta ante repetición | Order terminal: se confirma el mensaje sin efectos. Order `CREATED` con PaymentAttempt activo y lease vigente de otro consumidor: no se procesa ni se elimina el mensaje, se pospone su visibilidad. Order `CREATED` con lease vencido: se reclama el lease y se reanuda el mismo PaymentAttempt (`ADR-010`) |

### 3. PaymentAttempt único activo

| Aspecto | Decisión |
|---|---|
| Identidad | `paymentAttemptId = <orderId>-<paymentAttemptNo>` |
| Almacenamiento | Atributos del item Order; el inicio de pago exige que no exista `paymentAttemptId` (`ADR-005`), por lo que solo un consumidor puede abrir el intento |
| Ante el proveedor | El `paymentAttemptId` se envía como clave de idempotencia; el Payment Mock devuelve el mismo resultado para la misma clave (`ADR-011`) |
| Reintentos transitorios | Reutilizan el mismo `paymentAttemptId`: es la misma operación (`ALT-004`) |
| Reintento autorizado con identidad propia | El modelo lo admite incrementando `paymentAttemptNo`. Esta versión no abre un segundo intento: `paymentAttemptNo` es siempre 1 |
| Vigencia | La vida de la Order |
| Respuesta ante repetición | Se conserva el intento existente; no se abre otro (`AC-024`) |

### 4. Reprocesamiento de Order terminal

| Aspecto | Decisión |
|---|---|
| Identidad | `orderId` |
| Almacenamiento | `status` de la Order |
| Mecanismo | Toda transición exige `status = CREATED` (`ADR-005`). El consumidor y el proceso de expiración consultan el estado antes de actuar y tratan el fallo de condición como "ya resuelto" |
| Respuesta ante repetición | Se conserva el estado terminal; no hay pago, venta, cambio de inventario ni registro de auditoría adicionales (`AC-025`) |

## Rationale

- Registrar la clave en la misma transacción que el efecto elimina la ventana entre "comprobar" y "ejecutar": nunca hay efecto sin registro ni registro sin efecto.
- Vincular la clave al `customerId` impide que un usuario reutilice o adivine claves de otro (`EVAL-008`).
- El hash del contenido evita que una clave reutilizada por error devuelva una Order que no corresponde a lo solicitado.
- El estado de la Order es la identidad natural del trabajo asíncrono: sobrevive a republicaciones y a entregas duplicadas sin almacén adicional.
- El lease distingue "otro consumidor está procesando" de "el consumidor anterior murió", lo que permite cumplir `AC-024` sin dejar Orders atascadas.

## Consequences

- El cliente está obligado a enviar `Idempotency-Key` en `API-004`.
- Cada compra añade una lectura consistente y un item con TTL.
- Un lease mal dimensionado retrasa la reanudación tras una caída; su duración se alinea con el visibility timeout (`ADR-010`).
- La idempotencia del pago depende del contrato del proveedor; con un proveedor real que no la ofrezca habría que rediseñar este frente.

## Risks and mitigations

| Risk | Mitigation |
|---|---|
| Crecimiento del almacén de idempotencia por abuso | TTL, rate limiting por identidad (`ADR-013`), el registro solo se crea cuando la reserva tiene éxito |
| Dos consumidores invocan el pago para el mismo intento tras vencer un lease | El proveedor es idempotente por `paymentAttemptId`; la aplicación del resultado es condicional |
| Reloj desalineado entre instancias al evaluar el lease | Lease holgado respecto del tiempo de procesamiento; `RISK-006` |

## Items to verify

| Item | Qué comprobar | Impacto si difiere |
|---|---|---|
| Demora del borrado por TTL | Demora típica en el servicio y soporte en DynamoDB Local | Ninguno funcional: solo afecta almacenamiento |
| Idempotencia nativa de la escritura transaccional por token de cliente | Ventana de validez | Solo se usa para reintentos del SDK; no es el mecanismo de negocio |

## Depends on

- `FG-004`: tratamiento de la repetición de una solicitud rechazada síncronamente.
- `FG-003`: qué ocurre con un PaymentAttempt cuyo resultado es desconocido cuando la Order se cierra.
