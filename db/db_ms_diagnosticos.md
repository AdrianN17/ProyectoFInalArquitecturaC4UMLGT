# DB Schema — MS Diagnósticos

## Write Model — Azure SQL (Liquibase Changelog)

**Base de datos:** `hepaq_diagnosticos`
**Tecnología:** Azure SQL Database · Spring Data JPA · Liquibase 4.x

```xml
<?xml version="1.0" encoding="UTF-8"?>
<databaseChangeLog
    xmlns="http://www.liquibase.org/xml/ns/dbchangelog"
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xsi:schemaLocation="http://www.liquibase.org/xml/ns/dbchangelog
        http://www.liquibase.org/xml/ns/dbchangelog/dbchangelog-4.20.xsd">

    <!-- ================================================ -->
    <!-- CHANGESET 001 — Tabla principal: diagnosticos     -->
    <!-- ================================================ -->
    <changeSet id="001-create-table-diagnosticos" author="hepaq">
        <preConditions onFail="MARK_RAN">
            <not><tableExists tableName="diagnosticos"/></not>
        </preConditions>

        <createTable tableName="diagnosticos"
                     remarks="Tabla de diagnósticos médicos — Write Model (Commands)">

            <column name="id" type="uniqueidentifier"
                    remarks="Identificador único UUID del diagnóstico">
                <constraints primaryKey="true" nullable="false"/>
            </column>

            <column name="paciente_id" type="varchar(20)"
                    remarks="DNI o código del paciente">
                <constraints nullable="false"/>
            </column>

            <column name="medico_id" type="varchar(50)"
                    remarks="Código del médico que emite el diagnóstico">
                <constraints nullable="false"/>
            </column>

            <column name="consulta_id" type="uniqueidentifier"
                    remarks="Consulta médica asociada al diagnóstico">
                <constraints nullable="false"/>
            </column>

            <column name="codigo_cie" type="varchar(10)"
                    remarks="Código del catálogo CIE-10 (ej. I10, J45.0, E11)">
                <constraints nullable="false"/>
            </column>

            <column name="descripcion" type="nvarchar(500)"
                    remarks="Descripción libre del diagnóstico (complementa el código CIE)"/>

            <column name="tipo" type="varchar(20)"
                    remarks="PRESUNTIVO | DEFINITIVO | DIFERENCIAL | SECUNDARIO">
                <constraints nullable="false"/>
            </column>

            <column name="estado" type="varchar(20)"
                    defaultValue="ACTIVO"
                    remarks="ACTIVO | RESUELTO | CRONICO | ELIMINADO">
                <constraints nullable="false"/>
            </column>

            <column name="notas_medicas" type="nvarchar(max)"
                    remarks="Notas clínicas adicionales del médico"/>

            <column name="tratamiento_sugerido" type="nvarchar(max)"
                    remarks="Plan de tratamiento sugerido para este diagnóstico"/>

            <column name="motivo_eliminacion" type="nvarchar(500)"
                    remarks="Motivo del soft delete"/>

            <column name="fecha_registro" type="datetime2(0)"
                    defaultValueComputed="GETUTCDATE()">
                <constraints nullable="false"/>
            </column>

            <column name="fecha_actualizacion" type="datetime2(0)"
                    defaultValueComputed="GETUTCDATE()">
                <constraints nullable="false"/>
            </column>

            <column name="eliminado_por" type="varchar(100)"
                    remarks="Usuario que realizó la eliminación lógica"/>

            <column name="version" type="int" defaultValueNumeric="0">
                <constraints nullable="false"/>
            </column>
        </createTable>
    </changeSet>

    <!-- ================================================ -->
    <!-- CHANGESET 002 — Índices de rendimiento            -->
    <!-- ================================================ -->
    <changeSet id="002-indexes-diagnosticos" author="hepaq">

        <createIndex indexName="idx_diagnosticos_paciente_id"
                     tableName="diagnosticos">
            <column name="paciente_id"/>
        </createIndex>

        <createIndex indexName="idx_diagnosticos_consulta_id"
                     tableName="diagnosticos">
            <column name="consulta_id"/>
        </createIndex>

        <createIndex indexName="idx_diagnosticos_codigo_cie"
                     tableName="diagnosticos">
            <column name="codigo_cie"/>
        </createIndex>

        <createIndex indexName="idx_diagnosticos_estado"
                     tableName="diagnosticos">
            <column name="estado"/>
        </createIndex>

        <!-- Historia de diagnósticos activos de un paciente -->
        <createIndex indexName="idx_diagnosticos_paciente_estado"
                     tableName="diagnosticos">
            <column name="paciente_id"/>
            <column name="estado"/>
            <column name="fecha_registro" descending="true"/>
        </createIndex>

        <!-- Diagnósticos por consulta y tipo -->
        <createIndex indexName="idx_diagnosticos_consulta_tipo"
                     tableName="diagnosticos">
            <column name="consulta_id"/>
            <column name="tipo"/>
        </createIndex>
    </changeSet>

    <!-- ================================================ -->
    <!-- CHANGESET 003 — Constraints de dominio            -->
    <!-- ================================================ -->
    <changeSet id="003-constraints-diagnosticos" author="hepaq">

        <sql>
            ALTER TABLE diagnosticos
            ADD CONSTRAINT chk_diagnosticos_tipo
            CHECK (tipo IN ('PRESUNTIVO', 'DEFINITIVO', 'DIFERENCIAL', 'SECUNDARIO'));
        </sql>

        <sql>
            ALTER TABLE diagnosticos
            ADD CONSTRAINT chk_diagnosticos_estado
            CHECK (estado IN ('ACTIVO', 'RESUELTO', 'CRONICO', 'ELIMINADO'));
        </sql>

        <!-- Validar formato básico del código CIE (letra + dígitos) -->
        <sql>
            ALTER TABLE diagnosticos
            ADD CONSTRAINT chk_diagnosticos_codigo_cie
            CHECK (codigo_cie LIKE '[A-Z][0-9]%');
        </sql>
    </changeSet>

    <!-- ================================================ -->
    <!-- CHANGESET 004 — Tabla de auditoría                -->
    <!-- ================================================ -->
    <changeSet id="004-create-table-diagnosticos-auditoria" author="hepaq">

        <createTable tableName="diagnosticos_auditoria"
                     remarks="Historial de cambios de estado de diagnósticos">

            <column name="id" type="uniqueidentifier">
                <constraints primaryKey="true" nullable="false"/>
            </column>

            <column name="diagnostico_id" type="uniqueidentifier">
                <constraints nullable="false"/>
            </column>

            <column name="estado_anterior" type="varchar(20)"/>
            <column name="estado_nuevo"    type="varchar(20)">
                <constraints nullable="false"/>
            </column>

            <column name="accion"       type="varchar(50)"
                    remarks="INSERT | UPDATE_ESTADO | SOFT_DELETE"/>
            <column name="usuario_id"   type="varchar(100)"/>
            <column name="timestamp"    type="datetime2(0)"
                    defaultValueComputed="GETUTCDATE()">
                <constraints nullable="false"/>
            </column>
        </createTable>

        <addForeignKeyConstraint
            baseTableName="diagnosticos_auditoria"
            baseColumnNames="diagnostico_id"
            constraintName="fk_diag_auditoria_diagnostico_id"
            referencedTableName="diagnosticos"
            referencedColumnNames="id"
            onDelete="CASCADE"/>
    </changeSet>

</databaseChangeLog>
```

