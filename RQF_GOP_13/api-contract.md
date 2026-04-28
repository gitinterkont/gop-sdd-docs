# API Contract — Forma 202: Informe Mensual de Producción — Pozos de Petróleo y Gas

**Feature:** RQF_GOP_13  
**Versión:** 1.0  
**Fecha:** 2026-04-28  
**Estado:** Propuesta — Pendiente aprobación del arquitecto humano  
**Fuente:** RQF_GOP_13_Forma202 V1 + BP_GOP_13_Forma202 V1  

---

## 1. Resumen

Este contrato define la interfaz HTTP REST entre frontend y backend para la gestión completa de la **Forma 202 — Informe Mensual de Producción por Pozo y Formación Productora**. Cubre el ciclo de vida completo: creación automática, precarga desde AVM, edición por operador, cargue masivo de pozos históricos, validación cruzada con Formas 204/205, firma electrónica, envío, revisión ANH, aprobación y generación de PDF.

**OpenAPI Spec:** ver archivo adjunto `openapi.yaml` (OpenAPI 3.1, validable).

---

## 2. Convenciones

| Convención | Valor |
|---|---|
| Base URL | `/api/v1/production/form-202` |
| Formato de IDs | UUID v4 (string) |
| Fechas | ISO 8601 UTC (`2026-04-28T14:30:00Z`) |
| Paginación | `page` (1-based), `pageSize` (default 20, max 100), `sort`, `filter` |
| Autenticación | Bearer JWT (`Authorization: Bearer <token>`) |
| Content-Type | `application/json` (salvo bulk-upload: `multipart/form-data`) |
| Errores | Estructura uniforme `{ code, message, details[], traceId }` |
| Nombres de ruta | kebab-case, recursos en plural donde aplica |
| Versionado | En path (`/api/v1/...`) |

---

## 3. Actores y Roles

| Rol | Código | Permisos sobre Forma 202 |
|---|---|---|
| Operador GOP | `OperatorAgent` | Registrar, Modificar, Firmar, Enviar. Solo formas de sus contratos asignados. |
| Supervisor ANH Producción | `AnhProductionSupervisor` | Revisar, Aprobar, Devolver, Firmar. Solo formas de contratos/campos asignados. |
| Admin GOP | `AnhGopAdministrator` | Acceso total a todas las operadoras y formularios. CRUD completo. |

---

## 4. Tabla de Endpoints

| # | Método | Ruta | Descripción | Roles Autorizados | HTTP Éxito |
|---|--------|------|-------------|-------------------|------------|
| 1 | GET | `/` | Listar Formas 202 (filtradas por contexto del usuario) | Todos | 200 |
| 2 | POST | `/` | Crear instancia de Forma 202 | `AnhGopAdministrator` | 201 |
| 3 | GET | `/{id}` | Obtener detalle de una Forma 202 (encabezado + resumen) | Todos | 200 |
| 4 | PATCH | `/{id}/header` | Actualizar campos del encabezado (Paso 1) | `OperatorAgent` | 200 |
| 5 | POST | `/{id}/preload` | Disparar precarga de datos desde AVM | `OperatorAgent` | 202 |
| 6 | GET | `/{id}/well-productions` | Obtener grilla de producción por pozo (paginada) | Todos | 200 |
| 7 | POST | `/{id}/well-productions` | Agregar fila de pozo manualmente | `OperatorAgent` | 201 |
| 8 | PUT | `/{id}/well-productions/{wpId}` | Actualizar fila de producción de un pozo | `OperatorAgent` | 200 |
| 9 | DELETE | `/{id}/well-productions/{wpId}` | Eliminar fila de pozo agregada manualmente (solo pre-envío) | `OperatorAgent` | 204 |
| 10 | POST | `/{id}/well-productions/bulk-upload` | Cargue masivo desde archivo Excel | `OperatorAgent` | 202 |
| 11 | GET | `/bulk-upload-template` | Descargar plantilla Excel para cargue masivo | `OperatorAgent` | 200 |
| 12 | GET | `/{id}/totals` | Obtener fila TOTAL calculada | Todos | 200 |
| 13 | POST | `/{id}/cross-validations` | Ejecutar validación cruzada con Formas 204/205 | `OperatorAgent`, `AnhProductionSupervisor` | 200 |
| 14 | GET | `/{id}/cross-validations` | Consultar resultado de última validación cruzada | Todos | 200 |
| 15 | POST | `/{id}/actions/sign-and-submit` | Firmar electrónicamente y enviar | `OperatorAgent` | 200 |
| 16 | POST | `/{id}/actions/approve` | Aprobar y firmar por Supervisor ANH | `AnhProductionSupervisor` | 200 |
| 17 | POST | `/{id}/actions/return` | Devolver para corrección | `AnhProductionSupervisor` | 200 |
| 18 | GET | `/{id}/change-log` | Consultar log de modificaciones sobre datos precargados | `AnhProductionSupervisor`, `AnhGopAdministrator` | 200 |
| 19 | GET | `/{id}/pdf` | Descargar PDF generado | Todos | 200 |
| 20 | GET | `/{id}/versions` | Consultar historial de versiones | Todos | 200 |

---

## 5. Detalle por Endpoint

