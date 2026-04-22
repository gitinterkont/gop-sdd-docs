# API Contract — RQF_GOP_25: Registro de Pozo Antiguo / Perforado

**Versión:** 1.0
**Fecha:** Abril 2026
**Feature:** RQF_GOP_25_RegistroPozoAntiguo
**Estado:** Propuesta — pendiente aprobación del arquitecto humano
**Complemento:** `openapi.yaml` (especificación validable OpenAPI 3.1)

---

## 1. Resumen

Este contrato define la interfaz HTTP entre frontend y backend para la funcionalidad de **Registro de Pozo Antiguo/Perforado** en GOP 360°. Cubre el ciclo completo: creación de borrador, edición, envío a aprobación ANH, aprobación/devolución, eliminación, generación de PDF y gestión de adjuntos de registros eléctricos.

**Recurso principal:** `legacy-well-registrations`
**Base path:** `/api/v1`

---

## 2. Actores y Autorización por Endpoint

| Rol | Código interno | Contexto | Endpoints autorizados |
|-----|---------------|----------|----------------------|
| Agente Operadora | `OperatorAgent` | Operadora | Crear, Editar, Consultar, Eliminar, Enviar, Subir adjuntos |
| Coordinador Operadora | `OperatorCoordinator` | Operadora | Editar (Borrador/Devuelto), Consultar |
| Ingeniero ANH | `AnhOperationsEngineer` | ANH | Consultar, Aprobar, Devolver |
| Admin GOP ANH | `AnhGopAdministrator` | ANH | Todos (acceso total) |

---

## 3. Tabla de Endpoints

| # | Método | Ruta | Descripción | Roles | HTTP Éxito |
|---|--------|------|-------------|-------|------------|
| 1 | `POST` | `/legacy-well-registrations` | Crear borrador de registro | `OperatorAgent`, `AnhGopAdministrator` | `201` |
| 2 | `GET` | `/legacy-well-registrations` | Listar registros (paginado, filtrado) | Todos los roles | `200` |
| 3 | `GET` | `/legacy-well-registrations/{id}` | Obtener detalle de un registro | Todos los roles | `200` |
| 4 | `PUT` | `/legacy-well-registrations/{id}` | Actualizar registro (Borrador/Devuelto) | `OperatorAgent`, `OperatorCoordinator`, `AnhGopAdministrator` | `200` |
| 5 | `DELETE` | `/legacy-well-registrations/{id}` | Eliminar registro (Borrador/Devuelto) | `OperatorAgent`, `AnhGopAdministrator` | `204` |
| 6 | `POST` | `/legacy-well-registrations/{id}/submit` | Enviar a aprobación ANH | `OperatorAgent`, `AnhGopAdministrator` | `200` |
| 7 | `POST` | `/legacy-well-registrations/{id}/approve` | Aprobar registro (ANH) | `AnhOperationsEngineer`, `AnhGopAdministrator` | `200` |
| 8 | `POST` | `/legacy-well-registrations/{id}/return` | Devolver con observaciones (ANH) | `AnhOperationsEngineer`, `AnhGopAdministrator` | `200` |
| 9 | `GET` | `/legacy-well-registrations/{id}/pdf` | Descargar PDF del registro | Todos los roles | `200` |
| 10 | `POST` | `/legacy-well-registrations/{id}/log-attachments` | Subir adjunto de registro eléctrico | `OperatorAgent`, `OperatorCoordinator`, `AnhGopAdministrator` | `201` |
| 11 | `GET` | `/legacy-well-registrations/{id}/log-attachments/{attachmentId}` | Descargar adjunto | Todos los roles | `200` |
| 12 | `DELETE` | `/legacy-well-registrations/{id}/log-attachments/{attachmentId}` | Eliminar adjunto | `OperatorAgent`, `OperatorCoordinator`, `AnhGopAdministrator` | `204` |
| 13 | `POST` | `/legacy-well-registrations/preview-uwi` | Previsualizar UWI antes de guardar | `OperatorAgent`, `AnhGopAdministrator` | `200` |

**Endpoints de referencia (compartidos, definidos en otros contratos):**

