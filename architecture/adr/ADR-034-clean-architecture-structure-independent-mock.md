---
id: ADR-034
title: Clean Architecture structure with independent Payment Mock project and updated ports
status: ACCEPTED
priority: HIGH
supersedes: ADR-015
human_decision: CONFIRMED_WITH_CHANGE
source_ids: [TC-007, TC-013, TC-015, NFR-012, DEL-001, TC-001, TC-002, TC-003, TC-008, TC-017, NFR-003, NFR-013]
related_adrs: [ADR-024, ADR-025, ADR-026, ADR-027, ADR-028, ADR-030, ADR-032, ADR-035, ADR-038, ADR-039]
---

# ADR-034 — Clean Architecture structure with independent Payment Mock project and updated ports

Reemplaza a ADR-015.

## Human decision applied

Respuesta humana vinculante a ADR-015 (CONFIRMED_WITH_CHANGE), incorporada íntegramente:

> Se confirma para la aplicación `ticketing` el build multi-módulo (`domain`, `application`, `infrastructure`, `bootstrap`) con la dirección de dependencias impuesta por el compilador, `domain` y `application` sin tipos de Spring, del SDK de AWS ni de HTTP, puertos de entrada por caso de uso, puertos de salida con intención de negocio y una única imagen con los roles `api` y `worker`. Se descartan el módulo único con paquetes por capa y los módulos por funcionalidad. Se añaden tres cambios.
>
> Payment Mock como proyecto independiente. `payment-mock` no forma parte del build de `ticketing`. Es un proyecto independiente dentro del mismo repositorio, en su propia carpeta, con su propio build, Dockerfile y pruebas, sin código compartido con `ticketing` en ninguna dirección. No existe una librería común de DTOs: el adaptador de pago de `infrastructure` define sus propios tipos y solo se acopla al contrato HTTP del mock, que se documenta en un OpenAPI propio. La aplicación accede al mock exclusivamente a través del puerto `Payment gateway` y su adaptador HTTP. La cobertura mínima del 90 % se mide solo sobre `ticketing`; el mock mantiene sus propias pruebas fuera de esa métrica. Docker Compose construye ambos proyectos por separado y la aplicación alcanza el mock por nombre de servicio. ADR-011 debe ajustarse de "módulo" a "proyecto independiente".
>
> Puertos actualizados. El catálogo de puertos debe incorporar lo aprobado en esta revisión: procesar el aprovisionamiento de un Event y consultar su estado, y publicar en la cola de aprovisionamiento (ADR-004); el barrido de republicación (ADR-006); el proceso de reversos de pago y la operación de cancelación obligatoria en el `Payment gateway` (FG-003); la operación de cuarentena y el bloqueo de Order activa por cliente y Event como parte de las transiciones del `Order lifecycle store` (ADR-005, ADR-013); y la idempotencia de la creación de Events (ADR-007). El rol `worker` ejecuta cada proceso periódico con planificación independiente conforme a ADR-009.
>
> Ubicación de las reglas nuevas. La regla de una Order activa por cliente y Event, el criterio de cuarentena, la política de reverso de pagos y el ciclo de vida del aprovisionamiento (`PROVISIONING`, `ENABLED`, `FAILED`) son reglas de negocio y residen en `domain`. El adaptador de DynamoDB solo las expresa como condiciones y transacciones, conforme a la regla de frontera que separa la decisión de negocio de su atomicidad.

Correspondencia con los sucesores: ADR-004 → ADR-024, ADR-005 → ADR-025, ADR-006 → ADR-026, ADR-007 → ADR-027, ADR-009 → ADR-028, ADR-011 → ADR-030, ADR-013 → ADR-032. Además: `AV-006` (cobertura sobre `ticketing`), circuit breaker en adaptadores con resultado tipado de dependencia no disponible (ADR-035).

## Context

`TC-007` exige capas Domain, Use Cases e Infrastructure separadas; `TC-013` código en inglés; `TC-015` y `NFR-012` SOLID; `DEL-001` repositorio con esa estructura; `TC-017` un Payment Mock independiente.

## Options considered

### Option A — Build multi-módulo de `ticketing` con dependencias impuestas por el compilador, y `payment-mock` como proyecto separado

- A favor: dependencia prohibida no compila; independencia del mock verificable en el repositorio.
- En contra: más configuración de build; dos builds.

### Option B — Módulo único con paquetes por capa y pruebas de arquitectura

- En contra: la separación depende de una prueba. Descartada.

### Option C — Módulos por funcionalidad

- En contra: frontera artificial entre Order y Ticket que comparten transacción. Descartada.

### Option D — `payment-mock` como módulo del mismo build con DTOs compartidos

- A favor: un solo build.
- En contra: acopla el mock a `ticketing` y debilita `TC-017`; la decisión humana lo prohíbe. Descartada.

## Decision

Se adopta la **Option A**.

### Proyectos y módulos

