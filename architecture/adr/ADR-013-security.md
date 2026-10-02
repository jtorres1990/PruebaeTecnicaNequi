---
id: ADR-013
title: Security
status: PROPOSED
priority: HIGH
source_ids: [FR-018, FR-019, BR-023, VAL-011, NFR-010, TC-016, AC-027, AC-028, ALT-007, ERR-006, ERR-009, EVAL-006, EVAL-007, EVAL-008, MF-004]
related_adrs: [ADR-007, ADR-014, ADR-016, ADR-018]
---

# ADR-013 — Security

## Context

Las operaciones protegidas requieren un JWT emitido por Amazon Cognito; el backend actúa como Resource Server y obtiene los roles del claim `cognito:groups` (`FR-018`, `TC-016`). `ADMIN` crea Events; `CUSTOMER` consulta, compra y consulta solo sus Orders (`FR-019`, `AC-028`). La propiedad se deriva del JWT (`VAL-011`) y una Order ajena debe ser indistinguible de una inexistente (`BR-023`, `AC-027`). `EVAL-006`, `EVAL-007` y `EVAL-008` evalúan seguridad general, secretos y resistencia a reintentos maliciosos y abuso.

## Options considered

### Option A — Validación del JWT en la aplicación como Resource Server, con autorización en dos niveles

La aplicación valida firma y claims contra el emisor configurado, mapea grupos a autoridades, autoriza por rol en el punto de entrada y por propiedad en el caso de uso.

- A favor: cumple `FR-018` literalmente; funciona igual en local y en AWS; la autorización por propiedad queda en el núcleo, independiente del transporte.
- En contra: cada instancia valida tokens; la protección contra abuso volumétrico necesita además un control en el borde.

### Option B — Validación delegada exclusivamente al borde

Un gateway valida el JWT y la aplicación confía en cabeceras inyectadas.

- A favor: aplicación más simple.
- En contra: contradice "el backend actúa como Resource Server"; no existe gateway en el entorno local; cualquier acceso que evite el borde queda sin autenticar.

### Option C — Autorización por propiedad en la consulta de persistencia

Incluir el propietario en la clave de la Order para que una Order ajena sea físicamente inalcanzable.

- A favor: el aislamiento lo impone el almacén.
- En contra: el consumidor y el proceso de expiración solo conocen el `orderId`; obligaría a propagar el propietario en el mensaje y en el índice. La opción A ya logra la indistinguibilidad con una comparación en el caso de uso.

## Decision

Se adopta la **Option A**, complementada en AWS con controles de borde (`ADR-018`).

### Validación del JWT

- Resource Server reactivo configurado con el emisor y su conjunto de claves públicas.
- Validaciones: firma con algoritmo asimétrico esperado, emisor, expiración con tolerancia de reloj de 60 s, tipo de token de acceso, y cliente permitido.
- Sin sesión de servidor; cada solicitud se autentica por su token.
- La configuración se expresa solo como URL de emisor, URL de claves y nombres de claims, de modo que el proveedor local sea intercambiable (`ADR-014`).

### Mapeo de grupos a autoridades

- Cada valor de `cognito:groups` igual a `ADMIN` o `CUSTOMER` se convierte en la autoridad de aplicación correspondiente. Otros grupos se ignoran. Un token sin grupos reconocidos no tiene autoridades.

### Autorización por operación

| Operación | Autoridad requerida |
|---|---|
| `API-001` Crear Event | `ADMIN` |
| `API-002` Listar Events | `ADMIN` o `CUSTOMER` (`AV-005`) |
| `API-003` Consultar disponibilidad | `ADMIN` o `CUSTOMER` (`AV-005`) |
| `API-004` Iniciar compra | `CUSTOMER` |
| `API-005` Consultar Order | `CUSTOMER`, y además propiedad |

Sin token o con token inválido: no autenticado. Con token válido sin la autoridad requerida: prohibido.

### Propietario de la Order

- `customerId` = claim de sujeto del JWT, tomado en el punto de entrada y pasado al caso de uso como identidad autenticada. Ninguna operación acepta un identificador de usuario en ruta, consulta o cuerpo (`VAL-011`).

### Order ajena e inexistente indistinguibles