### 5.1 GET `/` — Listar Formas 202

**Propósito:** Obtener listado paginado de instancias de Forma 202 visibles para el usuario autenticado, filtradas automáticamente por su alcance de datos (tenant, contratos asignados).

**Query Parameters:**

| Parámetro | Tipo | Requerido | Descripción |
|---|---|---|---|
| `page` | integer | No (default: 1) | Página actual |
| `pageSize` | integer | No (default: 20, max: 100) | Elementos por página |
| `sort` | string | No | Campo de ordenamiento. Valores: `reportMonth`, `reportYear`, `fieldName`, `status`, `createdAt`. Prefijo `-` para descendente. |
| `operatorId` | uuid | No | Filtrar por operador (solo Admin/Supervisor) |
| `contractId` | uuid | No | Filtrar por contrato |
| `fieldId` | uuid | No | Filtrar por campo |
| `formationId` | uuid | No | Filtrar por formación |
| `reportYear` | integer | No | Filtrar por año |
| `reportMonth` | integer | No | Filtrar por mes (1-12) |
| `status` | string | No | Filtrar por estado: `Initial`, `Registration`, `Submitted`, `Approved` |

**Response 200:**

```json
{
  "items": [
    {
      "id": "00000000-0000-0000-0000-000000000001",
      "consecutiveNumber": "F202-CampoRubiales-Colorado-Exploitation-03-2026-001",
      "operatorName": "Operadora Ejemplo S.A.",
      "contractCode": "E&P-001",
      "fieldName": "Campo Rubiales",
      "formationName": "Colorado",
      "modalityType": "Exploitation",
      "reportMonth": 3,
      "reportYear": 2026,
      "status": "Registration",
      "version": 1,
      "wellCount": 45,
      "totalOilMonthlyBbl": 125000.50,
      "totalGasMonthlyKpc": 89000.25,
      "createdAt": "2026-04-01T05:00:00Z",
      "submittedAt": null,
      "approvedAt": null
    }
  ],
  "page": 1,
  "pageSize": 20,
  "total": 1,
  "totalPages": 1
}
```

---

### 5.2 POST `/` — Crear instancia de Forma 202

**Propósito:** Crear manualmente una instancia de Forma 202. Uso principal: creación por sistema (job del 1.° del mes) o por Admin GOP.

**Request Body:**

```json
{
  "contractId": "00000000-0000-0000-0000-000000000010",
  "fieldId": "00000000-0000-0000-0000-000000000020",
  "producingFormationId": "00000000-0000-0000-0000-000000000030",
  "productionModalityId": "00000000-0000-0000-0000-000000000040",
  "reportMonth": 3,
  "reportYear": 2026
}
```

**Response 201:**

```json
{
  "id": "00000000-0000-0000-0000-000000000001",
  "consecutiveNumber": "F202-CampoRubiales-Colorado-Exploitation-03-2026-001",
  "status": "Initial",
  "version": 0,
  "createdAt": "2026-04-01T05:00:00Z"
}
```

**Reglas de validación:**
- La combinación (campo + formación + modalidad + mes/año) debe ser única. Si ya existe → 409 Conflict.
- El contrato debe estar asignado al operador del usuario autenticado (o Admin tiene acceso global).
- El campo debe pertenecer al contrato.
- La formación debe existir y estar asociada al campo.
- La modalidad debe estar vigente para la formación.
- `reportMonth` entre 1 y 12; `reportYear` ≥ 2020 y ≤ año actual.

**Errores:**

| HTTP | code | Escenario |
|---|---|---|
| 400 | `VALIDATION_ERROR` | Campos faltantes o inválidos |
| 401 | `UNAUTHORIZED` | Token ausente o inválido |
| 403 | `FORBIDDEN` | Sin permiso para crear formas |
| 409 | `DUPLICATE_FORM` | Ya existe una Forma 202 para esa combinación |
| 422 | `INVALID_COMBINATION` | La combinación campo/formación/modalidad no es válida |

---

### 5.3 GET `/{id}` — Obtener detalle de Forma 202

**Propósito:** Obtener información completa del encabezado y metadatos de una Forma 202.

**Response 200:**

```json
{
  "id": "00000000-0000-0000-0000-000000000001",
  "consecutiveNumber": "F202-CampoRubiales-Colorado-Exploitation-03-2026-001",
  "radicadoNumber": null,
  "status": "Registration",
  "version": 1,
  "header": {
    "reportMonth": 3,
    "reportYear": 2026,
    "operatorId": "00000000-0000-0000-0000-000000000050",
    "operatorName": "Operadora Ejemplo S.A.",
    "contractId": "00000000-0000-0000-0000-000000000010",
    "contractCode": "E&P-001",
    "fieldId": "00000000-0000-0000-0000-000000000020",
    "fieldName": "Campo Rubiales",
    "producingFormationId": "00000000-0000-0000-0000-000000000030",
    "formationName": "Colorado",
    "productionModalityId": "00000000-0000-0000-0000-000000000040",
    "modalityType": "Exploitation",
    "trapType": "Structural",
    "observations": null
  },
  "summary": {
    "wellCount": 45,
    "wellsFromAvm": 40,
    "wellsManual": 3,
    "wellsBulkLoaded": 2,
    "totalOilMonthlyBbl": 125000.50,
    "totalWaterMonthlyBbl": 45000.00,
    "totalGasMonthlyKpc": 89000.25,
    "hasModifications": true,
    "modificationCount": 7
  },
  "crossValidation": {
    "oilStatus": "Warning",
    "oilForm202Total": 125000.50,
    "oilForm204Total": 125100.00,
    "oilDifference": -99.50,
    "gasStatus": "Passed",
    "gasForm202Total": 89000.25,
    "gasForm205Total": 89000.25,
    "gasDifference": 0.00,
    "executedAt": "2026-04-05T10:30:00Z"
  },
  "signatures": {
    "operator": null,
    "supervisor": null
  },
  "workflow": {
    "currentStep": "OperatorRegistration",
    "deadlineAt": "2026-04-10T23:59:59Z",
    "daysRemaining": 5,
    "returnCount": 0
  },
  "createdAt": "2026-04-01T05:00:00Z",
  "submittedAt": null,
  "approvedAt": null,
  "pdfAvailable": false
}
```

