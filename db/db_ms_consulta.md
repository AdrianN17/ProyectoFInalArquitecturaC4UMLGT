# DB Schema — MS Consultas

## Write Model — Azure SQL (Liquibase Changelog)

**Base de datos:** `hepaq_consultas`
**Tecnología:** Azure SQL Database · Spring Data JPA · Liquibase 4.x

```xml
<?xml version="1.0" encoding="UTF-8"?>
<databaseChangeLog
    xmlns="http://www.liquibase.org/xml/ns/dbchangelog"
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xsi:schemaLocation="http://www.liquibase.org/xml/ns/dbchangelog
        http://www.liquibase.org/xml/ns/dbchangelog/dbchangelog-4.20.xsd">

    <!-- ================================================ -->
    <!-- CHANGESET 001 — Tabla principal: consultas        -->
    <!-- ================================================ -->
    <changeSet id="001-create-table-consultas" author="hepaq">
        <preConditions onFail="MARK_RAN">
            <not><tableExists tableName="consultas"/></not>
        </preConditions>

        <createTable tableName="consultas"
                     remarks="Tabla de consultas/atenciones médicas — Write Model (Commands)">

            <column name="id" type="uniqueidentifier"
                    remarks="Identificador único UUID de la consulta">
                <constraints primaryKey="true" nullable="false"/>
            </column>

            <column name="paciente_id" type="varchar(20)"
                    remarks="DNI o código del paciente">
                <constraints nullable="false"/>
            </column>

            <column name="medico_id" type="varchar(50)"
                    remarks="Código del médico responsable">
                <constraints nullable="false"/>
            </column>

            <column name="cita_id" type="uniqueidentifier"
                    remarks="Referencia a la cita médica que originó esta consulta">
                <constraints nullable="false"/>
            </column>

            <column name="fecha_hora" type="datetime2(0)"
                    remarks="Fecha y hora de la consulta">
                <constraints nullable="false"/>
            </column>

            <column name="tipo" type="varchar(20)"
                    remarks="PRIMERA_VEZ | CONTROL | SEGUIMIENTO | EMERGENCIA">
                <constraints nullable="false"/>
            </column>

            <column name="estado" type="varchar(20)"
                    defaultValue="PENDIENTE"
                    remarks="PENDIENTE | EN_CURSO | COMPLETADA | CANCELADA">
                <constraints nullable="false"/>
            </column>

            <column name="motivo_consulta" type="nvarchar(500)"
                    remarks="Motivo de la consulta referido por el paciente"/>

            <column name="anamnesis" type="nvarchar(max)"
                    remarks="Descripción de síntomas y antecedentes (historia clínica)"/>

            <column name="examen_fisico" type="nvarchar(max)"
                    remarks="Hallazgos del examen físico: PA, FC, temperatura, peso, etc."/>

            <column name="observaciones" type="nvarchar(max)"
                    remarks="Observaciones clínicas del médico"/>

            <column name="tratamiento" type="nvarchar(max)"
                    remarks="Plan de tratamiento indicado por el médico"/>

            <column name="motivo_cancelacion" type="nvarchar(500)"
                    remarks="Razón de cancelación si aplica"/>

            <column name="fecha_creacion" type="datetime2(0)"
                    defaultValueComputed="GETUTCDATE()">
                <constraints nullable="false"/>
            </column>

            <column name="fecha_actualizacion" type="datetime2(0)"
                    defaultValueComputed="GETUTCDATE()">
                <constraints nullable="false"/>
            </column>

            <column name="creado_por" type="varchar(100)"
                    remarks="Usuario o servicio que creó el registro"/>

            <column name="version" type="int" defaultValueNumeric="0">
                <constraints nullable="false"/>
            </column>
        </createTable>
    </changeSet>

    <!-- ================================================ -->
    <!-- CHANGESET 002 — Índices de rendimiento            -->
    <!-- ================================================ -->
    <changeSet id="002-indexes-consultas" author="hepaq">

        <createIndex indexName="idx_consultas_paciente_id"
                     tableName="consultas">
            <column name="paciente_id"/>
        </createIndex>

        <createIndex indexName="idx_consultas_medico_id"
                     tableName="consultas">
            <column name="medico_id"/>
        </createIndex>

        <!-- Unique: una cita solo puede generar una consulta -->
        <createIndex indexName="uidx_consultas_cita_id"
                     tableName="consultas"
                     unique="true">
            <column name="cita_id"/>
        </createIndex>

        <createIndex indexName="idx_consultas_estado"
                     tableName="consultas">
            <column name="estado"/>
        </createIndex>

        <createIndex indexName="idx_consultas_fecha_hora"
                     tableName="consultas">
            <column name="fecha_hora"/>
        </createIndex>

        <!-- Índice compuesto: historia clínica de un paciente por fecha -->
        <createIndex indexName="idx_consultas_paciente_fecha"
                     tableName="consultas">
            <column name="paciente_id"/>
            <column name="fecha_hora" descending="true"/>
        </createIndex>
    </changeSet>

    <!-- ================================================ -->
    <!-- CHANGESET 003 — Constraints de dominio            -->
    <!-- ================================================ -->
    <changeSet id="003-constraints-consultas" author="hepaq">

        <sql>
            ALTER TABLE consultas
            ADD CONSTRAINT chk_consultas_tipo
            CHECK (tipo IN ('PRIMERA_VEZ', 'CONTROL', 'SEGUIMIENTO', 'EMERGENCIA'));
        </sql>

        <sql>
            ALTER TABLE consultas
            ADD CONSTRAINT chk_consultas_estado
            CHECK (estado IN ('PENDIENTE', 'EN_CURSO', 'COMPLETADA', 'CANCELADA'));
        </sql>
    </changeSet>

    <!-- ================================================ -->
    <!-- CHANGESET 004 — Tabla de sígnos vitales           -->
    <!-- ================================================ -->
    <changeSet id="004-create-table-signos-vitales" author="hepaq">

        <createTable tableName="signos_vitales"
                     remarks="Registro de signos vitales por consulta">

            <column name="id" type="uniqueidentifier">
                <constraints primaryKey="true" nullable="false"/>
            </column>

            <column name="consulta_id" type="uniqueidentifier">
                <constraints nullable="false"/>
            </column>

            <column name="presion_sistolica"  type="int"     remarks="mmHg"/>
            <column name="presion_diastolica" type="int"     remarks="mmHg"/>
            <column name="frecuencia_cardiaca" type="int"    remarks="lpm"/>
            <column name="frecuencia_respiratoria" type="int" remarks="rpm"/>
            <column name="temperatura"        type="decimal(4,1)" remarks="°C"/>
            <column name="peso_kg"            type="decimal(5,2)" remarks="kg"/>
            <column name="talla_cm"           type="decimal(5,1)" remarks="cm"/>
            <column name="saturacion_oxigeno" type="decimal(4,1)" remarks="%"/>
            <column name="glucosa_capilar"    type="int"     remarks="mg/dL"/>

            <column name="registrado_por" type="varchar(100)"/>
            <column name="fecha_registro" type="datetime2(0)"
                    defaultValueComputed="GETUTCDATE()">
                <constraints nullable="false"/>
            </column>
        </createTable>

        <addForeignKeyConstraint
            baseTableName="signos_vitales"
            baseColumnNames="consulta_id"
            constraintName="fk_signos_consulta_id"
            referencedTableName="consultas"
            referencedColumnNames="id"
            onDelete="CASCADE"/>
    </changeSet>

</databaseChangeLog>
```

