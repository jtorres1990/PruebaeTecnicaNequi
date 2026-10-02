Estoy desarrollando una prueba técnica para una vacante mediante una metodología multiagente. Quiero continuar exactamente desde el punto descrito a continuación, sin reiniciar el análisis ni reinterpretar decisiones ya tomadas.

# 1. Objetivo del proyecto

Construir una solución para una:

**Plataforma de Procesamiento de Eventos de Ticketing**

El sistema debe manejar eventos de conciertos, teatro y deportes, con alta concurrencia y problemas críticos como:

- compras duplicadas;
- venta del mismo asiento a múltiples usuarios;
- reservas temporales;
- procesamiento asíncrono;
- consistencia de inventario;
- baja latencia;
- idempotencia;
- tolerancia a entrega `at-least-once`.

Stack obligatorio/priorizado:

- Java 25
- Spring Boot 4.x
- Spring WebFlux
- Clean Architecture
- Amazon DynamoDB
- DynamoDB Local
- Amazon SQS Standard
- LocalStack
- Amazon Cognito
- JWT
- Docker / Docker Compose
- Payment Mock
- JUnit 5
- Mockito
- reactor-test

Terraform y arquitectura AWS productiva son diferenciales importantes, pero pertenecen a etapas posteriores.

---

# 2. Metodología multiagente

La filosofía acordada es:

**El software controla el workflow; los agentes resuelven incertidumbre; sistemas determinísticos verifican; humanos aprueban decisiones importantes.**

Reglas:

- No usar un superagente.
- Los agentes se comunican mediante artefactos versionados.
- El contexto de cada agente debe estar aislado.
- Las decisiones humanas se representan mediante YAML.
- Los artefactos generados no se sobrescriben.
- Se mantiene trazabilidad mediante IDs estables.
- No se almacenan cadenas de pensamiento; solo decisiones, evidencia y resultados.
- Las decisiones humanas tienen precedencia sobre interpretaciones del agente.

Pipeline previsto:

Requirements Analyst  
→ Feature Spec  
→ Human Functional Gate  
→ Consolidación  
→ Architect  
→ Human Architecture Gate  
→ Domain Developer  
→ Reactive Adapters Developer  
→ Platform/IaC  
→ QA/Resilience  
→ Security Reviewer  
→ Documentation/Demo

---

# 3. Artefactos existentes

Existen actualmente:

`requirements/Prueba2026.md`

Requerimiento original.

`feature-spec/ticketing.feature-spec.md`

Feature Specification actual.

Versión interna:

`version: 2`

`human-review/ticketing.functional-review.yaml`

Revisión humana con 21 decisiones `HV-*`.

El Human Review ya fue completado.

Debe quedar finalmente:

```yaml
review:
  status: APPROVED
  reviewer: human
  reviewed_at: 2026-10-01
```

Y:

```yaml
gate:
  pending_blocking_items: []
  non_blocking_pending_items: []
  architecture_can_start: true
```

Antes de consolidar se detectaron estas correcciones menores:

1. Cambiar `REVIWED` por `APPROVED`.
2. Usar fecha ISO `2026-10-01`.
3. Cambiar listas YAML vacías de `null` a `[]`.
4. En HV-012 cambiar `Reason` por `reason`.
5. En HV-014 usar correctamente el claim:
   `cognito:groups`.
6. En HV-009 fijar definitivamente el estado exitoso de Order como:
   `CONFIRMED`.
7. HV-005 puede mencionar directamente que `ADMIN` define las cortesías.

---

# 4. Decisiones humanas aprobadas

## HV-001 — Ticket vs Order

Una `Order` agrupa uno o más `Ticket`.

Todos los tickets de una misma Order se procesan como una unidad indivisible.

No se permiten ventas parciales.

Los estados:

- AVAILABLE
- RESERVED
- PENDING_CONFIRMATION
- SOLD
- COMPLIMENTARY

pertenecen a `Ticket`.

`Order` tiene su propia máquina de estados.

---

# 5. Modelo de dominio acordado

Cardinalidades:

```text
Event 1 -> N Ticket

Event 1 -> N Order

Order 1 -> N requested Ticket

Order 1 -> 0..1 active Reservation

Reservation 1 -> N Ticket
```

Cada Order pertenece a exactamente un Event.

Una Order no puede contener tickets de eventos diferentes.

Cada Ticket pertenece a un solo Event.

Un Ticket puede pertenecer como máximo a una Reservation activa.

Las operaciones sobre una Order son:

**all-or-nothing**.

---

# 6. Tickets individuales

No se administra únicamente una cantidad agregada.