**Errores:**

| HTTP | code | Escenario |
|---|---|---|
| 401 | `UNAUTHORIZED` | Token ausente o inválido |
| 403 | `FORBIDDEN` | Sin acceso a esta forma (fuera de su alcance) |
| 404 | `NOT_FOUND` | Forma no existe |

---

### 5.4 PATCH `/{id}/header` — Actualizar encabezado

**Propósito:** Actualizar campos editables del encabezado (Paso 1 del wizard). Solo permitido cuando el estado es `Registration`.

**Request Body (parcial — solo campos a actualizar):**

```json
{
  "trapType": "Stratigraphic",
  "observations": "Se reportan 3 pozos inactivos por mantenimiento preventivo."
}
```

**Campos editables:** `trapType`, `observations`.

**Response 200:** Retorna el objeto `header` actualizado (misma estructura que en GET `/{id}`).

**Reglas:**
- Solo permitido en estado `Registration`.
- `trapType` debe ser uno de: `Structural`, `Stratigraphic`, `Mixed`.
- `observations` máximo 2000 caracteres.

**Errores:**

| HTTP | code | Escenario |
|---|---|---|
| 400 | `VALIDATION_ERROR` | Valor de trapType inválido |
| 403 | `FORBIDDEN` | Sin permiso |
| 404 | `NOT_FOUND` | Forma no existe |
| 409 | `INVALID_STATE` | La forma no está en estado Registration |

---

### 5.5 POST `/{id}/preload` — Disparar precarga desde AVM

**Propósito:** Invoca la integración con AVM para precargar la grilla de producción con los pozos activos y sus datos del mes para la combinación campo + formación + modalidad seleccionada.

**Request Body:** Ninguno (los parámetros se derivan del encabezado de la forma).

**Response 202:**

```json
{
  "preloadId": "00000000-0000-0000-0000-000000000099",
  "status": "Completed",
  "wellsPreloaded": 40,
  "wellsSkipped": 0,
  "warnings": [],
  "executedAt": "2026-04-05T08:00:00Z"
}
```

**Response 202 (con warnings):**

```json
{
  "preloadId": "00000000-0000-0000-0000-000000000099",
  "status": "CompletedWithWarnings",
  "wellsPreloaded": 38,
  "wellsSkipped": 2,
  "warnings": [
    {
      "wellName": "Rubiales-45",
      "reason": "Well state in AVM does not match GOP Operations catalog",
      "avmState": "SUSPENDED",
      "gopState": null
    }
  ],
  "executedAt": "2026-04-05T08:00:00Z"
}
```

**Reglas:**
- Solo permitido en estado `Registration`.
- Si la precarga ya fue ejecutada, ejecutar de nuevo **sobrescribe los datos precargados que no hayan sido modificados** por el operador. Los datos modificados se preservan.
- Si la conexión con AVM falla, retorna 502 con indicación de usar registro manual (RN-25).

**Errores:**

| HTTP | code | Escenario |
|---|---|---|
| 403 | `FORBIDDEN` | Sin permiso |
| 404 | `NOT_FOUND` | Forma no existe |
| 409 | `INVALID_STATE` | No está en estado Registration |
| 502 | `AVM_CONNECTION_ERROR` | No se pudo conectar con AVM |

---

### 5.6 GET `/{id}/well-productions` — Obtener grilla de producción

**Propósito:** Obtener las filas de producción por pozo (Paso 2 del wizard), paginadas.

**Query Parameters:**

| Parámetro | Tipo | Requerido | Descripción |
|---|---|---|---|
| `page` | integer | No (default: 1) | Página |
| `pageSize` | integer | No (default: 50, max: 500) | Registros por página. Max alto por campos con +2000 pozos. |
| `sort` | string | No | `wellName`, `oilMonthlyBbl`, `status`. Prefijo `-` desc. |
| `wellStateFilter` | string | No | Filtrar por estado del pozo |
| `dataOriginFilter` | string | No | `Avm`, `Manual`, `BulkLoad` |
| `search` | string | No | Búsqueda por nombre de pozo |

**Response 200:**

