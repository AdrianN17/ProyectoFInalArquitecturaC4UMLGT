# HEPAQ — Descripción del Modelo C4

> Sistema de Gestión Médica · EsSalud

---

## Propósito

Este documento describe la arquitectura del sistema **HEPAQ** mediante el modelo C4 (Context, Container, Component, Code). Su objetivo es proveer una visión clara, estructurada y progresiva del sistema para facilitar la comunicación entre equipos técnicos y no técnicos, guiar decisiones de diseño y servir como referencia oficial de arquitectura durante el ciclo de vida del proyecto.

---

## Alcance

El modelo cubre los cuatro niveles de abstracción del modelo C4 aplicados al sistema HEPAQ:

| Nivel | Nombre | Descripción |
|-------|--------|-------------|
| C1 | System Context | Actores externos y sistemas con los que HEPAQ interactúa |
| C2 | Container Diagram | Contenedores desplegables que componen HEPAQ (frontends, microservicios, bases de datos, infraestructura) |
| C3 | Component Diagram | Componentes internos de cada microservicio (controllers, services, adapters, repositories) |
| C4 | Code Diagram | Clases, interfaces, enumeraciones y patrones de diseño por microservicio |

El alcance incluye los siguientes microservicios:

- **MS Citas** — Programación y seguimiento de citas médicas
- **MS Consultas** — Gestión de atenciones médicas e historia clínica
- **MS Diagnósticos** — Registro y consulta de diagnósticos médicos
- **MS Exámenes** — Gestión de laboratorios y estudios médicos
- **MS Notificaciones** — Envío de alertas, correos y notificaciones del sistema

---

## Audiencia

| Perfil | Niveles relevantes |
|--------|--------------------|
| Arquitectos de software | C1, C2, C3, C4 |
| Desarrolladores backend | C3, C4 |
| Líderes técnicos | C1, C2, C3 |
| Product Owners / Stakeholders | C1, C2 |
| Equipo de operaciones / DevOps | C2 |
| Auditores y revisores de seguridad | C1, C2, C3 |

---

## Elementos del Modelo

### Actores (Personas)

| Elemento | Descripción |
|----------|-------------|
| **Médico** | Realiza consultas, registra diagnósticos y solicita exámenes médicos |
| **Digitador / Personal Administrativo** | Registra pacientes y coordina la programación de atenciones médicas |
| **Paciente** | Programa citas, consulta resultados y realiza el seguimiento de su atención médica |

### Sistemas Externos

| Elemento | Descripción |
|----------|-------------|
| **RENIEC** | Servicio de validación y consulta de identidad de los pacientes |
| **Sistema de Aseguramiento EsSalud** | Verificación de cobertura y vigencia de los asegurados |
| **Microsoft Entra ID** | Proveedor de identidad y autenticación OAuth2/OpenID Connect |
| **Directorio Médico** | Fuente de información de médicos habilitados |
| **SMTP Server (Microsoft 365)** | Servidor de correo electrónico para notificaciones |

### Contenedores Principales

| Elemento | Tecnología | Descripción |
|----------|------------|-------------|
| **HEPAQ Web** | Angular | Portal para personal médico y administrativo |
| **HEPAQ Mobile** | Angular + Capacitor | Portal para pacientes |
| **Azure API Management** | APIM | Gateway centralizado con autenticación, autorización y enrutamiento |
| **MS Citas** | Java Spring Boot WebFlux | Programación y seguimiento de citas médicas |
| **MS Consultas** | Java Spring Boot WebFlux | Gestión de atenciones médicas e historia clínica |
| **MS Diagnósticos** | Java Spring Boot WebFlux | Registro y consulta de diagnósticos |
| **MS Exámenes** | Java Spring Boot WebFlux | Gestión de laboratorios y estudios médicos |
| **MS Notificaciones** | Java Spring Boot WebFlux | Envío de correos, SMS, push notifications y alertas |
| **Azure Cache for Redis** | Redis Cluster | Cache distribuido para consultas frecuentes |
| **Azure Service Bus** | Message Broker | Eventos de dominio y sincronización CQRS |
| **Azure Key Vault** | Secrets Management | Gestión segura de secretos, certificados y claves |
| **Azure Storage Account** | Blob Storage | Imágenes médicas, documentos clínicos y archivos adjuntos |

---
---

## C1 — Nivel 1: System Context

---

| Metadato | Valor |
|----------|-------|
| **Nivel C4** | C1 — System Context |
| **Sistema** | HEPAQ |
| **Módulo documentado** | Vista global del sistema y sus actores |
| **Diagrama fuente** | [c1/c1.puml](c1/c1.puml) |
| **Versión** | v1 |
| **Fecha** | 2026-06-20 |
| **Estado** | Aprobado |

