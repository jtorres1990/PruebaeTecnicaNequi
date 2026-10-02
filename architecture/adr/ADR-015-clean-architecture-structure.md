---
id: ADR-015
title: Clean Architecture structure
status: PROPOSED
priority: HIGH
source_ids: [TC-007, TC-013, TC-015, NFR-012, DEL-001, TC-001, TC-002, TC-003, TC-008, NFR-003]
related_adrs: [ADR-005, ADR-016, ADR-019, ADR-020]
---

# ADR-015 — Clean Architecture structure

## Context

`TC-007` exige capas Domain, Use Cases e Infrastructure claramente separadas; `TC-013` código en inglés; `TC-015` y `NFR-012` SOLID y patrones apropiados; `DEL-001` un repositorio con esa estructura. El stack es Java 25, Spring Boot 4.x y WebFlux (`TC-001` a `TC-003`).

Hay que decidir módulos, capas, puertos y dirección de dependencias, sin listar clases.

## Options considered

### Option A — Build multi-módulo con la dirección de dependencias impuesta por el compilador

Módulos `domain`, `application`, `infrastructure`, `bootstrap` y, aparte, `payment-mock`.

- A favor: una dependencia prohibida no compila; la separación es evidente al abrir el repositorio; permite medir cobertura por capa.
- En contra: más configuración de build.

### Option B — Módulo único con paquetes por capa y reglas verificadas por pruebas de arquitectura

- A favor: build más simple.
- En contra: la separación depende de una prueba, no del compilador; es fácil filtrar anotaciones del framework al dominio.

### Option C — Módulos por funcionalidad (events, orders, payments), cada uno con sus capas

- A favor: escala mejor con muchos equipos.
- En contra: las transiciones abarcan Order y Ticket en una sola transacción, por lo que la frontera entre funcionalidades sería artificial; excesivo para el tamaño de la solución.

## Decision

Se adopta la **Option A**, complementada con pruebas de arquitectura que verifican las reglas dentro de `infrastructure`.

### Módulos y capas

| Módulo | Capa | Responsabilidad | Puede depender de |
|---|---|---|---|
| `domain` | Domain | Modelo (Event, Ticket, Order, Reservation, PaymentAttempt), máquinas de estado de Ticket y Order, invariantes de §5.1, validaciones de dominio, políticas (vigencia de la Reservation, margen de corte, clasificación de resultado de pago), errores de dominio | Solo la biblioteca estándar de Java |
| `application` | Use Cases | Casos de uso, puertos de entrada y de salida, orquestación reactiva | `domain` y los tipos reactivos de Reactor |
| `infrastructure` | Infrastructure | Adaptadores de entrada (HTTP, consumidor de SQS, planificador) y de salida (DynamoDB, SQS, pago, reloj, identificadores), seguridad, mapeo de errores, observabilidad | `application`, `domain`, frameworks y SDKs |
| `bootstrap` | Infrastructure | Composición, configuración por entorno, activación de roles `api` y `worker` | Todos los anteriores |
| `payment-mock` | Servicio independiente | Simulador de pagos (`ADR-011`) | Ninguno de los anteriores |

Dirección de dependencias: `bootstrap` → `infrastructure` → `application` → `domain`. Nunca al revés. `domain` y `application` no contienen anotaciones ni tipos de Spring, del SDK de AWS ni de HTTP.

### Puertos de entrada (casos de uso)

Crear Event; listar Events; consultar disponibilidad; iniciar compra; consultar Order; procesar Order; expirar Reservations; limpiar Events incompletos. Cada uno recibe datos ya validados sintácticamente y la identidad autenticada cuando aplica; devuelve tipos reactivos (`TC-008`).

### Puertos de salida

