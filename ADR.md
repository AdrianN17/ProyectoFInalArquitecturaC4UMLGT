# Architecture Decision Records — HEPAQ
## Plataforma de Gestión Médica EsSalud

> **Formato:** [MADR](https://adr.github.io/madr/) (Markdown Any Decision Records)
> **Proyecto:** HEPAQ — EsSalud Medical Management Platform
> **Equipo:** Arquitectura & Backend
> **Última actualización:** 2026-06-18

---

## Índice

| ID | Título | Estado |
|----|--------|--------|
| [ADR-001](#adr-001) | Arquitectura de Microservicios como estilo principal | ✅ Aceptado |
| [ADR-002](#adr-002) | Arquitectura Hexagonal por microservicio | ✅ Aceptado |
| [ADR-003](#adr-003) | CQRS con modelos de escritura y lectura separados | ✅ Aceptado |
| [ADR-004](#adr-004) | Comunicación asíncrona mediante Event-Driven Architecture | ✅ Aceptado |
| [ADR-005](#adr-005) | Programación reactiva con Spring WebFlux | ✅ Aceptado |
| [ADR-006](#adr-006) | Microsoft Azure como plataforma de nube | ✅ Aceptado |
| [ADR-007](#adr-007) | Azure API Management como API Gateway | ✅ Aceptado |
| [ADR-008](#adr-008) | Microsoft Entra ID para identidad y control de acceso | ✅ Aceptado |
| [ADR-009](#adr-009) | Java 21 con Virtual Threads (Project Loom) | ✅ Aceptado |
| [ADR-010](#adr-010) | Azure SQL Database para el modelo de escritura (Write Model) | ✅ Aceptado |
| [ADR-011](#adr-011) | Azure Cosmos DB para el modelo de lectura (Read Model) | ✅ Aceptado |
| [ADR-012](#adr-012) | Azure Cache for Redis como caché distribuida | ✅ Aceptado |
| [ADR-013](#adr-013) | Liquibase YAML para migraciones de esquema SQL | ✅ Aceptado |
| [ADR-014](#adr-014) | UUID v4 como clave primaria en todas las entidades | ✅ Aceptado |
| [ADR-015](#adr-015) | Optimistic Locking con campo `version` | ✅ Aceptado |
| [ADR-016](#adr-016) | Soft Delete como estrategia de eliminación | ✅ Aceptado |
| [ADR-017](#adr-017) | Azure Blob Storage para archivos médicos adjuntos | ✅ Aceptado |
| [ADR-018](#adr-018) | Azure Table Storage para el catálogo CIE-10 | ✅ Aceptado |
| [ADR-019](#adr-019) | Azure Service Bus para mensajería entre microservicios | ✅ Aceptado |
| [ADR-020](#adr-020) | Azure Key Vault para gestión de secretos | ✅ Aceptado |
| [ADR-021](#adr-021) | MapStruct para mapeo entre capas | ✅ Aceptado |
| [ADR-022](#adr-022) | OpenAPI 3.0.3 como contrato de API | ✅ Aceptado |
| [ADR-023](#adr-023) | Thymeleaf para plantillas de correo electrónico | ✅ Aceptado |
| [ADR-024](#adr-024) | Chain of Responsibility para validación de citas | ✅ Aceptado |
| [ADR-025](#adr-025) | Template Method para casos de uso de consultas | ✅ Aceptado |
| [ADR-026](#adr-026) | Decorator para caché en MS Diagnósticos | ✅ Aceptado |
| [ADR-027](#adr-027) | Strategy + Factory para procesamiento de exámenes | ✅ Aceptado |
| [ADR-028](#adr-028) | Composite para notificaciones multi-canal | ✅ Aceptado |
| [ADR-029](#adr-029) | Angular + Capacitor como stack frontend unificado | ✅ Aceptado |
| [ADR-030](#adr-030) | RBAC mediante roles de Entra ID (no ABAC) | ✅ Aceptado |
| [ADR-031](#adr-031) | Modelo de auditoría mediante tablas dedicadas | ✅ Aceptado |
| [ADR-032](#adr-032) | Estrategia de retry con backoff exponencial | ✅ Aceptado |
| [ADR-033](#adr-033) | C4 Model para documentación de arquitectura | ✅ Aceptado |

---

## ADR-001

### Arquitectura de Microservicios como estilo principal

**Estado:** Aceptado
**Fecha:** 2025-03-01
**Decisores:** Equipo de Arquitectura

#### Contexto

HEPAQ requiere gestionar dominios médicos claramente diferenciados: citas, consultas, diagnósticos, exámenes y notificaciones. Cada uno tiene lógica, datos y ciclos de vida distintos. El sistema debe escalar de forma independiente, permitir despliegues sin afectar otras funcionalidades y soportar equipos autónomos.

#### Decisión

Descomponer el sistema en **5 microservicios independientes**, uno por dominio de negocio. Cada MS tiene su propio repositorio de datos, sus propias dependencias de despliegue y no comparte base de datos con ningún otro.

#### Alternativas consideradas

| Alternativa | Motivo de rechazo |
|---|---|
| Monolito modular | No permite escalar partes costosas (ej. MS Exámenes con Blob Storage) de forma independiente |
| SOA clásica | Acoplamiento a través de ESB (Enterprise Service Bus) — bus centralizado = punto único de fallo |
| Serverless Functions | Dificultad para mantener estado, cold starts inadmisibles para servicios médicos críticos |

#### Consecuencias

**Positivas:**
- Despliegue independiente por dominio
- Escalabilidad horizontal granular
- Aislamiento de fallos (un MS caído no derrumba el sistema completo)
- Equipos autónomos por dominio

**Negativas:**
- Mayor complejidad operacional (5 servicios, 5 bases de datos, observabilidad distribuida)
- Eventual consistencia entre dominios
- Latencia adicional en llamadas inter-servicio

---

## ADR-002

### Arquitectura Hexagonal por microservicio

**Estado:** Aceptado
**Fecha:** 2025-03-05
**Decisores:** Equipo de Arquitectura

#### Contexto

La lógica de dominio tiende a acoplarse a frameworks (Spring Data, Azure SDKs) y detalles de infraestructura. Esto dificulta las pruebas unitarias y hace costosos los cambios tecnológicos. En un contexto médico, la lógica de negocio debe ser el núcleo estable e independiente.

#### Decisión

Aplicar **Arquitectura Hexagonal (Ports & Adapters)** en cada microservicio. La estructura interna de cada MS es:

```
domain/        → Entidades, Value Objects, Enums, lógica pura
application/   → Casos de uso, puertos (interfaces)
infrastructure/
  ├── adapters/  → JPA, Cosmos, Redis, ServiceBus, REST Clients
  ├── rest/      → Controllers, DTOs, Mappers (entrada)
  └── config/    → Beans de Spring, configuraciones
```

Los puertos son interfaces Java en la capa `application`; los adaptadores en `infrastructure` implementan esas interfaces.

#### Alternativas consideradas

| Alternativa | Motivo de rechazo |
|---|---|
| Arquitectura en capas tradicional (Controller → Service → Repository) | Acoplamiento implícito entre capas; dificulta pruebas sin Spring Context |
| Clean Architecture estricta | Mayor overhead de carpetas y abstracciones para el tamaño del proyecto |

#### Consecuencias

**Positivas:**
- Dominio 100% testeable sin levantar Spring
- Cambiar de Azure SQL a PostgreSQL implica solo crear un nuevo adaptador
- Responsabilidades claras y separación de concerns

**Negativas:**
- Mayor número de interfaces e implementaciones por MS
- Curva de aprendizaje para developers sin experiencia en el patrón

---

## ADR-003

### CQRS con modelos de escritura y lectura separados

**Estado:** Aceptado
**Fecha:** 2025-03-10
**Decisores:** Equipo de Arquitectura

#### Contexto

Las operaciones de escritura (agendar cita, registrar diagnóstico) tienen requisitos de consistencia transaccional muy distintos a las operaciones de lectura (historial de paciente, lista de citas del día). Un modelo único genera índices de lectura en tablas relacionales que degradan la escritura, y consultas complejas que mezclan lógica transaccional con proyecciones de UI.

#### Decisión

Implementar **CQRS** en cada microservicio:

- **Write Side:** Azure SQL Database (modelo relacional normalizado, transacciones ACID, Liquibase para migraciones)
- **Read Side:** Azure Cosmos DB (documentos desnormalizados, optimizados para consultas de UI por `pacienteId`)

La sincronización se realiza vía eventos en Azure Service Bus: al completarse un comando, se publica un evento; un consumer actualiza la proyección en Cosmos DB.

#### Alternativas consideradas

| Alternativa | Motivo de rechazo |
|---|---|
| CRUD simple con un modelo | No escala para reportes clínicos; N+1 queries en historial de paciente |
| Event Sourcing completo | Complejidad excesiva; auditoría requerida se cubre con tablas de auditoría dedicadas |
| Read replica de SQL | Latencia de replicación similar, pero sin flexibilidad de esquema para proyecciones |

#### Consecuencias

**Positivas:**
- Escritura optimizada transaccionalmente (SQL relacional)
- Lectura sin joins, sub-10ms en Cosmos (indexación por `pacienteId`)
- Escalabilidad independiente de read/write paths

**Negativas:**
- Consistencia eventual en el Read Model (~segundos de lag)
- Complejidad de sincronización y gestión de eventos fallidos

---

## ADR-004

### Comunicación asíncrona mediante Event-Driven Architecture

**Estado:** Aceptado
**Fecha:** 2025-03-10
**Decisores:** Equipo de Arquitectura

#### Contexto

Varios flujos médicos requieren encadenar microservicios: al agendar una cita → notificar al paciente; al completar una consulta → crear diagnóstico; al completar un examen → notificar resultados. El acoplamiento síncrono directo entre MS aumenta la fragilidad del sistema.

#### Decisión

Usar **comunicación asíncrona mediante eventos** publicados en Azure Service Bus:

- Un MS publica un evento de dominio al completar una operación
- Los MS interesados suscriben a ese evento y reaccionan de forma independiente
- No hay llamadas REST directas entre microservicios (excepto hacia sistemas externos: RENIEC, Directorio Médico)

Los eventos siguen el formato: `{entidad}.{accion}` (ej. `cita.agendada`, `consulta.completada`, `examen.resultado-disponible`).

#### Alternativas consideradas

| Alternativa | Motivo de rechazo |
|---|---|
| REST síncrono entre MS | Acoplamiento temporal; si MS Notificaciones cae, MS Citas también falla |
| gRPC entre MS | Acoplamiento de contrato binario; dificultad de debug; overkill para frecuencia actual |
| Apache Kafka | Infraestructura externa; Azure Service Bus ya disponible en el tenant de EsSalud |

#### Consecuencias

**Positivas:**
- Desacoplamiento temporal entre productores y consumidores
- MS Notificaciones puede caer y recuperarse procesando mensajes en cola
- Fácil incorporar nuevos consumidores sin modificar el productor

**Negativas:**
- Trazabilidad distribuida requiere correlation IDs y distributed tracing
- Orden de eventos no garantizado salvo sesiones de Service Bus

---

## ADR-005

### Programación reactiva con Spring WebFlux

**Estado:** Aceptado
**Fecha:** 2025-03-12
**Decisores:** Equipo Backend

#### Contexto

Los microservicios deben manejar múltiples solicitudes concurrentes (médicos, pacientes, sistemas de laboratorio) con baja latencia. El modelo de threading clásico (un hilo por petición) genera contención bajo carga. Java 21 introduce Virtual Threads como alternativa, pero Spring WebFlux ya es la apuesta de Spring Boot para reactive.

#### Decisión

Usar **Spring Boot 3.x con WebFlux (Project Reactor)** para todos los microservicios. Los endpoints retornan `Mono<T>` o `Flux<T>`. Las operaciones de base de datos usan Spring Data R2DBC (SQL) y Spring Data for Azure Cosmos DB (reactive).

Adicionalmente, habilitar **Virtual Threads (Project Loom)** de Java 21 en operaciones de bloqueo restantes (Jakarta Mail, SDKs sin soporte reactivo nativo).

#### Alternativas consideradas

| Alternativa | Motivo de rechazo |
|---|---|
| Spring MVC (blocking) | No aprovecha el hardware; 200 threads concurrentes vs miles de virtual threads |
| Quarkus | Menor madurez del ecosistema en el contexto de EsSalud; curva de adopción del equipo |
| Micronaut | Similar a Quarkus; preferencia del equipo por Spring ecosystem |

#### Consecuencias

**Positivas:**
- Alto throughput con pocos recursos de cómputo
- Backpressure nativo en streams de datos
- Integración natural con Azure Cosmos DB SDK reactivo

**Negativas:**
- Curva de aprendizaje de programación reactiva (Mono/Flux, operadores)
- Stack traces reactivos son más difíciles de leer en debugging
- Algunas librerías legacy no son compatibles (requieren wrappers con `Schedulers.boundedElastic()`)

---

## ADR-006

### Microsoft Azure como plataforma de nube

**Estado:** Aceptado
**Fecha:** 2025-02-15
**Decisores:** Dirección TI EsSalud, Equipo de Arquitectura

#### Contexto

EsSalud tiene un acuerdo institucional con Microsoft Azure y su equipo de infraestructura ya opera servicios en este proveedor. La plataforma debe aprovechar servicios administrados para reducir la carga operacional del equipo de desarrollo.

#### Decisión

Utilizar **Microsoft Azure** como único proveedor de nube, aprovechando servicios PaaS:

| Necesidad | Servicio Azure |
|---|---|
| Base de datos relacional | Azure SQL Database (General Purpose) |
| Base de datos documental | Azure Cosmos DB (API NoSQL) |
| Caché distribuida | Azure Cache for Redis |
| Mensajería | Azure Service Bus (Standard) |
| Almacenamiento de archivos | Azure Blob Storage |
| Catálogo CIE-10 | Azure Table Storage |
| API Gateway | Azure API Management |
| Identidad | Microsoft Entra ID |
| Secretos | Azure Key Vault |
| Contenedores | Azure Container Apps |

#### Alternativas consideradas

| Alternativa | Motivo de rechazo |
|---|---|
| AWS | Sin acuerdo institucional vigente; migración costosa de identidad |
| GCP | Menor adopción en el ecosistema de salud peruano |
| On-premise | Alto costo de CapEx; sin elasticidad para picos de demanda |

#### Consecuencias

**Positivas:**
- Servicios administrados reducen overhead operacional
- Integración nativa entre servicios Azure (Entra ID → Key Vault → SQL → Cosmos)
- SLA garantizados por Microsoft
- Cumplimiento normativo ISO 27001, SOC 2, HIPAA (relevante para datos médicos)

**Negativas:**
- Vendor lock-in con Azure
- Costos variables según uso
- Dependencia de disponibilidad regional de Azure

---

## ADR-007

### Azure API Management como API Gateway

**Estado:** Aceptado
**Fecha:** 2025-03-15
**Decisores:** Equipo de Arquitectura

#### Contexto

Los microservicios exponen APIs que deben ser accesibles desde el frontend web (Angular), la app móvil (Capacitor) y potencialmente desde sistemas externos. Se necesita un punto de entrada único para: enrutamiento, autenticación centralizada, rate limiting, transformación de requests, y observabilidad de APIs.

#### Decisión

Usar **Azure API Management (APIM)** como único punto de entrada a todos los microservicios. APIM maneja:

- Validación de tokens JWT emitidos por Entra ID (política `validate-jwt`)
- Rate limiting por cliente (política `rate-limit-by-key`)
- CORS centralizado
- Transformación de cabeceras y routing hacia MS internos
- Caché de respuestas GET frecuentes (con política `cache-lookup`)
- Generación de portal de desarrolladores con las specs OpenAPI

Los MS internos no son accesibles directamente desde internet; solo APIM tiene acceso a ellos mediante VNet Integration.

#### Alternativas consideradas

| Alternativa | Motivo de rechazo |
|---|---|
| Spring Cloud Gateway | Requiere mantener otra aplicación Java; duplica esfuerzo con APIM ya disponible |
| Nginx / Kong | Requiere gestión de infraestructura adicional |
| Acceso directo a MS | Sin seguridad centralizada; duplicación de lógica de autenticación en cada MS |

#### Consecuencias

**Positivas:**
- Autenticación delegada al gateway; los MS confían en el token ya validado
- Dashboard de métricas de API nativo (latencia, error rate, throughput)
- Versionado de APIs transparente para los clientes

**Negativas:**
- Latencia adicional (~2-5ms) por pasar por APIM
- Costo de la tier Standard/Premium de APIM
- Políticas XML de APIM tienen una curva de aprendizaje

---

## ADR-008

### Microsoft Entra ID para identidad y control de acceso

**Estado:** Aceptado
**Fecha:** 2025-03-15
**Decisores:** Equipo de Seguridad, Equipo de Arquitectura

#### Contexto

El sistema maneja datos médicos sensibles. Los usuarios (médicos, digitadores, pacientes) deben autenticarse de forma segura. La gestión manual de contraseñas y tokens es un riesgo de seguridad. EsSalud ya tiene cuentas institucionales en Entra ID (ex Azure AD).

#### Decisión

Usar **Microsoft Entra ID** como Identity Provider (IdP):

- Flujo: **Authorization Code Flow + PKCE** para web/mobile
- Tokens: **JWT (Access Token + ID Token)**, firmados por Entra ID
- Autorización: Roles embebidos en el claim `roles` del JWT (`Medico`, `Digitador`, `Paciente`, `Admin`)
- Los MS validan el JWT con la clave pública de Entra ID (JWKS endpoint)
- Spring Security Resource Server con `spring-security-oauth2-resource-server`

#### Alternativas consideradas

| Alternativa | Motivo de rechazo |
|---|---|
| Keycloak self-hosted | Requiere operar el servidor de identidad; alta disponibilidad costosa |
| Auth0 | Costo adicional; sin integración nativa con cuentas institucionales EsSalud |
| Gestión propia de usuarios y JWT | Riesgo de seguridad inaceptable para datos médicos |

#### Consecuencias

**Positivas:**
- SSO con cuentas institucionales existentes de EsSalud
- MFA gestionado por Microsoft
- Rotación automática de claves de firma
- Cumplimiento OWASP A07:2021 (Identification and Authentication Failures)

**Negativas:**
- Dependencia de disponibilidad de Entra ID
- Gestión de grupos y roles debe mantenerse en el portal de Azure

---

## ADR-009

### Java 21 con Virtual Threads (Project Loom)

**Estado:** Aceptado
**Fecha:** 2025-03-01
**Decisores:** Equipo Backend

#### Contexto

Se necesita un lenguaje con ecosistema maduro para servicios backend médicos, con soporte LTS garantizado y amplia adopción en el mercado peruano. Java 21 es la versión LTS más reciente, con la inclusión de Virtual Threads como característica estable.

#### Decisión

Usar **Java 21 (LTS)** como lenguaje de todos los microservicios, con:

- **Spring Boot 3.x** (requiere mínimo Java 17)
- **Virtual Threads** habilitados (`spring.threads.virtual.enabled=true`) para operaciones de bloqueo
- **Records** para DTOs y Value Objects inmutables
- **Sealed classes** para modelar estados de dominio exhaustivos
- **Pattern matching** (`instanceof`, `switch` expressions) para lógica de dominio

#### Alternativas consideradas

| Alternativa | Motivo de rechazo |
|---|---|
| Java 17 LTS | Virtual Threads no estables (Preview); perder mejoras de Java 21 |
| Kotlin | Menor disponibilidad de developers Kotlin senior en mercado peruano |
| Node.js | Ecosistema menos maduro para lógica transaccional y tipo-seguro |
| Go | Sin Spring/JPA/Liquibase ecosystem; reentrenamiento completo del equipo |

#### Consecuencias

**Positivas:**
- Virtual Threads permite simplificar código reactivo en integraciones blocking (SMTP, SDK legacy)
- Records reducen boilerplate en DTOs
- LTS garantiza soporte hasta 2031

**Negativas:**
- Virtual Threads requiere librerías sin thread-local pinning (algunas librerías legacy incompatibles)
- Mayor consumo de memoria que Go en baseline

---

## ADR-010

### Azure SQL Database para el modelo de escritura (Write Model)

**Estado:** Aceptado
**Fecha:** 2025-03-10
**Decisores:** Equipo de Arquitectura, DBA

#### Contexto

Las operaciones de escritura (crear cita, registrar diagnóstico, actualizar estado) requieren integridad transaccional estricta: constraints, FK, transacciones ACID. Los datos médicos no pueden quedar en estado inconsistente.

#### Decisión

Usar **Azure SQL Database** (SQL Server PaaS) como almacén del Write Model:

- Tier: **General Purpose** (puede escalar a Business Critical si se requiere HA síncrona)
- Acceso: Spring Data JPA + Hibernate (ORM) + Liquibase para migraciones
- Connection pool: HikariCP (incluido en Spring Boot)
- Cada microservicio tiene su **propia base de datos** (no base de datos compartida)

#### Alternativas consideradas

| Alternativa | Motivo de rechazo |
|---|---|
| PostgreSQL (Azure Database for PostgreSQL) | Sin acuerdo de soporte institucional; mayor experiencia del equipo en SQL Server |
| MongoDB como único store | Sin soporte nativo de transacciones multi-document en todos los casos de uso |
| Azure Cosmos DB for NoSQL solo | Sin ACID multi-item garantizado para operaciones complejas |

#### Consecuencias

**Positivas:**
- ACID garantizado para operaciones médicas críticas
- Familiar para el equipo de DBA de EsSalud
- Full-text search, JSON functions de SQL Server disponibles
- Geo-replication y point-in-time restore nativo

**Negativas:**
- Costos de licenciamiento incluidos en Azure SQL (más caro que PostgreSQL)
- Esquema rígido requiere migraciones para cambios

---

## ADR-011

### Azure Cosmos DB para el modelo de lectura (Read Model)

**Estado:** Aceptado
**Fecha:** 2025-03-10
**Decisores:** Equipo de Arquitectura

#### Contexto

Las consultas de lectura (historial médico de paciente, citas del día de un médico, lista de diagnósticos) requieren respuesta sub-100ms y estructuras desnormalizadas adaptadas a la UI. Un modelo relacional con joins complejos no cumple estos requisitos de latencia.

#### Decisión

Usar **Azure Cosmos DB (API NoSQL)** como almacén del Read Model:

- **Partition key:** `/pacienteId` en todos los contenedores (el acceso más frecuente es "dame todo de este paciente")
- **Consistency level:** Session Consistency (balance entre consistencia y latencia)
- **Indexing:** Automático en todos los campos (ajustable por contenedor)
- Los documentos son proyecciones desnormalizadas del agregado completo (ej. `CitaDocument` incluye nombre del médico, especialidad)
- Actualización mediante consumers de eventos de Azure Service Bus

#### Alternativas consideradas

| Alternativa | Motivo de rechazo |
|---|---|
| Azure SQL Read Replica | Misma estructura relacional; joins igualmente costosos para proyecciones de UI |
| Elasticsearch | Excelente para búsqueda, pero overhead de operación adicional; Cosmos ya disponible |
| Redis como único Read Model | Datos efímeros; no adecuado como fuente de verdad del Read Model |

#### Consecuencias

**Positivas:**
- Latencia P99 < 10ms con partition key correcto
- Schema flexible: agregar campos al documento sin migración
- Escalabilidad horizontal ilimitada (RU/s autoscale)
- SDK reactivo nativo para Spring WebFlux

**Negativas:**
- Consistencia eventual (segundos de lag respecto al Write Model)
- Costos por RU/s (Request Units); queries sin partition key consumen más RU
- Sin soporte de JOINs ni transacciones entre contenedores

---

## ADR-012

### Azure Cache for Redis como caché distribuida

**Estado:** Aceptado
**Fecha:** 2025-03-20
**Decisores:** Equipo de Arquitectura

#### Contexto

Algunas consultas de Cosmos DB son repetitivas y caras en RU/s (ej. catálogo de médicos, lista de tipos de examen, diagnósticos recientes de un paciente). Adicionalmente, las sesiones de APIM y los tokens necesitan almacenamiento temporal de alta velocidad.

#### Decisión

Usar **Azure Cache for Redis** como caché distribuida:

- **TTL estándar:** 5 minutos para proyecciones médicas frecuentes
- **Patrón:** Cache-Aside (la aplicación gestiona la escritura/invalidación del caché)
- **Implementación en MS Diagnósticos:** Patrón Decorator sobre `DiagnosticoRepositoryPort`
- **Serialización:** JSON (Jackson) para compatibilidad entre instancias del MS
- **Tier:** Basic (desarrollo) / Standard C1 (producción con replicación)

#### Alternativas consideradas

| Alternativa | Motivo de rechazo |
|---|---|
| Caché en memoria (Caffeine) | No compartido entre instancias del MS; inconsistencias en entornos escalados |
| Azure Cosmos DB TTL | TTL en documentos, no caché de aplicación; diferente propósito |
| Memcached | Sin soporte de estructuras de datos avanzadas; menos integrado con Azure |

#### Consecuencias

**Positivas:**
- Reducción de carga en Cosmos DB (menos RU/s consumidos)
- Respuestas de catálogos en sub-1ms
- Compartido entre múltiples instancias del mismo MS (horizontal scaling)

**Negativas:**
- Invalidación de caché puede ser compleja (cache stampede bajo carga)
- Coste adicional de la instancia Redis
- Datos en caché pueden estar ligeramente desactualizados (eventual consistency adicional)

---

## ADR-013

### Liquibase YAML para migraciones de esquema SQL

**Estado:** Aceptado
**Fecha:** 2025-03-10
**Decisores:** Equipo Backend, DBA

#### Contexto

Los esquemas de base de datos evolucionan con el producto. Es necesario versionar los cambios de esquema, garantizar idempotencia y poder aplicar migraciones de forma reproducible en todos los entornos (dev, staging, producción).

#### Decisión

Usar **Liquibase 4.x** con formato de changelog **YAML** (no XML ni SQL nativo):

- Un archivo `changelog_ms_<nombre>.yaml` por microservicio
- Cada changeset tiene `preConditions` con `onFail: MARK_RAN` para idempotencia
- La migración se ejecuta automáticamente al arrancar el MS (`spring.liquibase.enabled=true`)
- Formato YAML elegido sobre XML por mayor legibilidad y soporte nativo de comentarios

El changelog de MS Notificaciones incluye un changeset `006` con `context: "!test"` para datos semilla que no se aplican en entornos de prueba.

#### Alternativas consideradas

| Alternativa | Motivo de rechazo |
|---|---|
| Flyway | Menos flexible para precondiciones y rollbacks; Liquibase más maduro en contextos enterprise |
| SQL scripts manuales | Sin versionado, sin idempotencia, sin rollback estructurado |
| Liquibase XML | Más verboso que YAML; YAML preferido por el equipo |

#### Consecuencias

**Positivas:**
- Migraciones reproducibles y versionadas en Git
- Idempotencia garantizada: se puede ejecutar múltiples veces sin efectos secundarios
- Rollback automático en caso de error durante startup

**Negativas:**
- Liquibase agrega tiempo al startup del MS
- Los changesets son inmutables una vez aplicados en producción (requiere nuevos changesets para correcciones)

---

## ADR-014

### UUID v4 como clave primaria en todas las entidades

**Estado:** Aceptado
**Fecha:** 2025-03-05
**Decisores:** Equipo Backend, DBA

#### Contexto

Las entidades de dominio (Cita, Consulta, Diagnóstico, Examen, Notificación) necesitan identificadores únicos. La generación de estos IDs no debe depender de la base de datos para soportar CQRS (el ID debe conocerse antes del INSERT, al construir el evento de dominio).

#### Decisión

Usar **UUID v4** (`java.util.UUID.randomUUID()`) como clave primaria en todas las entidades:

- Tipo en Azure SQL: `uniqueidentifier`
- Generación: en la capa de dominio/aplicación (no auto-generado por la DB)
- Representación externa: string lowercase con guiones (`xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx`)

#### Alternativas consideradas

| Alternativa | Motivo de rechazo |
|---|---|
| IDENTITY autoincremental (int/bigint) | Dependencia de la DB para conocer el ID; incompatible con CQRS y eventos pre-persistencia |
| ULID (Universally Unique Lexicographically Sortable ID) | Menor soporte nativo en Azure SQL; UUID más estándar |
| UUID v7 (time-ordered) | Estándar aún joven al momento de la decisión (RFC 9562 junio 2024) |

#### Consecuencias

**Positivas:**
- IDs generados en la aplicación antes de persistir (compatible con CQRS)
- Sin colisiones entre bases de datos distribuidas
- Opacidad: no revela conteos ni secuencias de negocio

**Negativas:**
- Mayor espacio de almacenamiento (16 bytes vs 4/8 bytes de int/bigint)
- Índices clustered por UUID fragmentan páginas en SQL Server (mitigado con índices non-clustered o NEWSEQUENTIALID en casos críticos)

---

## ADR-015

### Optimistic Locking con campo `version`

**Estado:** Aceptado
**Fecha:** 2025-03-15
**Decisores:** Equipo Backend

#### Contexto

En un entorno médico, dos usuarios pueden intentar modificar el mismo registro simultáneamente (ej. dos médicos actualizando la misma cita). Es necesario detectar conflictos de escritura concurrente sin usar locks pesimistas que degraden el throughput.

#### Decisión

Agregar un campo `version INT DEFAULT 0 NOT NULL` a todas las entidades mutables. JPA/Hibernate maneja automáticamente el incremento y la validación mediante `@Version`:

```java
@Version
private int version;
```

En caso de conflicto, Hibernate lanza `OptimisticLockException`, que se convierte en HTTP 409 Conflict en la capa REST.

#### Alternativas consideradas

| Alternativa | Motivo de rechazo |
|---|---|
| Pessimistic Locking (SELECT FOR UPDATE) | Bloqueos en DB degradan concurrencia; riesgo de deadlocks |
| Sin control de concurrencia | Condiciones de carrera ("last write wins") inadmisibles en datos médicos |
| ETag en HTTP | Complementario pero no suficiente sin versión en DB |

#### Consecuencias

**Positivas:**
- Sin bloqueos en base de datos; alta concurrencia
- Detección explícita de conflictos
- Compatible con Spring Data JPA sin código adicional

**Negativas:**
- El cliente debe reintentar la operación al recibir HTTP 409
- Requiere que todos los UPDATE incluyan la versión en el payload

---

## ADR-016

### Soft Delete como estrategia de eliminación

**Estado:** Aceptado
**Fecha:** 2025-03-15
**Decisores:** Equipo de Arquitectura, Área Legal EsSalud

#### Contexto

Los datos médicos están sujetos a regulaciones de retención (Ley 26842 - Ley General de Salud, resoluciones ministeriales de historia clínica). Eliminar físicamente registros médicos puede ser ilegal. Los auditores requieren trazabilidad de quién, cuándo y por qué se eliminó un registro.

#### Decisión

Implementar **Soft Delete** en todas las entidades que pueden ser "eliminadas":

- Campo: `estado VARCHAR(20)` con valor `ELIMINADO` (no campo booleano `deleted`)
- Campo adicional: `motivo_eliminacion NVARCHAR(500)` y `eliminado_por VARCHAR(100)`
- Los queries de lectura filtran implícitamente `estado != 'ELIMINADO'`
- Spring Data JPA: `@Where(clause = "estado != 'ELIMINADO'")` o filtros en QueryDSL

El estado `ELIMINADO` se trata como un estado más del ciclo de vida, no como un flag separado.

#### Alternativas consideradas

| Alternativa | Motivo de rechazo |
|---|---|
| Hard delete + tabla de archivo | Complejidad de mover datos entre tablas; particionamiento más costoso |
| Campo `deleted BIT` | Menos expresivo; no captura el motivo; no integra con el modelo de estados |
| Ninguna eliminación permitida | No refleja la realidad del dominio (diagnósticos erróneos deben poderse anular) |

#### Consecuencias

**Positivas:**
- Cumplimiento regulatorio de retención de datos médicos
- Auditabilidad completa
- Rollback de eliminaciones accidentales sin restaurar backups

**Negativas:**
- Las queries siempre deben filtrar por estado; riesgo de mostrar datos "eliminados" si se omite el filtro
- Las tablas crecen indefinidamente (requiere política de archivado periódico a cold storage)

---

## ADR-017

### Azure Blob Storage para archivos médicos adjuntos

**Estado:** Aceptado
**Fecha:** 2025-03-20
**Decisores:** Equipo de Arquitectura

#### Contexto

Los exámenes médicos generan archivos binarios: imágenes DICOM, radiografías (PNG/JPEG), PDFs de laboratorio, videos de ecografía (MP4). Estos archivos pueden pesar hasta 50 MB. Almacenarlos en la base de datos (VARBINARY) o en el sistema de archivos del MS es inviable.

#### Decisión

Almacenar archivos en **Azure Blob Storage** (container `examenes-adjuntos`):

- La tabla `adjuntos_examen` en Azure SQL almacena solo los **metadatos** (nombre, content-type, tamaño, URL del blob)
- El MS Exámenes genera **SAS URLs** (Shared Access Signatures) con TTL corto (1 hora) para que el frontend descargue directamente desde Blob Storage sin pasar por el MS
- Constraint de tamaño máximo: **50 MB por archivo** (`tamanio_bytes <= 52428800`)
- Content types permitidos: `image/jpeg`, `image/png`, `application/pdf`, `application/dicom`, `video/mp4`

#### Alternativas consideradas

| Alternativa | Motivo de rechazo |
|---|---|
| VARBINARY(MAX) en SQL | Degradación masiva de performance de la DB; backups enormes |
| Sistema de archivos del servidor | Sin HA, sin CDN, sin replicación geográfica |
| Azure Files (SMB) | Diseñado para compartir entre VMs, no para acceso HTTP de archivos médicos |

#### Consecuencias

**Positivas:**
- Costo de almacenamiento muy bajo (< $0.02/GB/mes tier Cool)
- Descarga directa desde Blob Storage sin pasar por el MS (sin bottleneck)
- Replicación geográfica (GRS) para datos médicos críticos
- Soft delete nativo de Blob Storage como capa adicional de protección

**Negativas:**
- SAS URLs expiran; el cliente debe renovar si el TTL caduca
- Consistencia entre metadatos en SQL y el blob real requiere transaccional outbox o cleanup job

---

## ADR-018

### Azure Table Storage para el catálogo CIE-10

**Estado:** Aceptado
**Fecha:** 2025-03-20
**Decisores:** Equipo de Arquitectura

#### Contexto

El catálogo CIE-10 (Clasificación Internacional de Enfermedades) contiene ~14,000 códigos. Es un dato de referencia de solo lectura que se actualiza máximo una vez al año con nuevas revisiones. Almacenarlo en Azure SQL como tabla relacional es innecesariamente costoso para un dato de lookup.

#### Decisión

Almacenar el catálogo CIE-10 en **Azure Table Storage**:

- **PartitionKey:** Primera letra del código (A-Z, U para códigos especiales) → 26 particiones
- **RowKey:** Código CIE completo (ej. `I10`, `J45.0`, `E11.9`)
- Campos: `descripcion`, `categoriaCapitulo`, `subcategoria`, `activo`
- Actualización: script de importación anual desde OPS/OMS
- Acceso desde MS Diagnósticos: `Spring Data Azure Tables` o REST SDK

#### Alternativas consideradas

| Alternativa | Motivo de rechazo |
|---|---|
| Tabla `cie10` en Azure SQL | Costo de DTU para datos de solo lectura; tabla de 14k filas no justifica licencia SQL |
| Cosmos DB container separado | Más caro que Table Storage para acceso por clave primaria simple |
| Archivo JSON en Blob Storage | Sin capacidad de búsqueda por código; habría que cargar todo el archivo |

#### Consecuencias

**Positivas:**
- Costo mínimo (Table Storage es el servicio Azure más barato: ~$0.045/GB/mes)
- Acceso por clave O(1) por `PartitionKey + RowKey`
- Sin gestión de esquema ni migraciones

**Negativas:**
- Sin soporte de queries complejas (solo filter simple por partition/row key)
- Búsqueda por descripción requiere carga parcial o integración con Azure Cognitive Search

---

## ADR-019

### Azure Service Bus para mensajería entre microservicios

**Estado:** Aceptado
**Fecha:** 2025-03-10
**Decisores:** Equipo de Arquitectura

#### Contexto

La comunicación asíncrona entre microservicios requiere un broker de mensajes confiable con garantías de entrega (at-least-once), soporte de dead-letter queue para mensajes fallidos, y posibilidad de reintentos automáticos.

#### Decisión

Usar **Azure Service Bus (tier Standard)** con:

- **Topics y Subscriptions** para comunicación pub/sub (1 publicador → N consumidores)
- **Dead Letter Queue (DLQ)** automática para mensajes que fallan después de N intentos
- **Lock duration:** 5 minutos (tiempo máximo para procesar un mensaje antes de reencolar)
- **Max delivery count:** 10 intentos antes de mover a DLQ
- Formato de mensaje: JSON con envelope `{ "eventType": "cita.agendada", "occurredAt": "ISO8601", "payload": {...} }`
- Correlación: header `correlation-id` propagado por todos los MS para distributed tracing

#### Alternativas consideradas

| Alternativa | Motivo de rechazo |
|---|---|
| Apache Kafka (Azure HDInsight) | Mucho más costoso; overkill para el volumen de mensajes de EsSalud |
| Azure Event Hub | Optimizado para telemetría de alto volumen; demasiado para inter-MS médico |
| RabbitMQ | Requiere operar el broker; Azure Service Bus es PaaS administrado |

#### Consecuencias

**Positivas:**
- At-least-once delivery garantizado
- DLQ para análisis de mensajes fallidos
- Integración nativa con Azure Monitor para alertas de DLQ
- Sessions para garantizar orden cuando sea necesario

**Negativas:**
- Tier Standard tiene límites de throughput (1000 operaciones de mensajería brokered/hora gratis)
- Mensajes grandes (> 256 KB) requieren Claim Check pattern (referencia a Blob)

---

## ADR-020

### Azure Key Vault para gestión de secretos

**Estado:** Aceptado
**Fecha:** 2025-03-15
**Decisores:** Equipo de Seguridad

#### Contexto

Los microservicios requieren secretos: connection strings de Azure SQL, claves de Redis, credenciales SMTP, SAS keys de Blob Storage. Almacenar estos valores en archivos de configuración o variables de entorno en texto plano es un riesgo de seguridad (OWASP A02:2021 - Cryptographic Failures).

#### Decisión

Almacenar **todos los secretos** en **Azure Key Vault**:

- Los MS acceden a Key Vault en startup usando **Managed Identity** (sin credenciales explícitas)
- Integración: `spring-cloud-azure-starter-keyvault-secrets` (property source automático)
- Los secretos se referencian en `application.yml` como `${azure.keyvault.secret-name}`
- Rotación de secretos: Key Vault notifica via Event Grid; los MS recargan en caliente
- Acceso: solo las Managed Identities de los Container Apps tienen permisos `get` y `list`

#### Alternativas consideradas

| Alternativa | Motivo de rechazo |
|---|---|
| Variables de entorno en Container Apps | Visibles en el portal de Azure sin auditoría de acceso |
| Hashicorp Vault | Requiere operar el servidor de Vault; Azure Key Vault es PaaS |
| Archivos `.env` cifrados | Gestión manual de claves de cifrado; anti-patrón |

#### Consecuencias

**Positivas:**
- Cumplimiento OWASP A02 (no hay credenciales en código ni en entorno plano)
- Audit log de cada acceso a cada secreto
- Rotación sin redespliegue del MS

**Negativas:**
- Latencia adicional en startup (llamada a Key Vault para cargar propiedades)
- Requiere configurar Managed Identity por cada MS en Container Apps

---

## ADR-021

### MapStruct para mapeo entre capas

**Estado:** Aceptado
**Fecha:** 2025-03-12
**Decisores:** Equipo Backend

#### Contexto

En Arquitectura Hexagonal, los objetos de dominio (entidades) no deben exponerse directamente en la capa REST ni persistirse directamente con JPA. Se necesita transformar entre: `Entity ↔ JPA Entity`, `Entity ↔ DTO (REST)`, `Entity ↔ Document (Cosmos)`. Este mapeo manual es repetitivo y propenso a errores.

#### Decisión

Usar **MapStruct 1.5+** para generar mappers en tiempo de compilación:

```java
@Mapper(componentModel = "spring")
public interface CitaMapper {
    CitaDTO toDTO(Cita cita);
    Cita toDomain(AgendarCitaRequest request);
    CitaJpaEntity toJpaEntity(Cita cita);
}
```

MapStruct genera implementaciones Java en tiempo de compilación (procesador de anotaciones), sin reflection en runtime.

#### Alternativas consideradas

| Alternativa | Motivo de rechazo |
|---|---|
| ModelMapper | Reflection en runtime; más lento; mappings implícitos difíciles de depurar |
| Mapeo manual | Repetitivo y propenso a bugs en campos nuevos; no escala |
| BeanUtils (Spring) | Reflection; sin seguridad de tipos en compilación |

#### Consecuencias

**Positivas:**
- Compilación falla si hay campos no mapeados (seguridad de tipos)
- Cero overhead en runtime (código generado en bytecode)
- Integración nativa con Spring IoC (`componentModel = "spring"`)

**Negativas:**
- Requiere annotación explícita cuando los nombres de campos difieren
- Los mappers generados no son legibles por humanos (código generado en `target/`)

---

## ADR-022

### OpenAPI 3.0.3 como contrato de API

**Estado:** Aceptado
**Fecha:** 2025-03-25
**Decisores:** Equipo de Arquitectura, Equipo Frontend

#### Contexto

Los equipos frontend (Angular web, Capacitor mobile) necesitan un contrato claro y versionado de las APIs de los microservicios. Sin un contrato formal, los cambios de API rompen silenciosamente el frontend.

#### Decisión

Definir el contrato de cada MS usando **OpenAPI 3.0.3** en archivos YAML:

- Un archivo `openapi_ms_<nombre>.yaml` por microservicio
- El contrato es **spec-first** (el YAML define la API, el código implementa el contrato)
- Importado en Azure API Management para generar el portal de desarrolladores
- Los clients del frontend se generan automáticamente desde el spec con `openapi-generator`
- Versionado en el path: `/v1/citas`, `/v1/consultas`, etc.

#### Alternativas consideradas

| Alternativa | Motivo de rechazo |
|---|---|
| GraphQL | Overkill para las necesidades actuales; mayor complejidad en authorization granular |
| gRPC (Protobuf) | No nativo en browsers; requiere gRPC-Web; APIM sin soporte completo |
| Sin contrato formal (código primero) | Acoplamiento frágil con frontend; diffs entre entornos |

#### Consecuencias

**Positivas:**
- Generación automática de clientes TypeScript para Angular
- Documentación siempre actualizada y ejecutable (Swagger UI en APIM)
- Detección temprana de breaking changes en CI/CD

**Negativas:**
- El spec YAML debe mantenerse sincronizado con el código (riesgo de drift)
- Curva de aprendizaje de OpenAPI 3.x para el equipo

---

## ADR-023

### Thymeleaf para plantillas de correo electrónico

**Estado:** Aceptado
**Fecha:** 2025-04-01
**Decisores:** Equipo Backend

#### Contexto

MS Notificaciones envía correos HTML a pacientes y médicos: confirmaciones de citas, cancelaciones, resultados de exámenes. Los templates HTML deben soportar variables dinámicas (nombre del paciente, fecha, médico) y estar almacenados en base de datos para modificarse sin redespliegue.

#### Decisión

Usar **Thymeleaf 3.x** como motor de plantillas:

- Las plantillas se almacenan en la tabla `plantillas_notificacion` (campo `cuerpo_template`)
- Thymeleaf procesa la plantilla en memoria con el contexto de variables (`pacienteNombre`, `fechaCita`, etc.)
- El puerto `EmailTemplatePort` abstrae el rendering; el adaptador `ThymeleafEmailTemplateAdapter` implementa la lógica
- Envío SMTP vía Jakarta Mail con `spring-boot-starter-mail`

#### Alternativas consideradas

| Alternativa | Motivo de rechazo |
|---|---|
| FreeMarker | Menos integrado con el ecosistema Spring; sintaxis más compleja |
| Mustache | Muy simple; sin expresiones ni condicionales necesarios en templates médicos |
| Templates HTML estáticos con String.format() | Frágil, sin escape XSS automático, difícil de mantener |

#### Consecuencias

**Positivas:**
- Escape automático de variables (previene XSS en contenido de email)
- Templates editables en DB sin redespliegue del MS
- Soporte de condicionales e iteraciones en templates HTML

**Negativas:**
- Thymeleaf sin contexto web completo requiere configuración manual del `TemplateEngine` bean
- Plantillas HTML complejas requieren conocimiento de sintaxis Thymeleaf por el equipo de contenido

---

## ADR-024

### Chain of Responsibility para validación de citas

**Estado:** Aceptado
**Fecha:** 2025-04-05
**Decisores:** Equipo Backend

#### Contexto

Agendar una cita requiere múltiples validaciones: verificar que el paciente existe en RENIEC, validar su seguro en el Sistema de Aseguramiento EsSalud, y comprobar la disponibilidad del médico. Si `AgendarCitaUseCase` invoca directamente estos tres servicios, queda acoplado a tres puertos externos con lógica if-else explícita.

#### Decisión

Aplicar el patrón **Chain of Responsibility** para la validación:

```
AgendarCitaUseCase
    → ReniecValidacionHandler → SeguroValidacionHandler → DisponibilidadMedicoHandler
```

Cada handler encapsula su propia validación y llama al siguiente. Si falla, lanza una excepción de dominio específica. El `AgendarCitaUseCase` solo conoce el primer handler de la cadena.

#### Consecuencias

**Positivas:**
- Agregar nuevas validaciones = agregar un nuevo handler sin modificar el use case
- Cada handler es unitariamente testeable de forma independiente
- Orden explícito de validaciones (primero RENIEC, luego seguro, luego disponibilidad)

**Negativas:**
- Mayor número de clases
- El debugging requiere seguir la cadena de llamadas

---

## ADR-025

### Template Method para casos de uso de consultas

**Estado:** Aceptado
**Fecha:** 2025-04-05
**Decisores:** Equipo Backend

#### Contexto

Los tres casos de uso de MS Consultas (`RegistrarConsultaUseCase`, `EditarConsultaUseCase`, `CancelarConsultaUseCase`) comparten un esqueleto común: validar, persistir, publicar evento. Solo los pasos `persistir()` y `publicarEvento()` varían entre implementaciones.

#### Decisión

Aplicar el patrón **Template Method** con `ConsultaUseCaseTemplate` como clase abstracta que define el método `ejecutar(cmd)` con el algoritmo fijo y declara como abstractos los pasos variables:

```java
public abstract class ConsultaUseCaseTemplate {
    public final void ejecutar(ConsultaCommand cmd) {
        validar(cmd);      // implementado en la clase base
        persistir(cmd);    // abstracto: cada subclase implementa
        publicarEvento(cmd); // abstracto: cada subclase implementa
    }
}
```

#### Consecuencias

**Positivas:**
- Algoritmo de ejecución documentado y no duplicado
- Nuevos casos de uso solo sobreescriben los pasos variables

**Negativas:**
- Herencia (vs composición); las subclases quedan acopladas a la clase base
- Si el esqueleto cambia, impacta todas las subclases

---

## ADR-026

### Decorator para caché en MS Diagnósticos

**Estado:** Aceptado
**Fecha:** 2025-04-05
**Decisores:** Equipo Backend

#### Contexto

MS Diagnósticos debe cachear los diagnósticos frecuentes de un paciente en Redis. Si la lógica de caché se coloca en el caso de uso (`ObtenerDiagnosticosUseCase`), el use case viola SRP mezclando lógica de negocio con lógica de infraestructura.

#### Decisión

Aplicar el patrón **Decorator** sobre `DiagnosticoRepositoryPort`:

```
DiagnosticoCacheDecorator (implementa DiagnosticoRepositoryPort)
    → intenta leer de Redis
    → si miss → delega a DiagnosticoJpaAdapter
    → guarda resultado en Redis
```

El use case solo conoce `DiagnosticoRepositoryPort`; no sabe si hay caché o no.

#### Consecuencias

**Positivas:**
- El use case es ajeno a la existencia del caché (SRP cumplido)
- El Decorator puede activarse/desactivarse mediante configuración de Spring (`@ConditionalOnProperty`)
- Testeable: se puede inyectar el JPA adapter directo sin Decorator en tests

**Negativas:**
- Una capa adicional de indirección
- La invalidación del caché debe coordinarse con las operaciones de escritura

---

## ADR-027

### Strategy + Factory para procesamiento de exámenes

**Estado:** Aceptado
**Fecha:** 2025-04-05
**Decisores:** Equipo Backend

#### Contexto

Cada tipo de examen (`LABORATORIO`, `IMAGEN`, `ECOGRAFIA`) tiene un flujo de procesamiento distinto: diferente validación de resultados, diferente procesamiento de adjuntos, diferentes reglas de notificación. Un `switch` en el use case acopla todos los casos en una clase.

#### Decisión

Aplicar **Strategy + Factory Method**:

- Interfaz `ExamenProcesadorStrategy` con `procesar(Examen)`
- Implementaciones: `LaboratorioExamenStrategy`, `ImagenExamenStrategy`, `EcografiaExamenStrategy`
- `ExamenStrategyFactory` selecciona la estrategia correcta según `examen.getTipo()`
- El use case delega en la estrategia; no conoce los tipos concretos

#### Consecuencias

**Positivas:**
- Agregar nuevo tipo de examen = nueva clase Strategy + registro en Factory
- Cada estrategia es testeable de forma independiente

**Negativas:**
- Proliferación de clases para tipos similares

---

## ADR-028

### Composite para notificaciones multi-canal

**Estado:** Aceptado
**Fecha:** 2025-04-05
**Decisores:** Equipo Backend

#### Contexto

Algunos eventos requieren notificar por múltiples canales simultáneamente (EMAIL + SMS). El canal `EMAIL_SMS` no es un canal primitivo, sino la composición de dos canales. Si el use case tiene lógica `if canal == EMAIL_SMS → enviar email Y enviar SMS`, se rompe OCP.

#### Decisión

Aplicar el patrón **Composite** sobre `NotificacionStrategy`:

```
CompositeNotificacionStrategy (implementa NotificacionStrategy)
    → List<NotificacionStrategy> [EmailNotificacionStrategy, SmsNotificacionStrategy]
    → al llamar enviar(), itera y delega en cada hijo
```

`NotificacionStrategyFactory` construye el composite para el canal `EMAIL_SMS`.

#### Consecuencias

**Positivas:**
- El use case trata canal simple y multi-canal de forma idéntica
- Agregar nuevo canal = nueva clase + registro en Factory; sin modificar el composite

**Negativas:**
- Si un canal del composite falla, ¿se detiene el composite o continúa? Requiere política de error explícita

---

## ADR-029

### Angular + Capacitor como stack frontend unificado

**Estado:** Aceptado
**Fecha:** 2025-03-01
**Decisores:** Equipo Frontend, Dirección TI EsSalud

#### Contexto

Se requiere una aplicación web y una aplicación móvil (Android/iOS). Mantener dos codebases independientes (Angular para web, React Native para móvil) duplica el esfuerzo de desarrollo y mantenimiento.

#### Decisión

Usar **Angular** como framework SPA único para la aplicación web, y **Capacitor** (de Ionic) para empaquetar la misma aplicación Angular como app nativa para Android e iOS:

- Una sola codebase Angular para web, Android e iOS
- Capacitor provee acceso a APIs nativas (cámara, notificaciones push) mediante plugins
- Comunicación con backend: clientes TypeScript generados desde OpenAPI specs

#### Alternativas consideradas

| Alternativa | Motivo de rechazo |
|---|---|
| Angular Web + React Native Mobile | Dos codebases diferentes; doble esfuerzo de mantenimiento |
| Flutter | Lenguaje Dart; sin experiencia en el equipo |
| Ionic (sin Capacitor) | Ionic 7+ usa Capacitor nativamente; mismo stack |

#### Consecuencias

**Positivas:**
- Una sola codebase para web y móvil
- Reutilización de lógica de negocio, servicios y componentes
- El equipo frontend solo necesita conocer Angular + TypeScript

**Negativas:**
- Performance nativa inferior a React Native o Flutter en animaciones complejas
- Tamaño de bundle mayor que una app nativa pura

---

## ADR-030

### RBAC mediante roles de Entra ID (no ABAC)

**Estado:** Aceptado
**Fecha:** 2025-03-15
**Decisores:** Equipo de Seguridad, Arquitectura

#### Contexto

El sistema tiene tres tipos de usuarios con permisos distintos: Médico (acceso a datos de sus pacientes), Digitador (registro de citas y datos administrativos), Paciente (acceso solo a sus propios datos). Se debe definir el modelo de autorización.

#### Decisión

Usar **RBAC (Role-Based Access Control)** con 4 roles definidos en Entra ID:

| Rol | Permisos clave |
|---|---|
| `Medico` | Leer/escribir consultas, diagnósticos y exámenes de sus pacientes |
| `Digitador` | CRUD de citas; sin acceso a datos clínicos |
| `Paciente` | Solo lectura de sus propios datos (filtrado por `pacienteId == sub`) |
| `Admin` | Acceso total; gestión de plantillas de notificación |

Los roles se incluyen en el claim `roles` del JWT de Entra ID. Los MS verifican el claim con Spring Security `@PreAuthorize("hasRole('Medico')")`.

**ABAC (Attribute-Based)** se implementa solo para el caso Paciente: el MS verifica `pacienteId == jwt.sub`.

#### Consecuencias

**Positivas:**
- Simple de implementar y entender
- Los roles se gestionan en Entra ID sin código en los MS
- Spring Security tiene soporte nativo para roles JWT

**Negativas:**
- RBAC puro no cubre "el médico solo ve sus propios pacientes" → requiere lógica ABAC adicional en los MS para el rol Médico
- Cambios de permisos requieren actualizar Entra ID app registration

---

## ADR-031

### Modelo de auditoría mediante tablas dedicadas

**Estado:** Aceptado
**Fecha:** 2025-03-15
**Decisores:** DBA, Área Legal EsSalud

#### Contexto

La normativa de salud peruana exige trazabilidad de cambios en datos médicos: quién creó un diagnóstico, quién modificó una consulta, quién canceló una cita. Los campos `creado_por`, `fecha_creacion`, `fecha_actualizacion` en la tabla principal no son suficientes para capturar el historial completo de cambios.

#### Decisión

Crear **tablas de auditoría dedicadas** (`_auditoria`) para las entidades críticas:

- `citas_auditoria`: historial de cambios de estado de citas
- `diagnosticos_auditoria`: historial de cambios de estado de diagnósticos

Cada registro de auditoría captura: estado anterior, estado nuevo, motivo, usuario, timestamp.

La auditoría se inserta desde la capa de aplicación (no mediante triggers de DB) para mantener la lógica en código versionado.

#### Alternativas consideradas

| Alternativa | Motivo de rechazo |
|---|---|
| Triggers de SQL Server | Lógica fuera del control del equipo de desarrollo; difícil de testear |
| Event Sourcing completo | Complejidad excesiva para el alcance actual |
| Campos `updated_by` en la tabla principal | No captura historial; solo el último estado |

#### Consecuencias

**Positivas:**
- Historial completo y trazable de cambios de estado
- La auditoría es parte del modelo de dominio (no un efecto secundario)
- Cumplimiento de requisitos legales de historia clínica

**Negativas:**
- Doble escritura en cada operación de cambio de estado (tabla principal + tabla auditoría)
- Las tablas de auditoría crecen significativamente en entornos de alto volumen

---

## ADR-032

### Estrategia de retry con backoff exponencial

**Estado:** Aceptado
**Fecha:** 2025-04-10
**Decisores:** Equipo Backend

#### Contexto

Las llamadas a sistemas externos (RENIEC, Directorio Médico, SMTP, Azure Service Bus) pueden fallar de forma transitoria (timeouts, throttling, errores 5xx). Sin retry automático, un error transitorio propaga un error al usuario que podría haberse recuperado automáticamente.

#### Decisión

Implementar **retry con backoff exponencial + jitter** en todas las llamadas a sistemas externos:

- Máximo **3 reintentos** (4 intentos en total)
- Backoff: `2^n * 100ms + random(0-100ms)` → 100ms, 300ms, 700ms
- Solo reintentar en errores **retriable**: HTTP 429, 503, 504, timeouts de red
- **No reintentar** en: HTTP 400, 401, 403, 404, 422 (errores de cliente, no de infraestructura)
- Implementación: `reactor.util.retry.Retry.backoff()` (WebFlux) o Spring Retry (`@Retryable`)

Para Azure Service Bus consumer, usar el **DLQ** nativo de Service Bus como mecanismo de retry long-term (hasta 10 intentos, luego DLQ).

#### Consecuencias

**Positivas:**
- Recuperación automática de fallos transitorios sin intervención manual
- Jitter evita thundering herd (todos los MS reintentando al mismo tiempo)
- DLQ garantiza que ningún mensaje médico se pierda silenciosamente

**Negativas:**
- La latencia percibida puede aumentar hasta ~700ms en el peor caso con reintentos
- Requiere configuración cuidadosa para no reintentar operaciones no idempotentes

---

## ADR-033

### C4 Model para documentación de arquitectura

**Estado:** Aceptado
**Fecha:** 2025-03-01
**Decisores:** Equipo de Arquitectura

#### Contexto

La arquitectura del sistema es compleja (5 MS, múltiples servicios Azure, patrones de diseño). Se necesita una forma de documentarla que sea comprensible para distintas audiencias: directivos (nivel alto), developers (nivel detalle), y auditores externos.

#### Decisión

Usar el **C4 Model** (Simon Brown) con 4 niveles de abstracción, documentado en **PlantUML** (formato `.puml`):

| Nivel | Archivo | Audiencia |
|---|---|---|
| C1 - System Context | `c1/c1.puml` | Directivos, stakeholders |
| C2 - Container | `c2/c2.puml` | Arquitectos, TI |
| C3 - Component | `c3/c3_ms_*.puml` | Tech leads, developers seniors |
| C4 - Code | `c4/c4_ms_*.puml` | Developers |

Complementado con diagramas de clases **Mermaid** (`uml/`) para mayor detalle de la estructura del código.

#### Alternativas consideradas

| Alternativa | Motivo de rechazo |
|---|---|
| UML completo (Enterprise Architect) | Demasiado detallado para comunicar la arquitectura a distintas audiencias |
| Diagrama de bloques ad-hoc | Sin estándar; difícil de mantener; no escala |
| ArchiMate | Mayor formalidad que la necesaria para el tamaño del equipo |

#### Consecuencias

**Positivas:**
- Un nivel por audiencia: no sobrecargar directivos con detalles de código
- PlantUML es texto (versionable en Git, diff-able)
- El C4 Model es un estándar reconocido internacionalmente

**Negativas:**
- Los diagramas PlantUML requieren mantenerse sincronizados con el código real
- Curva de aprendizaje de la sintaxis PlantUML para el equipo

---

*Documento generado para el proyecto HEPAQ — EsSalud Medical Management Platform*
*Todos los ADRs han sido revisados y aceptados por el Equipo de Arquitectura*