```json
{
  "items": [
    {
      "id": "00000000-0000-0000-0000-000000000100",
      "wellId": "00000000-0000-0000-0000-000000000200",
      "wellName": "Rubiales-01",
      "municipalityDaneCode": "50590",
      "productionMethod": "BES",
      "daysInMonth": 30.00,
      "cumulativeDays": 3650.50,
      "oilDailyBopd": 150.25,
      "oilMonthlyBbl": 4507.50,
      "oilCumulativeBbl": 1250000.00,
      "oilFieldFactor": 0.98750,
      "waterDailyBwpd": 50.00,
      "waterMonthlyBbl": 1500.00,
      "waterCumulativeBbl": 450000.00,
      "waterFieldFactor": 1.00050,
      "gasDailyKpc": 25.50,
      "gasMonthlyKpc": 765.00,
      "gasCumulativeKpc": 230000.00,
      "gasFieldFactor": 1.01200,
      "bswPercent": 24.950,
      "apiGravity": 12.5,
      "gasOilRatio": 169.78,
      "wellStateEndOfMonth": "Active",
      "wellTypeEndOfMonth": "Producer",
      "dataOrigin": "Avm",
      "isModified": true,
      "modificationCount": 2
    }
  ],
  "page": 1,
  "pageSize": 50,
  "total": 45,
  "totalPages": 1
}
```

---

### 5.7 POST `/{id}/well-productions` — Agregar pozo manualmente

**Propósito:** Agregar una fila de producción para un pozo histórico no disponible en AVM (cargue individual).

**Request Body:**

```json
{
  "wellName": "Rubiales-Legacy-01",
  "municipalityDaneCode": "50590",
  "productionMethod": "BM",
  "daysInMonth": 0.00,
  "cumulativeDays": 8500.00,
  "oilDailyBopd": 0.00,
  "oilMonthlyBbl": 0.00,
  "oilCumulativeBbl": 850000.00,
  "oilFieldFactor": 0.98500,
  "waterDailyBwpd": 0.00,
  "waterMonthlyBbl": 0.00,
  "waterCumulativeBbl": 320000.00,
  "waterFieldFactor": 1.00000,
  "gasDailyKpc": 0.00,
  "gasMonthlyKpc": 0.00,
  "gasCumulativeKpc": 95000.00,
  "gasFieldFactor": 1.00000,
  "bswPercent": 0.000,
  "apiGravity": 0.0,
  "gasOilRatio": 0.00,
  "wellStateEndOfMonth": "Abandoned",
  "wellTypeEndOfMonth": "Producer"
}
```

**Response 201:** Retorna el objeto creado con `id`, `dataOrigin: "Manual"`.

**Reglas de validación:**
- Solo permitido en estado `Registration`.
- RN-12: `wellName` se valida contra nombres aprobados en Forma 103 (GOP Operaciones). Si no coincide, se permite con warning y trazabilidad.
- RN-11: Para el primer registro (version 0), los acumulados se aceptan manualmente.
- Todos los campos numéricos ≥ 0.
- `bswPercent` entre 0 y 100 (RN-20).
- Factores de campo: 5 decimales, > 0.
- `municipalityDaneCode`: exactamente 5 dígitos.
- `productionMethod`: uno de `FN`, `BES`, `BCP`, `BM`, `BH`, `GL`, `Other`.
- `wellStateEndOfMonth`: uno de `Active`, `Inactive`, `TemporarilySuspended`, `Abandoned`, `TemporarilyAbandoned`, `SuspendedWithApprovedPlan`, `TemporarilyAbandonedWithApprovedPlan`, `NonCompliant`.
- `wellTypeEndOfMonth`: uno de `Producer`, `Injector`, `Disposal`, `Monitor`, `Other`.

**Errores:**

| HTTP | code | Escenario |
|---|---|---|
| 400 | `VALIDATION_ERROR` | Campos inválidos |
| 403 | `FORBIDDEN` | Sin permiso |
| 404 | `NOT_FOUND` | Forma no existe |
| 409 | `INVALID_STATE` | No está en Registration |
| 409 | `DUPLICATE_WELL` | Ya existe una fila para ese pozo en esta forma |

---

### 5.8 PUT `/{id}/well-productions/{wpId}` — Actualizar fila de producción

**Propósito:** Modificar datos de producción de un pozo (precargado o manual). Cada cambio sobre un dato precargado de AVM genera entrada en el log de cambios (RN-09).

**Request Body:** Misma estructura que POST (§5.7), todos los campos.

**Response 200:** Retorna el objeto actualizado. Si el dato original vino de AVM y se modificó, `isModified: true`.

**Reglas adicionales:**
- Solo permitido en estado `Registration`.
- RN-18: Si ya existe acumulado del mes anterior, el acumulado nuevo no puede ser menor.
- RN-09: Para campos precargados desde AVM, el sistema registra automáticamente: valor original AVM, valor nuevo, campo modificado, pozo, usuario y timestamp.
- Los factores de campo deben estar dentro de rangos razonables (no negativos).

**Errores:**

| HTTP | code | Escenario |
|---|---|---|
| 400 | `VALIDATION_ERROR` | Datos inválidos |
| 400 | `CUMULATIVE_DECREASE` | El acumulado propuesto es menor al del mes anterior |
| 403 | `FORBIDDEN` | Sin permiso |
| 404 | `NOT_FOUND` | Forma o fila de pozo no encontrada |
| 409 | `INVALID_STATE` | No está en Registration |

