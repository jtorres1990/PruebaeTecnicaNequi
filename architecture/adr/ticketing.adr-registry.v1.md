---
artifact: adr-registry
schema_version: 1.0
feature: ticketing-event-processing
version: 1
source:
  human_review:
    artifact: human-review/ticketing.architecture-review.yaml
    status: APPROVED
    reviewed_at: 2026-10-04
  architecture: architecture/ticketing.architecture.v2.md
authoritative_over: frontmatter `status` of every ADR file in architecture/adr/
generated_at: 2026-10-04
---

# ADR Registry v1 — Ticketing Event Processing

## Autoridad

Este registro es la fuente autoritativa del estado de cada ADR. **Prevalece sobre el campo `status` del frontmatter de los archivos ADR-001 a ADR-021**, que permanecen intactos con `status: PROPOSED` por la regla de inmutabilidad de esta consolidación. Los ADR-022 a ADR-040 se crearon ya en estado `ACCEPTED`, coherente con este registro.

Reglas aplicadas:

- `CONFIRMED` → el ADR original queda `ACCEPTED`.
- `CONFIRMED_WITH_CHANGE` → el ADR original queda `SUPERSEDED` y su sucesor, con `supersedes` en el frontmatter, queda `ACCEPTED`. El sucesor incorpora íntegramente la respuesta humana.
- Las referencias que un ADR aceptado original (ADR-003, ADR-008) hace a ADR reemplazados se resuelven a sus sucesores según la tabla.

## Registro

