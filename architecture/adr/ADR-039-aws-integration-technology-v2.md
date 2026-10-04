---
id: ADR-039
title: AWS integration technology for DynamoDB and SQS with two consumers, heartbeat and pause control
status: ACCEPTED
priority: MEDIUM
supersedes: ADR-020
human_decision: CONFIRMED_WITH_CHANGE
source_ids: [TC-001, TC-002, TC-003, TC-004, TC-005, TC-008, TC-010, NFR-003, FR-007]
related_adrs: [ADR-023, ADR-024, ADR-027, ADR-029, ADR-034, ADR-035, ADR-037, ADR-040]
---

# ADR-039 — AWS integration technology for DynamoDB and SQS with two consumers, heartbeat and pause control

Reemplaza a ADR-020.

## Human decision applied

Respuesta humana vinculante a ADR-020 (CONFIRMED_WITH_CHANGE), incorporada íntegramente:

> Se confirman los clientes asíncronos del SDK oficial de AWS adaptados a tipos reactivos solo dentro de los adaptadores, el cliente de bajo nivel de DynamoDB con mapeo explícito en el adaptador y un consumidor reactivo propio de SQS. Se descartan la integración de alto nivel de Spring para AWS y el cliente de mapeo de objetos. Se añaden cinco precisiones.
>
> Dos consumidores. Las colas de Orders y de aprovisionamiento tienen cada una su propio bucle reactivo, con concurrencia y apagado ordenado propios, conforme a ADR-004, ADR-010 y ADR-018.
>
> Pausa y reanudación. El bucle del consumidor de Orders y el proceso de reversos admiten pausa y reanudación gobernadas por el estado del circuit breaker del Payment Mock, conforme a ADR-016.
>
> Heartbeat de visibilidad. El consumidor de aprovisionamiento extiende la visibilidad del mensaje mientras el trabajo progresa, conforme a ADR-010.
>
> Motivos de cancelación. Las escrituras transaccionales solicitan el item que no cumplió la condición, para distinguir un Ticket inexistente de uno no disponible y dar prioridad al registro de idempotencia en la carrera con la misma clave, conforme a ADR-007 y ADR-016.
>
> Operaciones adicionales. El adaptador implementa la escritura por lotes con reintento de items no procesados (ADR-004) y el conteo en paralelo por shards del índice de disponibles (ADR-021).

Correspondencia: ADR-004 → ADR-024, ADR-007 → ADR-027, ADR-010 → ADR-029, ADR-016 → ADR-035, ADR-018 → ADR-037, ADR-021 → ADR-040.

## Context

API y consumidores no bloqueantes (`NFR-003`, `TC-003`) sobre Java 25 y Spring Boot 4.x. Se necesita control fino de condiciones, transacciones, motivos de cancelación, escritura y lectura por lotes, recuento por shards, visibilidad y contador de recepciones.

## Options considered

### Option A — Clientes asíncronos del SDK oficial adaptados a tipos reactivos, con consumidores reactivos propios

- A favor: no bloqueante; control total; depende solo del SDK.
- En contra: más código de adaptador.

### Option B — Integración de alto nivel de Spring para AWS

- En contra: compatibilidad con Spring Boot 4.x `TO_VERIFY`; menos control sobre visibilidad, heartbeat y motivos de cancelación. Descartada.

### Option C — Cliente de mapeo de objetos para DynamoDB

- En contra: claves sobrecargadas y condiciones heterogéneas se expresan mejor a bajo nivel. Descartada.

## Decision

Se adopta la **Option A**.

### DynamoDB (`CMP-010`)

- Cliente asíncrono de bajo nivel para todas las operaciones de `ticketing.data-model.v2.md` §5; mapeo explícito en el adaptador.
- Escrituras transaccionales con solicitud del item que no cumplió la condición; el adaptador devuelve un resultado tipado con el motivo por item (idempotencia, bloqueo, Ticket inexistente, Ticket no disponible, guardián de Order, conflicto).
- Escritura por lotes con reintento de items no procesados y backoff acotado (`AP-002`, `AP-027`).
- Lectura por lotes con consistencia fuerte para la verificación del aprovisionamiento (`AP-025`).
- Recuento en paralelo por shards de `GSI2` con concurrencia acotada (`AP-021`) y sondeo con parada en el primer resultado (`AP-005`).

### SQS

| Bucle | Componente | Concurrencia | Particularidades |
|---|---|---|---|
| Orders | `CMP-012` | Configurable (16) | Pausa y reanudación por eventos de estado del circuit breaker del Payment Mock; en semiabierto recibe solo los mensajes de prueba permitidos; contador aproximado de recepciones; cambio de visibilidad para backoff y para posponer |
| Aprovisionamiento | `CMP-025` | 1 | Heartbeat: extiende la visibilidad a 120 s cada 30 s mientras el caso de uso informa progreso; se detiene si no hay progreso en 60 s |

Ambos bucles: long polling, procesamiento con concurrencia acotada, eliminación o cambio de visibilidad según la clasificación de ADR-029, continuidad ante errores y apagado ordenado propio (dejan de recibir y completan lo que está en vuelo, ADR-037). El proceso de reversos (`CMP-024`) también se pausa con el circuito del Payment Mock abierto (ADR-035).

- Publicación (`CMP-011`): un adaptador con dos puertos (Orders y aprovisionamiento), con la política de timeout, circuit breaker y retry de ADR-035.
- Los puertos exponen `Mono` y `Flux` (`TC-008`); el SDK se adapta solo dentro de los adaptadores.
- Endpoint, región y credenciales por configuración; en AWS por el rol de la tarea.
- Reintentos del SDK acotados y sin apilar con los de aplicación.

## Rationale

- Distinguir motivos por item es imprescindible para `UNKNOWN_TICKETS`, `TICKETS_UNAVAILABLE`, `ACTIVE_ORDER_EXISTS` y la prioridad de la idempotencia.
- Dos bucles independientes impiden que el aprovisionamiento consuma la concurrencia de las compras.

## Consequences

- El adaptador de DynamoDB concentra la complejidad de expresiones; se prueba de forma unitaria y de integración (ADR-038).
- La pausa y el heartbeat son responsabilidad propia.

## Risks and mitigations

| Risk | Mitigation |
|---|---|
| `RISK-012` incompatibilidad del SDK con Java 25 | Prueba mínima temprana contra los emuladores |
| Errores en expresiones manuales | Pruebas unitarias e integración |

## Items to verify

| Item | Qué comprobar | Impacto si difiere |
|---|---|---|
| SDK oficial de AWS con Java 25 y cliente HTTP asíncrono | Compatibilidad | Si no, el stack obligatorio entra en conflicto y se escala |
| Solicitud del item que falló la condición en transacciones | Soporte en el SDK y en DynamoDB Local | Lectura de respaldo |
| Lectura por lotes consistente en DynamoDB Local | Soporte | Lecturas individuales consistentes en local |
| Cambio de visibilidad y contador de recepciones en LocalStack fijado | Soporte | Alternativas de ADR-036 |

## Depends on

- Sin dependencias de `FG-*` ni `AV-*` abiertas.