---

### 5.9 DELETE `/{id}/well-productions/{wpId}` — Eliminar fila manual

**Propósito:** Eliminar una fila de pozo agregada manualmente **que aún no ha sido enviada**. No aplica a pozos precargados desde AVM.

**Response 204:** Sin contenido.

**Reglas:**
- Solo permitido en estado `Registration`.
- Solo filas con `dataOrigin = "Manual"` o `"BulkLoad"`.
- RN-13: Si el pozo ya fue incluido en una Forma 202 de un mes previo para este campo/formación, **no se puede eliminar**.
- Pozos precargados desde AVM nunca se eliminan (retorna 422).

**Errores:**

| HTTP | code | Escenario |
|---|---|---|
| 403 | `FORBIDDEN` | Sin permiso |
| 404 | `NOT_FOUND` | Fila no encontrada |
| 409 | `INVALID_STATE` | No está en Registration |
| 422 | `CANNOT_DELETE_AVM_WELL` | No se puede eliminar un pozo precargado desde AVM |
| 422 | `CANNOT_DELETE_HISTORICAL_WELL` | Pozo tiene historial en meses anteriores |

---

### 5.10 POST `/{id}/well-productions/bulk-upload` — Cargue masivo

**Propósito:** Cargar múltiples filas de pozos históricos desde un archivo Excel (.xlsx).

**Request:** `Content-Type: multipart/form-data`

| Campo | Tipo | Requerido | Descripción |
|---|---|---|---|
| `file` | binary | Sí | Archivo Excel (.xlsx), máximo 10 MB |

**Response 202:**

```json
{
  "bulkLoadId": "00000000-0000-0000-0000-000000000300",
  "status": "Completed",
  "totalRecords": 150,
  "successfulRecords": 148,
  "failedRecords": 2,
  "errors": [
    {
      "row": 45,
      "wellName": "Pozo-X",
      "field": "bswPercent",
      "message": "Value 105.5 exceeds maximum allowed (100)"
    },
    {
      "row": 89,
      "wellName": "Pozo-Y",
      "field": "municipalityDaneCode",
      "message": "DANE code '999' does not match expected 5-digit format"
    }
  ],
  "warnings": [
    {
      "row": 12,
      "wellName": "Pozo-Legacy-12",
      "message": "Well name not found in Form 103 approved names"
    }
  ]
}
```

**Reglas:**
- Solo permitido en estado `Registration`.
- El archivo debe seguir la estructura de la plantilla descargable (§5.11).
- Máximo 10 MB (campos con +2000 pozos como La Cira).
- Cada fila se valida individualmente con las mismas reglas que POST individual (§5.7).
- Filas válidas se insertan; filas inválidas se reportan en `errors`.
- Si un pozo del archivo ya existe en la grilla, se actualiza (upsert por nombre de pozo).

---

### 5.11 GET `/bulk-upload-template` — Descargar plantilla Excel

**Propósito:** Descargar la plantilla Excel con las columnas predefinidas y listas de valores válidos.

**Response 200:** `Content-Type: application/vnd.openxmlformats-officedocument.spreadsheetml.sheet`

Archivo binario (.xlsx) con:
- Hoja 1: Plantilla de datos (22 columnas con encabezados y validaciones de Excel).
- Hoja 2: Catálogos de referencia (métodos de producción, estados de pozo, tipos de pozo).

---

### 5.12 GET `/{id}/totals` — Obtener fila TOTAL

**Propósito:** Obtener los valores calculados de la fila TOTAL de la grilla.

**Response 200:**

```json
{
  "totalDaysInMonth": 1350.00,
  "totalCumulativeDays": 164250.00,
  "totalOilDailyBopd": 6750.25,
  "totalOilMonthlyBbl": 202507.50,
  "totalOilCumulativeBbl": 56250000.00,
  "avgOilFieldFactor": 0.98850,
  "totalWaterDailyBwpd": 2250.00,
  "totalWaterMonthlyBbl": 67500.00,
  "totalWaterCumulativeBbl": 20250000.00,
  "avgWaterFieldFactor": 1.00030,
  "totalGasDailyKpc": 1147.50,
  "totalGasMonthlyKpc": 34425.00,
  "totalGasCumulativeKpc": 10350000.00,
  "avgGasFieldFactor": 1.01000,
  "totalBswPercent": 24.98,
  "totalApiGravity": 12.8,
  "totalGasOilRatio": 170.00,
  "wellCount": 45,
  "calculatedAt": "2026-04-05T10:35:00Z"
}
```

**Reglas de cálculo:**
- Campos con prefijo `total`: sumatoria de todas las filas.
- Campos con prefijo `avg`: promedio aritmético de factores de campo.
- `totalBswPercent` = `100 - ((totalOilMonthlyBbl / (totalOilMonthlyBbl + totalWaterMonthlyBbl)) × 100)`. Si denominador = 0 → `null`.
- `totalApiGravity` = promedio ponderado por volumen vía gravedad específica: `141.5 / (Σ(Vi × 141.5 / (APIi + 131.5)) / ΣVi) - 131.5`. Si ΣVi = 0 → `null`.
- `totalGasOilRatio` = `(totalGasMonthlyKpc × 1000) / totalOilMonthlyBbl`. Si totalOilMonthlyBbl = 0 → `null`.
- Valores `null` se representan como `null` en JSON (no como 0).