| ID | Título | Estado original | Estado consolidado | Reemplazado por | Decisión humana aplicada |
|---|---|---|---|---|---|
| ADR-001 | DynamoDB data model | PROPOSED | SUPERSEDED | ADR-022 | CONFIRMED_WITH_CHANGE: Ticket con partition key propia (`TICKET#<eventId>#<ticketId>`, `#META`); disponibilidad por GSI con sharding calculado conservado en el Event; consultas paginadas y filtrables por sección; transacción multi-partición; soporte de 50.000 Ticket y prueba concentrada > 1.000 usuarios; escritura por lotes antes de habilitar |
| ADR-002 | Atomic multi-ticket reservation | PROPOSED | SUPERSEDED | ADR-023 | CONFIRMED_WITH_CHANGE: única `TransactWriteItems` multi-partición con Order, Reservation, idempotencia y auditoría; condición existe + pertenece al Event + `AVAILABLE`; Event fuera e inmutable; conflictos con reintento acotado; condición fallida = rechazo funcional |
| ADR-003 | Tickets per Order limit | PROPOSED | ACCEPTED | — | CONFIRMED: mínimo 1, máximo 10 configurable (desplegado 10), sin repetidos; validado en el contrato y en el caso de uso; rechazo completo sin efectos |
| ADR-004 | Event creation at capacity scale | PROPOSED | SUPERSEDED | ADR-024 | CONFIRMED_WITH_CHANGE: creación asíncrona con SQS y worker; `PROVISIONING`/`ENABLED`/`FAILED`; definición compacta; `ticketId` determinista; lease y comprobación antes de cada lote; habilitación verificada; cola propia con DLQ; limpieza y detección de estancados; estado visible para `ADMIN` |
| ADR-005 | Order, Reservation and Ticket consistency per state transition | PROPOSED | SUPERSEDED | ADR-025 | CONFIRMED_WITH_CHANGE: una `TransactWriteItems` por transición; guardián `CREATED`; propiedad y estado de origen del Ticket; lectura de Order solo del item con lectura fuerte; cuarentena técnica no expuesta; trade-off transaccional aceptado |
| ADR-006 | Persistence plus enqueue | PROPOSED | SUPERSEDED | ADR-026 | CONFIRMED_WITH_CHANGE: publicación directa con compensación; presupuesto 3 intentos / 500 ms / 2 s; barrido de republicación a los 30 s; índice disperso con sharding; expiración como última garantía |
| ADR-007 | Idempotency | PROPOSED | SUPERSEDED | ADR-027 | CONFIRMED_WITH_CHANGE: carrera con la misma clave resuelta por relectura con prioridad de la idempotencia; idempotencia de creación de Events vinculada al `ADMIN` |
| ADR-008 | Payment versus expiration race | PROPOSED | ACCEPTED | — | CONFIRMED: gana la primera transición terminal; `expiresAt` autoridad única; confirmación exige `expiresAt > ahora`; margen de corte; aprobación no aplicable no confirma, se audita y alerta; tratamiento según `FG-003` y `AV-003`. Su sección "Parte del diseño que depende de FG-003" queda activa (índice de reversos como rango `REVERSAL#` de `GSI3`) |
| ADR-009 | Expiration process | PROPOSED | SUPERSEDED | ADR-028 | CONFIRMED_WITH_CHANGE: proceso periódico sin líder; planificación y concurrencia propias por proceso; cuarentena fuera del índice de expiración |
| ADR-010 | SQS operational policy | PROPOSED | SUPERSEDED | ADR-029 | CONFIRMED_WITH_CHANGE: política de Orders confirmada; cola de aprovisionamiento con DLQ, visibilidad 120 s con heartbeat, 5 recepciones, concurrencia baja; capacidad de consumo con alarma de antigüedad a 2 min y autoescalado |
| ADR-011 | Payment Mock | PROPOSED | SUPERSEDED | ADR-030 | CONFIRMED_WITH_CHANGE: proyecto independiente con OpenAPI propio; cancelación obligatoria idempotente; cancelación anticipada; inspección de cancelaciones |
| ADR-012 | Audit trail | PROPOSED | SUPERSEDED | ADR-031 | CONFIRMED_WITH_CHANGE: catálogo ampliado (reversos, aprobación tardía, cuarentena, aprovisionamiento fallido); inmutabilidad por convención con PITR; mejora productiva con bloqueo de objetos |
| ADR-013 | Security | PROPOSED | SUPERSEDED | ADR-032 | CONFIRMED_WITH_CHANGE: consulta de aprovisionamiento solo `ADMIN`; clave de creación vinculada al `ADMIN`; límite de cuerpo bajo; gestión en puerto interno; una Order activa por cliente y Event mediante item de bloqueo |
| ADR-014 | Local identity provider | PROPOSED | SUPERSEDED | ADR-033 | CONFIRMED_WITH_CHANGE: sujetos arbitrarios y > 1.000 identidades para la carga; identidades `ADMIN`+`CUSTOMER` y sin grupos |
| ADR-015 | Clean Architecture structure | PROPOSED | SUPERSEDED | ADR-034 | CONFIRMED_WITH_CHANGE: `payment-mock` como proyecto independiente sin código compartido; catálogo de puertos actualizado; reglas nuevas en `domain` |
| ADR-016 | Error model and reactive retry | PROPOSED | SUPERSEDED | ADR-035 | CONFIRMED_WITH_CHANGE: alineación de códigos y respuestas (`ACTIVE_ORDER_EXISTS`, `EVENT_NOT_ON_SALE`, 202, cursor); circuit breakers para Payment Mock y publicación en SQS con pausa del consumo y 503 antes de reservar |
| ADR-017 | Local topology | PROPOSED | SUPERSEDED | ADR-036 | CONFIRMED_WITH_CHANGE: recursos adicionales de `infra-init`; build independiente del mock; generación previa de > 1.000 tokens; LocalStack fijado a un tag anterior al 23-03-2026 con alternativas |
| ADR-018 | AWS target topology | PROPOSED | SUPERSEDED | ADR-037 | CONFIRMED_WITH_CHANGE: escalado por antigüedad; apagado ordenado; control de bots; catálogo de alarmas; mejoras productivas declaradas |
| ADR-019 | Test strategy | PROPOSED | SUPERSEDED | ADR-038 | CONFIRMED_WITH_CHANGE: datos de carga de 50.000 Ticket y > 1.000 identidades; invariantes ampliadas; pruebas deterministas de mecanismos; escenarios de resiliencia; mock fuera de la cobertura |
| ADR-020 | AWS integration technology for DynamoDB and SQS | PROPOSED | SUPERSEDED | ADR-039 | CONFIRMED_WITH_CHANGE: dos consumidores; pausa y reanudación; heartbeat; motivos de cancelación por item; escritura por lotes y conteo por shards |
| ADR-021 | Availability and inventory read model | PROPOSED | SUPERSEDED | ADR-040 | CONFIRMED_WITH_CHANGE: lectura desde GSI de disponibles con sharding; cantidad disponible y página por cursor filtrable por sección; conteo paralelo cacheado 1 s; sondeo de agotados con parada en el primer resultado |
| ADR-022 | DynamoDB data model with per-Ticket partitioning and sharded indexes | — (nuevo) | ACCEPTED | — | Sucesor de ADR-001 |
| ADR-023 | Atomic multi-ticket reservation across partitions | — (nuevo) | ACCEPTED | — | Sucesor de ADR-002 |
| ADR-024 | Asynchronous Event provisioning at capacity scale | — (nuevo) | ACCEPTED | — | Sucesor de ADR-004 |
| ADR-025 | Order, Reservation and Ticket consistency per state transition, with quarantine and payment reversal marking | — (nuevo) | ACCEPTED | — | Sucesor de ADR-005 |
| ADR-026 | Persistence plus enqueue with bounded publish budget and republish sweep | — (nuevo) | ACCEPTED | — | Sucesor de ADR-006 |
| ADR-027 | Idempotency for purchases, Event creation, messages and payments | — (nuevo) | ACCEPTED | — | Sucesor de ADR-007 |
| ADR-028 | Expiration process with isolated scheduling of worker periodic jobs | — (nuevo) | ACCEPTED | — | Sucesor de ADR-009 |
| ADR-029 | SQS operational policy for the Orders and Event provisioning queues | — (nuevo) | ACCEPTED | — | Sucesor de ADR-010 |
| ADR-030 | Payment Mock as an independent project with mandatory cancellation | — (nuevo) | ACCEPTED | — | Sucesor de ADR-011 |
| ADR-031 | Audit trail with extended cause catalog and declared immutability limits | — (nuevo) | ACCEPTED | — | Sucesor de ADR-012 |
| ADR-032 | Security, authorization and abuse controls including one active Order per customer and Event | — (nuevo) | ACCEPTED | — | Sucesor de ADR-013 |
| ADR-033 | Local identity provider with load-test and deterministic identities | — (nuevo) | ACCEPTED | — | Sucesor de ADR-014 |
| ADR-034 | Clean Architecture structure with independent Payment Mock project and updated ports | — (nuevo) | ACCEPTED | — | Sucesor de ADR-015 |
| ADR-035 | Error model, reactive retry and circuit breakers | — (nuevo) | ACCEPTED | — | Sucesor de ADR-016 |
| ADR-036 | Local topology with provisioning queue, independent mock build and pinned LocalStack | — (nuevo) | ACCEPTED | — | Sucesor de ADR-017 |
| ADR-037 | AWS target topology with backlog-age scaling, graceful shutdown, bot control and alarm catalog | — (nuevo) | ACCEPTED | — | Sucesor de ADR-018 |
| ADR-038 | Test strategy with capacity-scale load data, extended invariants and resilience scenarios | — (nuevo) | ACCEPTED | — | Sucesor de ADR-019 |
| ADR-039 | AWS integration technology for DynamoDB and SQS with two consumers, heartbeat and pause control | — (nuevo) | ACCEPTED | — | Sucesor de ADR-020 |
| ADR-040 | Availability read model over a sharded index with paginated pages and cached count | — (nuevo) | ACCEPTED | — | Sucesor de ADR-021 |