- El caso de uso lee la Order por ID; si no existe **o** su `customerId` difiere del sujeto autenticado, produce el mismo resultado de dominio "no encontrada".
- La respuesta HTTP es idéntica en estado, cuerpo y cabeceras en ambos casos; un identificador con formato inválido produce la misma respuesta.
- En ambos casos se ejecuta la misma lectura; no hay camino corto que diferencie tiempos de forma apreciable.
- Los identificadores de Order son aleatorios, no secuenciales.

### Secretos y credenciales

| Aspecto | Local | AWS objetivo |
|---|---|---|
| Credenciales de AWS | Valores ficticios por variables de entorno para los emuladores | Roles de tarea IAM; sin claves estáticas |
| API key del Payment Mock | Variable de entorno desde un archivo no versionado | Gestor de secretos, inyectado en la tarea |
| Validación de JWT | Solo claves públicas; el backend no guarda secreto de cliente | Igual |
| Repositorio e imagen | Sin secretos; solo un archivo de ejemplo con valores ficticios | Igual |

Logs: nunca se registran tokens, cabecera de autorización ni API keys.

### Reintentos maliciosos y abuso de recursos

- **Idempotencia obligatoria** en la compra, vinculada al sujeto y al contenido (`ADR-007`): repetir una solicitud no multiplica efectos.
- **Límites de entrada**: máximo de Ticket por Order (`FG-001`), capacidad máxima por Event (`FG-002`), tamaño máximo de cuerpo, formato estricto de identificadores, tamaño de página acotado.
- **Rate limiting**: en AWS, reglas por tasa en el borde (`ADR-018`). En la aplicación, un limitador por sujeto autenticado en `API-004`, en memoria y no bloqueante, como defensa local y adicional; responde "demasiadas solicitudes".
- **Vigencia de la Reservation**: el inventario retenido se libera a los diez minutos.
- **Errores sin detalle interno** (`ADR-016`).
- **Payment Mock**: llamada autenticada con API key y accesible solo por red interna.
- **Mensajes**: sin datos personales; solo los roles de la aplicación pueden publicar y consumir (`ADR-018`).
- **CORS**: deshabilitado por defecto; no hay frontend en el alcance.
- **Transporte**: TLS terminado en el borde en AWS; HTTP en local.

## Rationale

- Validar en la aplicación es lo que la especificación pide y lo que mantiene idéntico el modelo en ambos entornos.
- Autorizar la propiedad en el caso de uso garantiza que ninguna vía de entrada futura pueda saltarse la regla.
- Tratar "no existe" y "no es tuya" como un único resultado de dominio hace la indistinguibilidad verificable con una prueba unitaria (`AC-027`).
- Las defensas contra abuso combinan idempotencia (ya obligatoria), límites de tamaño y limitación por tasa, que son las que responden a `EVAL-008` sin introducir reglas de negocio nuevas.

## Consequences

- El limitador en memoria es aproximado cuando hay varias instancias; la limitación autoritativa es la del borde.
- Un `ADMIN` que no pertenezca también al grupo `CUSTOMER` no puede comprar ni consultar Orders.
- No se introduce un límite de Reservation activas por CUSTOMER; sería una regla funcional nueva (`RISK-010`).
- La rotación de claves del emisor depende de la caché de claves del Resource Server.

## Risks and mitigations

| Risk | Mitigation |
|---|---|
| `RISK-010` acaparamiento de inventario mediante reservas repetidas | Rate limiting por sujeto, máximo por Order, expiración a los diez minutos; un límite por CUSTOMER requeriría decisión funcional |
| `RISK-013` el token local no reproduce exactamente el de Cognito | Prueba de contrato de claims; perfil contra un user pool real (`ADR-014`) |
| Uso de un token de identidad en lugar de uno de acceso | Validación del tipo de token |

## Items to verify

| Item | Qué comprobar | Impacto si difiere |
|---|---|---|
| Claims del token de acceso de Cognito | Presencia y nombre de grupos, tipo de token, cliente y sujeto; ausencia de audiencia en el token de acceso | Ajustar los validadores de claims |
| Soporte de Resource Server reactivo en Spring Boot 4.x | Dependencia, propiedades y forma de registrar validadores y conversor de autoridades | Solo afecta la implementación |
| Librería de rate limiting compatible con el stack | Compatibilidad con Spring Boot 4.x y Java 25 | Si no hay, el limitador se implementa como componente propio |

## Depends on

- `AV-005`: acceso de lectura de `ADMIN` a Events y disponibilidad.
- `FG-001`, `FG-002`: límites de entrada.