### Descripción

El diagrama de contexto del sistema (C1) muestra a HEPAQ como una caja negra y describe cómo los actores humanos y los sistemas externos se relacionan con él. No detalla tecnologías internas; su propósito es comunicar el entorno del sistema.

### Actores y relaciones

| Actor / Sistema | Tipo | Relación con HEPAQ |
|-----------------|------|---------------------|
| **Médico** | Person | Gestiona consultas, diagnósticos y exámenes |
| **Digitador** | Person | Administra pacientes y agenda atenciones |
| **Paciente** | Person | Programa citas y consulta resultados médicos |
| **RENIEC** | Sistema externo | HEPAQ valida la identidad del paciente |
| **Sistema de Aseguramiento EsSalud** | Sistema externo | HEPAQ verifica la cobertura del asegurado |

---
---

## C2 — Nivel 2: Container Diagram

---

| Metadato | Valor |
|----------|-------|
| **Nivel C4** | C2 — Container Diagram |
| **Sistema** | HEPAQ |
| **Módulo documentado** | Todos los contenedores del sistema |
| **Diagrama fuente** | [c2/c2.puml](c2/c2.puml) |
| **Versión** | v1 |
| **Fecha** | 2026-06-20 |
| **Estado** | Aprobado |

### Descripción

El diagrama de contenedores (C2) desglosa HEPAQ en unidades desplegables. Muestra los frontends, el API Gateway, los microservicios con patrón CQRS, las bases de datos y la infraestructura compartida en Azure.

### Patrón arquitectónico: CQRS por microservicio

Cada microservicio core implementa CQRS con segregación física de almacenamiento:

| Dominio | Modelo escritura (Commands) | Modelo lectura (Queries) |
|---------|-----------------------------|--------------------------|
| Citas | Azure SQL — Citas | Cosmos DB — Citas |
| Consultas | Azure SQL — Consultas | Cosmos DB — Consultas |
| Diagnósticos | Azure SQL — Diagnósticos | Cosmos DB — Diagnósticos + Azure Tables CIE |
| Exámenes | Azure SQL — Exámenes | Cosmos DB — Exámenes |

### Infraestructura compartida

| Componente | Tecnología | Propósito |
|------------|------------|-----------|
| **Azure Cache for Redis** | Redis Cluster | Cache distribuido para consultas frecuentes |
| **Azure Service Bus** | Message Broker | Eventos de dominio y sincronización CQRS |
| **Azure Key Vault** | Secrets Management | Gestión segura de secretos y certificados |
| **Azure Storage Account** | Blob Storage | Imágenes médicas y archivos adjuntos |

### Flujo de autenticación

```
Usuario → Frontend (Web/Mobile)
         → Microsoft Entra ID (OAuth2/OIDC)
         → Azure API Management (valida JWT)
         → Microservicio correspondiente
```

---
---

## C3 — Nivel 3: Component Diagram · MS Citas

---

| Metadato | Valor |
|----------|-------|
| **Nivel C4** | C3 — Component Diagram |
| **Sistema** | HEPAQ |
| **Módulo documentado** | MS Citas |
| **Diagrama fuente** | [c3/c3_ms_citas.puml](c3/c3_ms_citas.puml) |
| **Versión** | v1 |
| **Fecha** | 2026-06-20 |
| **Estado** | Aprobado |

### Descripción

Desglosa los componentes internos del microservicio **MS Citas** (Spring Boot WebFlux). Implementa el patrón CQRS con separación entre el lado de comandos y el lado de consultas, además de un consumidor de eventos desde Azure Service Bus.

### Componentes

| Componente | Tipo | Responsabilidad |
|------------|------|-----------------|
| `CitasController` | REST Controller | Endpoints REST generados con OpenAPI |
| `CitaDelegate` | Delegate | Desacopla la lógica del controlador |
| `CitaMapper` | MapStruct | Transformación DTO ↔ Dominio ↔ Entity |
| `AgendarCitaService` | Command Service | Registrar cita |
| `CancelarCitaService` | Command Service | Cancelar cita |
| `ReagendarCitaService` | Command Service | Reagendar cita |
| `CitaEventPublisher` | Domain Event Publisher | Publicación de eventos de dominio |
| `ListarCitaQuery` | Query Service | Consulta listado de citas |
| `ObtenerCitaQuery` | Query Service | Consulta detalle de cita |
| `AgendarCitaConsumer` | Azure Service Bus Consumer | Consume eventos para registrar citas |
| `CitaJpaAdapter` | Outbound Adapter | Persistencia Azure SQL |
| `CitaCosmosAdapter` | Outbound Adapter | Persistencia Cosmos DB |
| `CitaRedisAdapter` | Outbound Adapter | Administración de cache |
| `MedicosRedisAdapter` | Outbound Adapter | Cache de Directorio Médico |
| `ReniecWebClient` | Outbound Adapter | Integración con RENIEC |
| `SeguroWebClient` | Outbound Adapter | Integración con Sistema de Aseguramiento EsSalud |
| `ServiceBusPublisher` | Azure Service Bus Adapter | Publicación de eventos de dominio |

