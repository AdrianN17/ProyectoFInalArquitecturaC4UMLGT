# DB Schema — MS Notificaciones

## Write Model — Azure SQL (Liquibase Changelog)

**Base de datos:** `hepaq_notificaciones`
**Tecnología:** Azure SQL Database · Spring Data JPA · Liquibase 4.x
> MS Notificaciones **no usa Cosmos DB** — solo escribe en Azure SQL para auditoría.

```xml
<?xml version="1.0" encoding="UTF-8"?>
<databaseChangeLog
    xmlns="http://www.liquibase.org/xml/ns/dbchangelog"
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xsi:schemaLocation="http://www.liquibase.org/xml/ns/dbchangelog
        http://www.liquibase.org/xml/ns/dbchangelog/dbchangelog-4.20.xsd">

    <!-- ================================================ -->
    <!-- CHANGESET 001 — Tabla de auditoría: notificaciones -->
    <!-- ================================================ -->
    <changeSet id="001-create-table-notificaciones" author="hepaq">
        <preConditions onFail="MARK_RAN">
            <not><tableExists tableName="notificaciones"/></not>
        </preConditions>

        <createTable tableName="notificaciones"
                     remarks="Registro de auditoría de todas las notificaciones enviadas">

            <column name="id" type="uniqueidentifier"
                    remarks="Identificador único UUID de la notificación">
                <constraints primaryKey="true" nullable="false"/>
            </column>

            <column name="paciente_id" type="varchar(20)"
                    remarks="DNI o código del paciente destinatario">
                <constraints nullable="false"/>
            </column>

            <column name="destinatario" type="varchar(320)"
                    remarks="Email o número de teléfono del destinatario">
                <constraints nullable="false"/>
            </column>

            <column name="destinatario_secundario" type="varchar(320)"
                    remarks="Email CC o número alternativo (nullable)"/>

            <column name="asunto" type="nvarchar(500)"
                    remarks="Asunto del mensaje (email) o título (SMS/push)">
                <constraints nullable="false"/>
            </column>

            <column name="cuerpo" type="nvarchar(max)"
                    remarks="Contenido completo del mensaje enviado"/>

            <column name="tipo" type="varchar(50)"
                    remarks="CITA_AGENDADA | CITA_CANCELADA | CITA_REAGENDADA | CONSULTA_CANCELADA | EXAMEN_COMPLETADO | DIAGNOSTICO_REGISTRADO">
                <constraints nullable="false"/>
            </column>

            <column name="canal" type="varchar(10)"
                    remarks="EMAIL | SMS | PUSH">
                <constraints nullable="false"/>
            </column>

            <column name="estado" type="varchar(15)"
                    defaultValue="PENDIENTE"
                    remarks="PENDIENTE | ENVIADA | FALLIDA | REINTENTANDO">
                <constraints nullable="false"/>
            </column>

            <column name="intentos" type="int" defaultValueNumeric="0"
                    remarks="Número de intentos de envío realizados">
                <constraints nullable="false"/>
            </column>

            <column name="max_intentos" type="int" defaultValueNumeric="3"
                    remarks="Máximo de intentos permitidos antes de marcar como FALLIDA definitiva">
                <constraints nullable="false"/>
            </column>

            <column name="error_mensaje" type="nvarchar(1000)"
                    remarks="Descripción del error en el último intento fallido"/>

            <column name="id_evento_origen" type="varchar(200)"
                    remarks="ID del evento del Service Bus que originó la notificación (trazabilidad)"/>

            <column name="fecha_envio" type="datetime2(0)"
                    remarks="Timestamp del envío exitoso (nullable hasta que se envíe)"/>

            <column name="fecha_creacion" type="datetime2(0)"
                    defaultValueComputed="GETUTCDATE()">
                <constraints nullable="false"/>
            </column>

            <column name="fecha_actualizacion" type="datetime2(0)"
                    defaultValueComputed="GETUTCDATE()">
                <constraints nullable="false"/>
            </column>
        </createTable>
    </changeSet>

    <!-- ================================================ -->
    <!-- CHANGESET 002 — Tabla de intentos                 -->
    <!-- ================================================ -->
    <changeSet id="002-create-table-notificaciones-intentos" author="hepaq">

        <createTable tableName="notificaciones_intentos"
                     remarks="Historial detallado de cada intento de envío">

            <column name="id" type="uniqueidentifier">
                <constraints primaryKey="true" nullable="false"/>
            </column>

            <column name="notificacion_id" type="uniqueidentifier">
                <constraints nullable="false"/>
            </column>

            <column name="numero_intento" type="int">
                <constraints nullable="false"/>
            </column>

            <column name="exitoso" type="bit">
                <constraints nullable="false"/>
            </column>

            <column name="error_detalle" type="nvarchar(2000)"
                    remarks="Stack trace o mensaje de error del proveedor SMTP/SMS"/>

            <column name="latencia_ms" type="int"
                    remarks="Tiempo de respuesta del servidor SMTP/SMS en milisegundos"/>

            <column name="fecha_intento" type="datetime2(3)"
                    defaultValueComputed="GETUTCDATE()">
                <constraints nullable="false"/>
            </column>
        </createTable>

        <addForeignKeyConstraint
            baseTableName="notificaciones_intentos"
            baseColumnNames="notificacion_id"
            constraintName="fk_intentos_notificacion_id"
            referencedTableName="notificaciones"
            referencedColumnNames="id"
            onDelete="CASCADE"/>
    </changeSet>

    <!-- ================================================ -->
    <!-- CHANGESET 003 — Tabla de plantillas               -->
    <!-- ================================================ -->
    <changeSet id="003-create-table-plantillas-notificacion" author="hepaq">

        <createTable tableName="plantillas_notificacion"
                     remarks="Plantillas Thymeleaf para emails y mensajes SMS">

            <column name="id" type="uniqueidentifier">
                <constraints primaryKey="true" nullable="false"/>
            </column>

            <column name="codigo" type="varchar(100)"
                    remarks="Código único de la plantilla. Ej: CITA_AGENDADA_EMAIL">
                <constraints nullable="false" unique="true"
                             uniqueConstraintName="uidx_plantillas_codigo"/>
            </column>

            <column name="tipo" type="varchar(50)"
                    remarks="Tipo de notificación al que aplica">
                <constraints nullable="false"/>
            </column>

            <column name="canal" type="varchar(10)"
                    remarks="EMAIL | SMS | PUSH">
                <constraints nullable="false"/>
            </column>

            <column name="asunto_template" type="nvarchar(500)"
                    remarks="Plantilla del asunto con variables Thymeleaf: {{variableNombre}}"/>

            <column name="cuerpo_template" type="nvarchar(max)"
                    remarks="Cuerpo HTML o texto con variables Thymeleaf">
                <constraints nullable="false"/>
            </column>

            <column name="idioma" type="varchar(5)"
                    defaultValue="es"
                    remarks="Código de idioma ISO 639-1 (es, en)">
                <constraints nullable="false"/>
            </column>

            <column name="activa" type="bit" defaultValueBoolean="true">
                <constraints nullable="false"/>
            </column>

            <column name="fecha_creacion" type="datetime2(0)"
                    defaultValueComputed="GETUTCDATE()">
                <constraints nullable="false"/>
            </column>

            <column name="fecha_actualizacion" type="datetime2(0)"
                    defaultValueComputed="GETUTCDATE()">
                <constraints nullable="false"/>
            </column>
        </createTable>
    </changeSet>

    <!-- ================================================ -->
    <!-- CHANGESET 004 — Índices de rendimiento            -->
    <!-- ================================================ -->
    <changeSet id="004-indexes-notificaciones" author="hepaq">

        <createIndex indexName="idx_notif_paciente_id"
                     tableName="notificaciones">
            <column name="paciente_id"/>
        </createIndex>

        <createIndex indexName="idx_notif_estado"
                     tableName="notificaciones">
            <column name="estado"/>
        </createIndex>

        <!-- Crítico para el scheduler de reintentos -->
        <createIndex indexName="idx_notif_estado_intentos"
                     tableName="notificaciones">
            <column name="estado"/>
            <column name="intentos"/>
            <column name="max_intentos"/>
        </createIndex>

        <createIndex indexName="idx_notif_tipo"
                     tableName="notificaciones">
            <column name="tipo"/>
        </createIndex>

        <createIndex indexName="idx_notif_fecha_creacion"
                     tableName="notificaciones">
            <column name="fecha_creacion" descending="true"/>
        </createIndex>

        <createIndex indexName="idx_notif_intentos_notif_id"
                     tableName="notificaciones_intentos">
            <column name="notificacion_id"/>
        </createIndex>

        <createIndex indexName="idx_plantillas_tipo_canal"
                     tableName="plantillas_notificacion">
            <column name="tipo"/>
            <column name="canal"/>
            <column name="idioma"/>
            <column name="activa"/>
        </createIndex>
    </changeSet>

    <!-- ================================================ -->
    <!-- CHANGESET 005 — Constraints y datos maestros      -->
    <!-- ================================================ -->
    <changeSet id="005-constraints-notificaciones" author="hepaq">

        <sql>
            ALTER TABLE notificaciones
            ADD CONSTRAINT chk_notif_tipo
            CHECK (tipo IN (
                'CITA_AGENDADA', 'CITA_CANCELADA', 'CITA_REAGENDADA',
                'CONSULTA_CANCELADA', 'EXAMEN_COMPLETADO', 'DIAGNOSTICO_REGISTRADO'
            ));
        </sql>

        <sql>
            ALTER TABLE notificaciones
            ADD CONSTRAINT chk_notif_canal
            CHECK (canal IN ('EMAIL', 'SMS', 'PUSH'));
        </sql>

        <sql>
            ALTER TABLE notificaciones
            ADD CONSTRAINT chk_notif_estado
            CHECK (estado IN ('PENDIENTE', 'ENVIADA', 'FALLIDA', 'REINTENTANDO'));
        </sql>

        <sql>
            ALTER TABLE notificaciones
            ADD CONSTRAINT chk_notif_intentos
            CHECK (intentos >= 0 AND intentos &lt;= max_intentos + 1);
        </sql>
    </changeSet>

    <!-- ================================================ -->
    <!-- CHANGESET 006 — Datos iniciales: plantillas       -->
    <!-- ================================================ -->
    <changeSet id="006-data-plantillas-iniciales" author="hepaq">

        <insert tableName="plantillas_notificacion">
            <column name="id"               value="a1b2c3d4-0001-0001-0001-000000000001"/>
            <column name="codigo"           value="CITA_AGENDADA_EMAIL"/>
            <column name="tipo"             value="CITA_AGENDADA"/>
            <column name="canal"            value="EMAIL"/>
            <column name="asunto_template"  value="HEPAQ - Su cita ha sido agendada para el {{fechaCita}}"/>
            <column name="cuerpo_template"  value="&lt;html&gt;&lt;body&gt;&lt;h2&gt;Estimado/a {{pacienteNombre}},&lt;/h2&gt;&lt;p&gt;Su cita de tipo &lt;strong&gt;{{tipoCita}}&lt;/strong&gt; con el &lt;strong&gt;{{medicoNombre}}&lt;/strong&gt; ha sido confirmada para el &lt;strong&gt;{{fechaCita}}&lt;/strong&gt;.&lt;/p&gt;&lt;/body&gt;&lt;/html&gt;"/>
            <column name="idioma"           value="es"/>
            <column name="activa"           valueBoolean="true"/>
        </insert>

        <insert tableName="plantillas_notificacion">
            <column name="id"               value="a1b2c3d4-0002-0002-0002-000000000002"/>
            <column name="codigo"           value="CITA_CANCELADA_EMAIL"/>
            <column name="tipo"             value="CITA_CANCELADA"/>
            <column name="canal"            value="EMAIL"/>
            <column name="asunto_template"  value="HEPAQ - Su cita del {{fechaCita}} ha sido cancelada"/>
            <column name="cuerpo_template"  value="&lt;html&gt;&lt;body&gt;&lt;h2&gt;Estimado/a {{pacienteNombre}},&lt;/h2&gt;&lt;p&gt;Lamentamos informarle que su cita del &lt;strong&gt;{{fechaCita}}&lt;/strong&gt; ha sido cancelada. Motivo: {{motivoCancelacion}}.&lt;/p&gt;&lt;/body&gt;&lt;/html&gt;"/>
            <column name="idioma"           value="es"/>
            <column name="activa"           valueBoolean="true"/>
        </insert>

        <insert tableName="plantillas_notificacion">
            <column name="id"               value="a1b2c3d4-0003-0003-0003-000000000003"/>
            <column name="codigo"           value="EXAMEN_COMPLETADO_EMAIL"/>
            <column name="tipo"             value="EXAMEN_COMPLETADO"/>
            <column name="canal"            value="EMAIL"/>
            <column name="asunto_template"  value="HEPAQ - Sus resultados de {{tipoExamen}} están disponibles"/>
            <column name="cuerpo_template"  value="&lt;html&gt;&lt;body&gt;&lt;h2&gt;Estimado/a {{pacienteNombre}},&lt;/h2&gt;&lt;p&gt;Sus resultados del examen &lt;strong&gt;{{tipoExamen}}&lt;/strong&gt; solicitado el {{fechaSolicitud}} ya están disponibles. Por favor ingrese a la plataforma HEPAQ para consultarlos.&lt;/p&gt;&lt;/body&gt;&lt;/html&gt;"/>
            <column name="idioma"           value="es"/>
            <column name="activa"           valueBoolean="true"/>
        </insert>
    </changeSet>

</databaseChangeLog>
```