| Método | Ruta | Propósito en esta feature |
|--------|------|---------------------------|
| `GET` | `/contracts` | Lista de contratos por operadora (Sección 1.1) |
| `GET` | `/operators` | Lista de operadores históricos (Sección 1.2) |
| `GET` | `/fields` | Campos por contrato/operadora (Sección 2) |
| `GET` | `/cluster-locations` | Clusters por contrato (Sección 3) |
| `POST` | `/cluster-locations` | Crear cluster nuevo (RN-12) |
| `GET` | `/wells` | Pozos del contrato para selección de padre (RN-05) |
| `GET` | `/catalogs/{code}/items` | Ítems de catálogos cerrados |
| `GET` | `/political-divisions/departments` | Departamentos DANE |
| `GET` | `/political-divisions/departments/{code}/municipalities` | Municipios por departamento |
| `GET` | `/coordinate-reference-systems` | Sistemas de referencia disponibles |

---

## 4. Detalle por Endpoint

### 4.1 `POST /api/v1/legacy-well-registrations` — Crear borrador

**Propósito:** Crea un nuevo registro de pozo antiguo en estado `Draft`. Genera el UWI fiscalizado automáticamente. No requiere que todos los campos obligatorios estén completos (es borrador).

**Autenticación:** Bearer JWT
**Roles:** `OperatorAgent`, `AnhGopAdministrator`
**Idempotencia:** No idempotente. Cada invocación crea un nuevo registro.

**Request Body:** `application/json`

```json
{
  "currentContract": {
    "contractId": "3fa85f64-5717-4562-b3fc-2c963f66afa6"
  },
  "historicalContract": {
    "initialOperatorId": "7fa85f64-5717-4562-b3fc-2c963f66afa6",
    "initialOperatorFreeText": null,
    "initialContractName": "Contrato Asociación Rubiales",
    "initialContractTypeCode": "ASSOCIATION"
  },
  "technicalDetails": {
    "trajectoryTypeCode": "O",
    "parentWellId": null,
    "hasForm103": true,
    "wellName": "RUBIALES 157",
    "avmWellId": null,
    "laheeClassificationCode": "B3",
    "finalClassificationCode": null,
    "fieldId": "9fa85f64-5717-4562-b3fc-2c963f66afa6",
    "fieldNameFreeText": null,
    "angleTypeCode": "V",
    "form103ObjectiveCode": "PH",
    "currentWellStateCode": "ACTIVE",
    "currentObjectiveCode": "PH",
    "completionTypeCode": "CC"
  },
  "geographicLocation": {
    "departmentDaneCode": "50",
    "municipalityDaneCode": "568",
    "clusterLocationId": "1fa85f64-5717-4562-b3fc-2c963f66afa6"
  },
  "sgcUwi": "SGC-RUB-157-2008",
  "initialOriginCoordinates": {
    "datumType": "MAGNA_SIRGAS",
    "originCode": "EAST_CENTRAL",
    "surfaceNorthing": 855230.45,
    "surfaceEasting": 1143567.89,
    "bottomNorthing": 855230.45,
    "bottomEasting": 1143568.12
  },
  "nationalOriginCoordinates": {
    "surfaceNorthing": 2055230.45,
    "surfaceEasting": 5143567.89,
    "bottomNorthing": 2055230.45,
    "bottomEasting": 5143568.12
  },
  "geographicCoordinates": {
    "surfaceLatitude": 3.850000,
    "surfaceLongitude": -73.750000,
    "bottomLatitude": 3.850001,
    "bottomLongitude": -73.750001
  },
  "physicalInfo": {
    "trapTypeCode": "STRUCTURAL",
    "rotaryTableElevationFeet": 550.00,
    "groundElevationFeet": 540.00,
    "waterDepthMeters": null,
    "coastDistanceKm": null,
    "totalDepthMdFeet": 8500.00,
    "totalDepthTvdFeet": 8350.00,
    "totalDepthTvdssFeet": 7810.00,
    "drillingStartDate": "2008-03-15",
    "drillingEndDate": "2008-06-20",
    "completionDate": "2008-07-15",
    "form103ApprovalDate": "2008-08-01",
    "observations": "Pozo completado con producción estable desde 2008."
  },
  "completionIntervals": {
    "completionDescription": "PERFORATION",
    "completionDescriptionOther": null,
    "intervals": [
      {
        "fromFeet": 7800.00,
        "toFeet": 7825.00,
        "shotsPerFoot": 10,
        "status": "OPEN"
      },
      {
        "fromFeet": 7850.00,
        "toFeet": 7870.00,
        "shotsPerFoot": 10,
        "status": "OPEN"
      }
    ]
  },
  "productionTest": {
    "testDate": "2008-07-20",
    "oilRateBblPerDay": 1250.50,
    "gasRateKscfPerDay": 3200.00,
    "waterRateBblPerDay": 120.00,
    "wellheadPressurePsia": 450.00,
    "apiGravity": 32.5,
    "gorKscfPerBbl": 2.56,
    "bswPercent": 8.76
  },
  "injectivityTest": null,
  "monitoringTest": null,
  "formationsFound": [
    {
      "formationName": "Mirador",
      "topMdFeet": 6200.00,
      "topTvdFeet": 6100.00,
      "topTvdssFeet": 5560.00
    },
    {
      "formationName": "Barco",
      "topMdFeet": 7200.00,
      "topTvdFeet": 7100.00,
      "topTvdssFeet": 6560.00
    }
  ],
  "reservoirFormationName": "Mirador",
  "casingsPlaced": [
    {
      "holeDiameterCode": "26",
      "casingSizeCode": "20",
      "gradeCode": "K55",
      "weightLbsPerFtCode": "94",
      "shoeDepthFeet": 500.00,
      "cementTopFeet": 0.00
    },
    {
      "holeDiameterCode": "17.5",
      "casingSizeCode": "13.375",
      "gradeCode": "N80",
      "weightLbsPerFtCode": "68",
      "shoeDepthFeet": 3500.00,
      "cementTopFeet": 0.00
    }
  ],
  "logsRun": [
    {
      "logTypeCode": "GR",
      "initialDepthFeet": 500.00,
      "finalDepthFeet": 8500.00,
      "scale": 200
    }
  ],
  "freshwaterSands": [
    {
      "number": 1,
      "topFeet": 200.00,
      "baseFeet": 450.00
    }
  ]
}
```