---

## Catálogo CIE-10 — Azure Table Storage

**Storage Account:** `hepaqstorage`
**Table:** `CatalogoClE10`
**PartitionKey:** Categoría CIE (letra, ej. `"I"`, `"J"`, `"E"`)
**RowKey:** Código completo CIE (ej. `"I10"`, `"J45"`, `"E11"`)

### Entidad de ejemplo

```json
{
  "PartitionKey": "I",
  "RowKey": "I10",
  "Timestamp": "2026-01-01T00:00:00Z",
  "Descripcion": "Hipertensión esencial (primaria)",
  "DescripcionLarga": "Hipertensión arterial primaria o idiopática, sin causa secundaria identificable",
  "Categoria": "I",
  "Subcategoria": "I10-I15",
  "CodigoParent": "I",
  "EsTerminal": true,
  "Activo": true,
  "Version": "CIE-10-ES-2026"
}
```

---

## Read Model — Azure Cosmos DB

**Database:** `hepaq`
**Container:** `diagnosticos`
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
    { "path": "/consultaId/?" },
    { "path": "/codigoCIE/?" },
    { "path": "/tipo/?" },
    { "path": "/estado/?" },
    { "path": "/fechaRegistro/?" }
  ],
  "excludedPaths": [
    { "path": "/notasMedicas/?" },
    { "path": "/tratamientoSugerido/?" },
    { "path": "/_etag/?" }
  ],
  "compositeIndexes": [
    [
      { "path": "/pacienteId", "order": "ascending" },
      { "path": "/estado",     "order": "ascending" },
      { "path": "/fechaRegistro", "order": "descending" }
    ]
  ]
}
```

### Documento de ejemplo

```json
{
  "id": "660e8400-e29b-41d4-a716-446655440002",
  "pacienteId": "12345678",
  "medicoId": "MED-001",
  "consultaId": "550e8400-e29b-41d4-a716-446655440001",
  "codigoCIE": "I10",
  "estado": "ACTIVO",
  "tipo": "DEFINITIVO",
  "descripcion": "Hipertensión esencial (primaria)",

  "cie": {
    "codigo": "I10",
    "descripcion": "Hipertensión esencial (primaria)",
    "categoria": "I",
    "subcategoria": "I10-I15"
  },

  "paciente": {
    "nombre": "Juan Alberto Pérez López",
    "dni": "12345678",
    "edad": 51
  },

  "medico": {
    "nombre": "Dr. Carlos García Ríos",
    "especialidad": "Cardiología",
    "cmp": "45678"
  },

  "notas_medicas": "Paciente con antecedente familiar de HTA paterna. Sin nefropatía asociada.",
  "tratamiento_sugerido": "Enalapril 10mg c/24h. Dieta hiposódica < 2g/día. Control en 4 semanas.",

  "historialEstados": [
    {
      "estado": "ACTIVO",
      "timestamp": "2026-06-25T10:30:00Z",
      "usuario": "MED-001"
    }
  ],

  "fechaRegistro": "2026-06-25T10:30:00Z",
  "fechaActualizacion": "2026-06-25T10:30:00Z",
  "_ts": 1750869000,
  "_etag": "\"0x8DCB1A2B3C4D5E6F\""
}
```