Cada Event contiene ubicaciones/tickets individuales identificables.

Ejemplo:

```text
H1
H2
H3
...
```

Esto permite probar correctamente concurrencia cuando múltiples usuarios compiten por el mismo asiento.

La disponibilidad agregada se calcula a partir del estado de los Tickets.

---

# 7. Flujo de compra acordado

La simple selección visual de tickets:

**NO crea una reserva.**

Ejemplo:

Usuario selecciona:

```text
H1
H2
```

pero todavía no pulsa:

`Ir a pagar`

Los tickets siguen disponibles.

La reserva comienza solamente cuando el CUSTOMER inicia formalmente la compra.

Flujo:

```text
CUSTOMER selecciona tickets
        ↓
NO hay reserva backend
        ↓
CUSTOMER pulsa "Ir a pagar"
        ↓
backend valida TODOS los tickets
        ↓
intenta reserva atómica
        ↓
si alguno no está disponible
        ↓
rechaza toda la operación
```

Si todos están disponibles:

```text
Reservation creada
        ↓
inicia temporizador de 10 minutos
        ↓
Order CREATED
        ↓
Tickets RESERVED
        ↓
SQS
        ↓
procesamiento asíncrono
        ↓
Payment Mock
        ↓
Tickets PENDING_CONFIRMATION
```

Pago exitoso:

```text
Tickets → SOLD
Order → CONFIRMED
```

Pago rechazado / fallo definitivo:

```text
Tickets → AVAILABLE
Order → FAILED o REJECTED según causa
```

Expiración:

```text
10 minutos
↓
Tickets → AVAILABLE
Order → EXPIRED
```

Nunca existe cumplimiento parcial.

---

# 8. Reservation

La reserva comienza exactamente cuando el backend consigue reservar correctamente TODOS los tickets de la Order.

Conceptualmente:

```text
createdAt = T0
expiresAt = T0 + 10 minutos
```

La selección visual anterior no inicia el temporizador.

Una Reservation pertenece a una única Order y contiene exactamente los Tickets solicitados.

No hay expiraciones parciales.

---

# 9. Estados de Ticket

Estados obligatorios:

```text
AVAILABLE
RESERVED
PENDING_CONFIRMATION
SOLD
COMPLIMENTARY
```

Flujo principal:

```text
AVAILABLE
   ↓ reserva exitosa
RESERVED
   ↓ comienza procesamiento de pago
PENDING_CONFIRMATION
   ↓ pago OK
SOLD
```

`SOLD` es terminal.

`COMPLIMENTARY` es terminal.

`RESERVED` no representa venta.

`PENDING_CONFIRMATION` no representa venta.

Ante fallo o expiración los tickets regresan conjuntamente a `AVAILABLE` cuando corresponda.

---

# 10. Estados de Order

Máquina independiente de Ticket.

Estados mínimos ya definidos:

```text
CREATED
CONFIRMED
REJECTED
FAILED
EXPIRED
```

`CREATED` es el estado inicial.

`CONFIRMED` representa compra exitosa.

`REJECTED` representa imposibilidad funcional o de negocio.

Ejemplo:

ticket ya no disponible.

`FAILED` representa fallo técnico definitivo.

`EXPIRED` representa vencimiento de la Reservation.

Estados terminales:

```text
CONFIRMED
REJECTED
FAILED
EXPIRED
```

El Architect podrá evaluar si necesita estados intermedios adicionales, pero no debe inventarlos sin justificación funcional.

---

# 11. Payment Mock

Existe un servicio de pago simulado independiente.

Representa un proveedor externo.

Debe permitir simular como mínimo:

- pago exitoso;
- pago rechazado/fallido.

Ticketing solo invoca Payment Mock después de haber reservado correctamente todos los Tickets.

El mock debe permitir pruebas deterministas.

Tecnología y protocolo quedan para Architect.

---

# 12. Idempotencia

El sistema debe ser idempotente frente a:

- solicitudes repetidas;
- mensajes SQS duplicados;
- procesamiento repetido;
- pagos repetidos;
- estados terminales.

Un mensaje duplicado nunca puede producir:

- otra reserva;
- otro pago;
- otra venta;
- otro cambio de inventario;
- otra transición terminal.

Una Order puede tener máximo un intento de pago activo simultáneamente.

Si la Order ya está CONFIRMED:

un reprocesamiento debe mantener/devolver el resultado anterior.

Un nuevo intento de pago autorizado tras un fallo recuperable puede ser un nuevo `PaymentAttempt`, pero asociado a la misma Order.

Los mecanismos técnicos quedan para Architect.