**Response `201 Created`:**

```json
{
  "id": "00000000-0000-0000-0000-000000000001",
  "consecutiveNumber": "FOPA-RUBI-0425-2026-001",
  "status": "DRAFT",
  "fiscalizedUwi": "50568RUBI0157RUBIVOOPH–CC",
  "operatorId": "2fa85f64-5717-4562-b3fc-2c963f66afa6",
  "operatorName": "Ecopetrol S.A.",
  "createdAt": "2026-04-21T14:30:00Z",
  "createdBy": "john.doe@example.com",
  "updatedAt": "2026-04-21T14:30:00Z"
}
```

**Errores posibles:**

| HTTP | Código error | Condición |
|------|-------------|-----------|
| `400` | `VALIDATION_ERROR` | Payload malformado o tipos inválidos |
| `401` | `UNAUTHORIZED` | Token ausente o inválido |
| `403` | `FORBIDDEN` | Rol sin permiso de creación |
| `409` | `DUPLICATE_WELL_NAME` | Nombre del pozo ya existe en el sistema |
| `409` | `DUPLICATE_UWI` | UWI fiscalizado generado ya existe |
| `422` | `BUSINESS_RULE_VIOLATION` | Violación de regla de negocio (ej: contrato no pertenece al operador) |

---

### 4.2 `GET /api/v1/legacy-well-registrations` — Listar registros

**Propósito:** Lista registros de pozo antiguo con paginación, ordenamiento y filtros. Filtrado automático por tenant (operadora del usuario) salvo roles ANH.

**Autenticación:** Bearer JWT
**Roles:** Todos

**Query Parameters:**

| Parámetro | Tipo | Requerido | Default | Descripción |
|-----------|------|-----------|---------|-------------|
| `page` | `integer` | No | `1` | Página actual |
| `pageSize` | `integer` | No | `20` | Tamaño de página (max: 100) |
| `sort` | `string` | No | `createdAt:desc` | Campo y dirección de ordenamiento |
| `status` | `string` | No | — | Filtro por estado: `DRAFT`, `SUBMITTED`, `UNDER_REVIEW`, `APPROVED`, `RETURNED` |
| `contractId` | `uuid` | No | — | Filtro por contrato |
| `wellName` | `string` | No | — | Búsqueda parcial por nombre de pozo |
| `fiscalizedUwi` | `string` | No | — | Búsqueda parcial por UWI |
| `createdFrom` | `date` | No | — | Fecha de creación desde (ISO 8601) |
| `createdTo` | `date` | No | — | Fecha de creación hasta (ISO 8601) |

**Response `200 OK`:**

```json
{
  "items": [
    {
      "id": "00000000-0000-0000-0000-000000000001",
      "consecutiveNumber": "FOPA-RUBI-0425-2026-001",
      "status": "DRAFT",
      "wellName": "RUBIALES 157",
      "fiscalizedUwi": "50568RUBI0157RUBIVOOPH–CC",
      "contractCode": "E&P RUBIALES",
      "operatorName": "Ecopetrol S.A.",
      "fieldName": "RUBIALES",
      "createdAt": "2026-04-21T14:30:00Z",
      "createdBy": "john.doe@example.com",
      "updatedAt": "2026-04-21T14:30:00Z"
    }
  ],
  "page": 1,
  "pageSize": 20,
  "total": 1,
  "totalPages": 1
}
```