Totales: 40 ADR; 21 `ACCEPTED` vigentes (ADR-003, ADR-008, ADR-022 a ADR-040); 19 `SUPERSEDED`; 0 `REJECTED`; 0 `PROPOSED` vigentes.

## Correspondencia de referencias

| Referencia en un ADR original o en la revisión humana | Se resuelve a |
|---|---|
| ADR-001 | ADR-022 |
| ADR-002 | ADR-023 |
| ADR-004 | ADR-024 |
| ADR-005 | ADR-025 |
| ADR-006 | ADR-026 |
| ADR-007 | ADR-027 |
| ADR-009 | ADR-028 |
| ADR-010 | ADR-029 |
| ADR-011 | ADR-030 |
| ADR-012 | ADR-031 |
| ADR-013 | ADR-032 |
| ADR-014 | ADR-033 |
| ADR-015 | ADR-034 |
| ADR-016 | ADR-035 |
| ADR-017 | ADR-036 |
| ADR-018 | ADR-037 |
| ADR-019 | ADR-038 |
| ADR-020 | ADR-039 |
| ADR-021 | ADR-040 |

## Notas sobre los ADR aceptados sin cambio

- **ADR-003**: su sección de riesgos indica que un límite de Reservation activas por `CUSTOMER` "no se introduce" en ese ADR. La revisión humana a ADR-013 introduce la regla de una Order activa por cliente y por Event, registrada en ADR-032. Ambas decisiones son compatibles: ADR-003 limita el tamaño de una Order; ADR-032 limita las Orders activas simultáneas por cliente y Event. La mitigación de `RISK-010` se amplía en ADR-032.
- **ADR-008**: su sección condicionada a `FG-003` queda activa con la respuesta aprobada: la marca de reverso se escribe en la misma transacción de cierre y la Order se indexa en `GSI3` bajo `REVERSAL#<shard>`; el proceso de reversos y su política se detallan en ADR-025 y su planificación en ADR-028. Los valores temporales son los de `AV-003` (5 s, 15 s, 15 s).

## Validaciones y vacíos funcionales

| ID | Decisión humana | Aplicado en |
|---|---|---|
| AV-001 | CONFIRMED | ADR-040, ADR-022, `API-002` |
| AV-002 | CONFIRMED_WITH_CHANGE | ADR-024, ADR-040, `API-002`, `API-003`, `API-006` |
| AV-003 | CONFIRMED | ADR-008, ADR-025, ADR-026, ADR-028, ADR-029 |
| AV-004 | CONFIRMED | ADR-030, ADR-038 |
| AV-005 | CONFIRMED | ADR-032, ADR-033, `API-002`, `API-003`, `API-004`, `API-005`, `API-006` |
| AV-006 | CONFIRMED | ADR-038, ADR-034 |
| FG-001 | CONFIRMED | ADR-003, ADR-023, ADR-025, `API-004` |
| FG-002 | CONFIRMED_WITH_CHANGE | ADR-022, ADR-024, ADR-040, `API-001`, `API-003` |
| FG-003 | CONFIRMED_WITH_CHANGE | ADR-008, ADR-025, ADR-028, ADR-029, ADR-030, ADR-031, ADR-034 |
| FG-004 | CONFIRMED | ADR-027, ADR-023, ADR-035 |
| FG-005 | CONFIRMED_WITH_CHANGE | ADR-023, ADR-035, ADR-040, `API-003`, `API-004` |
