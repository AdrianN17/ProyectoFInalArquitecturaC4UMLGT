# DB Schema — MS Exámenes

## Write Model — Azure SQL (Liquibase Changelog)

**Base de datos:** `hepaq_examenes`
**Tecnología:** Azure SQL Database · Spring Data JPA · Liquibase 4.x

```xml
<?xml version="1.0" encoding="UTF-8"?>
<databaseChangeLog
    xmlns="http://www.liquibase.org/xml/ns/dbchangelog"
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xsi:schemaLocation="http://www.liquibase.org/xml/ns/dbchangelog
        http://www.liquibase.org/xml/ns/dbchangelog/dbchangelog-4.20.xsd">

    <!-- ================================================ -->
    <!-- CHANGESET 001 — Tabla principal: examenes         -->
    <!-- ================================================ -->
    <changeSet id="001-create-table-examenes" author="hepaq">
        <preConditions onFail="MARK_RAN">
            <not><tableExists tableName="examenes"/></not>
        </preConditions>

        <createTable tableName="examenes"
                     remarks="Tabla de exámenes médicos — Write Model (Commands)">

            <column name="id" type="uniqueidentifier"
                    remarks="Identificador único UUID del examen">
                <constraints primaryKey="true" nullable="false"/>
            </column>

            <column name="paciente_id" type="varchar(20)"
                    remarks="DNI o código del paciente">
                <constraints nullable="false"/>
            </column>

            <column name="medico_id" type="varchar(50)"
                    remarks="Código del médico que solicita el examen">
                <constraints nullable="false"/>
            </column>

            <column name="consulta_id" type="uniqueidentifier"
                    remarks="Consulta médica que generó la solicitud del examen">
                <constraints nullable="false"/>
            </column>

            <column name="tipo" type="varchar(20)"
                    remarks="LABORATORIO | IMAGEN | ECOGRAFIA | RAYOS_X | TOMOGRAFIA | RESONANCIA">
                <constraints nullable="false"/>
            </column>

            <column name="estado" type="varchar(20)"
                    defaultValue="PENDIENTE"
                    remarks="PENDIENTE | EN_PROCESO | COMPLETADO | CANCELADO | ELIMINADO">
                <constraints nullable="false"/>
            </column>

            <column name="descripcion" type="nvarchar(500)"
                    remarks="Nombre o descripción del examen solicitado"/>

            <column name="indicaciones_previas" type="nvarchar(1000)"
                    remarks="Instrucciones al paciente: ayuno, preparación, etc."/>

            <column name="resultados" type="nvarchar(max)"
                    remarks="Resultados del examen en texto libre"/>

            <column name="interpretacion" type="nvarchar(max)"
                    remarks="Interpretación clínica del especialista"/>

            <column name="motivo_cancelacion" type="nvarchar(500)"
                    remarks="Motivo de cancelación si aplica"/>

            <column name="motivo_eliminacion" type="nvarchar(500)"
                    remarks="Motivo del soft delete"/>

            <column name="fecha_solicitud" type="datetime2(0)"
                    defaultValueComputed="GETUTCDATE()"
                    remarks="Fecha en que se solicitó el examen">
                <constraints nullable="false"/>
            </column>

            <column name="fecha_resultado" type="datetime2(0)"
                    remarks="Fecha en que se registraron los resultados (nullable)"/>

            <column name="fecha_creacion" type="datetime2(0)"
                    defaultValueComputed="GETUTCDATE()">
                <constraints nullable="false"/>
            </column>

            <column name="fecha_actualizacion" type="datetime2(0)"
                    defaultValueComputed="GETUTCDATE()">
                <constraints nullable="false"/>
            </column>

            <column name="version" type="int" defaultValueNumeric="0">
                <constraints nullable="false"/>
            </column>
        </createTable>
    </changeSet>

    <!-- ================================================ -->
    <!-- CHANGESET 002 — Tabla de adjuntos                 -->
    <!-- ================================================ -->
    <changeSet id="002-create-table-adjuntos-examen" author="hepaq">
        <preConditions onFail="MARK_RAN">
            <not><tableExists tableName="adjuntos_examen"/></not>
        </preConditions>

        <createTable tableName="adjuntos_examen"
                     remarks="Metadatos de archivos adjuntos almacenados en Azure Blob Storage">

            <column name="id" type="uniqueidentifier">
                <constraints primaryKey="true" nullable="false"/>
            </column>

            <column name="examen_id" type="uniqueidentifier">
                <constraints nullable="false"/>
            </column>

            <column name="nombre_archivo" type="varchar(500)"
                    remarks="Nombre original del archivo subido">
                <constraints nullable="false"/>
            </column>

            <column name="url_blob" type="varchar(2000)"
                    remarks="URL completa del blob en Azure Blob Storage">
                <constraints nullable="false"/>
            </column>

            <column name="content_type" type="varchar(100)"
                    remarks="MIME type: image/jpeg, image/png, application/pdf, application/dicom">
                <constraints nullable="false"/>
            </column>

            <column name="tamanio_bytes" type="bigint"
                    remarks="Tamaño del archivo en bytes">
                <constraints nullable="false"/>
            </column>

            <column name="container_blob" type="varchar(200)"
                    remarks="Nombre del container en Azure Blob Storage"
                    defaultValue="examenes-adjuntos"/>

            <column name="subido_por" type="varchar(100)"
                    remarks="Usuario o servicio que subió el archivo"/>

            <column name="activo" type="bit" defaultValueBoolean="true"
                    remarks="Soft delete — false indica que fue eliminado">
                <constraints nullable="false"/>
            </column>

            <column name="fecha_subida" type="datetime2(0)"
                    defaultValueComputed="GETUTCDATE()">
                <constraints nullable="false"/>
            </column>
        </createTable>

        <addForeignKeyConstraint
            baseTableName="adjuntos_examen"
            baseColumnNames="examen_id"
            constraintName="fk_adjuntos_examen_id"
            referencedTableName="examenes"
            referencedColumnNames="id"
            onDelete="CASCADE"/>
    </changeSet>

    <!-- ================================================ -->
    <!-- CHANGESET 003 — Índices de rendimiento            -->
    <!-- ================================================ -->
    <changeSet id="003-indexes-examenes" author="hepaq">

        <createIndex indexName="idx_examenes_paciente_id"
                     tableName="examenes">
            <column name="paciente_id"/>
        </createIndex>

        <createIndex indexName="idx_examenes_consulta_id"
                     tableName="examenes">
            <column name="consulta_id"/>
        </createIndex>

        <createIndex indexName="idx_examenes_estado"
                     tableName="examenes">
            <column name="estado"/>
        </createIndex>

        <createIndex indexName="idx_examenes_tipo"
                     tableName="examenes">
            <column name="tipo"/>
        </createIndex>

        <!-- Exámenes pendientes de resultado por paciente -->
        <createIndex indexName="idx_examenes_paciente_estado_tipo"
                     tableName="examenes">
            <column name="paciente_id"/>
            <column name="estado"/>
            <column name="tipo"/>
            <column name="fecha_solicitud" descending="true"/>
        </createIndex>

        <!-- Adjuntos activos de un examen -->
        <createIndex indexName="idx_adjuntos_examen_id_activo"
                     tableName="adjuntos_examen">
            <column name="examen_id"/>
            <column name="activo"/>
        </createIndex>
    </changeSet>

    <!-- ================================================ -->
    <!-- CHANGESET 004 — Constraints de dominio            -->
    <!-- ================================================ -->
    <changeSet id="004-constraints-examenes" author="hepaq">

        <sql>
            ALTER TABLE examenes
            ADD CONSTRAINT chk_examenes_tipo
            CHECK (tipo IN ('LABORATORIO', 'IMAGEN', 'ECOGRAFIA', 'RAYOS_X', 'TOMOGRAFIA', 'RESONANCIA'));
        </sql>

        <sql>
            ALTER TABLE examenes
            ADD CONSTRAINT chk_examenes_estado
            CHECK (estado IN ('PENDIENTE', 'EN_PROCESO', 'COMPLETADO', 'CANCELADO', 'ELIMINADO'));
        </sql>

        <sql>
            ALTER TABLE adjuntos_examen
            ADD CONSTRAINT chk_adjuntos_content_type
            CHECK (content_type IN (
                'image/jpeg', 'image/png', 'image/gif', 'image/webp',
                'application/pdf', 'application/dicom', 'video/mp4'
            ));
        </sql>

        <sql>
            ALTER TABLE adjuntos_examen
            ADD CONSTRAINT chk_adjuntos_tamanio
            CHECK (tamanio_bytes > 0 AND tamanio_bytes &lt;= 52428800); -- 50 MB máximo
        </sql>
    </changeSet>

</databaseChangeLog>
```