---

### 4.3 `GET /api/v1/legacy-well-registrations/{id}` — Obtener detalle

**Propósito:** Retorna el registro completo con todas las secciones, tablas dinámicas, campos calculados, historial de observaciones y adjuntos.

**Autenticación:** Bearer JWT
**Roles:** Todos

**Path Parameters:** `id` — UUID del registro.

**Response `200 OK`:**

```json
{
  "id": "00000000-0000-0000-0000-000000000001",
  "consecutiveNumber": "FOPA-RUBI-0425-2026-001",
  "status": "DRAFT",
  "fiscalizedUwi": "50568RUBI0157RUBIVOOPH–CC",
  "operatorId": "2fa85f64-5717-4562-b3fc-2c963f66afa6",
  "operatorName": "Ecopetrol S.A.",
  "currentContract": {
    "contractId": "3fa85f64-5717-4562-b3fc-2c963f66afa6",
    "contractCode": "E&P RUBIALES",
    "contractType": "Exploración y Producción",
    "basinName": "Llanos Orientales"
  },
  "historicalContract": {
    "initialOperatorId": "7fa85f64-5717-4562-b3fc-2c963f66afa6",
    "initialOperatorName": "Meta Petroleum Corp.",
    "initialOperatorFreeText": null,
    "initialContractName": "Contrato Asociación Rubiales",
    "initialContractTypeCode": "ASSOCIATION",
    "initialContractTypeName": "Asociación"
  },
  "technicalDetails": {
    "trajectoryTypeCode": "O",
    "trajectoryTypeName": "Original",
    "parentWellId": null,
    "parentWellName": null,
    "hasForm103": true,
    "wellName": "RUBIALES 157",
    "avmWellId": null,
    "laheeClassificationCode": "B3",
    "laheeClassificationName": "B3",
    "finalClassificationCode": null,
    "fieldId": "9fa85f64-5717-4562-b3fc-2c963f66afa6",
    "fieldName": "RUBIALES",
    "fieldNameFreeText": null,
    "locationType": "CONTINENTAL",
    "angleTypeCode": "V",
    "angleTypeName": "Vertical",
    "form103ObjectiveCode": "PH",
    "form103ObjectiveName": "Productor Hidrocarburos",
    "currentWellStateCode": "ACTIVE",
    "currentWellStateName": "Activo",
    "currentObjectiveCode": "PH",
    "currentObjectiveName": "Productor Hidrocarburos",
    "completionTypeCode": "CC",
    "completionTypeName": "Casing cementado y cañoneado"
  },
  "geographicLocation": {
    "departmentDaneCode": "50",
    "departmentName": "Meta",
    "municipalityDaneCode": "568",
    "municipalityName": "Puerto Gaitán",
    "clusterLocationId": "1fa85f64-5717-4562-b3fc-2c963f66afa6",
    "clusterLocationName": "RUBIALES A"
  },
  "sgcUwi": "SGC-RUB-157-2008",
  "initialOriginCoordinates": {
    "datumType": "MAGNA_SIRGAS",
    "originCode": "EAST_CENTRAL",
    "originName": "Origen Este Central",
    "surfaceNorthing": 855230.45,
    "surfaceEasting": 1143567.89,
    "bottomNorthing": 855230.45,
    "bottomEasting": 1143568.12
  },
  "nationalOriginCoordinates": {
    "surfaceNorthing": 2055230.45,
    "surfaceEasting": 5143567.89,
    "bottomNorthing": 2055230.45,
    "bottomEasting": 5143568.12
  },
  "geographicCoordinates": {
    "surfaceLatitude": 3.850000,
    "surfaceLongitude": -73.750000,
    "bottomLatitude": 3.850001,
    "bottomLongitude": -73.750001
  },
  "physicalInfo": {
    "trapTypeCode": "STRUCTURAL",
    "trapTypeName": "Estructural",
    "rotaryTableElevationFeet": 550.00,
    "groundElevationFeet": 540.00,
    "waterDepthMeters": null,
    "coastDistanceKm": null,
    "totalDepthMdFeet": 8500.00,
    "totalDepthTvdFeet": 8350.00,
    "totalDepthTvdssFeet": 7810.00,
    "drillingStartDate": "2008-03-15",
    "drillingEndDate": "2008-06-20",
    "completionDate": "2008-07-15",
    "form103ApprovalDate": "2008-08-01",
    "observations": "Pozo completado con producción estable desde 2008."
  },
  "completionIntervals": {
    "completionDescription": "PERFORATION",
    "completionDescriptionName": "Cañoneo",
    "completionDescriptionOther": null,
    "intervals": [
      {
        "sequenceNumber": 1,
        "fromFeet": 7800.00,
        "toFeet": 7825.00,
        "shotsPerFoot": 10,
        "status": "OPEN",
        "thicknessFeet": 25.00
      },
      {
        "sequenceNumber": 2,
        "fromFeet": 7850.00,
        "toFeet": 7870.00,
        "shotsPerFoot": 10,
        "status": "OPEN",
        "thicknessFeet": 20.00
      }
    ],
    "totalOpenFeet": 45.00
  },
  "productionTest": {
    "testDate": "2008-07-20",
    "oilRateBblPerDay": 1250.50,
    "gasRateKscfPerDay": 3200.00,
    "waterRateBblPerDay": 120.00,
    "wellheadPressurePsia": 450.00,
    "apiGravity": 32.5,
    "gorKscfPerBbl": 2.56,
    "bswPercent": 8.76
  },
  "injectivityTest": null,
  "monitoringTest": null,
  "formationsFound": [
    {
      "sequenceNumber": 1,
      "formationName": "Mirador",
      "topMdFeet": 6200.00,
      "topTvdFeet": 6100.00,
      "topTvdssFeet": 5560.00,
      "baseMdFeet": 7200.00,
      "baseTvdFeet": 7100.00,
      "baseTvdssFeet": 6560.00,
      "thicknessMdFeet": 1000.00
    },
    {
      "sequenceNumber": 2,
      "formationName": "Barco",
      "topMdFeet": 7200.00,
      "topTvdFeet": 7100.00,
      "topTvdssFeet": 6560.00,
      "baseMdFeet": 8500.00,
      "baseTvdFeet": 8350.00,
      "baseTvdssFeet": 7810.00,
      "thicknessMdFeet": 1300.00
    }
  ],
  "reservoirFormationName": "Mirador",
  "casingsPlaced": [
    {
      "sequenceNumber": 1,
      "holeDiameterCode": "26",
      "casingSizeCode": "20",
      "gradeCode": "K55",
      "weightLbsPerFtCode": "94",
      "shoeDepthFeet": 500.00,
      "cementTopFeet": 0.00
    }
  ],
  "logsRun": [
    {
      "sequenceNumber": 1,
      "logTypeCode": "GR",
      "logTypeName": "Gamma Ray",
      "initialDepthFeet": 500.00,
      "finalDepthFeet": 8500.00,
      "scale": 200,
      "attachmentId": "afa85f64-5717-4562-b3fc-2c963f66afa6",
      "attachmentFileName": "GR_Log_Rubiales157.pdf"
    }
  ],
  "freshwaterSands": [
    {
      "number": 1,
      "topFeet": 200.00,
      "baseFeet": 450.00
    }
  ],
  "signatures": {
    "operatorSignature": {
      "engineerName": "Carlos Pérez",
      "professionalLicenseNumber": "PET-12345",
      "submittedAt": null
    },
    "anhSignature": {
      "engineerName": null,
      "professionalLicenseNumber": null,
      "approvedAt": null
    }
  },
  "observations": [],
  "createdAt": "2026-04-21T14:30:00Z",
  "createdBy": "john.doe@example.com",
  "updatedAt": "2026-04-21T14:30:00Z",
  "updatedBy": "john.doe@example.com"
}
```