---
---

## C3 — Nivel 3: Component Diagram · MS Consultas

---

| Metadato | Valor |
|----------|-------|
| **Nivel C4** | C3 — Component Diagram |
| **Sistema** | HEPAQ |
| **Módulo documentado** | MS Consultas |
| **Diagrama fuente** | [c3/c3_ms_consulta.puml](c3/c3_ms_consulta.puml) |
| **Versión** | v1 |
| **Fecha** | 2026-06-20 |
| **Estado** | Aprobado |

### Descripción

Desglosa los componentes internos del microservicio **MS Consultas** (Spring Boot WebFlux). Gestiona el registro, edición y cancelación de atenciones médicas, con CQRS y publicación de eventos de dominio.

### Componentes

| Componente | Tipo | Responsabilidad |
|------------|------|-----------------|
| `ConsultasController` | REST Controller | Endpoints REST generados con OpenAPI |
| `ConsultaDelegate` | Delegate | Desacopla la lógica del controlador |
| `ConsultaMapper` | MapStruct | Transformación DTO ↔ Dominio ↔ Entity |
| `RegistrarConsultaService` | Command Service | Registrar consulta médica |
| `EditarConsultaService` | Command Service | Editar consulta médica |
| `CancelarConsultaService` | Command Service | Cancelar consulta médica |
| `ConsultaEventPublisher` | Domain Event Publisher | Publicación de eventos de dominio |
| `ObtenerConsultaQuery` | Query Service | Consulta detalle de atención |
| `ListarConsultasQuery` | Query Service | Consulta listado de atenciones |
| `RegistrarConsultaConsumer` | Azure Service Bus Consumer | Consume eventos para registrar consultas |
| `ConsultaJpaAdapter` | Outbound Adapter | Persistencia Azure SQL |
| `ConsultaCosmosAdapter` | Outbound Adapter | Persistencia Cosmos DB |
| `ConsultaRedisAdapter` | Outbound Adapter | Administración de cache |
| `ServiceBusPublisher` | Azure Service Bus Adapter | Publicación de eventos de dominio |

---
---

## C3 — Nivel 3: Component Diagram · MS Diagnósticos

---

| Metadato | Valor |
|----------|-------|
| **Nivel C4** | C3 — Component Diagram |
| **Sistema** | HEPAQ |
| **Módulo documentado** | MS Diagnósticos |
| **Diagrama fuente** | [c3/c3_ms_diagnosticos.puml](c3/c3_ms_diagnosticos.puml) |
| **Versión** | v1 |
| **Fecha** | 2026-06-20 |
| **Estado** | Aprobado |

### Descripción

Desglosa los componentes internos del microservicio **MS Diagnósticos** (Spring Boot WebFlux). Maneja el registro y eliminación de diagnósticos, consulta del catálogo CIE-10 vía Azure Table Storage y publicación de eventos de dominio.

### Componentes

| Componente | Tipo | Responsabilidad |
|------------|------|-----------------|
| `DiagnosticosController` | REST Controller | Endpoints REST generados con OpenAPI |
| `DiagnosticoDelegate` | Delegate | Desacopla la lógica del controlador |
| `DiagnosticoMapper` | MapStruct | Transformación DTO ↔ Dominio ↔ Entity |
| `RegistrarDiagnosticoService` | Command Service | Registrar diagnóstico |
| `BorrarDiagnosticoService` | Command Service | Eliminar diagnóstico |
| `DiagnosticoEventPublisher` | Domain Event Publisher | Publicación de eventos de dominio |
| `ObtenerDiagnosticoQuery` | Query Service | Consulta detalle de diagnóstico |
| `ListarDiagnosticosQuery` | Query Service | Consulta listado de diagnósticos |
| `ListarCieQuery` | Query Service | Consulta catálogo CIE-10 |
| `BorrarDiagnosticosConsumer` | Azure Service Bus Consumer | Consume eventos para eliminación masiva de diagnósticos |
| `DiagnosticoJpaAdapter` | Outbound Adapter | Persistencia Azure SQL |
| `DiagnosticoCosmosAdapter` | Outbound Adapter | Persistencia Cosmos DB |
| `DiagnosticoRedisAdapter` | Outbound Adapter | Administración de cache |
| `CieTableAdapter` | Outbound Adapter | Consulta catálogo CIE en Azure Table Storage |
| `ServiceBusPublisher` | Azure Service Bus Adapter | Publicación de eventos de dominio |

