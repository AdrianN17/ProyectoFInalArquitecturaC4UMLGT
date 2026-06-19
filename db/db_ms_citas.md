# DB Schema — MS Citas

## Write Model — Azure SQL (Liquibase Changelog)

**Base de datos:** `hepaq_citas`
**Tecnología:** Azure SQL Database · Spring Data JPA · Liquibase 4.x

```xml
<?xml version="1.0" encoding="UTF-8"?>
<databaseChangeLog
    xmlns="http://www.liquibase.org/xml/ns/dbchangelog"
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xsi:schemaLocation="http://www.liquibase.org/xml/ns/dbchangelog
        http://www.liquibase.org/xml/ns/dbchangelog/dbchangelog-4.20.xsd">

    <!-- ================================================ -->
    <!-- CHANGESET 001 — Tabla principal: citas            -->
    <!-- ================================================ -->
    <changeSet id="001-create-table-citas" author="hepaq">
        <preConditions onFail="MARK_RAN">
            <not><tableExists tableName="citas"/></not>
        </preConditions>

        <createTable tableName="citas"
                     remarks="Tabla de citas médicas — Write Model (Commands)">

            <column name="id" type="uniqueidentifier"
                    remarks="Identificador único UUID de la cita">
                <constraints primaryKey="true" nullable="false"/>
            </column>

            <column name="paciente_id" type="varchar(20)"
                    remarks="DNI o código del paciente (validado contra RENIEC)">
                <constraints nullable="false"/>
            </column>

            <column name="medico_id" type="varchar(50)"
                    remarks="Código del médico asignado (Directorio Médico)">
                <constraints nullable="false"/>
            </column>

            <column name="fecha_hora" type="datetime2(0)"
                    remarks="Fecha y hora programada de la cita">
                <constraints nullable="false"/>
            </column>

            <column name="tipo" type="varchar(20)"
                    remarks="CONSULTA_MEDICA | LABORATORIO | ECOGRAFIA">
                <constraints nullable="false"/>
            </column>

            <column name="estado" type="varchar(20)"
                    defaultValue="PENDIENTE"
                    remarks="PENDIENTE | CONFIRMADA | CANCELADA | REPROGRAMADA">
                <constraints nullable="false"/>
            </column>

            <column name="motivo_cita" type="nvarchar(500)"
                    remarks="Descripción del motivo de la cita"/>

            <column name="motivo_cancelacion" type="nvarchar(500)"
                    remarks="Motivo de cancelación o reprogramación (nullable)"/>

            <column name="fecha_creacion" type="datetime2(0)"
                    defaultValueComputed="GETUTCDATE()"
                    remarks="Timestamp UTC de creación del registro">
                <constraints nullable="false"/>
            </column>

            <column name="fecha_actualizacion" type="datetime2(0)"
                    defaultValueComputed="GETUTCDATE()"
                    remarks="Timestamp UTC de última modificación">
                <constraints nullable="false"/>
            </column>

            <column name="creado_por" type="varchar(100)"
                    remarks="Usuario o servicio que creó el registro"/>

            <column name="version" type="int" defaultValueNumeric="0"
                    remarks="Control de versión optimista (optimistic locking)">
                <constraints nullable="false"/>
            </column>
        </createTable>
    </changeSet>

    <!-- ================================================ -->
    <!-- CHANGESET 002 — Índices de rendimiento            -->
    <!-- ================================================ -->
    <changeSet id="002-indexes-citas" author="hepaq">

        <createIndex indexName="idx_citas_paciente_id"
                     tableName="citas">
            <column name="paciente_id"/>
        </createIndex>

        <createIndex indexName="idx_citas_medico_id"
                     tableName="citas">
            <column name="medico_id"/>
        </createIndex>

        <createIndex indexName="idx_citas_fecha_hora"
                     tableName="citas">
            <column name="fecha_hora"/>
        </createIndex>

        <createIndex indexName="idx_citas_estado"
                     tableName="citas">
            <column name="estado"/>
        </createIndex>

        <!-- Índice compuesto: validar disponibilidad de médico en fecha -->
        <createIndex indexName="idx_citas_medico_fecha"
                     tableName="citas">
            <column name="medico_id"/>
            <column name="fecha_hora"/>
            <column name="estado"/>
        </createIndex>

        <!-- Índice compuesto: citas activas de un paciente -->
        <createIndex indexName="idx_citas_paciente_estado"
                     tableName="citas">
            <column name="paciente_id"/>
            <column name="estado"/>
        </createIndex>
    </changeSet>

    <!-- ================================================ -->
    <!-- CHANGESET 003 — Constraint: tipo y estado válidos -->
    <!-- ================================================ -->
    <changeSet id="003-constraints-citas" author="hepaq">

        <sql>
            ALTER TABLE citas
            ADD CONSTRAINT chk_citas_tipo
            CHECK (tipo IN ('CONSULTA_MEDICA', 'LABORATORIO', 'ECOGRAFIA'));
        </sql>

        <sql>
            ALTER TABLE citas
            ADD CONSTRAINT chk_citas_estado
            CHECK (estado IN ('PENDIENTE', 'CONFIRMADA', 'CANCELADA', 'REPROGRAMADA'));
        </sql>
    </changeSet>

    <!-- ================================================ -->
    <!-- CHANGESET 004 — Tabla de auditoría                -->
    <!-- ================================================ -->
    <changeSet id="004-create-table-citas-auditoria" author="hepaq">

        <createTable tableName="citas_auditoria"
                     remarks="Historial de cambios de estado de las citas">

            <column name="id" type="uniqueidentifier">
                <constraints primaryKey="true" nullable="false"/>
            </column>

            <column name="cita_id" type="uniqueidentifier">
                <constraints nullable="false"/>
            </column>

            <column name="estado_anterior" type="varchar(20)"/>
            <column name="estado_nuevo"    type="varchar(20)">
                <constraints nullable="false"/>
            </column>

            <column name="motivo"     type="nvarchar(500)"/>
            <column name="usuario_id" type="varchar(100)"/>
            <column name="timestamp"  type="datetime2(0)"
                    defaultValueComputed="GETUTCDATE()">
                <constraints nullable="false"/>
            </column>
        </createTable>

        <addForeignKeyConstraint
            baseTableName="citas_auditoria"
            baseColumnNames="cita_id"
            constraintName="fk_auditoria_cita_id"
            referencedTableName="citas"
            referencedColumnNames="id"
            onDelete="CASCADE"/>

        <createIndex indexName="idx_citas_auditoria_cita_id"
                     tableName="citas_auditoria">
            <column name="cita_id"/>
        </createIndex>
    </changeSet>

</databaseChangeLog>
```