**Errores:**

| HTTP | Código | Condición |
|------|--------|-----------|
| `401` | `UNAUTHORIZED` | Token ausente/inválido |
| `403` | `FORBIDDEN` | Registro fuera del alcance del usuario |
| `404` | `NOT_FOUND` | Registro no existe |

---

### 4.4 `PUT /api/v1/legacy-well-registrations/{id}` — Actualizar registro

**Propósito:** Actualiza un registro en estado `DRAFT` o `RETURNED`. Permite guardado parcial (campos nulos aceptados mientras no se envíe a aprobación).

**Autenticación:** Bearer JWT
**Roles:** `OperatorAgent`, `OperatorCoordinator`, `AnhGopAdministrator`

**Precondiciones:** `status` ∈ {`DRAFT`, `RETURNED`}

**Request Body:** Mismo esquema que `POST` (sección 4.1). Se envía el objeto completo (reemplazo total del payload).

**Response `200 OK`:**

```json
{
  "id": "00000000-0000-0000-0000-000000000001",
  "status": "DRAFT",
  "updatedAt": "2026-04-21T15:00:00Z",
  "updatedBy": "john.doe@example.com"
}
```

**Errores adicionales:**

| HTTP | Código | Condición |
|------|--------|-----------|
| `409` | `INVALID_STATE_TRANSITION` | Registro no está en estado editable |
| `409` | `CONCURRENT_MODIFICATION` | Conflicto de concurrencia (ETag mismatch) |

