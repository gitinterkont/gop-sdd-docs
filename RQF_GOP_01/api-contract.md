# API Contract — RQF_GOP_01: Creación de Pozo Nuevo

**Versión:** 1.0
**Fecha:** Abril 2026
**Feature:** RQF_GOP_01_CreacionPozoNuevo
**Estado:** Propuesta — pendiente aprobación del arquitecto humano
**Complemento:** `openapi.yaml` (especificación validable OpenAPI 3.1)

---

## 1. Resumen

Contrato HTTP para la funcionalidad de **Creación de Pozo Nuevo** en GOP 360°. Cubre el ciclo completo: creación de borrador, edición, finalización del registro, edición/eliminación post-creación (condicionada a Forma 101 no radicada), y preview de UWI.

**Característica arquitectónica clave:** esta funcionalidad **NO crea `Procedure` ni `WorkflowInstance`**. Crea directamente la entidad `Well` en estado FSM `Registered`. El `Procedure` se creará cuando el Agente inicie la Forma 101 sobre el pozo.

**Recurso principal:** `wells`
**Base path:** `/api/v1`

---

## 2. Actores y Autorización por Endpoint

| Rol | Código interno | Contexto | Endpoints autorizados |
|-----|---------------|----------|----------------------|
| Agente Operadora | `OperatorAgent` | Operadora | Crear, Editar, Consultar, Eliminar, Finalizar |
| Coordinador Operadora | `OperatorCoordinator` | Operadora | Editar (Borrador y Creado sin F101), Consultar |
| Admin GOP ANH | `AnhGopAdministrator` | ANH | Todos (acceso total cross-operadora; creación restringida a Estratigráfico — RN-15) |

---

## 3. Tabla de Endpoints

| # | Método | Ruta | Descripción | Roles | HTTP Éxito |
|---|--------|------|-------------|-------|------------|
| 1 | `POST` | `/wells` | Crear borrador de pozo nuevo | `OperatorAgent`, `AnhGopAdministrator` | `201` |
| 2 | `GET` | `/wells` | Listar pozos (paginado, filtrado) | Todos | `200` |
| 3 | `GET` | `/wells/{id}` | Obtener detalle de un pozo | Todos | `200` |
| 4 | `PUT` | `/wells/{id}` | Actualizar pozo (Borrador o Creado sin F101) | `OperatorAgent`, `OperatorCoordinator`, `AnhGopAdministrator` | `200` |
| 5 | `DELETE` | `/wells/{id}` | Eliminar pozo (Borrador o Creado sin F101) | `OperatorAgent`, `AnhGopAdministrator` | `204` |
| 6 | `POST` | `/wells/{id}/finalize` | Finalizar registro (validación completa, genera Well oficial) | `OperatorAgent`, `AnhGopAdministrator` | `200` |
| 7 | `POST` | `/wells/preview-uwi` | Previsualizar UWI sin persistir | `OperatorAgent`, `AnhGopAdministrator` | `200` |
| 8 | `GET` | `/wells/{id}/parent-well-data` | Obtener datos del pozo padre para ST/PR/G | `OperatorAgent`, `OperatorCoordinator`, `AnhGopAdministrator` | `200` |

**Endpoints de referencia (compartidos):**

| Método | Ruta | Propósito |
|--------|------|-----------|
| `GET` | `/contracts` | Contratos por operadora (Sección 1) |
| `GET` | `/operators` | Operadoras (para Admin ANH) |
| `GET` | `/fields` | Campos por contrato/operadora (Sección 2) |
| `GET` | `/cluster-locations` | Clusters por contrato (Sección 3) |
| `POST` | `/cluster-locations` | Crear cluster nuevo (RN habilitada) |
| `GET` | `/catalogs/{code}/items` | Catálogos cerrados |
| `GET` | `/political-divisions/departments` | Departamentos DANE |
| `GET` | `/political-divisions/departments/{code}/municipalities` | Municipios filtrados |

---

## 4. Detalle por Endpoint

### 4.1 `POST /api/v1/wells` — Crear borrador

**Propósito:** Crea un nuevo pozo en estado `DRAFT`. Genera el UWI fiscalizado y calcula el nombre compuesto server-side. No requiere completitud de todos los campos (borrador).

**Autenticación:** Bearer JWT
**Roles:** `OperatorAgent`, `AnhGopAdministrator`
**Restricción RN-15:** Si el usuario es ANH (`Role.Context = Anh`), `classificationCode` debe ser `STRATIGRAPHIC`.

**Request Body:** `application/json`