---

## Read Model — Azure Cosmos DB

**Database:** `hepaq`
**Container:** `citas`
**Partition Key:** `/pacienteId`
**Throughput:** 400 RU/s (autoscale hasta 4000)

### Indexing Policy

```json
{
  "indexingMode": "consistent",
  "automatic": true,
  "includedPaths": [
    { "path": "/pacienteId/?" },
    { "path": "/medicoId/?" },
    { "path": "/estado/?" },
    { "path": "/tipo/?" },
    { "path": "/fechaHora/?" }
  ],
  "excludedPaths": [
    { "path": "/motivoCita/?" },
    { "path": "/motivoCancelacion/?" },
    { "path": "/_etag/?" }
  ]
}
```

### Documento de ejemplo

```json
{
  "id": "550e8400-e29b-41d4-a716-446655440000",
  "pacienteId": "12345678",
  "medicoId": "MED-001",
  "fechaHora": "2026-06-25T09:00:00Z",
  "tipo": "CONSULTA_MEDICA",
  "estado": "CONFIRMADA",
  "motivoCita": "Control de presión arterial",
  "motivoCancelacion": null,

  "paciente": {
    "nombre": "Juan Alberto Pérez López",
    "dni": "12345678",
    "fechaNacimiento": "1975-04-12",
    "telefono": "+51987654321",
    "email": "juan.perez@email.com"
  },

  "medico": {
    "nombre": "Dr. Carlos García Ríos",
    "especialidad": "Cardiología",
    "consultorio": "C-204",
    "telefono": "+51901234567"
  },

  "historialEstados": [
    {
      "estado": "PENDIENTE",
      "timestamp": "2026-06-18T14:30:00Z",
      "usuario": "digitador@hepaq.pe"
    },
    {
      "estado": "CONFIRMADA",
      "timestamp": "2026-06-18T16:00:00Z",
      "usuario": "sistema"
    }
  ],

  "fechaCreacion": "2026-06-18T14:30:00Z",
  "fechaActualizacion": "2026-06-18T16:00:00Z",
  "_ts": 1750262400,
  "_etag": "\"0x8DC8F1A2B3C4D5E6\"",
  "_rid": "abc123==",
  "_self": "dbs/hepaq/colls/citas/docs/550e8400...",
  "_attachments": "attachments/"
}
```
