---
id: ADR-033
title: Local identity provider with load-test and deterministic identities
status: ACCEPTED
priority: MEDIUM
supersedes: ADR-014
human_decision: CONFIRMED_WITH_CHANGE
source_ids: [TC-016, TC-012, FR-018, FR-019, AC-027, AC-028, DEL-005, EVAL-006, NFR-001]
related_adrs: [ADR-032, ADR-036, ADR-038]
---

# ADR-033 — Local identity provider with load-test and deterministic identities

Reemplaza a ADR-014.

## Human decision applied

Respuesta humana vinculante a ADR-014 (CONFIRMED_WITH_CHANGE), incorporada íntegramente:

> Se confirma como proveedor local por defecto un emisor OIDC simulado en contenedor que emite tokens con los claims de Cognito que valida el backend, con el servidor de identidad completo como alternativa documentada y el user pool real de Cognito disponible por configuración. El backend depende solo de la URL del emisor, la URL de claves y los nombres de claims. El emisor local solo existe en el perfil local y las pruebas automatizadas de la capa web usan tokens simulados sin depender del contenedor. Se añaden dos cambios.
>
> Identidades para la prueba de carga. Por la regla de una Order activa por cliente y por Event y la limitación por tasa por sujeto de ADR-013, la prueba de carga requiere más de 1.000 identidades `CUSTOMER` distintas. El emisor local debe permitir emitir tokens para sujetos arbitrarios o generar de antemano un lote de identidades, y la vigencia de los tokens debe cubrir la duración de la prueba.
>
> Identidades deterministas adicionales. Además del `ADMIN` y los dos `CUSTOMER`, el emisor local define una identidad con los grupos `ADMIN` y `CUSTOMER`, que puede comprar conforme a AV-005, y una identidad sin grupos reconocidos, que debe recibir prohibido en todas las operaciones protegidas.

Decisiones aprobadas incorporadas: regla de una Order activa y limitador por sujeto (ADR-032); `AV-005`; generación previa de tokens en el perfil de carga (ADR-036); datos de carga (ADR-038).

## Context

`TC-016` fija Cognito como emisor y `TC-012` exige que el entorno local se levante completo con Docker Compose. La disponibilidad de Cognito en el emulador local es `TO_VERIFY`.

## Options considered

### Option A — Emisor OIDC simulado en contenedor con claims de forma Cognito

- A favor: sin cuenta ni conexión externa; sujetos arbitrarios; arranque rápido.
- En contra: no es Cognito; capacidades de la imagen `TO_VERIFY`.

### Option B — Cognito del emulador local de AWS

- En contra: disponibilidad `TO_VERIFY`; con LocalStack fijado a un tag antiguo (ADR-036) es aún menos probable. No es la opción por defecto.

### Option C — User pool real de Cognito

- En contra: el entorno local deja de ser autónomo. Disponible por configuración.

### Option D — Servidor de identidad completo con mapeo de claims

- En contra: pesado; generar más de 1.000 usuarios requiere aprovisionamiento previo. Alternativa documentada.

## Decision

Se adopta la **Option A** por defecto, **Option D** como alternativa documentada y **Option C** por configuración.

| Aspecto | Decisión |
|---|---|
| Contrato del backend | URL de emisor, URL de claves y nombres de claims (ADR-032) |
| Claims emitidos | `sub`, `cognito:groups`, tipo de token de acceso, cliente, emisor, expiración |
| Identidades deterministas | `admin` (`ADMIN`); `customer-a` y `customer-b` (`CUSTOMER`, para `AC-027`); `admin-customer` (`ADMIN` y `CUSTOMER`, puede comprar, `AV-005`); `no-groups` (sin grupos reconocidos, recibe 403 en toda operación protegida) |
| Identidades de carga | Emisión para sujetos arbitrarios con grupo `CUSTOMER`; el perfil de carga genera de antemano más de 1.000 tokens con sujetos distintos (ADR-036) |
| Vigencia | Configurable; para la carga, superior a la duración de la prueba más la rampa |
| Ámbito | Solo perfil local; nunca en AWS |
| Pruebas de capa web | Tokens simulados por el soporte de pruebas de seguridad, sin contenedor (ADR-038) |
| Obtención de tokens | Endpoint de token del emisor local, documentado en el README y en la colección |

## Rationale

- El backend valida un contrato de claims y una firma, reproducible sin Cognito.
- Sujetos arbitrarios son imprescindibles para que la regla de una Order activa y el limitador por sujeto no distorsionen la prueba de carga.

## Consequences

- El contrato de claims debe mantenerse idéntico entre el emisor local y Cognito (`RISK-013`).
- El emisor visto desde el host y desde la red de contenedores puede diferir; emisor esperado y URL de claves se configuran por separado.

## Risks and mitigations

| Risk | Mitigation |
|---|---|
| `RISK-013` divergencia de tokens | Prueba de contrato de claims; perfil contra user pool real |
| La imagen no admite sujetos arbitrarios o vigencias largas | Alternativa Option D con un lote importado, o generador propio de tokens firmado con la clave del emisor local |

## Items to verify

| Item | Qué comprobar | Impacto si difiere |
|---|---|---|
| Imagen del emisor OIDC simulado | Claims arbitrarios, sujetos arbitrarios, vigencia configurable, descubrimiento, claves, emisor fijo | Si no, Option D |
| Rendimiento de emisión de más de 1.000 tokens | Tiempo del paso previo de la carga | Generar los tokens antes del perfil de carga |
| Cognito en el emulador local | Disponibilidad | Si existe, puede sustituir a la Option A sin cambio de backend |

## Depends on

- `AV-005` (resuelto).
