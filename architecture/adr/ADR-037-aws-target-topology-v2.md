---
id: ADR-037
title: AWS target topology with backlog-age scaling, graceful shutdown, bot control and alarm catalog
status: ACCEPTED
priority: MEDIUM
supersedes: ADR-018
human_decision: CONFIRMED_WITH_CHANGE
source_ids: [EVAL-005, EVAL-011, EVAL-012, EVAL-013, EVAL-006, EVAL-007, EVAL-008, NFR-007, NFR-010, TC-004, TC-005, TC-016]
related_adrs: [ADR-022, ADR-024, ADR-025, ADR-026, ADR-028, ADR-029, ADR-031, ADR-032, ADR-035, ADR-036]
---

# ADR-037 — AWS target topology with backlog-age scaling, graceful shutdown, bot control and alarm catalog

Reemplaza a ADR-018. Detalle: `ticketing.aws-target.v2.md`. No define Terraform.

## Human decision applied

Respuesta humana vinculante a ADR-018 (CONFIRMED_WITH_CHANGE), incorporada íntegramente:

> Se confirma la topología objetivo con ECS sobre Fargate y dos servicios, `api` y `worker`, de la misma imagen; balanceador público con TLS y firewall de aplicación con reglas por tasa; tareas en subredes privadas con endpoints privados; un rol de tarea de mínimo privilegio por servicio; gestor de secretos; DynamoDB on-demand con cifrado y PITR; SQS Standard con cifrado y DLQ; Cognito como emisor; autoescalado independiente; observabilidad con logs, métricas, trazas y alarmas; y una cuenta de AWS por entorno. El Payment Mock solo se despliega en entornos no productivos. Se descartan las funciones sin servidor y Kubernetes gestionado. Se añaden cinco cambios.
>
> Escalado del worker. Además de los mensajes pendientes por tarea, el worker escala según la antigüedad del mensaje más antiguo de la cola de Orders conforme a ADR-010, que es la señal que protege la vigencia de la Reservation.
>
> Apagado ordenado. Ante la señal de parada de la tarea, el `api` deja de aceptar solicitudes y completa las que están en curso, incluida la publicación en SQS, lo que reduce la ventana de ADR-006 durante despliegues y reducciones de escala. El `worker` deja de recibir mensajes y completa los que tiene en vuelo dentro del tiempo de parada de la tarea.
>
> Control de bots. El firewall de aplicación incorpora control de bots como mitigación del acaparamiento mediante varias cuentas que ADR-013 deja al borde, aceptando su costo como trade-off declarado.
>
> Catálogo de alarmas. Se definen alarmas sobre la profundidad de las DLQ de Orders y de aprovisionamiento, la antigüedad del mensaje más antiguo, el retraso de expiración, la apertura de cada circuit breaker, los reversos de pago pendientes o agotados, las Orders en cuarentena y los aprovisionamientos fallidos.
>
> Mejoras de producción declaradas. La captura de cambios de DynamoDB hacia almacenamiento con bloqueo de objetos para la auditoría (ADR-012) y el precalentamiento de la capacidad de la tabla y sus índices antes de una venta masiva mediante warm throughput, cuya configuración vigente debe verificarse.

Correspondencia: ADR-005 → ADR-025, ADR-006 → ADR-026, ADR-009 → ADR-028, ADR-010 → ADR-029, ADR-012 → ADR-031, ADR-013 → ADR-032, ADR-016 → ADR-035.

## Context

`EVAL-005`, `EVAL-011`, `EVAL-012` y `EVAL-013` evalúan escalabilidad, resiliencia, IaC, experiencia cloud-native y operación. Son criterios de evaluación, no requisitos. La topología debe compartir el modelo lógico local (ADR-036).

## Options considered

### Option A — ECS sobre Fargate con servicios `api` y `worker`, balanceador con WAF

- A favor: misma imagen y procesos que en local; procesos de larga vida (consumidores, procesos periódicos); escalado independiente.
- En contra: coste base; más piezas de red.

### Option B — Funciones sin servidor

- En contra: modelo de ejecución distinto; procesos periódicos con precisión `TO_VERIFY`. Descartada.

### Option C — Kubernetes gestionado

- En contra: complejidad desproporcionada. Descartada.

## Decision

Se adopta la **Option A**.

