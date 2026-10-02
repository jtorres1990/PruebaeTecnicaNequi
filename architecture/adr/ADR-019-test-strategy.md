---
id: ADR-019
title: Test strategy
status: PROPOSED
priority: MEDIUM
source_ids: [NFR-013, TC-014, DEL-003, AC-007, AC-029, AC-030, AC-031, NFR-001, NFR-002, NFR-004, AC-014, AC-023, AC-024, AC-025, AC-009]
related_adrs: [ADR-002, ADR-005, ADR-008, ADR-011, ADR-014, ADR-015, ADR-017]
---

# ADR-019 — Test strategy

## Context

`NFR-013` y `DEL-003` exigen pruebas unitarias con cobertura mínima del 90 %, que incluyan casos de uso con dobles de repositorios, componentes reactivos y concurrencia, con JUnit 5, Mockito y reactor-test (`TC-014`). `AC-007` exige verificar que no hay sobreventa concurrente y `AC-029` a `AC-031` definen objetivos de la prueba de carga.

Hay que decidir cómo se alcanza la cobertura, cómo se prueba la concurrencia de forma determinista y cómo se ejecuta la prueba de carga. Este ADR no escribe pruebas.

## Options considered

### Option A — Pirámide con la puerta de cobertura en pruebas unitarias, concurrencia probada por invariante y carga con herramienta externa

- A favor: la puerta de cobertura no depende de Docker; la concurrencia se verifica sobre un resultado determinista aunque el entrelazado no lo sea; la carga se ejecuta contra el sistema real desplegado.
- En contra: los adaptadores necesitan pruebas unitarias con el cliente del SDK simulado, además de las de integración.

### Option B — Cobertura alcanzada principalmente con pruebas de integración sobre emuladores

- A favor: menos dobles; mayor fidelidad por prueba.
- En contra: la puerta de cobertura depende de contenedores; pruebas lentas; la especificación pide pruebas unitarias con dobles.

### Option C — Concurrencia probada solo en la prueba de carga

- A favor: sin pruebas específicas.
- En contra: no determinista ni repetible; `DEL-003` exige pruebas que verifiquen concurrencia.

## Decision

Se adopta la **Option A**.

### Niveles

| Nivel | Alcance | Herramientas |
|---|---|---|
| Unitarias de dominio | Máquinas de estado, invariantes, validaciones, políticas temporales | JUnit 5 |
| Unitarias de casos de uso | Orquestación con puertos simulados, incluidas rutas de error | JUnit 5, Mockito, reactor-test |
| Unitarias de adaptadores | Construcción de condiciones y transacciones, clasificación de errores, mapeo, con el cliente del SDK simulado | JUnit 5, Mockito, reactor-test |
| Capa web reactiva | Contrato HTTP, validación, seguridad por rol, mapeo de errores, con tokens simulados | Cliente de pruebas reactivo, soporte de pruebas de seguridad |
| Integración | Adaptadores contra DynamoDB Local y SQS emulado | Contenedores de prueba |
| Extremo a extremo | Flujos de `MF-001` a `MF-004` sobre el entorno de Docker Compose | Colección de solicitudes (`DEL-004`) |
| Carga | `AC-029` a `AC-031` | Herramienta de carga externa |

### Cobertura mínima

- Puerta: cobertura de líneas agregada de al menos 90 % sobre `domain`, `application` e `infrastructure`, medida solo con las pruebas unitarias y de capa web, que no requieren Docker.
- Exclusiones acotadas y explícitas: clase de arranque y clases de pura configuración de `bootstrap`.
- El build falla si no se alcanza.
- `payment-mock` se mide por separado con el mismo umbral.

### Concurrencia determinista

1. **Por invariante**: se lanzan N solicitudes simultáneas sobre el mismo Ticket (y conjuntos solapados de Ticket) y se afirma el invariante, que es determinista aunque el orden no lo sea: exactamente una Order creada, N − 1 rechazos, ningún Ticket reservado parcialmente (`AC-007`, `AC-016`).
2. **En casos de uso**: contra un doble en memoria del puerto Order lifecycle store que aplica la misma semántica condicional y atómica; ejecución repetida.
3. **En integración**: lo mismo contra DynamoDB Local, que es donde se verifica el mecanismo real de `ADR-002`.
4. **Carreras temporales** (`ADR-008`): con el reloj inyectado, se prueban ambos órdenes (confirmar y luego expirar; expirar y luego confirmar) y la ejecución simultánea, afirmando un único estado terminal y coherencia de Ticket (`AC-008`, `AC-009`, `AC-019`).
5. **Tiempo virtual** de reactor-test para periodicidad, backoff y timeouts, sin esperas reales.
6. **Idempotencia**: entrega repetida y simultánea del mismo mensaje, afirmando una sola transición y, mediante la inspección del Payment Mock, un único PaymentAttempt (`AC-023`, `AC-024`, `AC-025`).
7. **Detección de bloqueo**: un detector de llamadas bloqueantes en las pruebas reactivas (`NFR-003`).