**Headers de concurrencia:**
- Request: `If-Match: "<etag>"` (obligatorio)
- Response: `ETag: "<new_etag>"`

---

### 4.5 `DELETE /api/v1/legacy-well-registrations/{id}` — Eliminar registro

**Propósito:** Elimina un registro. Solo permitido en estados `DRAFT` o `RETURNED` (RN-28 + fila 7 tabla de estados).

**Autenticación:** Bearer JWT
**Roles:** `OperatorAgent`, `AnhGopAdministrator`
**Precondiciones:** `status` ∈ {`DRAFT`, `RETURNED`}

**Response:** `204 No Content`

**Errores:**

| HTTP | Código | Condición |
|------|--------|-----------|
| `409` | `INVALID_STATE_TRANSITION` | Estado no permite eliminación (`SUBMITTED`, `UNDER_REVIEW`, `APPROVED`) |

---

### 4.6 `POST /api/v1/legacy-well-registrations/{id}/submit` — Enviar a aprobación

**Propósito:** Valida completitud de todos los campos obligatorios, registra la firma del operador, y transiciona el estado a `SUBMITTED`.

**Autenticación:** Bearer JWT
**Roles:** `OperatorAgent`, `AnhGopAdministrator`
**Precondiciones:** `status` ∈ {`DRAFT`, `RETURNED`}

**Request Body:**

```json
{
  "operatorEngineerName": "Carlos Pérez",
  "professionalLicenseNumber": "PET-12345"
}
```

**Response `200 OK`:**

```json
{
  "id": "00000000-0000-0000-0000-000000000001",
  "status": "SUBMITTED",
  "submittedAt": "2026-04-21T16:00:00Z",
  "fiscalizedUwi": "50568RUBI0157RUBIVOOPH–CC"
}
```

**Errores adicionales:**

| HTTP | Código | Condición |
|------|--------|-----------|
| `422` | `INCOMPLETE_REGISTRATION` | Campos obligatorios incompletos. `details` contiene la lista de campos faltantes |
| `422` | `INVALID_PROFESSIONAL_LICENSE` | Matrícula profesional inválida o no corresponde a Ingeniero de Petróleos |
| `409` | `INVALID_STATE_TRANSITION` | Estado no permite envío |

---

### 4.7 `POST /api/v1/legacy-well-registrations/{id}/approve` — Aprobar registro

**Propósito:** El ingeniero ANH aprueba el registro. Registra su firma, transiciona a `APPROVED`, crea el `Well` en el dominio con el estado FSM mapeado, y genera el PDF oficial.

**Autenticación:** Bearer JWT
**Roles:** `AnhOperationsEngineer`, `AnhGopAdministrator`
**Precondiciones:** `status` = `SUBMITTED`

**Request Body:**

```json
{
  "anhEngineerName": "María González",
  "professionalLicenseNumber": "PET-67890"
}
```

**Response `200 OK`:**

```json
{
  "id": "00000000-0000-0000-0000-000000000001",
  "status": "APPROVED",
  "approvedAt": "2026-04-22T10:00:00Z",
  "wellId": "bfa85f64-5717-4562-b3fc-2c963f66afa6",
  "fiscalizedUwi": "50568RUBI0157RUBIVOOPH–CC",
  "pdfUrl": "/api/v1/legacy-well-registrations/00000000-0000-0000-0000-000000000001/pdf"
}
```

**Efectos colaterales (no visibles en response, documentados para backend):**
- Se crea entidad `Well` en `ops` con datos del registro
- Se crea `WellLocation` con coordenadas geográficas transformadas al CRS canónico
- Se registra `WellStateHistory` con `TriggerEvent = LegacyWellRegistration`
- Se pueblan `WellFormation`, `WellCasing` desde los datos del registro
- Se genera `ProcedurePdf` tipo `PostAnhApproval`
- Se envía notificación a la operadora (RN-29)

---

### 4.8 `POST /api/v1/legacy-well-registrations/{id}/return` — Devolver con observaciones

