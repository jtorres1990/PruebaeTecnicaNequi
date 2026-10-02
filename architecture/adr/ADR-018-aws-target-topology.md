---
id: ADR-018
title: AWS target topology
status: PROPOSED
priority: MEDIUM
source_ids: [EVAL-005, EVAL-011, EVAL-012, EVAL-013, EVAL-006, EVAL-007, NFR-007, NFR-010, TC-004, TC-005, TC-016]
related_adrs: [ADR-001, ADR-010, ADR-013, ADR-015, ADR-017]
---

# ADR-018 — AWS target topology

## Context

`EVAL-005`, `EVAL-011`, `EVAL-012` y `EVAL-013` evalúan escalabilidad, resiliencia, infraestructura como código, experiencia cloud-native y operación. La especificación deja la topología AWS a arquitectura (§31). El detalle completo está en `ticketing.aws-target.md`; este ADR registra las decisiones y sus alternativas. No define Terraform.

La topología objetivo debe compartir el modelo lógico del entorno local (`ADR-017`).

## Options considered

### Option A — Contenedores sin servidor: dos servicios en un orquestador de contenedores gestionado, tras un balanceador con firewall de aplicación

Servicio `api` y servicio `worker` de la misma imagen, en subredes privadas; balanceador de aplicación público con firewall web; DynamoDB y SQS gestionados; Cognito como emisor.

- A favor: la misma imagen y los mismos procesos que en local; escalado independiente de API y worker; sin gestión de servidores; adecuado para procesos de larga vida como el consumidor con long polling y el bucle de expiración.
- En contra: costo base por tareas siempre activas; más piezas de red que una opción totalmente sin servidor.

### Option B — Funciones sin servidor: API por gateway y funciones, consumidor por integración nativa de la cola

- A favor: escala a cero; integración nativa con SQS; menor costo en reposo.
- En contra: modelo de ejecución distinto del local; una aplicación WebFlux de larga vida no encaja de forma natural; arranques en frío en picos; el proceso de expiración pasaría a ser una función planificada con precisión mínima de planificación `TO_VERIFY`.

### Option C — Kubernetes gestionado

- A favor: máxima flexibilidad y portabilidad.
- En contra: costo y complejidad operativa desproporcionados para dos servicios.

## Decision

Se adopta la **Option A**.

| Tema | Decisión |
|---|---|
| Cómputo | Orquestador de contenedores gestionado con capacidad sin servidor. Dos servicios (`api`, `worker`) de la misma imagen. El Payment Mock solo en entornos no productivos |
| Entrada | Balanceador de aplicación público con TLS y firewall de aplicación web con reglas por tasa. Alternativa evaluada: gateway de API con autorizador JWT; se descarta como primario porque la validación del JWT ya es responsabilidad del backend (`FR-018`) y añade costo por solicitud |
| Red | Una VPC por entorno en al menos dos zonas. Subredes públicas solo para el balanceador y la salida; subredes privadas para las tareas. Acceso a DynamoDB, SQS, registro de imágenes, logs y secretos por endpoints privados de la VPC. Salida a Internet solo para lo que no tenga endpoint privado |
| Aislamiento de red | Grupos de seguridad por servicio: el balanceador solo alcanza `api`; `worker` no recibe tráfico entrante; el Payment Mock solo recibe de `worker` |
| IAM | Un rol de tarea por servicio con mínimo privilegio sobre la tabla, sus índices y la cola concretos; rol de ejecución separado. Sin claves estáticas |
| Secretos | Gestor de secretos, inyectados en la tarea |
| Datos | DynamoDB on-demand con cifrado en reposo y recuperación a un punto en el tiempo; SQS Standard con cifrado y DLQ |
| Identidad | User pool de Cognito con grupos `ADMIN` y `CUSTOMER` y un cliente de aplicación |
| Escalabilidad | `api`: autoescalado por seguimiento de objetivo sobre CPU y solicitudes por destino. `worker`: autoescalado por mensajes pendientes por tarea. Mínimo de dos tareas por servicio en producción |
| Observabilidad | Logs estructurados centralizados, métricas técnicas y de negocio, trazas distribuidas con propagación a través del mensaje, alarmas y panel |
| Costo | On-demand para carga impulsiva; long polling; retención de logs acotada; equilibrio entre salida a Internet y endpoints privados; entornos no productivos reducidos |
| Aislamiento de entornos | Una cuenta de AWS por entorno; como mínimo, VPC, tabla, colas, user pool y estado de infraestructura separados por entorno |

## Rationale

- La opción A es la única en la que el artefacto desplegado y su modelo de ejecución son idénticos al local, lo que da validez a las pruebas locales.
- Separar `api` y `worker` permite que un pico de solicitudes y un pico de trabajo pendiente escalen con métricas distintas (`NFR-007`, `EVAL-005`).
- Endpoints privados y roles por servicio aplican mínimo privilegio y reducen la superficie de salida (`EVAL-006`, `EVAL-012`).

## Consequences

- Existe un costo base aunque no haya tráfico.
- Las claves públicas de Cognito se obtienen por su endpoint público salvo que exista endpoint privado; por eso se mantiene una salida controlada.
- El proceso de expiración corre en todas las tareas `worker`; es seguro por diseño (`ADR-009`).
- La infraestructura como código es un criterio diferencial (`EVAL-011`), no un entregable obligatorio; su materialización corresponde al agente Platform/IaC.

## Risks and mitigations

| Risk | Mitigation |
|---|---|
| Diferencias entre emuladores y servicios reales (`RISK-007`) | Entorno de desarrollo en AWS para pruebas de integración y de carga |
| Costo de la salida a Internet y de los endpoints privados | Una sola salida en no productivos; revisar qué endpoints compensan |

## Items to verify

| Item | Qué comprobar | Impacto si difiere |
|---|---|---|
| Endpoint privado para Cognito | Disponibilidad en la región | Si existe, puede eliminarse la salida a Internet del `api` |
| Métrica de mensajes pendientes por tarea para autoescalado | Forma soportada de publicarla y usarla | Alternativa: escalado por pasos sobre la profundidad de la cola |
| Precios de DynamoDB, cómputo, salida y endpoints | Tarifas vigentes en la región | Solo afecta la estimación de costo |
| Exportación de métricas y trazas desde la aplicación | Mecanismo soportado por Spring Boot 4.x | Cambia el agente o colector, no el diseño |

## Depends on

- Sin dependencias de `FG-*`. `FG-003` añadiría la operación de cancelación hacia el proveedor, sin cambio de topología.
