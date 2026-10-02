---
id: ADR-008
title: Payment versus expiration race
status: PROPOSED
priority: HIGH
source_ids: [ST-002, ST-004, ST-007, ST-010, BR-002, BR-003, VAL-001, VAL-003, AC-008, AC-009, AC-019, FR-011, FR-015, FR-017]
related_adrs: [ADR-005, ADR-007, ADR-009, ADR-010, ADR-011]
---

# ADR-008 — Payment versus expiration race

## Context

El resultado del pago puede llegar en el mismo instante en que la Reservation vence. Debe existir un único resultado (`ST-004` / `ST-007` frente a `ST-002` / `ST-010`). `AC-019` confirma la compra cuando el Payment Mock aprueba "dentro de la vigencia de la Reservation"; `BR-003` y `VAL-001` impiden que una Reservation siga vigente más de diez minutos sin confirmación.

Hay además un caso que la especificación no cubre: el proveedor aprueba (o pudo haber aprobado) un pago cuya Order termina sin confirmarse.

## Options considered

### Option A — Gana la primera transición terminal confirmada, con guarda temporal estricta en la confirmación

La exclusión la da la guarda `status = CREATED` (`ADR-005`). Además, la confirmación exige que `expiresAt` sea posterior al instante actual. Se añade un margen de corte para no iniciar pagos que no pueden terminar a tiempo.

- A favor: un único resultado garantizado por el almacén; `expiresAt` es la autoridad única; la confirmación nunca ocurre después del vencimiento; el comportamiento no depende de cuánto tarde en pasar el proceso de expiración.
- En contra: una aprobación que llega justo después del vencimiento no confirma la compra y requiere reversar el pago; depende de relojes razonablemente sincronizados.

### Option B — Gana la primera transición terminal confirmada, sin guarda temporal

Solo la guarda `status = CREATED`. Una aprobación tardía confirma la compra si el proceso de expiración aún no ha actuado.

- A favor: menos pagos aprobados sin compra; más simple.
- En contra: la confirmación puede ocurrir después de los diez minutos, por un margen que depende de la periodicidad del proceso de expiración; contradice la lectura literal de `VAL-001` y de `AC-019`; el resultado deja de ser determinista respecto de `expiresAt`.

### Option C — El proceso de expiración respeta pagos en curso

No expirar una Order con PaymentAttempt activo hasta conocer el resultado.

- A favor: nunca hay aprobación sin compra.
- En contra: una Reservation podría superar los diez minutos de forma no acotada si el proveedor no responde; contradice `BR-002` y `VAL-001`. Se descarta.

## Decision

Se adopta la **Option A**.

1. **Único resultado**: toda transición terminal exige `status = CREATED`. De las transacciones concurrentes "Confirmar" y "Expirar", exactamente una se confirma; la otra falla por condición y su ejecutor relee la Order.
2. **Autoridad temporal**: `expiresAt`, fijado al crear la Reservation.
   - "Confirmar" exige `expiresAt` posterior al instante actual.
   - "Expirar" exige `expiresAt` anterior o igual al instante actual (`VAL-003`, `AC-009`).
   - "Rechazar" y "Fallar" no llevan guarda temporal: liberan los Ticket igual que una expiración; entre ellas y "Expirar" gana la primera confirmada.
3. **Margen de corte**: el consumidor no inicia un pago si faltan menos de 15 segundos para `expiresAt` (valor inicial configurable, igual al presupuesto máximo de la llamada de pago con sus reintentos). En ese caso confirma el mensaje sin efectos y la Order se cierra por expiración.
4. **Plazo de la llamada de pago**: la espera del resultado se limita al menor entre el timeout configurado y el tiempo restante hasta `expiresAt` menos un margen de aplicación.
5. **Aprobación que no puede aplicarse**: si el Payment Mock aprueba pero "Confirmar" falla porque la Order ya está `EXPIRED` o porque la guarda temporal no se cumple:
   - la Order no se confirma; si aún está en `CREATED` con `expiresAt` vencido, el consumidor ejecuta la transición "Expirar";
   - los Ticket quedan liberados y la Order permanece en su estado terminal (`AC-025`);
   - la aprobación se registra en la auditoría de la Order como anomalía y se emite una métrica con alarma;
   - **queda pendiente de `FG-003`** qué hacer con el pago aprobado.
6. **Resultado desconocido**: si la Order se cierra como `EXPIRED` o `FAILED` con un PaymentAttempt activo cuyo resultado nunca se conoció (timeouts), el proveedor pudo haber aprobado. Se trata igual que el punto 5 y depende también de `FG-003`.

### Parte del diseño que depende de FG-003

Si se aprueba la respuesta recomendada (reversar el pago):

- La transición que cierra una Order con PaymentAttempt activo y resultado no aplicado marca `paymentReversalPending` en la misma escritura transaccional y la indexa en `GSI3` bajo `REVERSAL#<shard>`.
- El worker solicita la cancelación del intento al proveedor (operación idempotente por `paymentAttemptId`, ver `ADR-011`), con reintento, y al confirmarla elimina la marca y añade un registro de auditoría.
- El puerto de pago incorpora la operación de cancelación.

Si no se aprueba, estos elementos no se implementan y la anomalía queda solo registrada y alertada.

## Rationale

- La guarda `status = CREATED` resuelve la exclusión sin locks, líderes ni coordinación entre el consumidor y el proceso de expiración.
- La guarda temporal hace el resultado independiente de la periodicidad del proceso de expiración: una misma secuencia de instantes produce siempre el mismo resultado funcional.
- El margen de corte y el plazo de la llamada reducen el caso "aprobado sin compra" a situaciones de fallo real (proveedor lento o caído), no al funcionamiento normal.
- La opción B sacrifica la regla de los diez minutos por comodidad. La opción C la rompe.

## Consequences

- Los Ticket pueden permanecer retenidos unos segundos después de `expiresAt`, hasta que actúa el proceso de expiración; ninguna confirmación puede ocurrir en ese intervalo (`AV-003`).
- Una Order encolada con mucho retraso puede expirar sin que se intente su pago.
- Existe un caso residual de pago aprobado sin compra, que requiere decisión funcional (`FG-003`).
- Las guardas temporales usan el reloj de la instancia que ejecuta la transacción (`RISK-006`).

## Risks and mitigations

| Risk | Mitigation |
|---|---|
| `RISK-004` pago aprobado o de resultado desconocido con Order no confirmada | Margen de corte, plazo de la llamada, idempotencia del proveedor, auditoría y alarma; reverso según `FG-003` |
| `RISK-006` desalineación de relojes | Sincronización horaria de la plataforma; márgenes de segundos frente a desalineaciones esperadas de milisegundos; reloj inyectable para pruebas deterministas (`ADR-019`) |

## Items to verify

| Item | Qué comprobar | Impacto si difiere |
|---|---|---|
| Sincronización horaria de las tareas en la plataforma de cómputo objetivo | Desalineación máxima esperada | Si es comparable al margen de corte, aumentar los márgenes |

## Depends on

- `FG-003`: tratamiento del pago aprobado o de resultado desconocido cuando la Order se cierra sin confirmarse.
- `AV-003`: tolerancia de liberación tras `expiresAt`, guarda temporal estricta y margen de corte.
