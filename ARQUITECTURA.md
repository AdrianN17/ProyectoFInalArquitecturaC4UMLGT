# HEPAQ — Documentación de Arquitectura

> **Plataforma de Gestión de Atenciones Médicas · EsSalud**
> Proyecto Final — Arquitectura de Software

---

## Tabla de Contenidos

1. [Visión General del Sistema](#1-visión-general-del-sistema)
2. [Actores y Contexto](#2-actores-y-contexto)
3. [Arquitectura Elegida](#3-arquitectura-elegida)
4. [Modelo C4](#4-modelo-c4)
5. [Microservicios](#5-microservicios)
6. [Patrones de Diseño](#6-patrones-de-diseño)
7. [Stack Tecnológico](#7-stack-tecnológico)
8. [Inventario de Componentes Azure](#8-inventario-de-componentes-azure)
9. [Decisiones Arquitectónicas (ADR)](#9-decisiones-arquitectónicas-adr)
10. [Modelo de Datos](#10-modelo-de-datos)
11. [Seguridad](#11-seguridad)
12. [APIs y Contratos](#12-apis-y-contratos)
13. [Estructura del Repositorio](#13-estructura-del-repositorio)

---

## 1. Visión General del Sistema

**HEPAQ** es una plataforma de gestión de atenciones médicas diseñada para el ecosistema de EsSalud. Permite a médicos, digitadores y pacientes gestionar de forma digital el ciclo completo de atención médica: desde la programación de citas hasta el registro de diagnósticos y resultados de exámenes.

### Objetivos del sistema

- Digitalizar el flujo completo de atención médica (citas → consulta → diagnóstico → exámenes)
- Garantizar alta disponibilidad y escalabilidad horizontal mediante microservicios en la nube Azure
- Asegurar trazabilidad y auditoría completa de todas las operaciones clínicas
- Notificar proactivamente a los pacientes sobre el estado de sus atenciones

---

## 2. Actores y Contexto

| Actor | Rol en el sistema |
|---|---|
| **Paciente** | Programa citas médicas, consulta resultados de exámenes y diagnósticos desde la app móvil |
| **Médico** | Realiza consultas, registra diagnósticos con códigos CIE-10 y solicita exámenes desde la web |
| **Digitador** | Registra pacientes, coordina agendamiento y administra datos clínicos desde la web |

### Sistemas externos integrados

| Sistema | Tipo | Uso |
|---|---|---|
| **RENIEC** | Externo | Validación de identidad del paciente (DNI) antes de agendar una cita |
| **Sistema de Aseguramiento EsSalud** | Externo | Verificación de cobertura y vigencia del seguro del paciente |
| **Directorio Médico** | Externo | Consulta de disponibilidad de médicos y slots de atención |
| **Microsoft Entra ID** | Externo | Proveedor de identidad OAuth2/OpenID Connect para autenticación |
| **SMTP (Microsoft 365)** | Externo | Envío de correos de notificación a pacientes |

---

## 3. Arquitectura Elegida

HEPAQ combina **cuatro estilos arquitectónicos complementarios**:

### 3.1 Arquitectura de Microservicios

El sistema se descompone en 5 microservicios independientes, cada uno con su propio ciclo de vida, base de datos y canal de comunicación. Cada servicio se despliega de forma autónoma en Azure y se comunica a través de un Gateway centralizado (APIM) y un bus de mensajes (Service Bus).

```
┌──────────────────────────────────────────────────────────┐
│                        HEPAQ                             │
│  ┌─────────┐  ┌──────────┐  ┌────────────┐              │
│  │MS Citas │  │MS Consul.│  │MS Diagnóst.│              │
│  └─────────┘  └──────────┘  └────────────┘              │
│  ┌──────────┐  ┌──────────────────────────┐             │
│  │MS Examen │  │   MS Notificaciones       │             │
│  └──────────┘  └──────────────────────────┘             │
└──────────────────────────────────────────────────────────┘
```

### 3.2 Arquitectura Hexagonal (Ports & Adapters)

Cada microservicio aplica internamente la **Arquitectura Hexagonal** de Alistair Cockburn. El dominio de negocio es el núcleo y no tiene dependencias hacia el exterior. Todo el acceso a infraestructura (bases de datos, mensajería, APIs externas) se realiza a través de interfaces (puertos) implementadas por adaptadores.

```
          ┌─────────────────────────────────┐
          │           ADAPTADORES           │
          │  (JPA, Cosmos, Redis, WebClient) │
          │  ┌───────────────────────────┐  │
          │  │         PUERTOS           │  │
          │  │  (interfaces: *Port)      │  │
          │  │  ┌─────────────────────┐  │  │
          │  │  │    APPLICATION      │  │  │
          │  │  │   (Use Cases)       │  │  │
          │  │  │  ┌───────────────┐  │  │  │
          │  │  │  │    DOMINIO    │  │  │  │
          │  │  │  │  (Entidades,  │  │  │  │
          │  │  │  │   Enums,      │  │  │  │
          │  │  │  │   Strategies) │  │  │  │
          │  │  │  └───────────────┘  │  │  │
          │  │  └─────────────────────┘  │  │
          │  └───────────────────────────┘  │
          └─────────────────────────────────┘
              ▲                         ▲
         REST/Service Bus           JPA/Redis/Cosmos
         (Adaptadores entrada)      (Adaptadores salida)
```

### 3.3 CQRS (Command Query Responsibility Segregation)

Cada microservicio separa las operaciones de **escritura (Commands)** de las de **lectura (Queries)**:

| Lado | Almacén | Tecnología | Responsabilidad |
|---|---|---|---|
| **Write (Command)** | Azure SQL Database | Spring Data JPA | Fuente de verdad, transacciones ACID |
| **Read (Query)** | Azure Cosmos DB | Spring Data Cosmos | Modelo desnormalizado, consultas rápidas |
| **Cache** | Azure Cache for Redis | Spring Data Redis | Cache-aside para acceso de milisegundos |

La sincronización entre el modelo de escritura y el de lectura se realiza **directamente dentro del mismo use case**, que actualiza ambos almacenes en la misma transacción lógica (eventual consistency controlada).

### 3.4 Arquitectura Orientada a Eventos (EDA)

Los microservicios se comunican de forma asíncrona a través de **Azure Service Bus**. Los comandos de registro (alta) y eliminación masiva se procesan como mensajes del bus, desacoplando los productores de los consumidores.

```
MS Citas ──── CitaAgendadaEvent ────► Service Bus ────► MS Notificaciones
                                                   ────► MS Consultas (RegistrarConsultaConsumer)
MS Consultas ── ConsultaCanceladaEvent ─► Service Bus ─► MS Notificaciones
MS Exámenes ─── ExamenRegistradoEvent ──► Service Bus ─► MS Notificaciones
MS Diagnósticos ─ DiagnosticoRegistradoEvent ► Service Bus ► MS Notificaciones
```

---

## 4. Modelo C4

La arquitectura se documenta usando el **Modelo C4** (Simon Brown) en 4 niveles de abstracción:

### Nivel 1 — Contexto del Sistema (`c1/c1.puml`)

Vista de alto nivel: HEPAQ como sistema único con sus actores (Paciente, Médico, Digitador) y sistemas externos (RENIEC, EsSalud).

### Nivel 2 — Contenedores (`c2/c2.puml`)

Descomposición en contenedores: frontends (Web Angular, Mobile Angular+Capacitor), API Gateway (APIM), los 5 microservicios, y toda la infraestructura compartida (Redis, Service Bus, Key Vault, Blob Storage).

### Nivel 3 — Componentes (`c3/`)

Por cada microservicio, detalle de sus componentes internos: Controllers, Delegates, Mappers, Services (Command/Query), Consumers, Adapters y Repositories.

| Archivo | Microservicio |
|---|---|
| `c3/c3_ms_citas.puml` | MS Citas |
| `c3/c3_ms_consulta.puml` | MS Consultas |
| `c3/c3_ms_diagnosticos.puml` | MS Diagnósticos |
| `c3/c3_ms_examenes.puml` | MS Exámenes |
| `c3/C3_ms_notificaciones.puml` | MS Notificaciones |

### Nivel 4 — Código (`c4/`)

Diagramas de clases PlantUML que modelan el dominio, use cases, puertos, adaptadores y los patrones de diseño aplicados en cada microservicio.

---

## 5. Microservicios

### 5.1 MS Citas

**Responsabilidad:** Gestión del ciclo de vida de citas médicas (agendar, reagendar, cancelar).

**Flujo principal:**
1. El paciente/digitador envía `POST /citas` al APIM
2. `AgendarCitaUseCase` ejecuta la cadena de validación (RENIEC → Seguro → Disponibilidad médica)
3. Selecciona la estrategia de agendamiento según el tipo de cita
4. Persiste en Azure SQL y sincroniza en Cosmos DB
5. Publica `CitaAgendadaEvent` al Service Bus

**Integraciones externas:** RENIEC · EsSalud · Directorio Médico

---

### 5.2 MS Consultas

**Responsabilidad:** Registro y gestión de consultas médicas (atenciones clínicas).

**Flujo principal:**
1. MS Citas publica `CitaAgendadaEvent`
2. `RegistrarConsultaConsumer` consume el evento y ejecuta `RegistrarConsultaUseCase`
3. El médico edita la consulta (anamnesis, examen físico, tratamiento) vía REST
4. Publica `ConsultaCanceladaEvent` cuando aplica

**Sin integraciones externas directas** — recibe contexto del evento del Service Bus.

---

### 5.3 MS Diagnósticos

**Responsabilidad:** Registro de diagnósticos médicos codificados con CIE-10.

**Flujo principal:**
1. El médico registra un diagnóstico vía `POST /diagnosticos` referenciando un código CIE-10
2. Se valida el código contra el catálogo en Azure Table Storage
3. Se persiste en Azure SQL; la capa de cache Redis actúa transparentemente (Decorator)
4. Publica `DiagnosticoRegistradoEvent`

**Catálogo CIE-10:** Almacenado en Azure Table Storage (lectura intensiva, solo lectura por el servicio).

---

### 5.4 MS Exámenes

**Responsabilidad:** Gestión de exámenes médicos (laboratorio, imagen, ecografía, etc.) y sus adjuntos.

**Flujo principal:**
1. MS Consultas publica el evento que origina el examen
2. `RegistrarExamenConsumer` ejecuta `RegistrarExamenUseCase`
3. La estrategia de procesamiento se selecciona según el tipo de examen (Laboratorio, Imagen, Ecografía)
4. Los archivos adjuntos (PDFs, imágenes DICOM) se almacenan en Azure Blob Storage
5. El médico ingresa resultados vía `PUT /examenes/{id}`

**Azure Blob Storage:** Almacenamiento de imágenes médicas y resultados con URLs SAS firmadas (15 min).

---

### 5.5 MS Notificaciones

**Responsabilidad:** Envío de notificaciones multi-canal (Email/SMS) basado en eventos del dominio.

**Flujo principal:**
1. Consume eventos de todos los demás MS desde Azure Service Bus
2. `GenerarNotificacionService` selecciona la estrategia y plantilla Thymeleaf
3. `CompositeNotificacionStrategy` permite enviar simultáneamente por email + SMS
4. Registra el resultado (éxito/fallo) en Azure SQL para auditoría
5. Un scheduler automático reintenta los envíos fallidos

**Sin endpoints REST para envío** — completamente event-driven.

---

## 6. Patrones de Diseño

### GoF — Patrones aplicados por microservicio

| Patrón | MS | Clase(s) | Propósito |
|---|---|---|---|
| **Strategy** | MS Citas | `AgendaStrategy` → `ConsultaMedicaStrategy`, `LaboratorioStrategy`, `EcografiaStrategy` | Lógica de agendamiento específica por tipo de cita |
| **Factory Method** | MS Citas, Exámenes, Notificaciones | `AgendaStrategyFactory`, `ExamenStrategyFactory`, `NotificacionStrategyFactory` | Selección de estrategia en tiempo de ejecución |
| **Chain of Responsibility** | MS Citas | `ValidacionCitaHandler` → `ReniecValidacionHandler` → `SeguroValidacionHandler` → `DisponibilidadMedicoHandler` | Cadena de validaciones previas al agendamiento |
| **Template Method** | MS Consultas | `ConsultaUseCaseTemplate` → subclases (`Registrar`, `Editar`, `Cancelar`) | Define el flujo `validar → persistir → publicarEvento`; subclases implementan pasos concretos |
| **Decorator** | MS Diagnósticos | `DiagnosticoCacheDecorator` envuelve `DiagnosticoRepositoryPort` | Agrega cache Redis de forma transparente sin modificar el adaptador JPA |
| **Strategy** | MS Exámenes | `ExamenProcesadorStrategy` → `LaboratorioExamenStrategy`, `ImagenExamenStrategy`, `EcografiaExamenStrategy` | Validación y procesamiento específico por tipo de examen |
| **Strategy** | MS Notificaciones | `NotificacionStrategy` → `EmailNotificacionStrategy`, `SmsNotificacionStrategy` | Envío por canal específico |
| **Composite** | MS Notificaciones | `CompositeNotificacionStrategy` | Agrupa múltiples estrategias de envío (Email + SMS) como una sola |

### Patrones Arquitectónicos adicionales

| Patrón | Aplicación |
|---|---|
| **Repository** | Todos los MS — `*RepositoryPort` con implementaciones `JpaAdapter`, `CosmosAdapter` |
| **Delegate** | Todos los MS — `*Delegate` desacopla el Controller del Use Case |
| **CQRS** | Todos los MS — servicios de comando separados de servicios de consulta |
| **Cache-Aside** | Todos los MS — Redis como cache de lectura con fallback a Cosmos DB |
| **Event-Driven** | Comunicación inter-MS — Azure Service Bus como broker |
| **Saga (simple)** | Flujo cita → consulta → diagnóstico/examen orquestado por eventos |

---

## 7. Stack Tecnológico

### Backend

| Tecnología | Versión | Uso |
|---|---|---|
| **Java** | 21 LTS | Lenguaje principal |
| **Spring Boot** | 3.x | Framework base |
| **Spring WebFlux** | 3.x | Programación reactiva (non-blocking I/O) |
| **Spring Data JPA** | 3.x | Acceso a Azure SQL (Write Model) |
| **Spring Data Cosmos** | 5.x | Acceso a Azure Cosmos DB (Read Model) |
| **Spring Data Redis** | 3.x | Cache distribuida |
| **MapStruct** | 1.5.x | Mapeo automático DTO ↔ Dominio ↔ Entity |
| **Liquibase** | 4.x | Migraciones de esquema Azure SQL |
| **Thymeleaf** | 3.x | Plantillas de email (MS Notificaciones) |
| **Jakarta Mail** | 2.x | Envío SMTP de correos |

### Infraestructura Azure

| Servicio Azure | Uso |
|---|---|
| **Azure API Management (APIM)** | Gateway centralizado: autenticación JWT, rate limiting, enrutamiento, documentación |
| **Azure SQL Database** | Write Model CQRS — fuente de verdad transaccional para cada MS |
| **Azure Cosmos DB** | Read Model CQRS — consultas rápidas con modelo desnormalizado |
| **Azure Cache for Redis** | Cache distribuida — reducción de latencia en consultas frecuentes |
| **Azure Service Bus** | Message broker — comunicación asíncrona entre microservicios |
| **Azure Blob Storage** | Almacenamiento de imágenes médicas, PDFs y resultados de exámenes |
| **Azure Table Storage** | Catálogo CIE-10 — lectura intensiva, sin actualizaciones frecuentes |
| **Azure Key Vault** | Gestión segura de secretos, certificados y cadenas de conexión |
| **Microsoft Entra ID** | Proveedor de identidad OAuth2/OpenID Connect — autenticación y autorización |

### Frontend

| Tecnología | Plataforma | Uso |
|---|---|---|
| **Angular** | Web (SPA) | Portal para médicos y personal administrativo |
| **Angular + Capacitor** | Mobile (iOS/Android) | App para pacientes |

### Herramientas y estándares

| Herramienta | Uso |
|---|---|
| **OpenAPI 3.0.3** | Especificación de contratos REST por microservicio |
| **PlantUML C4** | Diagramas de arquitectura C1 → C4 |
| **Mermaid** | Diagramas de clases UML para documentación |
| **Liquibase YAML** | Changelogs de base de datos versionados |

---

## 8. Inventario de Componentes Azure

> Inventario extraído del análisis de los diagramas **C4 Nivel 2** (contenedores) y **C4 Nivel 3** (componentes) de cada microservicio.

### 8.1 Catálogo de servicios Azure

| Servicio Azure | Categoría | Instancias en HEPAQ | Tier recomendado |
|---|---|---|---|
| **Microsoft Entra ID** | Identidad (SaaS) | 1 tenant compartido | P1 (MFA incluido) |
| **Azure API Management** | API Gateway (PaaS) | 1 instancia | Standard |
| **Azure Container Apps** | Compute — Contenedores (PaaS) | 1 entorno / **5 container apps** (una por MS) | Consumption (escala a cero) |
| **Azure SQL Database** | Base de datos relacional (PaaS) | **5 bases de datos** (una por MS) | General Purpose — 2 vCores |
| **Azure Cosmos DB** | Base de datos documental (PaaS) | 1 cuenta / **4 contenedores** | Serverless o Autoscale 1000 RU/s |
| **Azure Cache for Redis** | Caché distribuida (PaaS) | 1 clúster compartido | C1 Standard (con replicación) |
| **Azure Service Bus** | Message broker (PaaS) | 1 namespace / **5 topics** | Standard |
| **Azure Blob Storage** | Almacenamiento de objetos (PaaS) | 1 cuenta / 1 container `examenes-adjuntos` | General Purpose v2 — LRS |
| **Azure Table Storage** | Almacenamiento tabular NoSQL (PaaS) | 1 tabla `CatalogoClE10` (mismo Storage Account) | LRS |
| **Azure Key Vault** | Gestión de secretos (PaaS) | 1 vault compartido | Standard |
| **SMTP / Microsoft 365** | Correo saliente (SaaS externo) | Exchange Online del tenant EsSalud | — |

---

### 8.2 Uso por microservicio

| Servicio Azure | MS Citas | MS Consultas | MS Diagnósticos | MS Exámenes | MS Notificaciones |
|---|:---:|:---:|:---:|:---:|:---:|
| **APIM** (entrada HTTP) | ✓ | ✓ | ✓ | ✓ | — |
| **Azure SQL** (Write Model) | `citas` `citas_auditoria` | `consultas` `signos_vitales` | `diagnosticos` `diagnosticos_auditoria` | `examenes` `adjuntos_examen` | `notificaciones` `notificaciones_intentos` `plantillas_notificacion` |
| **Azure Cosmos DB** (Read Model) | container `citas` | container `consultas` | container `diagnosticos` | container `examenes` | — |
| **Azure Cache for Redis** | `CitaRedisAdapter` `MedicosRedisAdapter` | `ConsultaRedisAdapter` | `DiagnosticoRedisAdapter` | `ExamenRedisAdapter` | — |
| **Azure Service Bus** | Publica + consume | Publica + consume | Publica + consume | Publica + consume | Solo consume |
| **Azure Blob Storage** | — | — | — | `BlobStorageAdapter` | — |
| **Azure Table Storage** | — | — | `CieTableAdapter` | — | — |
| **Azure Key Vault** | ✓ | ✓ | ✓ | ✓ | ✓ |
| **Microsoft Entra ID** | Valida JWT vía APIM | Valida JWT vía APIM | Valida JWT vía APIM | Valida JWT vía APIM | — |

---

### 8.3 SDKs y adaptadores por servicio

| Servicio Azure | Librería / SDK | Adaptador en el MS | Puerto (interfaz) |
|---|---|---|---|
| Azure SQL Database | `spring-data-jpa` + Hibernate + HikariCP | `*JpaAdapter` → `*JpaRepository` | `*RepositoryPort` |
| Azure Cosmos DB | `spring-cloud-azure-starter-data-cosmos` | `*CosmosAdapter` → `*CosmosRepository` | `*ReadRepositoryPort` |
| Azure Cache for Redis | `spring-boot-starter-data-redis` | `*RedisAdapter` → `*RedisRepository` | `*CachePort` |
| Azure Service Bus | `azure-messaging-servicebus` | `ServiceBusPublisher`, `*Consumer` | `EventPublisherPort` |
| Azure Blob Storage | `azure-storage-blob` | `BlobStorageAdapter` | `AdjuntoStoragePort` |
| Azure Table Storage | `azure-data-tables` | `CieTableAdapter` → `CieTableRepository` | `CieRepositoryPort` |
| Azure Key Vault | `spring-cloud-azure-starter-keyvault-secrets` | PropertySource automático (startup) | — |
| Microsoft Entra ID | `spring-security-oauth2-resource-server` | `SecurityFilterChain` (validación JWT) | — |

---

### 8.4 Topología de Azure Service Bus

| Topic | MS Publicador | MS Suscriptores | Consumer en el MS destino |
|---|---|---|---|
| `citas.agendada` | MS Citas | MS Notificaciones, MS Consultas | `GenerarNotificacionConsumer`, `RegistrarConsultaConsumer` |
| `consultas.cancelada` | MS Consultas | MS Notificaciones | `GenerarNotificacionConsumer` |
| `diagnosticos.registrado` | MS Diagnósticos | MS Notificaciones | `GenerarNotificacionConsumer` |
| `examenes.registrado` | MS Exámenes | MS Notificaciones | `GenerarNotificacionConsumer` |
| `examenes.completado` | MS Exámenes | MS Notificaciones | `GenerarNotificacionConsumer` |
| `diagnosticos.borrar-lote` | MS Diagnósticos (internal) | MS Diagnósticos | `BorrarDiagnosticosConsumer` |
| `examenes.cancelar` | MS Exámenes (internal) | MS Exámenes | `CancelarExamenConsumer`, `BorrarExamenConsumer` |

---

### 8.5 Modelo de contenedores en Azure Cosmos DB

| Contenedor | Partition Key | MS propietario | Descripción del documento |
|---|---|---|---|
| `citas` | `/pacienteId` | MS Citas | Proyección desnormalizada: cita + nombre del médico + especialidad |
| `consultas` | `/pacienteId` | MS Consultas | Proyección: consulta + signos vitales embebidos |
| `diagnosticos` | `/pacienteId` | MS Diagnósticos | Proyección: diagnóstico + descripción CIE-10 embebida |
| `examenes` | `/pacienteId` | MS Exámenes | Proyección: examen + lista de adjuntos con URL SAS |

> **MS Notificaciones** no usa Cosmos DB — su única fuente de verdad es Azure SQL (registro de auditoría de envíos).

---

### 8.6 Componentes por tipo de acceso

```
Acceso HTTP (síncrono)                    Acceso asíncrono (mensajería)
──────────────────────                    ────────────────────────────
Cliente → APIM → MS                       MS → Service Bus Topic
                  ├── Azure SQL (write)              ↓
                  ├── Cosmos DB (read)    Service Bus Subscription
                  └── Redis (cache)                  ↓
                                          MS Consumer → Azure SQL (write)

Almacenamiento de archivos                Configuración segura
──────────────────────────                ────────────────────
MS Exámenes → BlobStorageAdapter          Todos los MS
              ↓                           → Key Vault (Managed Identity)
     Azure Blob Storage                   → Secretos inyectados como
     (URL SAS generada)                     propiedades Spring Boot

Catálogo estático                         Identidad
─────────────────                         ─────────
MS Diagnósticos → CieTableAdapter         Web/Mobile → Entra ID
                  ↓                       APIM valida JWT
        Azure Table Storage               MS confía en claims del token
        (CIE-10, ~14 000 códigos)
```

---

## 9. Decisiones Arquitectónicas (ADR)

### ADR-001 — Arquitectura Hexagonal por microservicio

**Contexto:** Necesidad de aislar la lógica de negocio del dominio médico de los detalles de infraestructura (JPA, Redis, Cosmos DB, HTTP clients).

**Decisión:** Cada microservicio implementa Arquitectura Hexagonal con tres capas:
- **Dominio:** Entidades, enums, interfaces de estrategias — sin dependencias externas
- **Puertos:** Interfaces definidas por el dominio (`*Port`)
- **Adaptadores:** Implementaciones concretas de los puertos (`*Adapter`)

**Consecuencias:** Alta testeabilidad (mocks de puertos), intercambiabilidad de infraestructura, separación clara de responsabilidades.

---

### ADR-002 — CQRS con modelos de datos separados

**Contexto:** Los patrones de acceso para escritura (transaccional, consistente) y lectura (consultas analíticas, listados paginados) son fundamentalmente diferentes.

**Decisión:**
- **Write Model:** Azure SQL Database — garantías ACID, integridad referencial
- **Read Model:** Azure Cosmos DB — esquema flexible, consultas por partición (`/pacienteId`)
- **Cache:** Azure Cache for Redis — cache-aside con TTL configurable

**Consecuencias:** Mayor complejidad operativa (dos modelos a sincronizar), pero rendimiento óptimo en ambos lados. La sincronización es responsabilidad del use case.

---

### ADR-003 — Comunicación asíncrona con Azure Service Bus

**Contexto:** El registro de consultas, exámenes y notificaciones no requiere respuesta sincrónica. El acoplamiento temporal entre microservicios debe minimizarse.

**Decisión:** Azure Service Bus como broker de mensajes. Los comandos de creación masiva y las notificaciones se procesan como mensajes asíncronos.

**Consecuencias:** Resiliencia ante fallos de un microservicio (dead-letter queue), desacoplamiento temporal, posibilidad de replay de mensajes.

---

### ADR-004 — Spring WebFlux para programación reactiva

**Contexto:** Alta concurrencia esperada (EsSalud atiende millones de asegurados). Los modelos de threading tradicionales (1 thread por request) no son eficientes.

**Decisión:** Spring WebFlux con Project Reactor (non-blocking I/O) en todos los microservicios.

**Consecuencias:** Mayor eficiencia en uso de threads, mejor comportamiento bajo carga, pero mayor curva de aprendizaje para el equipo.

---

### ADR-005 — Microsoft Entra ID para autenticación/autorización

**Contexto:** El sistema necesita autenticación robusta con soporte para múltiples roles (médico, digitador, paciente) y SSO con el ecosistema Microsoft de EsSalud.

**Decisión:** Microsoft Entra ID como Identity Provider con flujo OAuth2 Authorization Code + PKCE para frontends y validación de JWT Bearer en el APIM.

**Consecuencias:** Delegación de seguridad a un servicio gestionado, SSO con el ecosistema Microsoft, revocación centralizada de tokens.

---

### ADR-006 — Azure Table Storage para el catálogo CIE-10

**Contexto:** El catálogo CIE-10 tiene ~15,000 códigos, es de solo lectura en tiempo de ejecución y se consulta con alta frecuencia en el MS Diagnósticos.

**Decisión:** Azure Table Storage (PartitionKey = categoría CIE, RowKey = código). Cache local en memoria en el adaptador `CieTableAdapter`.

**Consecuencias:** Costo mínimo de almacenamiento, acceso O(1) por PartitionKey+RowKey, no requiere instancia de base de datos dedicada.

---

### ADR-007 — OpenAPI-first para diseño de APIs

**Contexto:** Múltiples equipos (frontend web, mobile, backend) desarrollan en paralelo. Se necesita un contrato claro antes de implementar.

**Decisión:** Especificación OpenAPI 3.0.3 como fuente de verdad para cada API. Los Controllers se generan con el plugin `openapi-generator-maven-plugin`.

**Consecuencias:** Contrato explícito y versionado, generación automática de stubs, documentación siempre sincronizada con la implementación.

---

### ADR-008 — Liquibase para migraciones de Azure SQL

**Contexto:** Los cambios de esquema deben ser reproducibles, versionados y auditables en todos los entornos (dev, QA, producción).

**Decisión:** Liquibase con changelogs en formato YAML, ejecutados automáticamente al iniciar cada microservicio con Spring Boot.

**Consecuencias:** Esquema versionado en el repositorio junto al código, rollback controlado, historial completo de cambios.

---

### ADR-009 — Patron Decorator para cache-aside en MS Diagnósticos

**Contexto:** La lógica de cache Redis debía añadirse al repositorio sin modificar el adaptador JPA existente y sin duplicar código en los use cases.

**Decisión:** `DiagnosticoCacheDecorator` implementa `DiagnosticoRepositoryPort` y envuelve la instancia del `DiagnosticoJpaAdapter`, añadiendo la lógica de cache de forma transparente.

**Consecuencias:** Open/Closed Principle respetado, la capa de cache es intercambiable, los use cases no saben si hay cache o no.

---

### ADR-010 — Composite para notificaciones multi-canal

**Contexto:** Algunos eventos requieren notificar al paciente por email Y SMS simultáneamente. La lógica de envío debe ser uniforme independientemente del número de canales.

**Decisión:** `CompositeNotificacionStrategy` implementa `NotificacionStrategy` y contiene una lista de estrategias. El `NotificacionStrategyFactory` construye el composite adecuado según el tipo de evento.

**Consecuencias:** Extensibilidad para nuevos canales (push notifications) sin modificar los use cases, fallos parciales manejados (un canal falla sin afectar al otro).

---

## 10. Modelo de Datos

### Resumen por microservicio

| MS | Azure SQL (Write) | Cosmos DB (Read) | Especial |
|---|---|---|---|
| **MS Citas** | `citas`, `citas_auditoria` | Container `citas` / PK `/pacienteId` | — |
| **MS Consultas** | `consultas`, `signos_vitales` | Container `consultas` / PK `/pacienteId` | — |
| **MS Diagnósticos** | `diagnosticos`, `diagnosticos_auditoria` | Container `diagnosticos` / PK `/pacienteId` | Azure Table Storage: `CatalogoClE10` |
| **MS Exámenes** | `examenes`, `adjuntos_examen` | Container `examenes` / PK `/pacienteId` | Azure Blob Storage: imágenes médicas |
| **MS Notificaciones** | `notificaciones`, `notificaciones_intentos`, `plantillas_notificacion` | — (solo auditoría SQL) | Sin Cosmos DB |

### Estrategia de Partition Key en Cosmos DB

Todos los contenedores usan `/pacienteId` como partition key porque:
- La mayoría de consultas son por paciente (`GET /citas?pacienteId=...`)
- Distribuye equitativamente la carga entre particiones lógicas
- Permite consultas eficientes sin cross-partition queries

### Soft Delete

Los registros médicos nunca se eliminan físicamente. Se usan columnas `estado = 'ELIMINADO'` y `motivo_eliminacion` para mantener trazabilidad clínica completa.

---

## 11. Seguridad

### Autenticación y Autorización

```
Paciente/Médico/Digitador
        │
        ▼
Microsoft Entra ID (OAuth2 Authorization Code + PKCE)
        │ Access Token (JWT)
        ▼
Azure API Management
  ├── Valida JWT firma y expiración
  ├── Verifica claims de rol (médico, digitador, paciente)
  ├── Rate limiting por cliente
  └── Enruta al microservicio correspondiente
```

### Roles y permisos por operación

| Operación | Paciente | Digitador | Médico |
|---|---|---|---|
| Agendar cita | ✓ | ✓ | — |
| Cancelar cita | ✓ (propia) | ✓ | ✓ |
| Registrar consulta | — | — | ✓ (event-driven) |
| Registrar diagnóstico | — | — | ✓ |
| Borrar diagnóstico | — | — | ✓ |
| Ver resultados de exámenes | ✓ (propios) | ✓ | ✓ |
| Consultar historial | ✓ (propio) | ✓ | ✓ |
| Admin notificaciones | — | — | — (admin) |

### Secretos y configuración sensible

- Todas las cadenas de conexión, API keys y certificados se almacenan en **Azure Key Vault**
- El APIM obtiene certificados de Key Vault en tiempo de ejecución
- Los microservicios acceden a Key Vault mediante **Managed Identity** (sin credenciales en código)
- Las URLs de descarga de adjuntos en Blob Storage son **SAS URLs firmadas con TTL de 15 minutos**

---

## 12. APIs y Contratos

Cada microservicio tiene su especificación OpenAPI 3.0.3 en la carpeta `openapi/`:

| Especificación | Base URL | Endpoints principales |
|---|---|---|
| `openapi/openapi_ms_citas.yaml` | `/citas/v1` | `POST /citas`, `GET /citas`, `GET /citas/{id}`, `PUT /citas/{id}/reagendar`, `PUT /citas/{id}/cancelar` |
| `openapi/openapi_ms_consulta.yaml` | `/consultas/v1` | `GET /consultas`, `GET /consultas/{id}`, `PUT /consultas/{id}`, `PUT /consultas/{id}/cancelar` |
| `openapi/openapi_ms_diagnosticos.yaml` | `/diagnosticos/v1` | `POST /diagnosticos`, `GET /diagnosticos`, `GET /diagnosticos/{id}`, `DELETE /diagnosticos/{id}`, `GET /cie`, `GET /cie/{codigo}` |
| `openapi/openapi_ms_examenes.yaml` | `/examenes/v1` | `GET /examenes`, `GET /examenes/{id}`, `PUT /examenes/{id}`, `GET /examenes/{id}/adjuntos`, `GET /examenes/{id}/adjuntos/{adjId}` |
| `openapi/openapi_ms_notificaciones.yaml` | `/notificaciones/v1` | `GET /notificaciones`, `GET /notificaciones/{id}`, `POST /notificaciones/{id}/reintentar`, `GET /plantillas`, `GET /health` |

### Convenciones de API

- **Versionado:** en la URL (`/v1`)
- **Autenticación:** `Authorization: Bearer <JWT>` en todos los endpoints (excepto `/health`)
- **Paginación:** parámetros `page` y `size` en listados, respuesta con `PagedDTO`
- **Errores:** estructura `ErrorDTO` con `timestamp`, `status`, `message`, `traceId`
- **Trazabilidad:** header `X-Trace-Id` propagado en todas las respuestas

---

## 13. Estructura del Repositorio

```
ProyectoFInalArquitecturaC4UMLGT/
│
├── README.md                          # Este archivo
│
├── c1/                                # C4 Nivel 1 — Contexto del sistema
│   └── c1.puml
│
├── c2/                                # C4 Nivel 2 — Contenedores
│   └── c2.puml
│
├── c3/                                # C4 Nivel 3 — Componentes por MS
│   ├── c3_ms_citas.puml
│   ├── c3_ms_consulta.puml
│   ├── c3_ms_diagnosticos.puml
│   ├── c3_ms_examenes.puml
│   └── C3_ms_notificaciones.puml
│
├── c4/                                # C4 Nivel 4 — Código (clases, patrones)
│   ├── c4_ms_citas.puml               # Strategy + Factory + Chain of Responsibility
│   ├── c4_ms_consulta.puml            # Template Method
│   ├── c4_ms_diagnosticos.puml        # Decorator
│   ├── c4_ms_examenes.puml            # Strategy + Factory
│   └── c4_ms_notificaciones.puml      # Strategy + Factory + Composite
│
├── uml/                               # Diagramas de clases Mermaid (detallados)
│   ├── uml_ms_citas.mmd
│   ├── uml_ms_consulta.mmd
│   ├── uml_ms_diagnosticos.mmd
│   ├── uml_ms_examenes.mmd
│   └── uml_ms_notificaciones.mmd
│
├── openapi/                           # Especificaciones OpenAPI 3.0.3
│   ├── openapi_ms_citas.yaml
│   ├── openapi_ms_consulta.yaml
│   ├── openapi_ms_diagnosticos.yaml
│   ├── openapi_ms_examenes.yaml
│   └── openapi_ms_notificaciones.yaml
│
└── db/                                # Esquemas de base de datos
    ├── db_ms_citas.md                 # Liquibase XML + Cosmos documento ejemplo
    ├── db_ms_consulta.md
    ├── db_ms_diagnosticos.md          # Incluye Azure Table Storage (CIE-10)
    ├── db_ms_examenes.md
    └── db_ms_notificaciones.md        # Solo SQL — sin Cosmos DB
```

---

*Documentación generada el 18 de junio de 2026 · HEPAQ — Proyecto Final Arquitectura de Software*
