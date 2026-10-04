---
id: ADR-030
title: Payment Mock as an independent project with mandatory cancellation
status: ACCEPTED
priority: MEDIUM
supersedes: ADR-011
human_decision: CONFIRMED_WITH_CHANGE
source_ids: [TC-017, FR-015, FR-017, BR-003, BR-020, AC-005, AC-019, AC-020, AC-021, AC-023, AC-024, ALT-004, ALT-005, ERR-008, TC-006, TC-012]
related_adrs: [ADR-008, ADR-025, ADR-027, ADR-029, ADR-032, ADR-034, ADR-035, ADR-036, ADR-038]
---

# ADR-030 — Payment Mock as an independent project with mandatory cancellation

Reemplaza a ADR-011. Contrato HTTP: `architecture/payment-mock.openapi.v1.yaml`.

## Human decision applied

Respuesta humana vinculante a ADR-011 (CONFIRMED_WITH_CHANGE), incorporada íntegramente:

> Se confirma el Payment Mock como servicio HTTP propio, idempotente por `paymentAttemptId`, con selección determinista del resultado mediante reglas configuradas en el propio mock a través de una API de control, con los comportamientos de aprobación, rechazo, error definitivo, errores transitorios seguidos de un resultado final y latencia añadida, estado en memoria y contenedor propio en Docker Compose. Se descartan el servidor de stubs genérico y la selección por un campo de la solicitud de compra. Se añaden cuatro cambios.
>
> Proyecto independiente. Conforme a ADR-015, el mock es un proyecto independiente dentro del mismo repositorio, con su propio build, Dockerfile y pruebas, y su contrato HTTP se documenta en un OpenAPI propio. No es un módulo del build de `ticketing`.
>
> Cancelación obligatoria. Conforme a FG-003, el mock expone la operación de cancelación de un PaymentAttempt. Es idempotente por `paymentAttemptId` y responde correctamente cualquiera que sea el estado del intento: aprobado, rechazado, inexistente o aún en tránsito.
>
> Cancelación anticipada. Si la cancelación llega antes que el cobro, el mock la registra y rechaza cualquier cobro posterior con el mismo `paymentAttemptId`.
>
> Inspección de cancelaciones. La API de control permite consultar también las cancelaciones recibidas por intento, para verificar en pruebas que un pago aprobado sin compra se reversa exactamente una vez.

Decisiones aprobadas incorporadas: `AV-004` (reglas en el mock, sin campos nuevos en la compra; la colección de solicitudes configura el mock antes de cada escenario de rechazo o fallo); `FG-003` (cancelación con registro anticipado); proyecto independiente sin código compartido (ADR-034); circuit breaker en el adaptador de pago (ADR-035); despliegue solo en entornos no productivos (ADR-037); demostración de fallos transitorios para el circuito (ADR-035, ADR-038).

## Context

`TC-017` exige un servicio simulado independiente con respuestas deterministas. El consumidor lo invoca tras reservar (`FR-015`) y aplica aprobación (`AC-019`), rechazo (`AC-020`) o fallo (`AC-021`). `FG-003` exige poder reversar un pago aprobado o de resultado desconocido cuya Order termina sin confirmarse.

## Options considered

### Option A — Servicio HTTP propio, proyecto independiente, reglas por API de control

- A favor: idempotencia, cancelación anticipada, latencia y fallos transitorios bajo control total; inspección de autorizaciones y cancelaciones.
- En contra: un segundo proyecto que mantener.

### Option B — Servidor de stubs genérico configurado por archivos

- En contra: idempotencia por clave, "falla N veces y luego aprueba" y cancelación anticipada exigen extensiones complejas. Descartada.

### Option C — Resultado seleccionado por un campo de la solicitud de compra

- En contra: modifica el contrato funcional de la compra (`AV-004`). Descartada.

## Decision

Se adopta la **Option A**.

### Proyecto

- Carpeta propia en el repositorio, build propio, Dockerfile propio, pruebas propias sin puerta de cobertura (ADR-038).
- Ningún código compartido con `ticketing` en ninguna dirección; no hay librería común de DTOs. El adaptador de pago de `ticketing` define sus propios tipos y solo se acopla al contrato HTTP.
- Contenedor propio en Docker Compose, construido desde su carpeta (ADR-036); alcanzado por nombre de servicio.
- Estado en memoria (volátil).

