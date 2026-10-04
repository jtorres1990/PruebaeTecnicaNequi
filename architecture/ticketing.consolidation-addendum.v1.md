---
feature: ticketing
artifact: consolidation-addendum
version: 1
date: 2026-10-04
status: ACCEPTED
applies_to:
  - architecture/ticketing.architecture.v2.md
  - architecture/adr/ADR-025-state-transition-consistency-quarantine-reversal.md
  - architecture/adr/ticketing.adr-registry.v1.md
---

# Ticketing — Addendum de consolidación v1

Este addendum registra confirmaciones humanas posteriores a la consolidación de la versión 2 de la arquitectura. No modifica ningún archivo existente; complementa los artefactos indicados en `applies_to` y prevalece sobre ellos en los puntos que trata.

## 1. Tamaño máximo de transacción (ADR-005 frente a ADR-013)

**Origen.** `ticketing.architecture.v2.md` §8 y §14.1 (observación 1) y `ADR-025` (Context) registraron como *interpretación* del Architect que la cifra "ninguna transición supera 13 items" de la respuesta humana a ADR-005 es una cota derivada y no una regla independiente.

**Confirmación humana (2026-10-04).** Se confirma la interpretación: la cifra de 13 items fue calculada con el conjunto de items de la versión 1 (N Ticket + Order + idempotencia + auditoría, con N = 10) y no constituye un límite normativo. Su propósito, que toda transición quepa holgadamente en el límite de `TransactWriteItems`, se mantiene.

**Cifras vigentes** (con el máximo de 10 Ticket por Order de ADR-003):

| Transición | Composición | Items máximos |
|---|---|---|
| Reserva (AP-008) | N Ticket + Order + `IDEM#…` + auditoría + `ACTIVE#<customerId>#<eventId>` | 14 |
| Transiciones terminales (AP-014, AP-015) | Order + N Ticket + auditoría + borrado del bloqueo | 13 |
| Inicio de pago (AP-012) | Order + N Ticket + auditoría | 12 |
| Creación de Event (AP-001) | Event + `IDEMEVT#…` + auditoría | 3 |

**Efecto.** La observación 1 de §14.1 queda cerrada como decisión confirmada, no como interpretación pendiente. No cambia ninguna regla de ADR-003, ADR-025 ni ADR-032, ni el modelo de datos v2, que ya usan estas cifras. El límite de items de `TransactWriteItems` sigue como `TO_VERIFY` (se asumen 100) en `ticketing.architecture.v2.md` §13.

## 2. Erratas

Sin cambios: las dos erratas de los artefactos de la versión 2 siguen registradas, con la redacción que prevalece, en `ticketing.architecture.v2.md` §14.2.
