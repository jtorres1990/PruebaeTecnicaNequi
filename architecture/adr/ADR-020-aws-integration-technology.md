---
id: ADR-020
title: AWS integration technology for DynamoDB and SQS
status: PROPOSED
priority: MEDIUM
source_ids: [TC-001, TC-002, TC-003, TC-004, TC-005, TC-008, TC-010, NFR-003, FR-007]
related_adrs: [ADR-002, ADR-010, ADR-015, ADR-016]
---

# ADR-020 — AWS integration technology for DynamoDB and SQS

## Context

La API y el consumidor deben ser no bloqueantes (`NFR-003`, `TC-003`) sobre Java 25 y Spring Boot 4.x (`TC-001`, `TC-002`). `ADR-002` y `ADR-005` necesitan control fino sobre expresiones de condición, escrituras transaccionales y motivos de cancelación por item. `ADR-010` necesita control fino sobre visibilidad, contador de recepciones y eliminación de mensajes.

La compatibilidad de las librerías de integración con Spring Boot 4.x no puede afirmarse de memoria.

## Options considered

### Option A — Clientes asíncronos del SDK oficial de AWS, adaptados a tipos reactivos, con un consumidor reactivo propio

Cliente asíncrono de bajo nivel para DynamoDB y cliente asíncrono para SQS, envueltos en `Mono` / `Flux` dentro de los adaptadores. El consumidor de SQS es un bucle reactivo propio de recepción con long polling.

- A favor: no bloqueante de extremo a extremo; acceso completo a condiciones, transacciones y motivos de cancelación; control total del ciclo de vida del mensaje; depende solo del SDK, no de una integración de terceros con Spring Boot 4.x.
- En contra: más código de adaptador (construcción de expresiones, mapeo de items, bucle de consumo, apagado ordenado).

### Option B — Integración de alto nivel de Spring para AWS, con listeners declarativos de SQS y plantillas de DynamoDB

- A favor: menos código; listeners declarativos; gestión de confirmación integrada.
- En contra: compatibilidad con Spring Boot 4.x `TO_VERIFY`; el modelo de listener no es nativamente reactivo; menos control sobre visibilidad por mensaje y sobre los motivos de cancelación de transacciones.

### Option C — Cliente mejorado de mapeo de objetos para DynamoDB

Usar el cliente de mapeo de objetos del SDK para todas las operaciones.

- A favor: mapeo declarativo de items.
- En contra: el diseño de tabla única con claves sobrecargadas y transacciones con condiciones heterogéneas se expresa con más claridad y control con el cliente de bajo nivel; el mapeo declarativo acerca anotaciones de persistencia al modelo.

## Decision

Se adopta la **Option A**.

- DynamoDB: cliente asíncrono de bajo nivel del SDK oficial para todas las operaciones de `ticketing.data-model.md` §5. El mapeo entre items y modelo de dominio es explícito y vive en el adaptador (`CMP-010`).
- SQS: cliente asíncrono del SDK oficial. El consumidor (`CMP-012`) es un bucle reactivo: recibir con long polling, procesar con concurrencia acotada, eliminar o cambiar visibilidad según la clasificación de `ADR-010`, continuar ante errores, y detenerse ordenadamente al apagar (deja de recibir y termina lo que está en vuelo).
- El SDK asíncrono se adapta a tipos reactivos solo dentro de los adaptadores; los puertos exponen `Mono` y `Flux` (`TC-008`).
- Configuración del cliente por entorno: endpoint, región y credenciales por configuración; en local apuntan a los emuladores, en AWS se resuelven por el rol de la tarea.
- Reintentos del SDK acotados y sin apilar con reintentos de aplicación (`ADR-016`).

## Rationale

- El control sobre condiciones y motivos de cancelación no es opcional: de él depende distinguir "Ticket no disponible", "repetición" y "conflicto" (`ADR-002`).
- Depender únicamente del SDK oficial reduce el riesgo de compatibilidad con un framework recién publicado.
- Un bucle de consumo propio hace explícito el backpressure y el tratamiento por mensaje que exige `ADR-010`.

## Consequences

- El adaptador de DynamoDB concentra la complejidad de expresiones; debe probarse de forma unitaria y de integración (`ADR-019`).
- El apagado ordenado del consumidor es responsabilidad propia.
- Si la Option B resulta compatible, podría adoptarse más adelante para el consumidor sin cambiar puertos ni casos de uso.

## Risks and mitigations

| Risk | Mitigation |
|---|---|
| `RISK-012` incompatibilidad del SDK con Java 25 o con el cliente HTTP asíncrono | Verificación al inicio del desarrollo con una prueba mínima contra los emuladores |
| Errores en la construcción manual de expresiones | Pruebas unitarias del adaptador y de integración contra DynamoDB Local |

## Items to verify

| Item | Qué comprobar | Impacto si difiere |
|---|---|---|
| SDK oficial de AWS para Java con Java 25 | Versión compatible y cliente HTTP asíncrono soportado | Si no es compatible, el stack obligatorio queda en conflicto y debe escalarse |
| Integración de Spring para AWS con Spring Boot 4.x | Compatibilidad | Solo relevante si se reconsidera la Option B |

## Depends on

- Sin dependencias de `FG-*` ni `AV-*`.