### Prueba de carga objetivo

| Aspecto | Decisión |
|---|---|
| Herramienta | Herramienta de carga basada en scripts, ejecutada como contenedor en un perfil opcional de Docker Compose (`ADR-017`) |
| Perfil de carga | Al menos 1.000 usuarios virtuales concurrentes y tasa de llegada sostenida de aproximadamente 200 solicitudes por segundo durante varios minutos, con rampa previa (`NFR-001`) |
| Mezcla | Mayoría de consultas de disponibilidad, más listado, inicio de compra y consulta de Order. La proporción exacta se fija en el script y se documenta con los resultados |
| Datos | Varios Events con capacidad de entre 1.000 y 2.000 Ticket; tokens de `CUSTOMER` generados previamente; Payment Mock en modo de porcentaje determinista |
| Umbrales | p95 de disponibilidad inferior a 500 ms (`AC-030`); p95 de inicio de compra inferior a 1 s (`AC-031`) |
| Verificación de consistencia | Al terminar, una comprobación recorre la tabla y afirma: ningún Ticket pertenece a más de una Order activa o confirmada; todo Ticket `SOLD` pertenece a exactamente una Order `CONFIRMED`; toda Order `CONFIRMED` tiene todos sus Ticket `SOLD`; toda Order terminal no confirmada no retiene Ticket; capacidad igual al total de Ticket (`AC-029`, `NFR-004`) |
| Entorno | Primero el entorno local; si el emulador no sostiene la carga, se repite contra un entorno de desarrollo en AWS |
| Interpretación | Los resultados valen para el entorno medido. Son objetivos de prueba, no capacidad productiva ni SLA |

## Rationale

- Separar la puerta de cobertura de los contenedores hace que el requisito del 90 % sea verificable en cualquier máquina.
- Afirmar invariantes en lugar de entrelazados convierte la prueba de concurrencia en determinista sin controlar el planificador.
- La verificación posterior a la carga es la única forma objetiva de demostrar `AC-029`: la ausencia de errores HTTP no prueba ausencia de sobreventa.

## Consequences

- Los adaptadores se prueban dos veces (unitaria e integración), con propósitos distintos.
- El doble en memoria del Order lifecycle store debe mantenerse fiel a la semántica del adaptador real; la prueba de integración es la que lo valida.
- La métrica de cobertura y las exclusiones requieren validación (`AV-006`).

## Risks and mitigations

| Risk | Mitigation |
|---|---|
| `RISK-008` el entorno local no alcanza los objetivos por limitaciones del emulador | Tamaño de datos realista y acotado; ejecución alternativa en AWS; el resultado se informa con su entorno |
| `RISK-007` el emulador no reproduce la exclusión transaccional | Verificación temprana; pruebas de integración repetibles contra AWS |
| `RISK-012` herramientas de prueba sin soporte de Java 25 | Verificación al inicio |

## Items to verify

| Item | Qué comprobar | Impacto si difiere |
|---|---|---|
| Herramienta de cobertura, Mockito y detector de bloqueo con Java 25 | Compatibilidad | Si el detector de bloqueo no es compatible, se omite; si la herramienta de cobertura no lo es, el requisito de `NFR-013` queda bloqueado y debe escalarse |
| Contenedores de prueba para DynamoDB Local y LocalStack | Disponibilidad de módulos e imágenes | Alternativa: ejecutar las pruebas de integración contra el Docker Compose ya levantado |
| Capacidad del entorno local para 200 solicitudes por segundo | Medición | Ejecutar la prueba objetivo en AWS |
| Herramienta de carga | Imagen y soporte de tasa de llegada constante y umbrales por percentil | Sustituir por otra equivalente |

## Depends on

- `AV-006`: métrica y alcance de la cobertura.
- `AV-004`: preparación de escenarios de pago mediante el mock.
