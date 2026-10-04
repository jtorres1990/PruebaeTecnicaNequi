---
id: ADR-038
title: Test strategy with capacity-scale load data, extended invariants and resilience scenarios
status: ACCEPTED
priority: MEDIUM
supersedes: ADR-019
human_decision: CONFIRMED_WITH_CHANGE
source_ids: [NFR-013, TC-014, DEL-003, DEL-004, AC-007, AC-029, AC-030, AC-031, NFR-001, NFR-002, NFR-004, AC-014, AC-023, AC-024, AC-025, AC-009, AC-019]
related_adrs: [ADR-022, ADR-023, ADR-024, ADR-025, ADR-026, ADR-027, ADR-029, ADR-030, ADR-032, ADR-033, ADR-034, ADR-035, ADR-036, ADR-040]
---

# ADR-038 — Test strategy with capacity-scale load data, extended invariants and resilience scenarios

Reemplaza a ADR-019. Este ADR no escribe pruebas.

## Human decision applied

Respuesta humana vinculante a ADR-019 (CONFIRMED_WITH_CHANGE), incorporada íntegramente:

> Se confirma la pirámide de pruebas con la puerta de cobertura del 90 % medida con pruebas que no requieren contenedores, la concurrencia verificada por invariante con solicitudes simultáneas, reloj inyectado y tiempo virtual, las pruebas de integración contra los emuladores, los flujos extremo a extremo con la colección de solicitudes y la prueba de carga con herramienta externa y verificación de invariantes al terminar. Se descartan la cobertura alcanzada principalmente con pruebas de integración y la concurrencia probada solo con la carga. Se añaden cinco cambios.
>
> Datos de carga. La prueba de carga incluye al menos un Event de 50.000 Ticket y un escenario concentrado sobre ese Event con más de 1.000 usuarios concurrentes conforme a ADR-001, usando más de 1.000 identidades `CUSTOMER` distintas conforme a ADR-014.
>
> Invariantes posteriores a la carga. Además de los originales, se verifica que cada bloqueo de Order activa corresponde a una Order en `CREATED` y que toda Order en `CREATED` tiene su bloqueo (ADR-013); que el índice de Orders pendientes de encolado queda vacío (ADR-006); que ninguna Order expiró por espera en cola (ADR-010); que no hay reversos de pago agotados (FG-003); y que la cantidad disponible informada coincide con los Ticket en `AVAILABLE` (ADR-021).
>
> Pruebas deterministas de los mecanismos aprobados. Se prueban la carrera de dos solicitudes con la misma `Idempotency-Key` que deben producir la misma Order (ADR-007); la reentrega del aprovisionamiento que no debe sobrescribir Ticket después de habilitar el Event (ADR-004); la cancelación anticipada que debe hacer rechazar el cobro posterior (FG-003); el barrido de republicación que no debe competir con la ruta síncrona (ADR-006); la imposibilidad de dos Orders activas del mismo cliente en el mismo Event (ADR-013); y las transiciones del circuit breaker con tiempo virtual (ADR-016).
>
> Escenarios de resiliencia extremo a extremo. Se documentan en el README y se demuestran sobre Docker Compose: detener el worker a mitad de un pago y verificar la reanudación por lease; detener el Payment Mock y verificar la apertura del circuito, la pausa del consumo y la recuperación; y detener la cola y verificar que la compra responde 503 sin modificar el inventario.
>
> Cobertura del Payment Mock. Conforme a ADR-015, `payment-mock` queda fuera de la métrica del 90 % y mantiene sus propias pruebas sin puerta de cobertura.
>
> Se mantiene como riesgo declarado que DynamoDB Local puede no sostener la carga objetivo; en ese caso la prueba se repite en un entorno de AWS y el resultado se informa junto con su entorno.

Correspondencia: ADR-001 → ADR-022, ADR-004 → ADR-024, ADR-006 → ADR-026, ADR-007 → ADR-027, ADR-010 → ADR-029, ADR-013 → ADR-032, ADR-014 → ADR-033, ADR-015 → ADR-034, ADR-016 → ADR-035, ADR-021 → ADR-040. Además `AV-006` (métrica y alcance de cobertura) y `AV-004` (configuración del mock previa a cada escenario en la colección).

## Context

`NFR-013` y `DEL-003` exigen pruebas unitarias con cobertura mínima del 90 % sobre casos de uso, componentes reactivos y concurrencia, con JUnit 5, Mockito y reactor-test (`TC-014`). `AC-007` exige verificar ausencia de sobreventa; `AC-029` a `AC-031` fijan los objetivos de la prueba de carga (objetivos de prueba, no capacidad productiva).