---

## Read Model — Azure Cosmos DB

**Database:** `hepaq`
**Container:** `examenes`
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
    { "path": "/tipo/?" },
    { "path": "/estado/?" },
    { "path": "/fechaSolicitud/?" },
    { "path": "/adjuntos/[]/id/?" }
  ],
  "excludedPaths": [
    { "path": "/resultados/?" },
    { "path": "/interpretacion/?" },
    { "path": "/indicacionesPrevias/?" },
    { "path": "/_etag/?" }
  ],
  "compositeIndexes": [
    [
      { "path": "/pacienteId",    "order": "ascending" },
      { "path": "/tipo",          "order": "ascending" },
      { "path": "/fechaSolicitud","order": "descending" }
    ]
  ]
}
```

### Documento de ejemplo

```json
{
  "id": "770e8400-e29b-41d4-a716-446655440003",
  "pacienteId": "12345678",
  "medicoId": "MED-001",
  "consultaId": "550e8400-e29b-41d4-a716-446655440001",
  "tipo": "LABORATORIO",
  "estado": "COMPLETADO",
  "descripcion": "Hemograma completo + perfil lipídico",
  "indicacionesPrevias": "Ayuno de 8 horas. No tomar medicamentos antes de la prueba.",

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

  "resultados": "Hemoglobina: 14.2 g/dL | Hematocrito: 42% | Leucocitos: 7200/mm³ | Plaquetas: 245000/mm³ | Colesterol Total: 210 mg/dL | HDL: 52 mg/dL | LDL: 138 mg/dL | Triglicéridos: 165 mg/dL",
  "interpretacion": "Hemograma dentro de rangos normales. Dislipidemia leve con LDL y triglicéridos levemente elevados. Se recomienda control dietético y evaluación cardiológica.",

  "adjuntos": [
    {
      "id": "880e8400-e29b-41d4-a716-446655440004",
      "nombreArchivo": "hemograma_12345678_20260625.pdf",
      "contentType": "application/pdf",
      "tamanioBytes": 204800,
      "containerBlob": "examenes-adjuntos",
      "urlBlob": "https://hepaqstorage.blob.core.windows.net/examenes-adjuntos/770e8400/hemograma_12345678_20260625.pdf",
      "subidoPor": "LAB-TECNICO-01",
      "fechaSubida": "2026-06-26T14:00:00Z"
    },
    {
      "id": "881e8400-e29b-41d4-a716-446655440005",
      "nombreArchivo": "perfil_lipidico_12345678_20260625.pdf",
      "contentType": "application/pdf",
      "tamanioBytes": 180224,
      "containerBlob": "examenes-adjuntos",
      "urlBlob": "https://hepaqstorage.blob.core.windows.net/examenes-adjuntos/770e8400/perfil_lipidico_12345678_20260625.pdf",
      "subidoPor": "LAB-TECNICO-01",
      "fechaSubida": "2026-06-26T14:00:00Z"
    }
  ],

  "totalAdjuntos": 2,
  "fechaSolicitud": "2026-06-25T10:30:00Z",
  "fechaResultado": "2026-06-26T14:00:00Z",
  "fechaCreacion": "2026-06-25T10:30:00Z",
  "fechaActualizacion": "2026-06-26T14:05:00Z",
  "_ts": 1750957200,
  "_etag": "\"0x8DCC2B3C4D5E6F7A\""
}
```