**Propósito:** El ingeniero ANH devuelve el registro a la operadora indicando observaciones de corrección.

**Autenticación:** Bearer JWT
**Roles:** `AnhOperationsEngineer`, `AnhGopAdministrator`
**Precondiciones:** `status` = `SUBMITTED`

**Request Body:**

```json
{
  "observations": "Las coordenadas de fondo no son consistentes con la profundidad total declarada. Revisar sección de Localización."
}
```

**Response `200 OK`:**

```json
{
  "id": "00000000-0000-0000-0000-000000000001",
  "status": "RETURNED",
  "returnedAt": "2026-04-22T11:00:00Z",
  "returnedBy": "maria.gonzalez@example.com"
}
```

**Errores:**

| HTTP | Código | Condición |
|------|--------|-----------|
| `422` | `EMPTY_OBSERVATIONS` | El campo `observations` es vacío o nulo |

---

### 4.9 `GET /api/v1/legacy-well-registrations/{id}/pdf` — Descargar PDF

**Propósito:** Descarga el PDF del registro. Si el estado es `APPROVED`, devuelve el PDF oficial con firmas. En otros estados, genera un preview sin firmas ANH.

**Autenticación:** Bearer JWT
**Roles:** Todos

**Response `200 OK`:**
- `Content-Type: application/pdf`
- `Content-Disposition: attachment; filename="FOPA-RUBI-0425-2026-001.pdf"`
- Body: contenido binario del PDF

**Reglas del PDF (RN-30):**
- Solo incluye las secciones diligenciadas (ej: si objetivo = PH, solo muestra Prueba de Producción)
- Campos condicionales de offshore se omiten si tipo de ubicación = Continental
- Campos calculados (espesor, total pies abiertos, bases de formaciones) incluidos
- Firmas incluidas según estado

---

### 4.10-4.12 Gestión de adjuntos de registros eléctricos

#### `POST /api/v1/legacy-well-registrations/{id}/log-attachments`

**Request:** `multipart/form-data`

| Campo | Tipo | Requerido | Descripción |
|-------|------|-----------|-------------|
| `file` | binary | Sí | Archivo del registro eléctrico |
| `logSequenceNumber` | integer | Sí | Número de secuencia del registro corrido al que se asocia |

**Response `201 Created`:**

```json
{
  "attachmentId": "afa85f64-5717-4562-b3fc-2c963f66afa6",
  "fileName": "GR_Log_Rubiales157.pdf",
  "sizeBytes": 2048576,
  "uploadedAt": "2026-04-21T15:30:00Z"
}
```

#### `GET .../log-attachments/{attachmentId}` — Response `200`: contenido binario.
#### `DELETE .../log-attachments/{attachmentId}` — Response `204`.

---

### 4.13 `POST /api/v1/legacy-well-registrations/preview-uwi` — Previsualizar UWI

**Propósito:** Calcula y muestra el UWI fiscalizado que se generará con los datos proporcionados, sin persistir nada. Permite al operador verificar antes de guardar.

**Request Body:**

```json
{
  "departmentDaneCode": "50",
  "municipalityDaneCode": "568",
  "wellName": "RUBIALES 157",
  "wellNumber": "0157",
  "clusterLocationName": "RUBIALES A",
  "angleTypeCode": "V",
  "trajectoryTypeCode": "O",
  "objectiveCode": "PH",
  "completionTypeCode": "CC"
}
```

**Response `200 OK`:**

```json
{
  "fiscalizedUwi": "50568RUBI0157RUBIVOOPH–CC",
  "components": {
    "departmentCode": "50",
    "municipalityCode": "568",
    "wellNameSigle": "RUBI",
    "wellNumber": "0157",
    "clusterCode": "RUBI",
    "angleCode": "V",
    "trajectoryCode": "O",
    "objectiveCode": "PH",
    "completionCode": "CC"
  },
  "isUnique": true,
  "conflictingUwi": null
}
```

---

## 5. Códigos de Error Estandarizados

Estructura uniforme de error:

```json
{
  "code": "BUSINESS_RULE_VIOLATION",
  "message": "El contrato seleccionado no pertenece a la operadora del usuario.",
  "details": [
    {
      "field": "currentContract.contractId",
      "rule": "RN-01",
      "message": "Contrato no válido para la operadora en sesión."
    }
  ],
  "traceId": "00000000-0000-0000-0000-000000000001"
}
```

**Catálogo de códigos:**

