---
id: ADR-032
title: Security, authorization and abuse controls including one active Order per customer and Event
status: ACCEPTED
priority: HIGH
supersedes: ADR-013
human_decision: CONFIRMED_WITH_CHANGE
source_ids: [FR-018, FR-019, BR-023, VAL-011, NFR-010, TC-016, AC-027, AC-028, ALT-007, ERR-006, ERR-009, EVAL-006, EVAL-007, EVAL-008, MF-004]
related_adrs: [ADR-003, ADR-022, ADR-023, ADR-024, ADR-025, ADR-027, ADR-033, ADR-035, ADR-037]
---

# ADR-032 — Security, authorization and abuse controls including one active Order per customer and Event

Reemplaza a ADR-013.

## Human decision applied

Respuesta humana vinculante a ADR-013 (CONFIRMED_WITH_CHANGE), incorporada íntegramente:

> Se confirma la validación del JWT en la aplicación como Resource Server (firma, emisor, expiración, tipo de token de acceso y cliente permitido), el mapeo de `cognito:groups` a las autoridades `ADMIN` y `CUSTOMER`, la autorización por rol en el punto de entrada y por propiedad en el caso de uso, el propietario tomado siempre del sujeto del JWT, y la respuesta idéntica para una Order ajena y una inexistente. Se confirman el manejo de secretos fuera del repositorio, sin claves estáticas en AWS y sin registrar tokens ni API keys, y las defensas contra abuso por idempotencia, límites de entrada y limitación por tasa. Se descartan la validación delegada exclusivamente al borde y la propiedad impuesta por la clave de persistencia. Se añaden cuatro cambios.
>
> Operaciones nuevas. La consulta del estado de aprovisionamiento de un Event introducida en ADR-004 requiere la autoridad `ADMIN`. La `Idempotency-Key` de la creación de Events introducida en ADR-007 se vincula al sujeto del `ADMIN` autenticado. La tabla de autorización por operación y OpenAPI deben incluirlas.
>
> Tamaño de cuerpo. Con la definición compacta del inventario aprobada en ADR-004, ninguna operación necesita cuerpos de varios MB. El límite de cuerpo en memoria del servidor se mantiene bajo y explícito, y una solicitud que lo supere se rechaza sin procesarse.
>
> Endpoints de gestión. Solo el endpoint de salud puede ser accesible desde el exterior. Las métricas y el resto de endpoints de gestión se exponen en un puerto interno, no publicado por el balanceador, y no deben revelar configuración ni secretos.
>
> Una Order activa por cliente y por Event. Un `CUSTOMER` no puede tener más de una Order en `CREATED` para el mismo Event. La transacción de reserva crea un item de bloqueo identificado por cliente y Event con la condición de que no exista, y toda transición terminal lo elimina en la misma escritura transaccional, conforme a ADR-005. Una segunda solicitud de compra para el mismo Event mientras la primera está activa se rechaza síncronamente, sin crear Reservation, Order ni Order ID y sin modificar el inventario. Una repetición con la misma `Idempotency-Key` sigue resolviéndose como repetición conforme a ADR-007 y no como segunda compra. Se acepta como limitación declarada que esta regla no impide el acaparamiento mediante varias cuentas, que queda a cargo del control de bots en el borde, y que una Order en cuarentena mantiene el bloqueo hasta su revisión manual. Esta regla funcional debe propagarse a la especificación funcional.

Decisiones aprobadas incorporadas: `AV-005` (un `ADMIN` lista Events, consulta disponibilidad y estado de aprovisionamiento; no compra ni consulta Orders salvo que sea también `CUSTOMER`); consulta de estado de ADR-024; idempotencia de ADR-027; transiciones con bloqueo de ADR-025; `ACTIVE_ORDER_EXISTS` 409 (ADR-035); control de bots en el borde (ADR-037); identidades locales (ADR-033).

## Context

Las operaciones protegidas requieren JWT de Cognito validado por el backend (`FR-018`, `TC-016`); `ADMIN` crea Events y `CUSTOMER` consulta, compra y consulta sus Orders (`FR-019`, `AC-028`); la propiedad se deriva del JWT (`VAL-011`) y una Order ajena es indistinguible de una inexistente (`BR-023`, `AC-027`). `EVAL-006` a `EVAL-008` evalúan seguridad, secretos y abuso.

## Options considered

### Option A — Resource Server en la aplicación, autorización por rol en la entrada y por propiedad en el caso de uso, bloqueo de Order activa en la transacción de reserva

- A favor: igual en local y en AWS; la propiedad y el bloqueo residen en el núcleo y en la atomicidad del almacén.
- En contra: cada instancia valida tokens; el abuso volumétrico necesita además el borde.

### Option B — Validación delegada exclusivamente al borde

- En contra: contradice "el backend actúa como Resource Server"; no hay borde en local. Descartada.

### Option C — Propiedad impuesta por la clave de persistencia

- En contra: el consumidor y la expiración solo conocen `orderId`. Descartada.

### Option D — Límite de Order activa comprobado con una lectura previa, fuera de la transacción

- A favor: sin item adicional.
- En contra: dos compras simultáneas del mismo cliente pasarían ambas la lectura; no es atómico. Descartada en favor del item de bloqueo condicional.

## Decision

Se adopta la **Option A**, complementada en AWS con controles de borde (ADR-037).

### Validación del JWT

Firma con algoritmo asimétrico esperado, emisor, expiración con tolerancia de 60 s, tipo de token de acceso y cliente permitido; sin sesión. Configuración solo por URL de emisor, URL de claves y nombres de claims (ADR-033).

### Mapeo de grupos