---

### 5.13 POST `/{id}/cross-validations` — Ejecutar validación cruzada

**Propósito:** Comparar totales de producción de la Forma 202 (suma de todas las formaciones del campo) contra Formas 204 (petróleo) y 205 (gas) del mismo campo/mes/año.

**Request Body:** Ninguno.

**Response 200:**

```json
{
  "oilValidation": {
    "status": "Warning",
    "form202Total": 125000.50,
    "form204Total": 125100.00,
    "difference": -99.50,
    "differencePercent": -0.08,
    "message": "La sumatoria de petróleo de Forma 202 difiere de Forma 204 en 99.50 BBL"
  },
  "gasValidation": {
    "status": "Passed",
    "form202Total": 89000.25,
    "form205Total": 89000.25,
    "difference": 0.00,
    "differencePercent": 0.00,
    "message": null
  },
  "executedAt": "2026-04-05T10:30:00Z"
}
```

**Valores de `status`:** `Passed`, `Warning`, `NotAvailable` (si la Forma 204/205 no existe aún).

**Reglas:**
- RN-17 / RN-19: La comparación es informativa (warning), no bloqueante.
- Si las Formas 204 o 205 no existen para el mismo campo/mes/año, el status es `NotAvailable`.

---

### 5.14 POST `/{id}/actions/sign-and-submit` — Firmar y Enviar

**Propósito:** El Operador GOP firma electrónicamente la Forma 202 y la envía para revisión del Supervisor ANH. Cambia estado de `Registration` → `Submitted`.

**Request Body:**

```json
{
  "engineerFullName": "Juan Pérez Rodríguez",
  "professionalLicenseNumber": "1234567",
  "confirmSignature": true
}
```

**Response 200:**

```json
{
  "id": "00000000-0000-0000-0000-000000000001",
  "status": "Submitted",
  "submittedAt": "2026-04-05T14:00:00Z",
  "signatureId": "00000000-0000-0000-0000-000000000400",
  "pdfGenerationStatus": "Queued",
  "radicadoNumber": null,
  "version": 1
}
```

**Reglas:**
- Solo permitido en estado `Registration`.
- Plazo: 7 días hábiles desde el 1.° del mes. Después de este plazo, el sistema envía automáticamente (no es vía API — es job programado).
- `confirmSignature` debe ser `true` (previene envío accidental).
- `engineerFullName` máximo 50 caracteres, obligatorio.
- `professionalLicenseNumber` exactamente 7 dígitos, obligatorio.
- El sistema genera el PDF con firma del operador (vía SSRS) y lo radica en ControlDoc de forma asíncrona.
- Se registra en `ProcedureSignature`: hash del contenido, timestamp, datos del firmante.

**Errores:**

| HTTP | code | Escenario |
|---|---|---|
| 400 | `VALIDATION_ERROR` | Datos del firmante inválidos |
| 400 | `SIGNATURE_NOT_CONFIRMED` | `confirmSignature` es false |
| 403 | `FORBIDDEN` | Sin permiso para firmar |
| 404 | `NOT_FOUND` | Forma no existe |
| 409 | `INVALID_STATE` | No está en Registration |
| 409 | `EMPTY_GRID` | No hay filas de producción registradas |

---

### 5.15 POST `/{id}/actions/approve` — Aprobar

**Propósito:** El Supervisor ANH aprueba la Forma 202 y firma electrónicamente. Cambia estado `Submitted` → `Approved`. La forma queda bloqueada para edición.

**Request Body:**

```json
{
  "engineerFullName": "María García López",
  "professionalLicenseNumber": "7654321",
  "confirmSignature": true
}
```

**Response 200:**

```json
{
  "id": "00000000-0000-0000-0000-000000000001",
  "status": "Approved",
  "approvedAt": "2026-04-15T16:00:00Z",
  "signatureId": "00000000-0000-0000-0000-000000000500",
  "pdfGenerationStatus": "Queued",
  "radicadoNumber": "RAD-2026-04-00123"
}
```

**Reglas:**
- Solo permitido en estado `Submitted`.
- El PDF final con doble firma se genera asincrónicamente (SSRS) y se actualiza en ControlDoc.
- La forma queda inmutable tras aprobación.

---

### 5.16 POST `/{id}/actions/return` — Devolver para corrección

**Propósito:** El Supervisor ANH devuelve la forma al operador para corrección. Cambia estado `Submitted` → `Registration` (estandarización transversal BP_GOP_13, PA-BP-06).

**Request Body:**

```json
{
  "returnReason": "Los valores de producción del pozo Rubiales-12 no coinciden con los registros de AVM. Verificar factor de campo de petróleo.",
  "affectedWellIds": [
    "00000000-0000-0000-0000-000000000200"
  ]
}
```

**Response 200:**