## Options considered

### Option A — Pirámide con puerta de cobertura sin contenedores, concurrencia por invariante, carga externa con invariantes

- A favor: puerta verificable en cualquier máquina; concurrencia determinista por resultado; carga contra el sistema real.
- En contra: adaptadores probados dos veces.

### Option B — Cobertura principalmente con integración sobre emuladores

- En contra: la puerta depende de contenedores; lenta. Descartada.

### Option C — Concurrencia probada solo con la carga

- En contra: no determinista. Descartada.

## Decision

Se adopta la **Option A**.

### Niveles

| Nivel | Alcance | Herramientas |
|---|---|---|
| Unitarias de dominio | Máquinas de estado; invariantes; validaciones; reglas de una Order activa, cuarentena, reverso, ciclo de aprovisionamiento; generación de `ticketId` y shards | JUnit 5 |
| Unitarias de casos de uso | Orquestación con puertos simulados, incluidas rutas de error y precedencia de códigos | JUnit 5, Mockito, reactor-test |
| Unitarias de adaptadores | Condiciones y transacciones, motivos por item, clasificación de errores, circuit breaker, cursor, mapeo | JUnit 5, Mockito, reactor-test |
| Capa web reactiva | Contrato HTTP, validación, seguridad por rol (cinco identidades de ADR-033), mapeo de errores | Cliente de pruebas reactivo, soporte de pruebas de seguridad |
| Integración | Adaptadores contra DynamoDB Local y SQS emulado | Contenedores de prueba |
| Extremo a extremo | `MF-001` a `MF-004`, aprovisionamiento asíncrono, rechazos y escenarios de resiliencia sobre Docker Compose | Colección de solicitudes (`DEL-004`), que configura el mock antes de cada escenario de rechazo o fallo (`AV-004`) |
| Carga | `AC-029` a `AC-031` | Herramienta de carga externa |

### Cobertura mínima (`AV-006`)

Cobertura de líneas agregada ≥ 90 % sobre `domain`, `application` e `infrastructure` de `ticketing`, medida solo con pruebas sin contenedores; exclusiones: clase de arranque y clases de pura configuración. El build falla si no se alcanza. `payment-mock` queda fuera de la métrica y mantiene sus propias pruebas sin puerta.

### Pruebas deterministas de concurrencia y mecanismos

| Mecanismo | Verificación |
|---|---|
| Sobreventa (`AC-007`, `AC-016`) | N solicitudes simultáneas sobre el mismo Ticket y conjuntos solapados: un ganador, N − 1 rechazos, ningún parcial; con doble en memoria y contra DynamoDB Local |
| Carrera de misma `Idempotency-Key` (ADR-027) | Dos solicitudes simultáneas con la misma clave y contenido producen la misma Order; con contenido distinto, una Order y un `IDEMPOTENCY_KEY_REUSED`; nunca `TICKETS_UNAVAILABLE` para la perdedora |
| Reentrega del aprovisionamiento (ADR-024) | Tras habilitar y reservar Ticket, una reentrega de `MSG-002` no obtiene el lease y no modifica ningún Ticket; un lease ajeno vencido se reclama y reanuda |
| Cancelación anticipada (`FG-003`, ADR-030) | Cancelación antes del cobro hace que la autorización posterior responda `DECLINED`; la inspección del mock muestra una única cancelación por intento |
| Barrido frente a ruta síncrona (ADR-026) | Con reloj inyectado, una Order de menos de 30 s no es republicada; una de más de 30 s sin `enqueuedAt` sí; una con tiempo restante inferior al margen de corte no |
| Una Order activa por cliente y Event (ADR-032) | Compras simultáneas del mismo cliente al mismo Event: una Order y `ACTIVE_ORDER_EXISTS` para las demás; tras un estado terminal, una nueva compra es posible |
| Circuit breaker (ADR-035) | Transiciones cerrado → abierto → semiabierto → cerrado con tiempo virtual; rechazos y errores de contrato no abren; pausa y reanudación del consumidor |
| Carreras temporales (ADR-008) | Confirmar y expirar en ambos órdenes y simultáneos; aprobación tardía; marca de reverso |
| Idempotencia de mensajes (`AC-023` a `AC-025`) | Entregas repetidas y simultáneas; un único PaymentAttempt según la inspección del mock |
| Cuarentena (ADR-025) | Cancelación por condición de Ticket con Order en `CREATED`: cuarentena, salida del índice de expiración, auditoría, sin escritura sobre Ticket |
| Detección de bloqueo | Detector de llamadas bloqueantes en pruebas reactivas (`NFR-003`) |