---

# 13. SQS

Decisión aprobada:

**Amazon SQS Standard**

Entorno local:

**LocalStack**

Se eligió Standard para demostrar explícitamente idempotencia bajo entrega:

`at-least-once`.

No depender de FIFO para resolver duplicados.

Detalles pendientes para Architect:

- visibility timeout;
- retries;
- DLQ;
- polling;
- manejo de poison messages.

---

# 14. DynamoDB

Persistencia objetivo:

**Amazon DynamoDB**

Local:

**DynamoDB Local**

Se almacenarán conceptualmente:

- Events
- Tickets
- Orders
- Reservations
- demás datos necesarios

Los siguientes elementos son responsabilidad del Architect:

- partition keys;
- sort keys;
- GSIs;
- access patterns;
- conditional writes;
- TransactWriteItems;
- consistency;
- modelo de tablas.

El Requirements Analyst no debe diseñarlos.

---

# 15. COMPLIMENTARY

Los Tickets `COMPLIMENTARY` se definen únicamente durante la creación del Event.

Solo `ADMIN` puede definirlos.

Ejemplo:

```text
capacidad = 500

480 AVAILABLE
20 COMPLIMENTARY
```

Los 20 COMPLIMENTARY:

- cuentan dentro de capacidad física;
- no están disponibles comercialmente;
- no representan venta;
- no pueden agregarse después;
- no se convierten desde AVAILABLE/RESERVED/etc.;
- son estado terminal.

---

# 16. Seguridad

Se utilizará:

**Amazon Cognito**

Autenticación:

**JWT**

Backend:

OAuth2 Resource Server.

Roles:

```text
ADMIN
CUSTOMER
```

Los grupos Cognito se obtienen mediante:

```text
cognito:groups
```

ADMIN puede:

- crear Events;
- definir COMPLIMENTARY durante la creación.

CUSTOMER puede:

- consultar Events;
- consultar disponibilidad;
- iniciar compras;
- consultar sus propias Orders.

La identidad del CUSTOMER debe derivarse del JWT, nunca de un `customerId` enviado libremente por el cliente.

Un CUSTOMER intentando consultar una Order ajena debe recibir funcionalmente el mismo resultado que una Order inexistente.

No revelar existencia de recursos de otros clientes.

Fuera de alcance:

- UI login;
- UI registro;
- recuperación contraseña;
- administración visual de usuarios.

---

# 17. Consulta de Order

Order existente en:

```text
CONFIRMED
REJECTED
FAILED
EXPIRED
```

sigue siendo consultable.

Order inexistente:

`NOT_FOUND`.

Order perteneciente a otro CUSTOMER:

también:

`NOT_FOUND`.

La representación HTTP concreta queda para arquitectura/OpenAPI.

---

# 18. Disponibilidad en tiempo real

Se decidió NO utilizar WebSocket ni SSE.

“Real time” significa:

**lectura actual bajo demanda.**

Cuando usuario consulta el Event:

se devuelve un snapshot actual de disponibilidad.

Ese snapshot es informativo.

No garantiza adquisición.

Al pulsar `Ir a pagar`:

el backend revalida autoritativamente todos los Tickets.

Si uno ya no está disponible:

se rechaza toda la Reservation.

Streaming continuo queda fuera de alcance.

---

# 19. Event disponible

Un Event está disponible para consulta cuando:

- todavía no ha ocurrido;
- está habilitado para consulta/venta.

Un Event futuro agotado sigue visible.

Ejemplo:

```text
Event futuro + tickets disponibles
→ visible

Event futuro + 0 tickets
→ visible como agotado

Event pasado
→ no aparece en consulta normal
```

No introducir por ahora estados adicionales de publicación.

---

# 20. Validaciones de Event

Campos mínimos:

- name;
- place;
- date/time;
- capacity.

Nombre obligatorio.

Lugar obligatorio.

Fecha/hora obligatoria.

Debe representar un instante futuro.

Referencia temporal:

**UTC**

No dejar `America/Bogota` quemado funcionalmente.

Capacidad:

entero > 0.

Debe cumplirse:

```text
Event.capacity
=
cantidad total de Tickets del Event
```

COMPLIMENTARY forma parte de esa capacidad.

---

# 21. Rendimiento

Objetivos de prueba acordados:

```text
>= 1.000 usuarios concurrentes

≈ 200 requests/second sostenidos

availability query:
p95 < 500 ms

inicio síncrono de compra/reserva:
p95 < 1 segundo

0 sobreventas

0 ventas duplicadas
```

Estos son objetivos para la prueba técnica.