```json
{
  "contractId": "3fa85f64-5717-4562-b3fc-2c963f66afa6",
  "trajectoryTypeCode": "O",
  "parentWellId": null,
  "classificationCode": "DEVELOPMENT",
  "subClassificationCode": null,
  "denomination": "RUBIALES",
  "consecutive": "157",
  "fieldId": "9fa85f64-5717-4562-b3fc-2c963f66afa6",
  "angleTypeCode": "V",
  "objectiveCode": "PH",
  "completionTypeCode": "CC",
  "departmentDaneCode": "50",
  "municipalityDaneCode": "568",
  "clusterLocationId": "1fa85f64-5717-4562-b3fc-2c963f66afa6"
}
```

**Response `201 Created`:**

```json
{
  "id": "00000000-0000-0000-0000-000000000001",
  "wellName": "RUBIALES 157O",
  "fiscalizedUwi": "50568RUBI0157RU0000VOOPH–CC",
  "status": "DRAFT",
  "operatorId": "2fa85f64-5717-4562-b3fc-2c963f66afa6",
  "operatorName": "Ecopetrol S.A.",
  "createdAt": "2026-04-22T14:30:00Z",
  "createdBy": "john.doe@example.com"
}
```

**Errores:**

| HTTP | Código | Condición |
|------|--------|-----------|
| `400` | `VALIDATION_ERROR` | Payload malformado |
| `401` | `UNAUTHORIZED` | Token ausente/inválido |
| `403` | `FORBIDDEN` | Rol sin permiso o ANH intenta clasificación ≠ Estratigráfico |
| `409` | `DUPLICATE_WELL_NAME` | Nombre compuesto calculado ya existe |
| `409` | `DUPLICATE_UWI` | UWI generado ya existe |
| `422` | `BUSINESS_RULE_VIOLATION` | Regla de negocio violada (RN-03 a RN-22) |

---

### 4.2 `GET /api/v1/wells` — Listar pozos

**Propósito:** Lista pozos con paginación, filtrado por tenant automático.

**Query Parameters:**

| Parámetro | Tipo | Default | Descripción |
|-----------|------|---------|-------------|
| `page` | `integer` | `1` | Página |
| `pageSize` | `integer` | `20` | Tamaño (max 100) |
| `sort` | `string` | `createdAt:desc` | Ordenamiento |
| `status` | `string` | — | `DRAFT`, `CREATED` |
| `contractId` | `uuid` | — | Filtro por contrato |
| `wellName` | `string` | — | Búsqueda parcial |
| `fiscalizedUwi` | `string` | — | Búsqueda parcial |
| `classificationCode` | `string` | — | Filtro por clasificación |
| `isLegacyWell` | `boolean` | — | `false` = solo nuevos, `true` = solo antiguos |

**Response `200 OK`:**

```json
{
  "items": [
    {
      "id": "00000000-0000-0000-0000-000000000001",
      "wellName": "RUBIALES 157O",
      "fiscalizedUwi": "50568RUBI0157RU0000VOOPH–CC",
      "status": "CREATED",
      "operatorName": "Ecopetrol S.A.",
      "contractCode": "E&P RUBIALES",
      "fieldName": "RUBIALES",
      "classificationCode": "DEVELOPMENT",
      "departmentName": "Meta",
      "municipalityName": "Puerto Gaitán",
      "hasFiledForm101": false,
      "createdAt": "2026-04-22T14:30:00Z"
    }
  ],
  "page": 1,
  "pageSize": 20,
  "total": 1,
  "totalPages": 1
}
```

---

### 4.3 `GET /api/v1/wells/{id}` — Detalle del pozo

**Response `200 OK`:**