### Prueba de carga objetivo

| Aspecto | Decisión |
|---|---|
| Herramienta | Herramienta de carga basada en scripts, en el perfil de carga de Docker Compose (ADR-036) |
| Perfil | ≥ 1.000 usuarios virtuales y ~200 solicitudes por segundo sostenidas varios minutos, con rampa (`NFR-001`) |
| Datos | Al menos un Event de 50.000 Ticket aprovisionado asíncronamente antes de la carga; escenario concentrado sobre ese Event con más de 1.000 usuarios; más de 1.000 identidades `CUSTOMER` distintas generadas por `load-token-generator` (ADR-033); Payment Mock en modo porcentaje |
| Mezcla | Mayoría de consultas de disponibilidad paginadas, más listado, compra y consulta de Order; proporción documentada con los resultados |
| Umbrales | p95 de disponibilidad < 500 ms (`AC-030`); p95 de inicio de compra < 1 s (`AC-031`) |
| Invariantes al terminar | Ningún Ticket en más de una Order activa o confirmada; todo `SOLD` en exactamente una Order `CONFIRMED`; toda `CONFIRMED` con todos sus Ticket `SOLD`; ninguna Order terminal no confirmada retiene Ticket; capacidad igual al total de Ticket (`AC-029`, `NFR-004`); cada bloqueo corresponde a una Order en `CREATED` y toda Order en `CREATED` tiene su bloqueo; `GSI4` vacío; ninguna Order `EXPIRED` sin PaymentAttempt habiendo sido encolada; `GSI3` rango `REVERSAL#EXHAUSTED` vacío; cantidad disponible informada por `API-003` igual al recuento de Ticket `AVAILABLE` tras el drenaje |
| Entorno | Primero local; si DynamoDB Local no sostiene la carga, se repite en AWS y se informa con su entorno |
| Interpretación | Objetivos de prueba, no capacidad productiva ni SLA |

### Escenarios de resiliencia (README, Docker Compose)

1. Detener `ticketing-worker` a mitad de un pago: al reiniciar, el lease vence y otro consumo reanuda el mismo PaymentAttempt; el mock registra un único intento.
2. Detener `payment-mock`: el circuito se abre, el consumo de Orders y los reversos se pausan, la expiración continúa; al reiniciar, el circuito pasa a semiabierto y el consumo se reanuda.
3. Detener `localstack`: la compra responde 503 con `Retry-After` sin modificar el inventario una vez abierto el circuito de publicación; al recuperar, las compras vuelven a crear Orders.

## Rationale

- La puerta sin contenedores hace verificable `NFR-013` en cualquier máquina.
- Afirmar invariantes convierte la concurrencia en determinista.
- Las invariantes ampliadas verifican los mecanismos añadidos en la revisión.

## Consequences

- El aprovisionamiento del Event de 50.000 Ticket forma parte de la preparación de la carga.
- El doble en memoria del Order lifecycle store debe reproducir el bloqueo, la cuarentena y la marca de reverso.

## Risks and mitigations

| Risk | Mitigation |
|---|---|
| `RISK-008` DynamoDB Local no sostiene la carga | Repetición en AWS; resultado con su entorno |
| `RISK-007` el emulador no reproduce la exclusión transaccional | Verificación temprana; integración contra AWS |
| `RISK-012` herramientas sin soporte de Java 25 | Verificación al inicio |

## Items to verify

| Item | Qué comprobar | Impacto si difiere |
|---|---|---|
| Herramienta de cobertura, Mockito, detector de bloqueo, pruebas de arquitectura con Java 25 | Compatibilidad | Si la cobertura no es compatible, `NFR-013` queda bloqueado y se escala |
| Contenedores de prueba para DynamoDB Local y LocalStack fijado | Disponibilidad | Integración contra el Docker Compose levantado |
| Capacidad local para 200 solicitudes por segundo y un Event de 50.000 Ticket | Medición | Ejecutar en AWS |
| Herramienta de carga | Tasa de llegada constante y umbrales por percentil | Sustituir |

## Depends on

- `AV-004`, `AV-006`, `FG-003` (resueltos).