---
---

## C3 — Nivel 3: Component Diagram · MS Exámenes

---

| Metadato | Valor |
|----------|-------|
| **Nivel C4** | C3 — Component Diagram |
| **Sistema** | HEPAQ |
| **Módulo documentado** | MS Exámenes |
| **Diagrama fuente** | [c3/c3_ms_examenes.puml](c3/c3_ms_examenes.puml) |
| **Versión** | v1 |
| **Fecha** | 2026-06-20 |
| **Estado** | Aprobado |

### Descripción

Desglosa los componentes internos del microservicio **MS Exámenes** (Spring Boot WebFlux). Gestiona el ciclo completo de exámenes médicos incluyendo registro, cancelación, edición y eliminación. Maneja adjuntos clínicos mediante Azure Blob Storage y múltiples consumidores de eventos.

### Componentes

| Componente | Tipo | Responsabilidad |
|------------|------|-----------------|
| `ExamenesController` | REST Controller | Endpoints REST generados con OpenAPI |
| `ExamenDelegate` | Delegate | Desacopla la lógica del controlador |
| `ExamenMapper` | MapStruct | Transformación DTO ↔ Dominio ↔ Entity |
| `RegistrarExamenService` | Command Service | Registrar examen |
| `CancelarExamenService` | Command Service | Cancelar examen |
| `EditarExamenService` | Command Service | Editar examen |
| `BorrarExamenService` | Command Service | Eliminar examen |
| `ExamenEventPublisher` | Domain Event Publisher | Publicación de eventos de dominio |
| `ObtenerExamenQuery` | Query Service | Consulta detalle de examen |
| `ListarExamenesQuery` | Query Service | Consulta listado de exámenes |
| `ObtenerAdjuntoQuery` | Query Service | Obtiene adjuntos del examen |
| `RegistrarExamenConsumer` | Azure Service Bus Consumer | Consume eventos para registrar exámenes |
| `CancelarExamenConsumer` | Azure Service Bus Consumer | Consume eventos para cancelar exámenes |
| `BorrarExamenConsumer` | Azure Service Bus Consumer | Consume eventos para eliminar exámenes |
| `ExamenJpaAdapter` | Outbound Adapter | Persistencia Azure SQL |
| `ExamenCosmosAdapter` | Outbound Adapter | Persistencia Cosmos DB |
| `ExamenRedisAdapter` | Outbound Adapter | Administración de cache |
| `BlobStorageAdapter` | Outbound Adapter | Gestión de adjuntos médicos en Azure Blob |
| `ServiceBusPublisher` | Azure Service Bus Adapter | Publicación de eventos de dominio |

---
---

## C3 — Nivel 3: Component Diagram · MS Notificaciones

---

| Metadato | Valor |
|----------|-------|
| **Nivel C4** | C3 — Component Diagram |
| **Sistema** | HEPAQ |
| **Módulo documentado** | MS Notificaciones |
| **Diagrama fuente** | [c3/C3_ms_notificaciones.puml](c3/C3_ms_notificaciones.puml) |
| **Versión** | v1 |
| **Fecha** | 2026-06-20 |
| **Estado** | Aprobado |

### Descripción

Desglosa los componentes internos del microservicio **MS Notificaciones** (Spring Boot WebFlux). Es un microservicio reactivo orientado exclusivamente a la recepción de eventos y al despacho de notificaciones por múltiples canales (correo electrónico, SMS). Registra auditoría de cada envío en Azure SQL.

### Componentes

| Componente | Tipo | Responsabilidad |
|------------|------|-----------------|
| `GenerarNotificacionConsumer` | Azure Service Bus Consumer | Consume eventos para generar notificaciones |
| `GenerarNotificacionService` | Command Service | Genera y envía notificaciones |
| `EmailTemplateService` | Template Service | Construcción de plantillas de correo |
| `JavaMailAdapter` | Spring Mail / Jakarta Mail | Envío de correos electrónicos vía SMTP/TLS |
| `NotificacionJpaAdapter` | Outbound Adapter | Persistencia de auditoría de notificaciones |
| `NotificacionJpaRepository` | Spring Data JPA | Acceso a auditoría de notificaciones en Azure SQL |

