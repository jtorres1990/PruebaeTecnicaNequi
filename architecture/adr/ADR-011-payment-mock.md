---
id: ADR-011
title: Payment Mock
status: PROPOSED
priority: MEDIUM
source_ids: [TC-017, FR-015, FR-017, BR-003, BR-020, AC-005, AC-019, AC-020, AC-021, AC-023, AC-024, ALT-004, ALT-005, ERR-008, TC-006, TC-012]
related_adrs: [ADR-007, ADR-008, ADR-010, ADR-013, ADR-016, ADR-017, ADR-019]
---

# ADR-011 — Payment Mock

## Context

`TC-017` exige un servicio simulado independiente con respuestas exitosas y fallidas deterministas. El consumidor lo invoca después de reservar todos los Ticket (`FR-015`) y aplica aprobación (`AC-019`), rechazo (`AC-020`) o fallo técnico (`AC-021`). La integración debe estar protegida (§4 de la especificación).

La especificación no define precio ni importe; tampoco un medio de pago en la solicitud de compra (§10). Hay que decidir protocolo, contrato, cómo se selecciona el resultado de forma determinista y cómo se despliega en local.

## Options considered

### Option A — Servicio HTTP propio e independiente, con reglas de resultado configuradas en el propio mock

Aplicación pequeña en un módulo y contenedor propios. El resultado se decide por reglas administradas mediante una API de control del mock, evaluadas sobre los datos que ya viajan en la solicitud de pago.

- A favor: no añade campos al contrato de compra; idempotencia por clave y simulación de latencia y fallos transitorios bajo control total; permite inspeccionar las invocaciones recibidas para verificar `AC-023` y `AC-024`; mismo toolchain que el resto.
- En contra: código adicional que mantener y probar; el escenario se prepara en dos pasos (configurar el mock y luego comprar).

### Option B — Servidor de stubs genérico configurado por archivos

Usar una herramienta de stubbing HTTP existente, con mappings estáticos.

- A favor: sin código propio.
- En contra: la idempotencia por clave, el "falla N veces y luego aprueba" y el reparto determinista por porcentaje para la prueba de carga exigen extensiones o plantillas complejas; la disponibilidad y capacidades de la imagen son `TO_VERIFY`.

### Option C — Resultado seleccionado por un campo de la solicitud de compra

El cliente envía un token de medio de pago de prueba que determina el resultado.

- A favor: cada solicitud es autocontenida; muy cómodo para la colección de solicitudes.
- En contra: añade una entrada a la operación de inicio de compra que §10 de la especificación no contempla; modifica el contrato funcional. Requeriría decisión humana.

## Decision

Se adopta la **Option A**.

**Protocolo**: HTTP con JSON, invocado por el adaptador de pago (`CMP-013`) con un cliente HTTP no bloqueante.

**Contrato de pago** (operaciones del mock, `CMP-019`):

| Operación | Entrada | Salida |
|---|---|---|
| Autorizar pago | Cabecera de clave de idempotencia = `paymentAttemptId`; cabecera de API key; cuerpo: `paymentAttemptId`, `orderId`, `eventId`, `customerRef`, `ticketIds` | Respuesta correcta con `status` `APPROVED` o `DECLINED`, `providerReference` y, en rechazo, un código de motivo. Errores de contrato o autenticación: respuesta de error de cliente. Fallo simulado: respuesta de error de servidor o sin respuesta dentro del plazo |
| Cancelar pago (depende de `FG-003`) | `paymentAttemptId`, cabecera de API key | Confirmación idempotente de la cancelación |

**Idempotencia**: el mock guarda el resultado por `paymentAttemptId`; una repetición con la misma clave devuelve el mismo resultado y no cuenta como un pago nuevo.

**Selección determinista del resultado**:

1. Reglas configuradas por la API de control, evaluadas en este orden de precedencia: por `orderId`, por `ticketId`, por `customerRef`, por `eventId`.
2. Si ninguna regla aplica y hay un porcentaje de rechazo configurado: se rechaza cuando un hash estable del `paymentAttemptId` cae bajo ese porcentaje. Es determinista por intento y sirve para la prueba de carga.
3. En otro caso, el resultado por defecto configurado (inicialmente `APPROVED`).

Comportamientos seleccionables: aprobar, rechazar, error definitivo, error transitorio durante las primeras N invocaciones seguido del resultado final, y latencia añadida.

**API de control** (solo para pruebas y demostración): definir y borrar reglas, fijar el resultado por defecto y el porcentaje, y consultar las invocaciones recibidas (intentos distintos y número de invocaciones por intento).

**Clasificación en el consumidor**: `APPROVED` → confirmar; `DECLINED` → `REJECTED`; error de cliente → `FAILED`; timeout, error de conexión o error de servidor → transitorio (`ADR-010`).

**Despliegue local**: contenedor propio en Docker Compose, estado en memoria, en la red interna; el consumidor lo alcanza por nombre de servicio. El puerto de control se publica al host para pruebas. La API key llega por variable de entorno (`ADR-013`, `ADR-017`).

**Sin importe**: la solicitud de pago no incluye importe porque la especificación no define precios.

## Rationale

- Un servicio propio permite garantizar las tres propiedades que el diseño necesita del proveedor: independencia (`TC-017`), determinismo e idempotencia por intento (`ADR-007`).
- Mantener la selección del resultado dentro del mock respeta el contrato funcional de la compra.
- La inspección de invocaciones convierte `AC-023` y `AC-024` en verificaciones objetivas.
- El modo por porcentaje permite una prueba de carga con mezcla de aprobaciones y rechazos reproducible.

## Consequences

- Hay un segundo artefacto desplegable en el repositorio, con sus propias pruebas.
- El estado del mock es volátil: al reiniciarlo pierde resultados y reglas. Aceptable para un simulador.
- En un entorno productivo el mock se sustituye por un proveedor real; el adaptador de pago es el único punto de cambio. El mock no se despliega en producción.
- La preparación de un escenario de rechazo o fallo requiere configurar el mock antes de comprar (`AV-004`).

## Risks and mitigations

| Risk | Mitigation |
|---|---|
| El mock no representa la semántica de un proveedor real (idempotencia, cancelación) | Contrato aislado tras un puerto; declarado como limitación |
| API de control expuesta | Solo en entornos no productivos; protegida por la misma API key y por red |

## Items to verify

| Item | Qué comprobar | Impacto si difiere |
|---|---|---|
| Cliente HTTP reactivo disponible en Spring Boot 4.x | Nombre del módulo y configuración de timeouts | Solo afecta la implementación del adaptador |

## Depends on

- `AV-004`: mecanismo de selección del resultado.
- `FG-003`: existencia de la operación de cancelación.