---

## Read Model — Azure Cosmos DB

**Database:** `hepaq`
**Container:** `consultas`
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
    { "path": "/citaId/?" },
    { "path": "/estado/?" },
    { "path": "/tipo/?" },
    { "path": "/fechaHora/?" }
  ],
  "excludedPaths": [
    { "path": "/anamnesis/?" },
    { "path": "/examenFisico/?" },
    { "path": "/tratamiento/?" },
    { "path": "/_etag/?" }
  ],
  "compositeIndexes": [
    [
      { "path": "/pacienteId", "order": "ascending" },
      { "path": "/fechaHora",  "order": "descending" }
    ]
  ]
}
```

### Documento de ejemplo

```json
{
  "id": "550e8400-e29b-41d4-a716-446655440001",
  "pacienteId": "12345678",
  "medicoId": "MED-001",
  "citaId": "550e8400-e29b-41d4-a716-446655440000",
  "fechaHora": "2026-06-25T09:00:00Z",
  "tipo": "CONTROL",
  "estado": "COMPLETADA",
  "motivoConsulta": "Control de presión arterial",

  "paciente": {
    "nombre": "Juan Alberto Pérez López",
    "dni": "12345678",
    "edad": 51,
    "email": "juan.perez@email.com"
  },

  "medico": {
    "nombre": "Dr. Carlos García Ríos",
    "especialidad": "Cardiología",
    "cmp": "45678"
  },

  "signosVitales": {
    "presionSistolica": 130,
    "presionDiastolica": 85,
    "frecuenciaCardiaca": 78,
    "temperatura": 36.8,
    "pesoKg": 82.5,
    "tallaCm": 172.0,
    "saturacionOxigeno": 98.0
  },

  "anamnesis": "Paciente refiere cefalea frontal de 3 días de evolución, sin náuseas. Antecedente de HTA diagnosticada hace 2 años.",
  "examenFisico": "PA 130/85 mmHg, FC 78 lpm, FR 16 rpm, T° 36.8°C. Consciente, orientado. Sin edemas.",
  "observaciones": "Hipertensión arterial estadio I con adecuado control parcial. Requiere ajuste de medicación.",
  "tratamiento": "Enalapril 10mg c/24h (aumentar dosis). Dieta hiposódica. Control en 4 semanas.",

  "diagnosticos": [
    {
      "codigoCIE": "I10",
      "descripcion": "Hipertensión esencial (primaria)",
      "tipo": "DEFINITIVO"
    }
  ],

  "examenesSolicitados": [
    {
      "examenId": "770e8400-e29b-41d4-a716-446655440003",
      "tipo": "LABORATORIO",
      "descripcion": "Hemograma completo + perfil lipídico"
    }
  ],

  "fechaCreacion": "2026-06-25T09:00:00Z",
  "fechaActualizacion": "2026-06-25T10:30:00Z",
  "_ts": 1750868400,
  "_etag": "\"0x8DC9A2B3C4D5E6F7\""
}
```