### Operaciones de pago (consumidas por `CMP-013`)

| Operación | Entrada | Salida |
|---|---|---|
| Autorizar | `Idempotency-Key = paymentAttemptId`, API key; cuerpo `paymentAttemptId`, `orderId`, `eventId`, `customerRef`, `ticketIds` | 200 con `status` `APPROVED` o `DECLINED`, `providerReference`, `reasonCode` en rechazo. 4xx de contrato o autenticación. 5xx o sin respuesta en fallos simulados |
| Cancelar | `paymentAttemptId` en la ruta, API key | 200 con `cancellationStatus`: `REVERSED` (estaba aprobado), `VOIDED` (estaba rechazado), `REGISTERED_BEFORE_CHARGE` (no existía o estaba en tránsito); siempre idempotente |

Reglas:

1. **Idempotencia de autorización**: el resultado se guarda por `paymentAttemptId`; una repetición devuelve el mismo resultado sin contar un pago nuevo, e informa si fue cancelado después.
2. **Idempotencia de cancelación**: repetir la cancelación devuelve el mismo `cancellationStatus` y no cuenta una cancelación nueva.
3. **Cancelación anticipada**: si la cancelación llega antes de que exista resultado de autorización (inexistente o en tránsito), se registra; cualquier autorización posterior con ese `paymentAttemptId` responde `DECLINED` con `reasonCode = ATTEMPT_CANCELLED`. Una autorización en tránsito que termina después de la cancelación también resuelve como `DECLINED`.
4. **Sin importe**: la especificación no define precios.

### Selección determinista del resultado (`AV-004`)

1. Reglas definidas por la API de control, con precedencia: `orderId`, `ticketId`, `customerRef`, `eventId`.
2. Porcentaje de rechazo determinista por hash estable del `paymentAttemptId` (prueba de carga).
3. Resultado por defecto configurado (inicialmente `APPROVED`).

Comportamientos seleccionables: aprobar, rechazar, error definitivo, errores transitorios durante las primeras N invocaciones seguidos de un resultado final, latencia añadida. Los fallos transitorios sostenidos permiten demostrar la apertura del circuit breaker.

### API de control (solo no productivo)

Definir, listar y borrar reglas; fijar resultado por defecto y porcentaje; consultar autorizaciones por intento (número de invocaciones y resultado); consultar cancelaciones por intento (número recibido y estado); reiniciar el estado. Protegida por la misma API key y por red.

### Clasificación en `ticketing`

`APPROVED` → confirmar; `DECLINED` → `REJECTED`; 4xx → `FAILED` sin reverso; timeout, conexión, 5xx o circuito abierto → transitorio (ADR-029, ADR-035). Cancelación: 200 → reverso confirmado; transitorio → reintento del proceso de reversos (ADR-025).

## Rationale

- La cancelación anticipada cierra el hueco del resultado desconocido: un cobro en tránsito nunca se aprueba después de una cancelación.
- La inspección de cancelaciones convierte "reversado exactamente una vez" en una verificación objetiva.
- El proyecto independiente garantiza `TC-017` en el código, no solo en el despliegue.

## Consequences

- Dos builds en el repositorio; Docker Compose construye ambos por separado.
- El estado volátil se pierde al reiniciar el mock; aceptable para un simulador. Un reinicio entre un cobro y su cancelación se trata como intento inexistente (`REGISTERED_BEFORE_CHARGE`).
- En producción el mock se sustituye por un proveedor real tras el mismo puerto; el proveedor real debe ofrecer idempotencia y cancelación equivalentes.

## Risks and mitigations

| Risk | Mitigation |
|---|---|
| El mock no representa a un proveedor real | Contrato aislado tras un puerto; limitación declarada |
| API de control expuesta | Solo no productivo, API key y red interna |

## Items to verify

| Item | Qué comprobar | Impacto si difiere |
|---|---|---|
| Cliente HTTP reactivo de Spring Boot 4.x | Módulo y configuración de timeouts | Solo implementación del adaptador |
| Herramienta de build del mock con Java 25 | Compatibilidad | Cambia la herramienta, no el contrato |

## Depends on

- `AV-004`, `FG-003` (resueltos).