---

## Auditoría de Notificaciones — Ejemplo de registros SQL

### Registro en tabla `notificaciones`

```json
{
  "id": "990e8400-e29b-41d4-a716-446655440005",
  "paciente_id": "12345678",
  "destinatario": "juan.perez@email.com",
  "destinatario_secundario": null,
  "asunto": "HEPAQ - Su cita ha sido agendada para el 25/06/2026 09:00",
  "tipo": "CITA_AGENDADA",
  "canal": "EMAIL",
  "estado": "ENVIADA",
  "intentos": 1,
  "max_intentos": 3,
  "error_mensaje": null,
  "id_evento_origen": "ServiceBusMsg-abc123xyz",
  "fecha_envio": "2026-06-18T14:30:45Z",
  "fecha_creacion": "2026-06-18T14:30:30Z",
  "fecha_actualizacion": "2026-06-18T14:30:45Z"
}
```

### Registro en tabla `notificaciones_intentos`

```json
{
  "id": "991e8400-e29b-41d4-a716-446655440006",
  "notificacion_id": "990e8400-e29b-41d4-a716-446655440005",
  "numero_intento": 1,
  "exitoso": true,
  "error_detalle": null,
  "latencia_ms": 342,
  "fecha_intento": "2026-06-18T14:30:45.123Z"
}
```