| Proyecto / módulo | Capa | Responsabilidad | Puede depender de |
|---|---|---|---|
| `ticketing/domain` | Domain | Modelo; máquinas de estado; invariantes; validaciones; políticas temporales; regla de una Order activa por cliente y Event; criterio de cuarentena; política de reverso; ciclo de aprovisionamiento; generación determinista de `ticketId` y de shards; errores de dominio | Solo la biblioteca estándar |
| `ticketing/application` | Use Cases | Casos de uso; puertos de entrada y salida; orquestación reactiva | `domain` y tipos de Reactor |
| `ticketing/infrastructure` | Infrastructure | Adaptadores HTTP, consumidores de SQS, planificador, DynamoDB, SQS, pago, reloj, identificadores, seguridad, errores, resiliencia, observabilidad | `application`, `domain`, frameworks, SDKs |
| `ticketing/bootstrap` | Infrastructure | Composición, configuración, activación de roles `api` y `worker` | Todos los anteriores |
| `payment-mock` (proyecto independiente) | Servicio independiente | Simulador (ADR-030) | Nada de `ticketing` |

`domain` y `application` sin tipos de Spring, del SDK de AWS ni de HTTP.

### Puertos de entrada

| Puerto | Rol | Componente |
|---|---|---|
| Create Event | `api` | `CMP-003` |
| Get Event provisioning status | `api` | `CMP-003` |
| List Events | `api` | `CMP-004` |
| Get Event availability | `api` | `CMP-004` |
| Start purchase | `api` | `CMP-005` |
| Get Order | `api` | `CMP-006` |
| Process Order | `worker` | `CMP-007` |
| Provision Event | `worker` | `CMP-022` |
| Expire Reservations | `worker` | `CMP-008` |
| Republish pending Orders | `worker` | `CMP-023` |
| Reverse payments | `worker` | `CMP-024` |
| Clean up provisioning | `worker` | `CMP-015` |

### Puertos de salida

| Puerto | Propósito |
|---|---|
| Event catalog | Crear un Event en `PROVISIONING` con su idempotencia y auditoría; obtener (con y sin consistencia fuerte); listar habilitados; tomar y renovar el lease de aprovisionamiento con progreso; habilitar; marcar `FAILED`; localizar estancados y fallidos; registrar republicación y purga |
| Ticket inventory | Escribir un lote de Ticket; verificar la existencia de todas las claves; purgar los Ticket de un Event `FAILED`; obtener una página de disponibles; contar disponibles; sondear agotado |
| Order lifecycle store | Transiciones atómicas de ADR-025 con resultado tipado (aplicado, condición no cumplida con motivo por item, conflicto): reservar y crear (con idempotencia y bloqueo), iniciar pago, reclamar lease, confirmar, cerrar con liberación (con eliminación del bloqueo y marca de reverso), marcar `enqueuedAt`, poner en cuarentena, registrar aprobación tardía, completar o agotar reverso, programar el siguiente intento de reverso |
| Order reader | Obtener una Order; buscar Reservation vencidas; buscar Orders pendientes de encolado; buscar reversos pendientes |
| Idempotency store | Obtener el registro de idempotencia de compra y de creación de Event |
| Order queue publisher | Publicar `MSG-001` |
| Provisioning queue publisher | Publicar `MSG-002` |
| Payment gateway | Autorizar y cancelar un PaymentAttempt; devuelve resultados tipados, incluido "dependencia no disponible" |
| Clock | Instante actual |
| Id generator | Identificadores de Order, Event y Reservation |

### Reglas de frontera

1. Las reglas de negocio residen en `domain`; el adaptador de DynamoDB solo las expresa como condiciones y transacciones.
2. La auditoría la construye el dominio y la persiste el adaptador en la misma transacción (ADR-031).
3. Los adaptadores de entrada traducen transporte a comandos y errores a respuestas, sin reglas de negocio.
4. Autorización por rol en el adaptador HTTP; propiedad en el caso de uso (ADR-032).
5. Errores de dominio como jerarquía cerrada con mapeo exhaustivo (ADR-035).
6. Retry y circuit breaker solo en adaptadores de `infrastructure` (ADR-035).
7. El adaptador de pago define sus propios tipos y solo conoce el contrato HTTP del mock.
8. Código, nombres y comentarios en inglés (`TC-013`).

### Unidades de despliegue

Una imagen de `ticketing` con roles `api` y `worker`. El rol `worker` ejecuta los consumidores de Orders y de aprovisionamiento y cuatro procesos periódicos con planificación independiente (ADR-028). Una imagen propia de `payment-mock`.

### Características del lenguaje

Records, interfaces selladas y pattern matching. Sin Virtual Threads (`TC-001` las declara condicionales; toda la E/S es reactiva).

## Rationale

- El compilador impone `TC-007`; los proyectos separados imponen `TC-017`.
- Ubicar las reglas nuevas en `domain` las hace probables sin infraestructura (ADR-038).

## Consequences

- El puerto Order lifecycle store crece en operaciones; sigue siendo de intención de negocio.
- La cobertura del 90 % se mide solo sobre `ticketing` (`AV-006`).

## Risks and mitigations

| Risk | Mitigation |
|---|---|
| `RISK-012` compatibilidad de herramientas con Java 25 y Spring Boot 4.x | Verificación al inicio |
| Lógica de negocio filtrada al adaptador | Pruebas de arquitectura y revisión |

## Items to verify

| Item | Qué comprobar | Impacto si difiere |
|---|---|---|
| Herramienta de build multi-proyecto con Java 25 y el plugin de Spring Boot 4.x | Compatibilidad | Cambia la herramienta, no la estructura |
| Librería de pruebas de arquitectura con Java 25 | Compatibilidad | Reglas internas por revisión |
| Imagen base de Java 25 | Disponibilidad | Cambia la imagen |

## Depends on

- `FG-003`, `AV-006` (resueltos).