---
---

## C4 — Nivel 4: Code Diagram · MS Citas

---

| Metadato | Valor |
|----------|-------|
| **Nivel C4** | C4 — Code Diagram |
| **Sistema** | HEPAQ |
| **Módulo documentado** | MS Citas — Clases de dominio y patrones de diseño |
| **Diagrama fuente** | [c4/c4_ms_citas.puml](c4/c4_ms_citas.puml) |
| **Versión** | v1 |
| **Fecha** | 2026-06-20 |
| **Estado** | Aprobado |

### Descripción

Detalla las clases, interfaces y enumeraciones del microservicio **MS Citas**. Documenta los patrones de diseño aplicados: **Strategy** para el tipo de agenda y **Chain of Responsibility** para la cadena de validaciones previa al agendamiento.

### Dominio

| Elemento | Tipo | Descripción |
|----------|------|-------------|
| `Cita` | Clase | Entidad raíz con `id`, `pacienteId`, `medicoId`, `fechaHora`, `tipo`, `estado` |
| `TipoCita` | Enum | `CONSULTA_MEDICA`, `LABORATORIO`, `ECOGRAFIA` |
| `EstadoCita` | Enum | `PENDIENTE`, `CONFIRMADA`, `CANCELADA`, `REPROGRAMADA` |
| `AgendaStrategy` | Interface | `+validar(cita)`, `+reservar(cita)` |
| `ConsultaMedicaStrategy` | Clase | Implementa `AgendaStrategy` para consulta médica |
| `LaboratorioStrategy` | Clase | Implementa `AgendaStrategy` para laboratorio |
| `EcografiaStrategy` | Clase | Implementa `AgendaStrategy` para ecografía |
| `ValidacionCitaHandler` | Clase abstracta | Cadena de responsabilidad: `+setNext()`, `+validar()` |
| `DisponibilidadMedicoHandler` | Clase | Valida disponibilidad del médico |
| `ReniecValidacionHandler` | Clase | Valida identidad del paciente vía RENIEC |
| `SeguroValidacionHandler` | Clase | Valida cobertura del seguro |

### Application

| Elemento | Tipo | Descripción |
|----------|------|-------------|
| `AgendarCitaUseCase` | Clase | Ejecuta el caso de uso de agendamiento |
| `ReagendarCitaUseCase` | Clase | Ejecuta el caso de uso de reagendamiento |
| `CancelarCitaUseCase` | Clase | Ejecuta el caso de uso de cancelación |
| `AgendarStrategyFactory` | Clase | Factoría que obtiene la estrategia según `TipoCita` |

### Ports

| Puerto | Tipo | Métodos |
|--------|------|---------|
| `MedicoPort` | Interface | `+obtenerDisponibilidad()` |
| `ReniecPort` | Interface | `+validarIdentidad()` |
| `SeguroPort` | Interface | `+validarCobertura()` |
| `EventPublisherPort` | Interface | `+publish(event)` |
| `CitaRepositoryPort` | Interface | `+guardar(cita)`, `+obtener(id)` |

### Adapters

| Adaptador | Implementa |
|-----------|------------|
| `MedicoAdapter` | `MedicoPort` |
| `ReniecAdapter` | `ReniecPort` |
| `SeguroAdapter` | `SeguroPort` |
| `ServiceBusPublisherAdapter` | `EventPublisherPort` |
| `CitaJpaAdapter` | `CitaRepositoryPort` |

### Patrones aplicados

| Patrón | Aplicación |
|--------|------------|
| **Strategy** | `AgendaStrategy` con implementaciones por tipo de cita |
| **Chain of Responsibility** | `ValidacionCitaHandler` para validaciones encadenadas antes del agendamiento |
| **Factory** | `AgendarStrategyFactory` para selección dinámica de estrategia |

---
---

## C4 — Nivel 4: Code Diagram · MS Consultas

---

| Metadato | Valor |
|----------|-------|
| **Nivel C4** | C4 — Code Diagram |
| **Sistema** | HEPAQ |
| **Módulo documentado** | MS Consultas — Clases de dominio y patrones de diseño |
| **Diagrama fuente** | [c4/c4_ms_consulta.puml](c4/c4_ms_consulta.puml) |
| **Versión** | v1 |
| **Fecha** | 2026-06-20 |
| **Estado** | Aprobado |

### Descripción

Detalla las clases, interfaces y enumeraciones del microservicio **MS Consultas**. Documenta el patrón **Template Method** aplicado al ciclo de vida de los casos de uso: validar → persistir → publicar evento.

### Dominio