```json
{
  "id": "00000000-0000-0000-0000-000000000001",
  "status": "Registration",
  "returnedAt": "2026-04-12T09:00:00Z",
  "correctionDeadlineAt": "2026-04-17T23:59:59Z",
  "returnCount": 1,
  "returnReason": "Los valores de producción del pozo Rubiales-12 no coinciden..."
}
```

**Reglas:**
- Solo permitido en estado `Submitted`.
- Solo permitido antes del día 19 del mes calendario.
- `returnReason` obligatorio, máximo 2000 caracteres.
- El operador tiene 3 días hábiles para corregir y reenviar. Si excede, envío automático por job.
- Se envía notificación SMTP al operador.

**Errores:**

| HTTP | code | Escenario |
|---|---|---|
| 400 | `VALIDATION_ERROR` | Razón de devolución faltante |
| 403 | `FORBIDDEN` | Sin permiso |
| 404 | `NOT_FOUND` | Forma no existe |
| 409 | `INVALID_STATE` | No está en Submitted |
| 409 | `RETURN_DEADLINE_EXCEEDED` | Fecha actual excede el día 19 del mes |

---

### 5.17 GET `/{id}/change-log` — Log de modificaciones

**Propósito:** Consultar el log de modificaciones realizadas sobre datos precargados de AVM (RN-10). Filtrable.

**Query Parameters:**

| Parámetro | Tipo | Requerido | Descripción |
|---|---|---|---|
| `page` | integer | No (default: 1) | Página |
| `pageSize` | integer | No (default: 50) | Elementos por página |
| `wellId` | uuid | No | Filtrar por pozo |
| `modifiedField` | string | No | Filtrar por campo modificado |

**Response 200:**

```json
{
  "items": [
    {
      "id": "00000000-0000-0000-0000-000000000600",
      "wellProductionId": "00000000-0000-0000-0000-000000000100",
      "wellName": "Rubiales-01",
      "modifiedField": "oilMonthlyBbl",
      "originalValue": "4500.00",
      "newValue": "4507.50",
      "modifiedBy": "john.doe@example.com",
      "modifiedByName": "John Doe",
      "modifiedAt": "2026-04-05T11:30:00Z",
      "justification": null
    }
  ],
  "page": 1,
  "pageSize": 50,
  "total": 7,
  "totalPages": 1
}
```

---

### 5.18 GET `/{id}/pdf` — Descargar PDF

**Propósito:** Descargar el PDF generado de la Forma 202. Disponible después del envío (firma única) o después de la aprobación (doble firma).

**Query Parameters:**

| Parámetro | Tipo | Requerido | Descripción |
|---|---|---|---|
| `type` | string | No (default: `latest`) | `operatorSigned`, `approved`, `latest` |

**Response 200:** `Content-Type: application/pdf`. Archivo binario.

**Errores:**

| HTTP | code | Escenario |
|---|---|---|
| 404 | `PDF_NOT_AVAILABLE` | No hay PDF generado aún |

---

### 5.19 GET `/{id}/versions` — Historial de versiones

**Propósito:** Consultar las versiones de la Forma 202. Cada envío genera nueva versión.

**Response 200:**

```json
{
  "items": [
    {
      "version": 1,
      "status": "Submitted",
      "submittedAt": "2026-04-05T14:00:00Z",
      "submittedBy": "john.doe@example.com",
      "returnedAt": "2026-04-12T09:00:00Z",
      "returnReason": "Verificar factor de campo..."
    },
    {
      "version": 0,
      "status": "Initial",
      "submittedAt": null,
      "submittedBy": null,
      "returnedAt": null,
      "returnReason": null
    }
  ]
}
```

---

## 6. Estructura Estándar de Errores

Todos los errores siguen esta estructura:

```json
{
  "code": "VALIDATION_ERROR",
  "message": "One or more validation errors occurred.",
  "details": [
    {
      "field": "bswPercent",
      "message": "Value must be between 0 and 100"
    }
  ],
  "traceId": "00000000-0000-0000-0000-000000000999"
}
```

**Catálogo de códigos de error:**

| code | HTTP | Descripción |
|---|---|---|
| `VALIDATION_ERROR` | 400 | Error de validación de datos de entrada |
| `SIGNATURE_NOT_CONFIRMED` | 400 | Firma no confirmada por el usuario |
| `CUMULATIVE_DECREASE` | 400 | Acumulado propuesto menor al mes anterior |
| `UNAUTHORIZED` | 401 | Token JWT ausente, inválido o expirado |
| `FORBIDDEN` | 403 | Usuario autenticado sin permisos para la acción/recurso |
| `NOT_FOUND` | 404 | Recurso no encontrado |
| `PDF_NOT_AVAILABLE` | 404 | PDF aún no generado |
| `INVALID_STATE` | 409 | Operación no permitida en el estado actual de la forma |
| `DUPLICATE_FORM` | 409 | Ya existe forma para esa combinación |
| `DUPLICATE_WELL` | 409 | Ya existe fila para ese pozo en la forma |
| `RETURN_DEADLINE_EXCEEDED` | 409 | No se puede devolver después del día 19 |
| `EMPTY_GRID` | 409 | No hay filas de producción para enviar |
| `INVALID_COMBINATION` | 422 | Combinación campo/formación/modalidad inválida |
| `CANNOT_DELETE_AVM_WELL` | 422 | Pozo precargado de AVM no eliminable |
| `CANNOT_DELETE_HISTORICAL_WELL` | 422 | Pozo con historial previo no eliminable |
| `RATE_LIMIT_EXCEEDED` | 429 | Límite de solicitudes excedido |
| `AVM_CONNECTION_ERROR` | 502 | Error de conexión con sistema AVM |
| `INTERNAL_ERROR` | 500 | Error interno del servidor |

