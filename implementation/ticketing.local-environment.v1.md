---
artifact: local-environment
schema_version: 1.0
feature: ticketing-event-processing
version: 1
prepared_by: human + assistant (sesión interactiva)
verified_at: 2026-10-04
status: READY
scope: entorno de desarrollo del backend (build, pruebas unitarias y de integración)
---

# Local Environment v1 — Ticketing

Entorno local preparado y verificado para que el agente `backend-developer` pueda compilar y ejecutar pruebas sin decisiones de toolchain pendientes. No cubre el entorno de Docker Compose de la aplicación (ADR-036), que corresponde al agente Platform.

## 1. Decisiones humanas (2026-10-04)

| ID | Decisión | Respuesta |
|---|---|---|
| ENV-001 | Herramienta de build de `ticketing` | **Maven Wrapper**, multi-módulo |
| ENV-002 | Obtención del JDK 25 | **Zulu 25 en zip, descomprimido en `D:\java`**, sin instalador y sin cambiar `JAVA_HOME` global |

Estas decisiones resuelven los `IV-*` de toolchain que el contrato del backend preveía (§7 Step 2). No deben volver a plantearse en el plan.

## 2. Toolchain

| Elemento | Valor | Verificación |
|---|---|---|
| JDK | Zulu 25.36.205, OpenJDK `25.0.4.1+1-LTS` | `java -version` correcto |
| Ruta del JDK | `D:\java\zulu25.36.205-ca-jdk25.0.4.1-win_x64` | — |
| Origen | `https://cdn.azul.com/zulu/bin/zulu25.36.205-ca-jdk25.0.4.1-win_x64.zip` | SHA-256 `5e91bc55aaa08750370d60b7030bb79193901ce1e93aba31becbf78c3ade5826`, igual al publicado por la API de Azul |
| Maven del wrapper | 3.9.16 (última 3.9.x estable a la fecha; 4.0.0 sigue en RC) | Compatibilidad con Spring Boot 4.x: `SPK` del backend |
| Maven instalado | 3.8.1 en `D:\apache-maven-3.8.1` | Arranca sobre JDK 25 (con avisos de acceso nativo de jansi). Solo se usa, si hace falta, para generar el wrapper |
| `JAVA_HOME` global | Sigue en Zulu 17 | No se modifica |
| Otros JDK presentes | Zulu 8, 17 y 21 en `D:\java` | No se usan para ticketing |

### Cómo invocar el build

El JDK 25 se selecciona por invocación, sin tocar la configuración global:

```bash
JAVA_HOME="D:\\java\\zulu25.36.205-ca-jdk25.0.4.1-win_x64" ./mvnw <goals>
```

```powershell
$env:JAVA_HOME = 'D:\java\zulu25.36.205-ca-jdk25.0.4.1-win_x64'; .\mvnw.cmd <goals>
```

El wrapper (`mvnw`, `mvnw.cmd`, `.mvn/wrapper/`) se crea dentro de `ticketing/` en el primer incremento del backend, fijado a Maven 3.9.16.

## 3. Docker

| Elemento | Valor |
|---|---|
| Motor | Docker Desktop 29.8.0, backend Linux |
| Recursos | 12 CPU, 8 GB de memoria |
| Disco | WSL en `D:\DockerData\DockerDesktopWSL`, límite ≈ 162 GB (ver §6) |

### Imágenes fijadas

| Uso | Imagen | Digest |
|---|---|---|
| DynamoDB Local | `amazon/dynamodb-local:3.3.1` (publicada 2026-07-31) | `sha256:ff89bd48ff32cd8d9be5fee8873b65b8854dc408f1afe881be6eb00247bc0dab` |
| SQS emulado | `localstack/localstack:4.14.0` (publicada 2026-02-26; último tag anterior al 2026-03-23) | `sha256:3ebc37595918b8accb852f8048fef2aff047d465167edd655528065b07bc364a` |

Ambas están descargadas localmente. Las pruebas de integración deben usar exactamente estos tags. `testcontainers/ryuk` la descarga Testcontainers bajo demanda.

## 4. Verificaciones realizadas

Ejecutadas con AWS CLI v2 contra contenedores efímeros, eliminados al terminar.

### 4.1 LocalStack 4.14.0 — item 15 de `ticketing.architecture.v2.md` §13

| Comprobación | Resultado |
|---|---|
| Arranque sin token de autenticación | **CONFIRMED**: edición `community`, versión `4.14.0`, `sqs: available`, sin errores de licencia |
| Cola Standard con `RedrivePolicy` (`maxReceiveCount: 2`) | **CONFIRMED** |
| `ApproximateReceiveCount` | **CONFIRMED**: 1 y 2 en recepciones sucesivas |
| `ChangeMessageVisibility` | **CONFIRMED**: visibilidad 0 hace el mensaje recibible de inmediato |
| Redrive a la DLQ al superar `maxReceiveCount` | **CONFIRMED**: tercera recepción vacía en la cola principal; mensaje presente en la DLQ |
| Long polling (`WaitTimeSeconds`) | **CONFIRMED**: la recepción espera en vez de volver de inmediato |

Conclusión: no hace falta token ni ElasticMQ. La entrada 15 de las aclaraciones funcionales (cambio condicionado a ElasticMQ) **no se activa**; HV-010, TC-005, TC-012 y DEL-005 no cambian.

### 4.2 DynamoDB Local 3.3.1 — items 1, 2 y 3 (parcial)

| Comprobación | Resultado |
|---|---|
| `TransactWriteItems` de 14 items en 14 partition keys distintas | **CONFIRMED** |
| Motivos de cancelación por item | **CONFIRMED** a nivel de servicio: `TransactionCanceledException` con la lista posicional `[None × 7, ConditionalCheckFailed, None × 6]` |
| Atomicidad ante la cancelación | **CONFIRMED**: los demás items quedan sin cambios |

Pendiente para los spikes del backend: exposición de los motivos y del item `ALL_OLD` en el SDK asíncrono de Java; aislamiento frente a escrituras concurrentes; lectura por lotes consistente de 100 claves (item 5); GSI disperso (item 11).

## 5. Fuera de este artefacto (spikes del backend)

| Item §13 | Motivo |
|---|---|
| 18–20 | Resilience4j/Reactor, SDK de AWS y funcionalidades de Spring Boot 4.x sobre Java 25 requieren código |
| 23–24 | Plugin de Spring Boot 4.x con Maven 3.9.16 y herramientas de prueba (cobertura, Mockito, detector de bloqueo, Testcontainers) con Java 25 requieren el build |
| — | Testcontainers sobre Docker Desktop en Windows (named pipe) se verifica en el primer incremento con pruebas de integración |

## 6. Observaciones

- El disco de Docker Desktop está en D: (`CustomWslDistroDir: D:\DockerData\DockerDesktopWSL`; `docker_data.vhdx` de 49,8 GB), no en C:. Límite ampliado por el usuario el 2026-10-04 de 64 GB a ≈ 162 GB (`DiskSizeMiB: 165536`); uso al momento de la ampliación ≈ 45 GB (imágenes 15,4 GB y caché de build 29,6 GB), de los que ≈ 33 GB son recuperables con limpieza de imágenes y caché no usadas. Las pruebas de integración añaden poco; con el nuevo límite hay margen amplio.
- La unidad C: está al 82 %. Solo el repositorio local de Maven (`%USERPROFILE%\.m2`, 1,6 GB actualmente) reside en C:. Si el espacio se vuelve un problema, moverlo a D: es una opción de configuración de usuario que no afecta al proyecto.
- Acceso de red verificado a Maven Central, Gradle Services y Docker Hub.