| Elemento | Tipo | Descripción |
|----------|------|-------------|
| `Consulta` | Clase | Entidad raíz con `id`, `pacienteId`, `medicoId`, `citaId`, `fechaHora`, `tipo`, `estado`, `motivoConsulta`, `observaciones` |
| `TipoConsulta` | Enum | `PRIMERA_VEZ`, `CONTROL`, `SEGUIMIENTO`, `EMERGENCIA` |
| `EstadoConsulta` | Enum | `PENDIENTE`, `EN_CURSO`, `COMPLETADA`, `CANCELADA` |

### Application

| Elemento | Tipo | Descripción |
|----------|------|-------------|
| `ConsultaUseCaseTemplate` | Clase abstracta | Define el esqueleto del flujo: `+ejecutar()`, `#validar()`, `#persistir()`, `#publicarEvento()` |
| `RegistrarConsultaUseCase` | Clase | Implementa persistir y publicarEvento para registro |
| `EditarConsultaUseCase` | Clase | Implementa persistir y publicarEvento para edición |
| `CancelarConsultaUseCase` | Clase | Implementa persistir y publicarEvento para cancelación |

### Ports

| Puerto | Tipo | Métodos |
|--------|------|---------|
| `MedicoPort` | Interface | `+obtenerMedico(medicoId)` |
| `CitaPort` | Interface | `+obtenerCita(citaId)` |
| `ConsultaRepositoryPort` | Interface | `+guardar()`, `+obtener(id)`, `+actualizar()` |
| `EventPublisherPort` | Interface | `+publish(event)` |
| `ConsultaReadRepositoryPort` | Interface | `+obtenerPorId(id)`, `+listar(filtros)` |
| `ConsultaCachePort` | Interface | `+obtener(id)`, `+invalidar(id)` |

### Adapters

| Adaptador | Implementa |
|-----------|------------|
| `MedicoWebClientAdapter` | `MedicoPort` |
| `CitaWebClientAdapter` | `CitaPort` |
| `ConsultaJpaAdapter` | `ConsultaRepositoryPort` |
| `ServiceBusPublisherAdapter` | `EventPublisherPort` |
| `ConsultaCosmosAdapter` | `ConsultaReadRepositoryPort` |
| `ConsultaRedisAdapter` | `ConsultaCachePort` |

### Patrones aplicados

| Patrón | Aplicación |
|--------|------------|
| **Template Method** | `ConsultaUseCaseTemplate` define el flujo; las subclases implementan los pasos específicos |

---
---

## C4 — Nivel 4: Code Diagram · MS Diagnósticos

---

| Metadato | Valor |
|----------|-------|
| **Nivel C4** | C4 — Code Diagram |
| **Sistema** | HEPAQ |
| **Módulo documentado** | MS Diagnósticos — Clases de dominio y patrones de diseño |
| **Diagrama fuente** | [c4/c4_ms_diagnosticos.puml](c4/c4_ms_diagnosticos.puml) |
| **Versión** | v1 |
| **Fecha** | 2026-06-20 |
| **Estado** | Aprobado |

### Descripción

Detalla las clases, interfaces y enumeraciones del microservicio **MS Diagnósticos**. Documenta el patrón **Decorator** aplicado al repositorio de diagnósticos para añadir comportamiento de caché sin modificar la implementación base.

### Dominio

| Elemento | Tipo | Descripción |
|----------|------|-------------|
| `Diagnostico` | Clase | Entidad raíz con `id`, `pacienteId`, `medicoId`, `consultaId`, `codigoCIE`, `descripcion`, `tipo`, `estado`, `fechaRegistro` |
| `CodigoCIE` | Clase | `codigo`, `descripcion`, `categoria` — representa un código CIE-10 |
| `TipoDiagnostico` | Enum | `PRESUNTIVO`, `DEFINITIVO`, `DIFERENCIAL`, `SECUNDARIO` |
| `EstadoDiagnostico` | Enum | `ACTIVO`, `RESUELTO`, `CRONICO`, `ELIMINADO` |

### Application

| Elemento | Tipo | Descripción |
|----------|------|-------------|
| `RegistrarDiagnosticoUseCase` | Clase | Ejecuta el caso de uso de registro de diagnóstico |
| `BorrarDiagnosticoUseCase` | Clase | Ejecuta el caso de uso de eliminación de diagnóstico |

### Ports

| Puerto | Tipo | Métodos |
|--------|------|---------|
| `DiagnosticoRepositoryPort` | Interface | `+guardar()`, `+obtener(id)`, `+eliminar(id)` |
| `DiagnosticoReadRepositoryPort` | Interface | `+obtenerPorId(id)`, `+listar(filtros)` |
| `CieRepositoryPort` | Interface | `+listar()`, `+obtenerPorCodigo(codigo)` |
| `EventPublisherPort` | Interface | `+publish(event)` |