```json
{
  "id": "00000000-0000-0000-0000-000000000001",
  "wellName": "RUBIALES 157O",
  "fiscalizedUwi": "50568RUBI0157RU0000VOOPH–CC",
  "status": "CREATED",
  "isLegacyWell": false,
  "operatorId": "2fa85f64-5717-4562-b3fc-2c963f66afa6",
  "operatorName": "Ecopetrol S.A.",
  "contract": {
    "contractId": "3fa85f64-5717-4562-b3fc-2c963f66afa6",
    "contractCode": "E&P RUBIALES",
    "contractType": "Exploración y Producción",
    "basinName": "LLANOS ORIENTALES"
  },
  "trajectoryTypeCode": "O",
  "trajectoryTypeName": "Original",
  "parentWellId": null,
  "parentWellName": null,
  "classificationCode": "DEVELOPMENT",
  "classificationName": "Desarrollo",
  "subClassificationCode": null,
  "denomination": "RUBIALES",
  "consecutive": "157",
  "fieldId": "9fa85f64-5717-4562-b3fc-2c963f66afa6",
  "fieldName": "RUBIALES",
  "locationType": "CONTINENTAL",
  "angleTypeCode": "V",
  "angleTypeName": "Vertical",
  "objectiveCode": "PH",
  "objectiveName": "Productor Hidrocarburos",
  "completionTypeCode": "CC",
  "completionTypeName": "Casing cementado y cañoneado",
  "departmentDaneCode": "50",
  "departmentName": "Meta",
  "municipalityDaneCode": "568",
  "municipalityName": "Puerto Gaitán",
  "clusterLocationId": "1fa85f64-5717-4562-b3fc-2c963f66afa6",
  "clusterLocationName": "RUBIALES A",
  "uwiComponents": {
    "departmentCode": "50",
    "municipalityCode": "568",
    "wellNameSigle": "RUBI",
    "wellNumber": "0157",
    "clusterCode": "RU0000",
    "angleCode": "V",
    "trajectoryCode": "O",
    "objectiveCode": "PH",
    "completionCode": "CC"
  },
  "hasFiledForm101": false,
  "currentStateCode": "REGISTERED",
  "currentStateName": "Registrado",
  "createdAt": "2026-04-22T14:30:00Z",
  "createdBy": "john.doe@example.com",
  "updatedAt": "2026-04-22T14:30:00Z",
  "updatedBy": "john.doe@example.com"
}
```

---

### 4.4 `PUT /api/v1/wells/{id}` — Actualizar pozo

**Precondiciones:** `status` ∈ {`DRAFT`, `CREATED`} **Y** `hasFiledForm101 = false` (RN-40).

**Request Body:** Mismo esquema que `POST` (sección 4.1).

**Headers de concurrencia:** `If-Match` / `ETag`.

**Errores adicionales:**

| HTTP | Código | Condición |
|------|--------|-----------|
| `409` | `WELL_LOCKED_BY_FORM101` | F101 radicada — pozo no editable (RN-40) |
| `409` | `CONCURRENT_MODIFICATION` | ETag mismatch |

---

### 4.5 `DELETE /api/v1/wells/{id}` — Eliminar pozo

**Precondiciones:** `status` ∈ {`DRAFT`, `CREATED`} **Y** `hasFiledForm101 = false` (RN-40).

**Response:** `204 No Content`

**Errores:**

| HTTP | Código | Condición |
|------|--------|-----------|
| `409` | `WELL_LOCKED_BY_FORM101` | F101 radicada — no eliminable |

---

### 4.6 `POST /api/v1/wells/{id}/finalize` — Finalizar registro

**Propósito:** Valida todos los 18 campos obligatorios, verifica unicidad de nombre y UWI, y transiciona de `DRAFT` a `CREATED`. El pozo queda disponible para F101.

**Precondición:** `status = DRAFT`

**Request Body:** Vacío (la data ya fue guardada vía PUT/POST).

**Response `200 OK`:**

```json
{
  "id": "00000000-0000-0000-0000-000000000001",
  "wellName": "RUBIALES 157O",
  "fiscalizedUwi": "50568RUBI0157RU0000VOOPH–CC",
  "status": "CREATED",
  "currentStateCode": "REGISTERED",
  "finalizedAt": "2026-04-22T15:00:00Z"
}
```

**Errores:**

| HTTP | Código | Condición |
|------|--------|-----------|
| `422` | `INCOMPLETE_WELL` | Campos obligatorios faltantes (lista en `details`) |
| `409` | `DUPLICATE_WELL_NAME` | Nombre duplicado |
| `409` | `DUPLICATE_UWI` | UWI duplicado |

---

### 4.7 `POST /api/v1/wells/preview-uwi` — Preview UWI

**Request Body:**

```json
{
  "departmentDaneCode": "50",
  "municipalityDaneCode": "568",
  "denomination": "RUBIALES",
  "consecutive": "157",
  "clusterLocationName": "RUBIALES A",
  "angleTypeCode": "V",
  "trajectoryTypeCode": "O",
  "objectiveCode": "PH",
  "completionTypeCode": "CC",
  "classificationCode": "DEVELOPMENT"
}
```

**Response `200 OK`:**