**Regla de seguridad:** Los mensajes de error NUNCA exponen stack traces, nombres de tablas, rutas de archivo, queries SQL ni detalles de infraestructura interna.

---

## 7. Seguridad

### 7.1 Autenticación
- Todos los endpoints requieren Bearer JWT válido.
- El JWT se emite por `gop.identity` con expiración corta (conforme CONSTITUTION-ba.md §7.1).
- El claim `sub` identifica al usuario; claims de rol y alcance determinan los permisos.

### 7.2 Autorización por Endpoint

| Endpoint | Operador GOP | Supervisor ANH | Admin GOP |
|---|---|---|---|
| GET `/` | ✅ (solo sus contratos) | ✅ (solo contratos asignados) | ✅ (global) |
| POST `/` | ❌ | ❌ | ✅ |
| GET `/{id}` | ✅ (propias) | ✅ (asignadas) | ✅ |
| PATCH `/{id}/header` | ✅ | ❌ | ❌ |
| POST `/{id}/preload` | ✅ | ❌ | ❌ |
| GET `/{id}/well-productions` | ✅ | ✅ | ✅ |
| POST `/{id}/well-productions` | ✅ | ❌ | ❌ |
| PUT `/{id}/well-productions/{wpId}` | ✅ | ❌ | ❌ |
| DELETE `/{id}/well-productions/{wpId}` | ✅ | ❌ | ❌ |
| POST `/{id}/well-productions/bulk-upload` | ✅ | ❌ | ❌ |
| GET `/bulk-upload-template` | ✅ | ❌ | ❌ |
| GET `/{id}/totals` | ✅ | ✅ | ✅ |
| POST/GET `/{id}/cross-validations` | ✅ | ✅ | ✅ |
| POST `/{id}/actions/sign-and-submit` | ✅ | ❌ | ❌ |
| POST `/{id}/actions/approve` | ❌ | ✅ | ❌ |
| POST `/{id}/actions/return` | ❌ | ✅ | ❌ |
| GET `/{id}/change-log` | ❌ | ✅ | ✅ |
| GET `/{id}/pdf` | ✅ | ✅ | ✅ |
| GET `/{id}/versions` | ✅ | ✅ | ✅ |

### 7.3 Filtrado por Tenant
- Operador GOP solo ve formas de contratos asignados a su usuario (RN-03).
- Supervisor ANH solo ve formas de contratos/campos asignados a su usuario.
- Admin GOP tiene visibilidad global.
- El filtrado se aplica en el backend; nunca se confía en filtros del frontend.

### 7.4 Rate Limiting
- Endpoints estándar de lectura: policy `read-standard` (conforme CONSTITUTION-ba.md §12.5).
- Endpoints de escritura/acciones: policy `api-standard`.
- Bulk upload: policy `export-heavy` (límite más estricto por la carga de procesamiento).
- Autenticación: policy `auth-strict`.

---

## 8. Idempotencia

| Endpoint | Idempotente | Mecanismo |
|---|---|---|
| GET (todos) | Sí | Por definición |
| POST `/` (crear) | No | Protección por constraint UNIQUE en BD |
| PATCH `/{id}/header` | Sí | PATCH es idempotente por naturaleza |
| POST `/{id}/preload` | Sí | Re-ejecutar no duplica datos; actualiza no-modificados |
| POST `/{id}/well-productions` | No | Protección por UNIQUE (pozo + forma) |
| POST `/{id}/well-productions/bulk-upload` | Sí | Upsert por nombre de pozo |
| POST `/{id}/cross-validations` | Sí | Recalcula sobre los mismos datos |
| POST `/{id}/actions/sign-and-submit` | Sí* | Si ya está en Submitted, retorna 200 con el estado actual |
| POST `/{id}/actions/approve` | Sí* | Si ya está en Approved, retorna 200 |
| POST `/{id}/actions/return` | No | Cada devolución genera nueva iteración |

---

## 9. Paginación y Ordenamiento

Todos los endpoints de listado siguen la convención estándar:

**Request:** `?page=1&pageSize=20&sort=-createdAt`

**Response:**
```json
{
  "items": [],
  "page": 1,
  "pageSize": 20,
  "total": 0,
  "totalPages": 0
}
```

- `page` es 1-based.
- `pageSize` mínimo 1, máximo depende del endpoint (100 por defecto, 500 para well-productions).
- `sort` acepta nombre de campo; prefijo `-` para descendente.
- Los valores por defecto se aplican si el parámetro se omite.

---

## 10. Referencia Complementaria

- **OpenAPI 3.1 Spec:** `openapi.yaml` (adjunto en este directorio)
- **Spec Frontend:** `spec-frontend.md`
- **Spec Backend:** `spec-backend.md`
- **Modelo de Datos:** `data-model.md` (raíz del repositorio)