### Adapters

| Adaptador | Implementa | Notas |
|-----------|------------|-------|
| `DiagnosticoCacheDecorator` | `DiagnosticoRepositoryPort` | Decorator que envuelve la persistencia con caché |
| `DiagnosticoJpaAdapter` | `DiagnosticoRepositoryPort` | Persistencia Azure SQL (delegado por el Decorator) |
| `DiagnosticoCosmosAdapter` | `DiagnosticoReadRepositoryPort` | Persistencia Cosmos DB |
| `DiagnosticoRedisAdapter` | — | Caché Redis (utilizado por el Decorator) |
| `CieTableAdapter` | `CieRepositoryPort` | Consulta catálogo CIE en Azure Table Storage |
| `ServiceBusPublisherAdapter` | `EventPublisherPort` | Publicación de eventos de dominio |

### Patrones aplicados

| Patrón | Aplicación |
|--------|------------|
| **Decorator** | `DiagnosticoCacheDecorator` añade capa de caché sobre `DiagnosticoJpaAdapter` sin modificarlo |

---
---

## C4 — Nivel 4: Code Diagram · MS Exámenes

---

| Metadato | Valor |
|----------|-------|
| **Nivel C4** | C4 — Code Diagram |
| **Sistema** | HEPAQ |
| **Módulo documentado** | MS Exámenes — Clases de dominio y patrones de diseño |
| **Diagrama fuente** | [c4/c4_ms_examenes.puml](c4/c4_ms_examenes.puml) |
| **Versión** | v1 |
| **Fecha** | 2026-06-20 |
| **Estado** | Aprobado |

### Descripción

Detalla las clases, interfaces y enumeraciones del microservicio **MS Exámenes**. Documenta el patrón **Strategy** aplicado al procesamiento de exámenes según su tipo, y la composición `Examen` → `AdjuntoExamen` para gestionar archivos adjuntos.

### Dominio

| Elemento | Tipo | Descripción |
|----------|------|-------------|
| `Examen` | Clase | Entidad raíz con `id`, `pacienteId`, `medicoId`, `consultaId`, `tipo`, `estado`, `descripcion`, `fechaSolicitud`, `fechaResultado` |
| `AdjuntoExamen` | Clase | `id`, `examenId`, `nombreArchivo`, `urlBlob`, `contentType`, `tamanioBytes` |
| `TipoExamen` | Enum | `LABORATORIO`, `IMAGEN`, `ECOGRAFIA`, `RAYOS_X`, `TOMOGRAFIA`, `RESONANCIA` |
| `EstadoExamen` | Enum | `PENDIENTE`, `EN_PROCESO`, `COMPLETADO`, `CANCELADO`, `ELIMINADO` |
| `ExamenProcesadorStrategy` | Interface | `+validar(examen)`, `+procesar(examen)` |
| `EcografiaExamenStrategy` | Clase | Implementa `ExamenProcesadorStrategy` para ecografía |
| `LaboratorioExamenStrategy` | Clase | Implementa `ExamenProcesadorStrategy` para laboratorio |
| `ImagenExamenStrategy` | Clase | Implementa `ExamenProcesadorStrategy` para imagen |

### Application

| Elemento | Tipo | Descripción |
|----------|------|-------------|
| `RegistrarExamenUseCase` | Clase | Ejecuta el caso de uso de registro de examen |
| `CancelarExamenUseCase` | Clase | Ejecuta el caso de uso de cancelación |
| `EditarExamenUseCase` | Clase | Ejecuta el caso de uso de edición |
| `BorrarExamenUseCase` | Clase | Ejecuta el caso de uso de eliminación |
| `ExamenStrategyFactory` | Clase | `+obtener(tipo : TipoExamen) : ExamenProcesadorStrategy` |

### Ports

| Puerto | Tipo | Métodos |
|--------|------|---------|
| `ExamenRepositoryPort` | Interface | `+guardar()`, `+obtener(id)`, `+actualizar()`, `+eliminar(id)` |
| `ExamenReadRepositoryPort` | Interface | `+obtenerPorId(id)`, `+listar(filtros)` |
| `ExamenCachePort` | Interface | `+obtener(id)`, `+invalidar(id)` |
| `EventPublisherPort` | Interface | `+publish(event)` |
| `BlobStoragePort` | Interface | `+subir(archivo)`, `+obtener(url)`, `+eliminar(url)` |