### Ejemplo de notificación fallida con reintento

```json
{
  "id": "992e8400-e29b-41d4-a716-446655440007",
  "paciente_id": "87654321",
  "destinatario": "maria.gomez@email.com",
  "tipo": "EXAMEN_COMPLETADO",
  "canal": "EMAIL",
  "estado": "FALLIDA",
  "intentos": 3,
  "max_intentos": 3,
  "error_mensaje": "Connection timeout: smtp.office365.com:587 (timeout after 30000ms)",
  "fecha_envio": null,
  "fecha_creacion": "2026-06-26T14:05:00Z",
  "fecha_actualizacion": "2026-06-26T14:35:00Z"
}
```

### Ejemplo plantilla en `plantillas_notificacion`

```json
{
  "id": "a1b2c3d4-0001-0001-0001-000000000001",
  "codigo": "CITA_AGENDADA_EMAIL",
  "tipo": "CITA_AGENDADA",
  "canal": "EMAIL",
  "asunto_template": "HEPAQ - Su cita ha sido agendada para el {{fechaCita}}",
  "cuerpo_template": "<html><body><h2>Estimado/a {{pacienteNombre}},</h2><p>Su cita de tipo <strong>{{tipoCita}}</strong> con el <strong>{{medicoNombre}}</strong> ha sido confirmada para el <strong>{{fechaCita}}</strong> en el consultorio <strong>{{consultorio}}</strong>.</p><p>Por favor llegue 10 minutos antes de su cita.</p><br/><p>Atentamente,<br/><strong>HEPAQ - EsSalud</strong></p></body></html>",
  "idioma": "es",
  "activa": true
}
```