NO deben presentarse como capacidad garantizada de producción.

---

# 22. Reporting

Fuera de alcance:

- reportes operativos;
- reportes contables;
- reportes inventario;
- exportaciones/reporting endpoints.

Solo preservar semántica:

```text
RESERVED
PENDING_CONFIRMATION
→ no ventas

SOLD
→ venta confirmada

COMPLIMENTARY
→ capacidad física pero no venta
```

---

# 23. Requirements Analyst

Existe un contrato de `requirements-analyst`.

El agente inicialmente:

- analiza el requerimiento;
- genera Feature Specification;
- genera Human Review YAML;
- usa IDs estables;
- no diseña arquitectura;
- no inventa requerimientos;
- distingue Evaluation Criteria de Product Requirements.

IDs:

```text
CAP-
FR-
BR-
VAL-
DS-
ST-
ALT-
ERR-
AC-
NFR-
TC-
DEL-
EVAL-
HV-
```

---

# 24. Modo de Consolidación

Se agregó al Requirements Analyst un:

**Modo de Revisión y Consolidación**

Objetivo:

NO volver a analizar desde cero.

Debe usar:

1. Human Functional Review
2. Feature Specification
3. Requerimiento original

en ese orden de autoridad para las decisiones revisadas.

Entradas:

```text
requirements/Prueba2026.md

feature-spec/ticketing.feature-spec.md

human-review/ticketing.functional-review.yaml
```

Salida:

```text
feature-spec/ticketing.feature-spec.v3.md
```

NO debe sobrescribir artefactos anteriores.

Las decisiones `CONFIRMED_WITH_CHANGE` deben propagarse a:

- Actors
- Capabilities
- Functional Requirements
- Business Rules
- Validations
- Domain States
- State Transitions
- Main Flows
- Alternative Flows
- Error Flows
- Acceptance Criteria
- NFR
- Technical Constraints
- Deliverables
- Evaluation Criteria
- Domain Model
- Traceability Matrix

Debe mantener IDs estables.

Toda regla modificada por revisión humana debe conservar referencia al `HV-*` correspondiente.

---

# 25. Regla importante de Evaluation Criteria

Los `EVAL-*` NO deben convertirse automáticamente en:

- FR
- NFR

Ejemplos:

- observabilidad;
- costos;
- gobernanza;
- Terraform;
- resiliencia;
- security evaluation.

Solo deben convertirse en requisito funcional/NFR si:

- el requerimiento original lo exige explícitamente; o
- Human Review lo incorporó explícitamente.

---

# 26. Gate esperado tras consolidación

La nueva Feature Spec debe quedar:

```text
READY_FOR_ARCHITECTURE
```

solo si:

- no hay HIGH pendientes;
- no hay contradicciones;
- las 21 HV fueron incorporadas;
- Ticket y Order state machines son coherentes;
- idempotencia está representada;
- all-or-nothing está representado;
- acceptance criteria de flujos críticos están completos;
- las restricciones tecnológicas están presentes.

---

# 27. Próximo paso inmediato

El siguiente paso NO es diseñar arquitectura todavía.

Primero:

**ejecutar el Requirements Analyst en Modo de Revisión y Consolidación.**

Instrucción sugerida:

```text
Ejecuta el Modo de Revisión y Consolidación para la funcionalidad de ticketing.

Utiliza la Human Functional Review aprobada como fuente autoritativa para todas
las ambigüedades revisadas.

Genera una nueva Feature Specification consolidada sin sobrescribir los
artefactos existentes.
```

Resultado esperado:

```text
feature-spec/ticketing.feature-spec.v3.md

Version: 3

Human Review decisions incorporated: 21

Pending Human Validations: 0

Pending blocking items: 0

Architecture readiness: READY_FOR_ARCHITECTURE
```

Después de revisar esa Feature Spec v3:

**el siguiente trabajo será diseñar el contrato del Architect Agent.**

El Architect deberá resolver posteriormente, entre otras cosas:

- modelo DynamoDB basado en access patterns;
- reserva atómica multi-ticket;
- consistencia Order/Reservation/Ticket;
- persistencia + publicación SQS;
- estrategia frente a fallo de enqueue;
- idempotencia;
- Payment Mock;
- carreras pago vs expiración;
- scheduler de expiración;
- SQS retries / DLQ / visibility timeout;
- Cognito + Spring Security;
- OpenAPI;
- diagramas;
- ADRs;
- Docker Compose;
- posible Terraform AWS;
- observabilidad y resiliencia.

No avanzar a estas decisiones hasta revisar primero la Feature Spec consolidada.