### Adapters

| Adaptador | Implementa |
|-----------|------------|
| `ExamenJpaAdapter` | `ExamenRepositoryPort` |
| `ExamenCosmosAdapter` | `ExamenReadRepositoryPort` |
| `ExamenRedisAdapter` | `ExamenCachePort` |
| `ServiceBusPublisherAdapter` | `EventPublisherPort` |
| `BlobStorageAdapter` | `BlobStoragePort` |

### Patrones aplicados

| Patrón | Aplicación |
|--------|------------|
| **Strategy** | `ExamenProcesadorStrategy` con implementaciones por tipo de examen |
| **Factory** | `ExamenStrategyFactory` para selección dinámica de la estrategia de procesamiento |

---
---

## C4 — Nivel 4: Code Diagram · MS Notificaciones

---

| Metadato | Valor |
|----------|-------|
| **Nivel C4** | C4 — Code Diagram |
| **Sistema** | HEPAQ |
| **Módulo documentado** | MS Notificaciones — Clases de dominio y patrones de diseño |
| **Diagrama fuente** | [c4/c4_ms_notificaciones.puml](c4/c4_ms_notificaciones.puml) |
| **Versión** | v1 |
| **Fecha** | 2026-06-20 |
| **Estado** | Aprobado |

### Descripción

Detalla las clases, interfaces y enumeraciones del microservicio **MS Notificaciones**. Documenta el patrón **Composite** para envío multicanal y el patrón **Strategy** para encapsular la lógica de cada canal de notificación (correo electrónico, SMS).

### Dominio

| Elemento | Tipo | Descripción |
|----------|------|-------------|
| `Notificacion` | Clase | Entidad con `id`, `pacienteId`, `destinatario`, `asunto`, `cuerpo`, `tipo`, `estado`, `fechaEnvio` |
| `TipoNotificacion` | Enum | `CITA_AGENDADA`, `CITA_CANCELADA`, `CITA_REAGENDADA`, `CONSULTA_CANCELADA`, `EXAMEN_COMPLETADO`, `DIAGNOSTICO_REGISTRADO` |
| `EstadoNotificacion` | Enum | `PENDIENTE`, `ENVIADA`, `FALLIDA` |
| `NotificacionStrategy` | Interface | `+enviar(notificacion : Notificacion)` |
| `EmailNotificacionStrategy` | Clase | Implementa envío por correo electrónico |
| `SmsNotificacionStrategy` | Clase | Implementa envío por SMS |
| `CompositeNotificacionStrategy` | Clase | `strategies : List<NotificacionStrategy>` — envío combinado por múltiples canales |

### Application

| Elemento | Tipo | Descripción |
|----------|------|-------------|
| `GenerarNotificacionUseCase` | Clase | `+ejecutar(cmd)` — coordina la generación y envío de la notificación |
| `NotificacionStrategyFactory` | Clase | `+obtener(tipo : TipoNotificacion)` — devuelve la estrategia adecuada |

### Ports

| Puerto | Tipo | Métodos |
|--------|------|---------|
| `EmailTemplatePort` | Interface | `+construir(tipo, datos)` |
| `NotificacionRepositoryPort` | Interface | `+guardar(notificacion)`, `+obtener(id)` |
| `NotificacionSenderPort` | Interface | `+enviar(notificacion)` |

### Adapters

| Adaptador | Implementa |
|-----------|------------|
| `ThymeleafEmailTemplateAdapter` | `EmailTemplatePort` |
| `NotificacionJpaAdapter` | `NotificacionRepositoryPort` |
| `JavaMailAdapter` | `NotificacionSenderPort` |

### Patrones aplicados

| Patrón | Aplicación |
|--------|------------|
| **Strategy** | `NotificacionStrategy` con implementaciones por canal (Email, SMS) |
| **Composite** | `CompositeNotificacionStrategy` agrupa múltiples estrategias y las ejecuta como una sola |
| **Factory** | `NotificacionStrategyFactory` para selección dinámica de estrategia por tipo de notificación |

---

## Resumen de Patrones de Diseño

| Microservicio | Patrón(es) aplicados |
|---------------|----------------------|
| MS Citas | Strategy · Chain of Responsibility · Factory |
| MS Consultas | Template Method |
| MS Diagnósticos | Decorator |
| MS Exámenes | Strategy · Factory |
| MS Notificaciones | Strategy · Composite · Factory |

---

## Control de Versiones del Documento

| Versión | Fecha | Estado | Autor |
|---------|-------|--------|-------|
| v1 | 2026-06-20 | Aprobado | Equipo de Arquitectura HEPAQ |