| Puerto | Propósito |
|---|---|
| Event catalog | Crear, habilitar, obtener y listar Events; sondear agotado |
| Ticket inventory | Escribir el inventario inicial y leer los Ticket de un Event |
| Order lifecycle store | Ejecutar atómicamente las transiciones de negocio de `ADR-005` (reservar y crear, iniciar pago, reclamar lease, confirmar, cerrar con liberación) y devolver un resultado tipado: aplicado, condición no cumplida, conflicto |
| Order reader | Obtener una Order por ID; buscar Reservation vencidas |
| Idempotency store | Obtener el registro de idempotencia |
| Order queue publisher | Publicar el mensaje de procesamiento |
| Payment gateway | Autorizar un PaymentAttempt (y cancelarlo, según `FG-003`) |
| Clock | Instante actual, para lógica temporal determinista |
| Id generator | Identificadores de Order, Event y Reservation |

El detalle de qué adaptador implementa cada puerto está en `ticketing.architecture.md` §6.

### Reglas de frontera

1. Las reglas de negocio (qué transición es válida, cuándo expira, qué causa se registra) viven en `domain`; la forma de hacerlas atómicas (condiciones y transacción) vive en el adaptador de DynamoDB, detrás del puerto Order lifecycle store.
2. El registro de auditoría lo construye el dominio como descripción de la transición y lo persiste el adaptador dentro de la misma transacción (`ADR-012`).
3. Los adaptadores de entrada traducen transporte a comandos y errores de dominio a respuestas; no contienen reglas de negocio.
4. La autorización por rol está en el adaptador HTTP; la autorización por propiedad está en el caso de uso (`ADR-013`).
5. Los errores de dominio son una jerarquía cerrada; el mapeo a HTTP es exhaustivo (`ADR-016`).
6. Todo el código, nombres y comentarios en inglés (`TC-013`).

### Unidades de despliegue

Una sola imagen de la aplicación con dos roles activables por configuración: `api` (puntos de entrada HTTP) y `worker` (consumidor de SQS, proceso de expiración y limpieza). En local y en AWS se ejecutan como procesos separados de la misma imagen.

### Características del lenguaje

Records para valores y comandos, interfaces selladas y pattern matching para resultados y errores. No se usan Virtual Threads: toda la E/S es no bloqueante sobre el modelo reactivo (`NFR-003`); `TC-001` las declara condicionales.

## Rationale

- Imponer la regla de dependencias con el compilador es la forma más fuerte de cumplir `TC-007` y la más fácil de demostrar.
- Un puerto de persistencia con operaciones de intención de negocio evita que el caso de uso conozca transacciones de DynamoDB y permite probarlo con un doble en memoria que respete la misma semántica (`ADR-019`).
- Dos roles sobre una imagen permiten escalar la API y el worker de forma independiente (`NFR-007`) sin duplicar código.

## Consequences

- El puerto Order lifecycle store es de grano grueso: un cambio en una transición afecta al dominio y al adaptador.
- `application` depende de los tipos de Reactor; se acepta como parte del contrato reactivo (`TC-008`).
- La herramienta de build y su compatibilidad con Java 25 deben verificarse antes de crear el esqueleto.

## Risks and mitigations

| Risk | Mitigation |
|---|---|
| `RISK-012` compatibilidad de herramientas con Java 25 y Spring Boot 4.x | Verificación al inicio del desarrollo; alternativas en "Items to verify" |
| Lógica de negocio que se filtra al adaptador de persistencia | Pruebas de arquitectura y revisión; el dominio decide, el adaptador solo expresa condiciones |

## Items to verify

| Item | Qué comprobar | Impacto si difiere |
|---|---|---|
| Herramienta de build | Versión que soporte Java 25 y el plugin de Spring Boot 4.x. Se propone Gradle multi-proyecto; Maven multi-módulo es equivalente | Cambia la herramienta, no la estructura |
| Librería de pruebas de arquitectura | Soporte del formato de clases de Java 25 | Si no hay, las reglas internas de `infrastructure` se verifican por revisión |
| Imagen base de ejecución para Java 25 | Disponibilidad | Cambia la imagen base |

## Depends on

- `FG-003`: operación de cancelación en el puerto Payment gateway.
