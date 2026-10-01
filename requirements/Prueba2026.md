# Plataforma de Procesamiento de Eventos de Ticketing

## Descripción del Contexto

Una empresa de venta de entradas para eventos (conciertos, teatro, deportes) necesita modernizar su sistema de procesamiento de compras. Actualmente enfrentan problemas con picos de demanda cuando se liberan entradas para eventos populares, donde miles de usuarios intentan comprar simultáneamente.

El sistema actual basado en arquitectura tradicional no escala adecuadamente y presenta problemas de consistencia: usuarios reportan compras duplicadas, asientos vendidos a múltiples personas, y timeouts durante alta concurrencia. La empresa ha decidido migrar hacia una arquitectura reactiva y basada en eventos que pueda manejar miles de solicitudes concurrentes de forma eficiente.

Necesitan un sistema que gestione las entradas en tiempo real, procese las compras de forma asíncrona mediante colas de mensajes y garantice que no se vendan más entradas de las disponibles, manteniendo tiempos de respuesta bajos incluso bajo alta carga.

Las entradas pueden encontrarse en uno de los siguientes estados, los cuales determinan su disponibilidad, su relación con un cliente y su impacto en los distintos tipos de reportes (operativos, contables y de inventario).

AVAILABLE (Disponible)

RESERVED (Reservada)

PENDING_CONFIRMATION (En Confirmación)

SOLD (Vendida)

COMPLIMENTARY (Cortesía)

### Notas generales

- Cada entrada solo puede tener un estado a la vez.
- Las transiciones de estado deben ser atómicas y auditables.
- Los estados RESERVED y PENDING_CONFIRMATION no representan ventas.
- El estado SOLD es final e irreversible.
- El estado COMPLIMENTARY es final, pero no contable.
### Objetivo

Desarrollar un sistema backend reactivo que gestione la disponibilidad  y el procesamiento de compras de entradas para eventos. La aplicación debe exponer una API reactiva que permita consultar eventos disponibles y realizar compras, delegando el procesamiento pesado a consumidores asíncronos que garanticen la consistencia del inventario.

El sistema debe demostrar capacidad de manejar operaciones concurrentes sin bloqueos, utilizar persistencia NoSQL para acceso rápido a datos, y orquestar flujos asíncronos mediante mensajería entre componentes.

### Requisitos Funcionales

1. **Gestión de Eventos:** Permitir crear y consultar eventos con información básica (nombre, fecha, lugar, capacidad total). Cada evento debe tener un inventario de entradas disponibles que se actualiza conforme se procesan compras.

2. **Reserva Temporal de Entradas:** Cuando un usuario inicia una compra, el sistema debe reservar temporalmente las entradas solicitadas (máximo 10 minutos) para evitar sobreventa. Si no se confirma la compra en ese tiempo, las entradas vuelven al inventario disponible.

3. **Procesamiento Asíncrono de Compras:** Las solicitudes de compra deben encolarse inmediatamente y retornar un identificador de orden. Un consumidor procesa estas órdenes de forma asíncrona, validando disponibilidad real, actualizando inventario y cambiando el estado de la orden.

4. **Consulta de Estado de Orden:** Permitir consultar el estado actual de una orden de compra (AVAILABLE, RESERVED, PENDING_CONFIRMATION, SOLD, COMPLIMENTARY) en cualquier momento usando su identificador.

5. **Control de Concurrencia**: Implementar mecanismos que garanticen que múltiples solicitudes concurrentes para el mismo evento no resulten en sobreventa de entradas. El sistema debe manejar correctamente condiciones de carrera.

6. **Liberación Automática de Reservas Expiradas:** Implementar un proceso que periódicamente identifique y libere reservas que superaron el tiempo límite sin confirmarse, devolviendo las entradas al inventario disponible.

7. C**onsulta Reactiva de Disponibilidad:** Exponer un endpoint que retorne en tiempo real la disponibilidad actual de entradas para eventos específicos, considerando tanto entradas vendidas como reservadas temporalmente.

## Requisitos Técnicos

### Stack Tecnológico Obligatorio:

- **Java 25**: Utilizar características modernas del lenguaje (Records, Pattern Matching, Virtual Threads si aplica)

- **Spring Boot 4.x**: Framework base de la aplicación

- **Spring WebFlux**: Para implementar la API reactiva con programación no bloqueante

- Base de datos para persistencia de eventos, órdenes e inventario (puede usar DynamoDB Local u otra solución de persistencia)