| Código | HTTP | Descripción |
|--------|------|-------------|
| `VALIDATION_ERROR` | `400` | Payload malformado, tipos inválidos |
| `UNAUTHORIZED` | `401` | Token ausente, expirado o inválido |
| `FORBIDDEN` | `403` | Rol sin permiso para la operación |
| `NOT_FOUND` | `404` | Recurso no existe (o fuera del alcance del tenant) |
| `DUPLICATE_WELL_NAME` | `409` | Nombre del pozo ya registrado |
| `DUPLICATE_UWI` | `409` | UWI fiscalizado ya existe |
| `INVALID_STATE_TRANSITION` | `409` | Acción no permitida en el estado actual |
| `CONCURRENT_MODIFICATION` | `409` | ETag no coincide (optimistic concurrency) |
| `BUSINESS_RULE_VIOLATION` | `422` | Regla de negocio violada (detallado en `details`) |
| `INCOMPLETE_REGISTRATION` | `422` | Faltan campos obligatorios para envío |
| `INVALID_PROFESSIONAL_LICENSE` | `422` | Matrícula profesional no válida |
| `EMPTY_OBSERVATIONS` | `422` | Observaciones vacías en devolución |
| `PAYLOAD_TOO_LARGE` | `413` | Archivo adjunto excede límite |
| `RATE_LIMIT_EXCEEDED` | `429` | Demasiadas solicitudes |
| `INTERNAL_ERROR` | `500` | Error interno del servidor |

---

## 6. Seguridad

### 6.1 Autenticación
- Mecanismo: Bearer Token JWT emitido por `gop.identity`.
- Header: `Authorization: Bearer <token>`.
- Todos los endpoints requieren autenticación. No existen endpoints públicos.

### 6.2 Autorización
- Verificación de rol **y** alcance de datos en cada endpoint.
- Usuarios operadora: solo ven registros de su operadora (`User.OperatorId` → filtro implícito).
- Usuarios ANH: ven registros según alcance de `UserRole` (global, por operador, por contrato, por campo).
- El `Coordinador` no puede ejecutar `/submit`, `/approve`, `/return`.
- El `Agente` no puede ejecutar `/approve`, `/return`.

### 6.3 Rate Limiting
- Endpoints de escritura: 30 requests/minuto por usuario.
- Endpoints de lectura: 120 requests/minuto por usuario.
- Upload de archivos: 10 requests/minuto por usuario.
- Respuesta al exceder: `429 Too Many Requests` con header `Retry-After`.

### 6.4 Validaciones de seguridad
- Inputs sanitizados contra inyección.
- Nombres de archivo de adjuntos validados (sin path traversal).
- Tamaño máximo de adjuntos: configurable (propuesta: 10 MB por archivo, PA-05 pendiente).
- Content-Type de adjuntos validado contra whitelist.
- No se exponen IDs internos de base de datos ni stack traces en respuestas de error.

---

## 7. Convenciones Transversales

| Concepto | Convención |
|----------|------------|
| IDs | UUID v4 (string) |
| Fechas | ISO 8601 UTC (`2026-04-21T14:30:00Z`) |
| Fechas solo-fecha | ISO 8601 (`2026-04-21`) |
| Rutas | kebab-case, recursos en plural |
| Versionado | Path: `/api/v1/...` |
| Paginación | `{ items, page, pageSize, total, totalPages }` |
| Concurrencia | Optimistic via `ETag` / `If-Match` en `PUT` |
| Tenant filtering | Implícito desde JWT claims; nunca se pasa como parámetro |

---

## 8. Puntos de Decisión Pendientes

| # | Punto | Impacto |
|---|-------|---------|
| PD-01 | PA-05: Formatos y tamaño máximo de adjuntos para registros eléctricos. Propuesta temporal: PDF/TIFF/LAS, max 10 MB | Afecta validación en endpoint 10 |
| PD-02 | Integración AVM (PA-01): si la API no está disponible, el endpoint de búsqueda AVM devuelve `501 Not Implemented` y el frontend muestra campo de texto libre como fallback | Afecta UX de selección de pozo |
| PD-03 | Integración SGC: en V1, el UWI SGC se captura como texto libre. Endpoint de búsqueda SGC no incluido hasta que la API esté disponible | Supuesto A-07 confirmado |
| PD-04 | Mapeo de estados HU → FSM del Well: definición exacta de la tabla de equivalencia a incluir en spec-backend | Supuesto A-04 confirmado |

---

## 9. Anexos

- `openapi.yaml` — Especificación OpenAPI 3.1 validable (archivo adjunto)
- `spec-frontend.md` — Especificación funcional del frontend
- `spec-backend.md` — Especificación funcional del backend