Cada valor de `cognito:groups` igual a `ADMIN` o `CUSTOMER` se convierte en la autoridad correspondiente; otros se ignoran; sin grupos reconocidos no hay autoridades (prohibido en toda operación protegida).

### Autorización por operación

| Operación | Autoridad |
|---|---|
| `API-001` Crear Event | `ADMIN`; la `Idempotency-Key` se vincula al sujeto del `ADMIN` |
| `API-002` Listar Events | `ADMIN` o `CUSTOMER` (`AV-005`) |
| `API-003` Consultar disponibilidad | `ADMIN` o `CUSTOMER` (`AV-005`) |
| `API-004` Iniciar compra | `CUSTOMER` (un `ADMIN` solo si también es `CUSTOMER`) |
| `API-005` Consultar Order | `CUSTOMER` y propiedad |
| `API-006` Consultar estado de aprovisionamiento | `ADMIN` |

Sin token o token inválido: 401. Token válido sin autoridad: 403.

### Propietario y Order ajena

`customerId` = sujeto del JWT; ninguna operación acepta un identificador de usuario. Inexistente o ajena producen el mismo resultado de dominio y la misma respuesta HTTP; un identificador malformado también; misma lectura en ambos casos; identificadores aleatorios.

### Una Order activa por cliente y por Event

| Aspecto | Decisión |
|---|---|
| Regla | Un `CUSTOMER` no puede tener más de una Order en `CREATED` para el mismo Event. Es regla de negocio y reside en `domain` (ADR-034) |
| Mecanismo | Item `ACTIVE#<customerId>#<eventId>` con `orderId`, creado en la transacción de reserva con condición "no existe" (ADR-023) |
| Liberación | Eliminado en toda transición terminal en la misma transacción (Confirmar, Rechazar, Fallar, Expirar; ADR-025) |
| Segunda compra | 409 `ACTIVE_ORDER_EXISTS`, sin Reservation, Order ni Order ID y sin modificar inventario |
| Repetición con la misma clave | Se resuelve como repetición (ADR-027), nunca como segunda compra |
| Cuarentena | El bloqueo se mantiene hasta la revisión manual |
| Limitaciones declaradas | No impide el acaparamiento con varias cuentas (control de bots en el borde, ADR-037); bloqueo retenido por una Order en cuarentena |

### Tamaño de cuerpo

Límite de cuerpo en memoria bajo y explícito: 256 KB (configurable), suficiente para la definición compacta máxima (ADR-024). Una solicitud que lo supera se rechaza con 413 `PAYLOAD_TOO_LARGE` sin procesarse.

### Endpoints de gestión

Solo el endpoint de salud es accesible desde el exterior (puerto de la aplicación, registrado en el balanceador). Métricas y demás endpoints de gestión en un puerto interno no publicado por el balanceador ni al host salvo en local; sin exposición de configuración, variables ni secretos.

### Secretos y credenciales

| Aspecto | Local | AWS |
|---|---|---|
| Credenciales de AWS | Valores ficticios por variables de entorno | Roles de tarea; sin claves estáticas |
| API key del Payment Mock | Archivo de entorno no versionado | Gestor de secretos |
| Token de LocalStack, si se usa la versión actual (ADR-036) | Archivo no versionado | No aplica |
| Repositorio e imagen | Sin secretos; solo ejemplo con valores ficticios | Igual |

Nunca se registran tokens, cabeceras de autorización ni API keys.

### Abuso

- Idempotencia obligatoria en compra y creación (ADR-027).
- Límites de entrada: 1..10 Ticket por Order (ADR-003), 50.000 Ticket por Event y límites de la definición (ADR-024), tamaño de cuerpo, formatos estrictos, tamaño de página acotado, cursor validado.
- Una Order activa por cliente y Event.
- Limitación por tasa: en el borde con reglas por tasa y control de bots (ADR-037); en la aplicación, limitador por sujeto en `API-004` (en memoria, no bloqueante) que responde 429.
- Vigencia de la Reservation de diez minutos.
- Errores sin detalle interno (ADR-035).
- Payment Mock con API key y solo en red interna; mensajes sin datos personales; CORS deshabilitado; TLS en el borde.

## Rationale

- La propiedad y el bloqueo en el núcleo y en la transacción no pueden saltarse por otra vía de entrada.
- El item de bloqueo condicional es la única forma atómica de impedir dos Orders activas simultáneas del mismo cliente en el mismo Event.

## Consequences

- La transacción de reserva añade un item (14 en total) y las terminales una eliminación (13).
- La prueba de carga necesita más de 1.000 identidades `CUSTOMER` distintas (ADR-033, ADR-038).
- El limitador en memoria es aproximado con varias instancias; el autoritativo es el borde.

## Risks and mitigations

| Risk | Mitigation |
|---|---|
| `RISK-010` acaparamiento | Máximo por Order, una Order activa por cliente y Event, limitación por tasa, control de bots, expiración |
| `RISK-013` token local distinto del de Cognito | Prueba de contrato de claims; perfil contra user pool real |
| `RISK-018` bloqueo retenido por Orders en cuarentena | Alarma y revisión manual |

## Items to verify

| Item | Qué comprobar | Impacto si difiere |
|---|---|---|
| Claims del token de acceso de Cognito | Grupos, tipo de token, cliente, sujeto; ausencia de audiencia | Ajustar validadores |
| Resource Server reactivo, límite de cuerpo en memoria y puerto de gestión separado en Spring Boot 4.x | Dependencias y propiedades | Solo implementación |
| Librería de limitación por tasa | Compatibilidad con el stack | Componente propio |

## Depends on

- `AV-005`, `FG-001`, `FG-002` (resueltos).