```json
{
  "fiscalizedUwi": "50568RUBI0157RU0000VOOPH–CC",
  "wellName": "RUBIALES 157O",
  "components": {
    "departmentCode": "50",
    "municipalityCode": "568",
    "wellNameSigle": "RUBI",
    "wellNumber": "0157",
    "clusterCode": "RU0000",
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

### 4.8 `GET /api/v1/wells/{id}/parent-well-data` — Datos del pozo padre

**Propósito:** Para trayectorias ST/PR/G, retorna los datos del pozo padre para precargar el formulario (RN-08/RN-10).

**Query Parameters:** `parentWellId` (uuid, requerido)

**Response `200 OK`:**

```json
{
  "parentWellId": "bfa85f64-5717-4562-b3fc-2c963f66afa6",
  "parentWellName": "RUBIALES 157O",
  "basinName": "LLANOS ORIENTALES",
  "locationType": "CONTINENTAL",
  "departmentDaneCode": "50",
  "departmentName": "Meta",
  "municipalityDaneCode": "568",
  "municipalityName": "Puerto Gaitán",
  "clusterLocationId": "1fa85f64-5717-4562-b3fc-2c963f66afa6",
  "clusterLocationName": "RUBIALES A",
  "angleTypeCode": "V",
  "objectiveCode": "PH",
  "completionTypeCode": "CC",
  "hasApprovedPluggingProgram": true,
  "pluggingProgramWarning": null
}
```

> **Nota RN-09:** `hasApprovedPluggingProgram` indica si el pozo padre tiene F108/109 aprobada. Si `false` y trayectoria = ST, el campo `pluggingProgramWarning` contiene el texto de advertencia. En V1, es warning no bloqueante (supuesto A-01 confirmado).

---

## 5. Códigos de Error Estandarizados

```json
{
  "code": "BUSINESS_RULE_VIOLATION",
  "message": "La denominación contiene siglas no permitidas.",
  "details": [
    {
      "field": "denomination",
      "rule": "RN-17",
      "message": "No se permiten las siglas ST, G, PR, P, ML ni la palabra Piloto."
    }
  ],
  "traceId": "00000000-0000-0000-0000-000000000099"
}
```

| Código | HTTP | Descripción |
|--------|------|-------------|
| `VALIDATION_ERROR` | `400` | Payload malformado |
| `UNAUTHORIZED` | `401` | Token ausente/inválido |
| `FORBIDDEN` | `403` | Sin permiso |
| `NOT_FOUND` | `404` | Pozo no existe (o fuera del tenant) |
| `DUPLICATE_WELL_NAME` | `409` | Nombre ya existe |
| `DUPLICATE_UWI` | `409` | UWI ya existe |
| `WELL_LOCKED_BY_FORM101` | `409` | F101 radicada — inmutable |
| `CONCURRENT_MODIFICATION` | `409` | ETag mismatch |
| `BUSINESS_RULE_VIOLATION` | `422` | Regla de negocio (con detalle) |
| `INCOMPLETE_WELL` | `422` | Campos obligatorios faltantes |
| `INVALID_CLASSIFICATION_FOR_ANH` | `422` | ANH solo puede crear Estratigráfico (RN-15) |
| `PARENT_WELL_REQUIRED` | `422` | Trayectoria ST/PR/G sin pozo padre (RN-03) |
| `FORBIDDEN_SIGLA_IN_NAME` | `422` | Denominación contiene sigla reservada (RN-17) |
| `RATE_LIMIT_EXCEEDED` | `429` | Límite excedido |
| `INTERNAL_ERROR` | `500` | Error interno |

---

## 6. Seguridad

### 6.1 Autenticación
- Bearer JWT emitido por `gop.identity`. Todos los endpoints requieren autenticación.

### 6.2 Autorización
- Filtrado por `User.OperatorId` para operadoras.
- ANH ve según alcance de `UserRole`.
- **RN-15:** Admin ANH solo puede crear con `classificationCode = STRATIGRAPHIC`.
- Coordinador no puede crear, eliminar ni finalizar.

### 6.3 Rate Limiting
- Escritura: 30 req/min por usuario.
- Lectura: 120 req/min por usuario.
- Preview UWI: 60 req/min por usuario.

---

## 7. Convenciones

| Concepto | Convención |
|----------|------------|
| IDs | UUID v4 |
| Fechas | ISO 8601 UTC |
| Rutas | kebab-case, plural |
| Paginación | `{ items, page, pageSize, total, totalPages }` |
| Concurrencia | ETag / If-Match en PUT |
| Tenant | Implícito desde JWT |

---

## 8. Puntos de Decisión Pendientes

| # | Punto | Impacto |
|---|-------|---------|
| PD-01 | PA-01: Fuente de datos maestros. Se abstrae como catálogos internos. | Bajo |
| PD-02 | Catálogo exacto de orígenes para pozos ANH estratigráficos (contratos VAF) | Endpoint `/contracts` filtrado |

---

## 9. Anexos

- `openapi.yaml` — Especificación OpenAPI 3.1
- `spec-frontend.md` — Especificación frontend
- `spec-backend.md` — Especificación backend