- Cola de mensajes para procesamiento asíncrono de órdenes (puede usar LocalStack u cualquier otra solución que cumpla esta función)

- **Docker:** Contenedorización de la aplicación y servicios de infraestructura

- **Clean Architecture:** Organizar el código en capas claramente separadas (Domain, Use Cases, Infrastructure)

### Especificaciones Técnicas:

- La API debe ser completamente reactiva retornando `Mono` y `Flux` donde corresponda

- Implementar manejo de errores reactivo con estrategias de retry cuando sea apropiado

- Configurar SQS para garantizar procesamiento al menos una vez (at-least-once delivery)

- Las operaciones de actualización de inventario deben usar optimistic locking o conditional writes

- Proveer un `docker-compose.yml` que levante todos los servicios necesarios

### Entregables

1. **Repositorio de código fuente** con estructura de Clean Architecture donde:

- Nombres de variables, clases, métodos y comentarios deben estar en inglés

- Las capas de dominio, casos de uso e infraestructura estén claramente separadas

- El código siga principios SOLID y patrones de diseño apropiados

2. **Archivo README.md** que incluya:

- Descripción breve de la solución implementada

- Instrucciones detalladas de instalación y configuración

- Comandos para levantar la aplicación con Docker

- Descripción de decisiones arquitectónicas relevantes

- Ejemplos de uso de los endpoints principales

3. **Tests unitarios** con cobertura mínima del 90% que incluyan:

- Tests de casos de uso con mocks de repositorios

- Tests de componentes reactivos (WebFlux controllers y services)

- Tests que verifiquen el manejo correcto de concurrencia

- Usar frameworks como JUnit 5, Mockito y reactor-test

4. **Colección de solicitudes** (Postman, Insomnia o archivo curl) demostrando los flujos principales del sistema

5. **Archivo docker-compose.yml** configurado con la aplicación y todas las dependencias (DynamoDB Local, SQS/LocalStack).

## Forma de Evaluación

La evaluación de la prueba se realizará mediante una reunión técnica en la cual el candidato presentará la solución implementada y explicará las decisiones de diseño, arquitectura y tecnología adoptadas durante el desarrollo.

Durante la sesión se revisará la solución de manera estructurada, contrastando los requerimientos planteados con la implementación realizada, poniendo especial énfasis en el razonamiento técnico, los trade-offs asumidos y la experiencia demostrada en el diseño de sistemas distribuidos, cloud-native y seguros.

### Criterios de Evaluación

Cumplimiento funcional de los requerimientos

Se validará que los distintos puntos de la prueba estén correctamente abordados, evaluando tanto la implementación como la coherencia de la solución propuesta.

Calidad del diseño y decisiones técnicas (valor diferencial)

Se valorará la capacidad del candidato para justificar por qué la solución fue diseñada de esa manera, incluyendo:

Elección de patrones arquitectónicos

Manejo de concurrencia y consistencia

Uso adecuado de asincronía y procesamiento basado en eventos

Consideraciones de escalabilidad, resiliencia y tolerancia a fallos ( Adicional)

Seguridad de la solución

Se evaluará el enfoque de seguridad aplicado al sistema, considerando aspectos como:

Manejo seguro de secretos y credenciales

Consideraciones frente a ataques comunes (reintentos maliciosos, idempotencia, abuso de recursos)

Explicación clara y estructurada de la solución

Se otorgará valor adicional a la capacidad de comunicar la solución de forma clara y ordenada, apoyándose en:

Diagramas de arquitectura

Diagramas de flujo o secuencia

Explicación de las interacciones entre componentes

Infraestructura como Código (valor diferencial)

La definición de la infraestructura mediante Terraform, junto con la explicación del código y de las decisiones tomadas (networking, seguridad, escalabilidad, aislamiento de entornos), será considerada un factor diferencial en la evaluación.

Experiencia Cloud-Native en AWS (alto valor agregado)

Se valorará especialmente la demostración de experiencia práctica en AWS, incluyendo:

Buenas prácticas de seguridad en la nube

Enfoque cloud-native orientado a operación y producción

Consideraciones de costos, observabilidad y gobernanza

Madurez técnica y experiencia práctica

Más allá de la implementación, se evaluará la capacidad del candidato para reflexionar sobre:

Limitaciones de la solución

Posibles mejoras

Qué decisiones cambiaría en un entorno productivo real