| Tema | Decisión |
|---|---|
| Cómputo | ECS sobre Fargate; servicios `api` y `worker` de la misma imagen; Payment Mock solo no productivo |
| Entrada | ALB público con TLS; WAF con reglas gestionadas, reglas por tasa y control de bots (coste aceptado) |
| Red | VPC por entorno, dos o más zonas; subredes privadas para tareas; endpoints privados para DynamoDB, SQS, registro de imágenes, logs, métricas y secretos; salida controlada para lo demás |
| Gestión | Solo el endpoint de salud en el puerto registrado en el ALB; métricas y gestión en puerto interno no registrado (ADR-032) |
| IAM | Rol de tarea por servicio con mínimo privilegio sobre la tabla, índices y colas concretas (`ticketing.aws-target.v2.md` §4); rol de ejecución separado; sin claves estáticas |
| Datos | DynamoDB on-demand, cifrado, PITR; SQS Standard cifrado con DLQ para Orders y aprovisionamiento |
| Identidad | User pool de Cognito con grupos `ADMIN` y `CUSTOMER` |
| Escalado `api` | Seguimiento de objetivo sobre CPU y solicitudes por destino |
| Escalado `worker` | Mensajes pendientes por tarea de la cola de Orders y antigüedad del mensaje más antiguo de esa cola (alarma a los 2 minutos, ADR-029); máximo de tareas acotado |
| Apagado ordenado | `api`: deja de aceptar, completa solicitudes en curso incluida la publicación en SQS; `worker`: detiene los bucles de recepción, completa los mensajes en vuelo y los ciclos en curso; tiempo de parada de la tarea superior al presupuesto de publicación (2 s) y al tope de procesamiento (30 s) |
| Observabilidad | Logs estructurados, métricas técnicas y de negocio, trazas propagadas por los mensajes, panel y catálogo de alarmas (abajo) |
| Aislamiento | Una cuenta por entorno |

### Catálogo de alarmas

| Alarma | Fuente |
|---|---|
| Profundidad de `ticketing-orders-dlq` > 0 | ADR-029 |
| Profundidad de `ticketing-event-provisioning-dlq` > 0 | ADR-029 |
| Antigüedad del mensaje más antiguo de `ticketing-orders` > 2 min | ADR-029 |
| Retraso de expiración > 15 s | ADR-028 |
| Apertura del circuit breaker del Payment Mock | ADR-035 |
| Apertura del circuit breaker de publicación en SQS | ADR-035 |
| Reversos de pago pendientes por encima de un umbral o con antigüedad excesiva | ADR-025 |
| Reversos de pago agotados > 0 | ADR-025 |
| Orders en cuarentena > 0 | ADR-025 |
| Aprovisionamientos fallidos > 0 | ADR-024 |
| Tasa de errores 5xx, p95 por encima de los objetivos de prueba, throttling de DynamoDB | ADR-018 original, mantenidas |

### Mejoras de producción declaradas (no incluidas)

- Captura de cambios de DynamoDB hacia almacenamiento con bloqueo de objetos y retención para la auditoría (ADR-031).
- Precalentamiento de la tabla y sus índices antes de una venta masiva mediante warm throughput (configuración `TO_VERIFY`).

## Rationale

- La antigüedad del mensaje más antiguo es la señal directamente ligada a la vigencia de la Reservation.
- El apagado ordenado reduce la ventana entre commit y publicación durante despliegues.
- El control de bots es la mitigación del acaparamiento con varias cuentas que ADR-032 deja al borde.

## Consequences

- El rol `worker` necesita permiso de envío a ambas colas (barrido y detección de estancados).
- Coste adicional del control de bots y de las alarmas (trade-off aceptado).
- Con el Payment Mock caído, la antigüedad de la cola crece y el worker escala hasta su máximo sin efecto útil; el máximo acota el coste.

## Risks and mitigations

| Risk | Mitigation |
|---|---|
| `RISK-007` emuladores frente a servicios reales | Entorno de desarrollo en AWS para integración y carga |
| Coste de salida a Internet, endpoints y control de bots | Una salida en no productivo; revisar endpoints; control de bots solo donde aporte |

## Items to verify

| Item | Qué comprobar | Impacto si difiere |
|---|---|---|
| Endpoint privado para Cognito | Disponibilidad | Eliminar la salida a Internet del `api` |
| Métricas de mensajes pendientes por tarea y de antigüedad para autoescalado | Forma soportada | Escalado por pasos sobre profundidad y antigüedad |
| Control de bots del WAF | Disponibilidad y precios | Reglas por tasa más estrictas |
| Warm throughput de DynamoDB | Configuración vigente para tabla e índices | Solo la mejora productiva |
| Tiempo máximo de parada de tareas Fargate | Valor vigente | Ajustar topes de procesamiento |
| Sincronización horaria de tareas Fargate | Desalineación | Ampliar márgenes temporales |

## Depends on

- `FG-003` (resuelto).
