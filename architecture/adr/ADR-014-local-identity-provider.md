---
id: ADR-014
title: Local identity provider
status: PROPOSED
priority: MEDIUM
source_ids: [TC-016, TC-012, FR-018, FR-019, AC-028, DEL-005, EVAL-006]
related_adrs: [ADR-013, ADR-017, ADR-019]
---

# ADR-014 — Local identity provider

## Context

`TC-016` fija Amazon Cognito como emisor de JWT y `TC-012` exige que el entorno local se levante completo con Docker Compose. Hay que decidir cómo se obtienen JWT válidos en local.

La disponibilidad de Cognito en el emulador local es `TO_VERIFY`; el diseño no puede depender de ella sin alternativa.

## Options considered

### Option A — Emisor OIDC simulado en contenedor, con tokens de forma Cognito

Un servidor OIDC ligero de pruebas, en Docker Compose, que expone descubrimiento y claves públicas y emite tokens con los claims que valida el backend (`sub`, `cognito:groups`, tipo de token, cliente, emisor).

- A favor: sin cuenta de AWS ni conexión externa; arranque rápido; usuarios y grupos deterministas para pruebas y colección de solicitudes; el backend valida exactamente igual que con Cognito porque solo depende de emisor, claves y claims.
- En contra: no es Cognito; la capacidad de la imagen elegida para emitir claims arbitrarios es `TO_VERIFY`.

### Option B — Cognito emulado por el emulador local de AWS

- A favor: máxima paridad de API.
- En contra: su disponibilidad en la edición sin licencia es `TO_VERIFY`; hacer depender el entorno local de ello bloquearía `TC-012`.

### Option C — User pool real de Cognito en una cuenta de desarrollo

- A favor: tokens reales.
- En contra: requiere cuenta, conectividad y aprovisionamiento previo; el entorno local deja de ser autónomo. `HV-010` prioriza no incurrir en costos de AWS para ejecutar localmente.

### Option D — Servidor de identidad completo con mapeo de claims

Un servidor de identidad de código abierto configurado con un realm importado y un mapeador que emita `cognito:groups`.

- A favor: producto maduro y bien documentado; flujos OIDC reales.
- En contra: contenedor pesado y arranque lento; más configuración que el problema requiere.

## Decision

Se adopta la **Option A** como proveedor local por defecto, con **Option D como alternativa documentada** si la imagen de la Option A no puede emitir los claims requeridos, y **Option C disponible por configuración** para validar contra Cognito real.

- El backend se configura únicamente con URL de emisor, URL de claves públicas y nombres de claims (`ADR-013`). Cambiar de proveedor es un cambio de configuración, no de código.
- El emisor local define al menos tres identidades deterministas: un `ADMIN`, y dos `CUSTOMER` distintos (necesarios para verificar `AC-027`).
- Obtención de tokens en local: solicitud directa al endpoint de token del emisor local, documentada en el README y en la colección de solicitudes.
- Las pruebas automatizadas de la capa web no dependen del contenedor: usan tokens simulados por el soporte de pruebas de seguridad (`ADR-019`).
- El emisor local solo existe en el perfil local; nunca se despliega en AWS.

## Rationale

- La especificación exige que el backend sea Resource Server de tokens de Cognito; lo que el backend valida es un contrato de claims y una firma, y ese contrato puede reproducirse localmente sin Cognito.
- Mantener tres opciones ordenadas evita que un dato no verificado bloquee el entorno local.

## Consequences

- Hay un contrato de claims explícito que debe mantenerse idéntico entre el emisor local y Cognito (`RISK-013`).
- El emisor visto desde el host y desde la red de contenedores puede tener nombres distintos; la validación del emisor y la obtención de claves deben configurarse por separado o usar un nombre resoluble desde ambos lados.
- Los flujos de registro, login y recuperación no se emulan; están fuera de alcance (§3.2 de la especificación).

## Risks and mitigations

| Risk | Mitigation |
|---|---|
| `RISK-013` divergencia entre el token local y el de Cognito | Prueba de contrato de claims; perfil de configuración para un user pool real |
| La imagen elegida no admite claims personalizados | Alternativa Option D ya definida |
| Uso accidental del emisor local fuera de local | Solo configurable en el perfil local; el emisor de AWS es el del user pool |

## Items to verify

| Item | Qué comprobar | Impacto si difiere |
|---|---|---|
| Imagen del emisor OIDC simulado | Que emita claims arbitrarios (grupos, tipo de token, cliente), exponga descubrimiento y claves, y permita fijar el emisor | Si no, usar la Option D |
| Cognito en el emulador local de AWS | Disponibilidad en la edición usada | Si está disponible, puede sustituir a la Option A sin cambiar el backend |
| Coherencia del emisor entre host y red de contenedores | Que el claim de emisor coincida con el valor esperado por el backend | Configurar por separado emisor esperado y URL de claves |

## Depends on

- Sin dependencias de `FG-*` ni `AV-*`.
