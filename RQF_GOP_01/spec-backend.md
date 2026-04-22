# Especificación Backend — RQF_GOP_01: Creación de Pozo Nuevo

**Versión:** 1.0
**Fecha:** Abril 2026
**Feature:** RQF_GOP_01_CreacionPozoNuevo
**Estado:** Propuesta — pendiente aprobación del arquitecto humano

---

## 1. Contexto y Objetivo

Implementar la lógica de negocio, persistencia y seguridad para la creación de pozos nuevos. Característica arquitectónica: **NO crea `Procedure` ni `WorkflowInstance`**. El Well se crea directamente en el esquema `ops` en estado FSM `Registered`. El backend es responsable de:
- Componer el nombre del pozo desde campos separados (denominación + consecutivo + sufijos) aplicando 22 reglas de nomenclatura.
- Generar el UWI fiscalizado algorítmicamente (RN-27 a RN-39).
- Validar unicidad de nombre y UWI.
- Persistir el Well con sus relaciones (`FiscalizedUwiComponent`, `WellStateHistory`).
- Controlar edición/eliminación condicionada a F101 no radicada (RN-40).

---

## 2. Actores, Roles y Permisos

### 2.1 Matriz de autorización

| Endpoint / Acción | OperatorAgent | OperatorCoordinator | AnhGopAdministrator |
|--------------------|:---:|:---:|:---:|
| Crear borrador (POST) | ✅ | ❌ | ✅ (solo Estratigráfico) |
| Listar (GET list) | ✅ | ✅ | ✅ |
| Ver detalle (GET id) | ✅ | ✅ | ✅ |
| Editar (PUT) — si no hay F101 | ✅ | ✅ | ✅ |
| Eliminar (DELETE) — si no hay F101 | ✅ | ❌ | ✅ |
| Finalizar (POST finalize) | ✅ | ❌ | ✅ |
| Preview UWI | ✅ | ✅ | ✅ |
| Datos pozo padre | ✅ | ✅ | ✅ |

### 2.2 Filtrado por tenant
- Operadora: todo filtrado por `User.OperatorId`.
- ANH: según alcance de `UserRole`.
- **RN-15:** al crear, si `User.Role.Context = Anh`, el backend **rechaza** cualquier `classificationCode ≠ STRATIGRAPHIC` con `422 INVALID_CLASSIFICATION_FOR_ANH`.

---

## 3. Criterios de Aceptación Backend

### CA-BE-01 — Composición de nombre server-side
**Given** `denomination = "RUBIALES"`, `consecutive = "157"`, `trajectoryTypeCode = "O"`
**When** se crea el pozo
**Then** el backend calcula `wellName = "RUBIALES-157O"` aplicando RN-04 a RN-22.

### CA-BE-02 — Generación de UWI server-side
**Given** datos de identificación completos
**When** se crea/actualiza
**Then** el UWI se genera algorítmicamente (RN-27 a RN-39), se verifica unicidad, y se persiste con sus componentes en `FiscalizedUwiComponent`.

### CA-BE-03 — Well en estado Registered
**Given** se finaliza un pozo nuevo
**When** se persiste
**Then** se crea `WellStateHistory` con `NewStateId = Registered`, `TriggerEvent = 'NewWellCreation'`, y se actualiza `Well.CurrentStateId`.

### CA-BE-04 — Bloqueo por F101
**Given** el pozo tiene una `Procedure` tipo F101 con `DomainStatus ∈ {Filed, UnderReview, Approved}`
**When** se intenta PUT o DELETE
**Then** se retorna `409 WELL_LOCKED_BY_FORM101`.

### CA-BE-05 — ST auto-nombre
**Given** trayectoria = ST, pozo padre = "Ballena-1"
**When** el backend calcula el nombre
**Then** cuenta los ST existentes del padre en BD, incrementa: "Ballena-1ST2" (si ya existe ST1).

---

## 4. Reglas de Negocio

### 4.1 Reglas de nomenclatura del pozo (RN-03 a RN-22)

| ID | Regla | Lógica backend |
|----|-------|----------------|
| RN-03 | ST/PR/G → seleccionar pozo padre del mismo contrato | Validar que `parentWellId` exista, pertenezca al mismo contrato, y esté en estado válido |
| RN-04 | ST → nombre = `[PadreName]ST[N]` | N = `COUNT(wells WHERE originalWell = padre AND trajectory = ST) + 1` |
| RN-05 | G → nombre = `[PadreName]G[N]` | Análogo |
| RN-06 | PR → nombre = `[PadreName]PR` | Sin consecutivo |
| RN-07 | P/ML/O → nombre = `[prefijo] [denominación]-[consecutivo][sufijo]` | Sufijo: P, ML, O según trayectoria |
| RN-08 | ST/PR/G → heredar: cuenca, ubicación, departamento, municipio, cluster del padre | Backend copia valores del padre; rechaza si el request los contradice |
| RN-09 | ST → validar programa de taponamiento del padre | V1: warning (campo en response), no bloquea |
| RN-10 | ST/PR/G → cargar datos del padre | Endpoint `parent-well-data` retorna atributos del `Well` padre |
| RN-11 | Exploratorio A3 → campo = "CAMPO EXPLORATORIO" | Backend fuerza `fieldId = null`, `fieldName = "CAMPO EXPLORATORIO"` |
| RN-12 | Exploratorio A2/A1 → dropdown campos + campo de A3 si F103 Lahee B3 | Backend consulta: ¿existe A3 en el contrato con Procedure F103 Approved y LaheeClassification = B3? Si sí, incluye su campo en la lista |
| RN-13 | Desarrollo → campos aprobados del contrato | Backend filtra campos con `Status = Commercial` |
| RN-14 | Estratigráfico → campo = "N/A" | Backend fuerza `fieldId = null` |
| RN-15 | ANH solo Estratigráfico | Backend verifica `Role.Context` al crear |
| RN-16 | Mensaje de nomenclatura al usuario | Solo UI; backend valida la estructura |
| RN-17 | Denominación sin siglas reservadas (ST, G, PR, P, ML, PILOTO) | Backend verifica que `denomination` no contenga estas palabras como token o sufijo |
| RN-18 | Sin caracteres después de la numeración | Backend verifica patrón del consecutivo: solo dígitos |
| RN-19 | Nombre único | Backend verifica `UNIQUE` en `Well.Name` (case-insensitive) |
| RN-20 | Desarrollo: ¿nombre del campo? | Si `useFieldNameAsPrefix = true`, el backend usa el nombre del campo como prefijo |
| RN-21 | Primer A3/Estratigráfico: nombre + "-1" | Backend detecta si es primero por contrato y fuerza consecutivo "1" |
| RN-22 | 2do+ Estratigráfico: consecutivo auto o nuevo nombre | Si `denomination` vacía pero consecutivo presente → auto-incrementa desde el último |

### 4.2 Reglas de UWI (RN-27 a RN-39)

Misma lógica que en `spec-backend.md` de RQF_GOP_25 (§4.4), con la diferencia de que aquí el UWI se genera al **crear** el pozo, no al aprobar un registro.

**Particularidades:**
- RN-30: Sigla para estratigráficos ANH: prefijo `ANH` + 2 primeras letras del nombre (ej: "ANH MORICHE" → `ANHMO`).
- RN-32: Si cluster = pozo → sufijo `C` en lugar de sigla normal.
- RN-34: Para ST, el consecutivo del UWI es el mismo que el del nombre (ST1, ST2...).

### 4.3 Regla de bloqueo post-F101 (RN-40)

**Determinación de `hasFiledForm101`:**
- El backend consulta: ¿existe un `Procedure` con `WellId = este pozo` y `FormDefinition.FormCode = 'F101'` y `DomainStatus ∈ {Filed, UnderReview, Approved}`?
- Si sí: PUT y DELETE retornan `409 WELL_LOCKED_BY_FORM101`.
- Este check se ejecuta en cada PUT/DELETE, no se cachea (la F101 puede radicarse entre requests).

---

## 5. Modelo de Datos Impactado

### 5.1 Entidades existentes utilizadas (sin cambios de esquema)

| Entidad | Uso |
|---------|-----|
| `ops.Well` | Se crea directamente con `IsLegacyWell = false`, `IsFinalized = false` (borrador) o `true` (finalizado) |
| `ops.FiscalizedUwiComponent` | Se crea con los 8 componentes del UWI |
| `ops.WellStateHistory` | Se crea al finalizar con `TriggerEvent = 'NewWellCreation'`, `NewStateId = Registered` |
| `catalog.FormDefinition` | Referenciado para verificar F101 radicada (RN-40) |
| `procedure.Procedure` | Consultado para verificar F101 (no creado por esta feature) |
| `core.Contract`, `core.Field`, `core.ClusterLocation`, `core.Operator` | Referenciados por FKs |
| `catalog.PoliticalDivision` | Departamentos y municipios |

### 5.2 Cambios propuestos

**Ninguno.** Todas las entidades necesarias ya existen en `data-model.md` v0.6. La feature utiliza `Well`, `FiscalizedUwiComponent` y `WellStateHistory` tal como están definidos. No se requieren entidades nuevas porque:
- No hay payload extenso (solo 18 campos que mapean directamente a atributos de `Well`).
- No hay Procedure.
- No hay tablas dinámicas.

### 5.3 Atributo `Well.Status` para borrador

El `data-model.md` v0.6 define `Well.IsFinalized` como flag. Para esta feature:
- `IsFinalized = false` → estado `DRAFT` (borrador, aún no finalizado).
- `IsFinalized = true` → estado `CREATED` (finalizado, disponible para F101).
- `Well.CurrentStateId` se puebla solo al finalizar (`Registered`).

---

## 6. Operaciones Transaccionales

### 6.1 Atómicas

| Operación | Entidades | Justificación |
|-----------|-----------|---------------|
| Crear borrador | `Well` + `FiscalizedUwiComponent` | Pozo y UWI son indisociables |
| Finalizar | `Well` (update `IsFinalized`) + `WellStateHistory` | Estado y log son indisociables |
| Eliminar | `Well` + `FiscalizedUwiComponent` + `WellStateHistory` (si existe) | Cascada completa |

### 6.2 Eventual consistency
- Validación contra Mapa de Tierras (RN-25/26): si el servicio no responde, se permite la creación con warning y se registra `ProcedureSpatialValidation` con `Result = Unknown`.
- Logs de auditoría: Audit.NET post-commit.

---

## 7. Integraciones Externas

| Sistema | Dato | Manejo de fallas |
|---------|------|------------------|
| SOLAR/VCH/VPAA (vía `gop.core`) | Operadoras, Contratos, Tipos, Cuencas | Catálogos precargados |
| SOLAR (vía `gop.core`) | Campos por contrato | Catálogos precargados |
| Mapa de Tierras (vía `gop.integration`) | Validación Dpto/Mpio vs polígono | Si no disponible: warning, no bloquea |
| DANE | Códigos DANE | Catálogo `PoliticalDivision` precargado |
| VAF (vía `gop.integration`) | Contratos para ANH estratigráficos | Si no disponible: contratos ya sincronizados en `core.Contract` |

---

## 8. NFRs Backend

### 8.1 Performance

| Métrica | Objetivo |
|---------|----------|
| Latencia p95 — POST crear | < 500ms |
| Latencia p95 — PUT actualizar | < 400ms |
| Latencia p95 — POST finalize | < 600ms |
| Latencia p95 — POST preview-uwi | < 200ms |
| Latencia p99 — cualquier endpoint | < 2s |

### 8.2 Escalabilidad

| Métrica | Estimación |
|---------|------------|
| Pozos nuevos por día | 10-50 |
| Concurrencia (usuarios creando) | < 30 |
| Crecimiento anual | ~2000-5000 pozos |

### 8.3 Disponibilidad
- SLA: 99.9%. RTO: 4h. RPO: 1h.

### 8.4 Observabilidad
- Métricas: pozos creados/finalizados por día, latencia, errores.
- Trazas: `traceId` en todos los logs.
- Logs: cada generación de UWI (con inputs), cada intento de nombre/UWI duplicado, cada bloqueo por F101.

---

## 9. Seguridad Backend — OWASP Top 10

### A01 — Broken Access Control
- Autorización por endpoint según §2.1. Tenant isolation por `User.OperatorId`.
- RN-15 enforced server-side (ANH solo Estratigráfico).
- Verificación de ownership en PUT/DELETE.
- Un Agente de Operadora A no puede ver ni modificar pozos de Operadora B (retorna `404`, no `403`).

### A02 — Cryptographic Failures
- TLS 1.2+ obligatorio.
- No hay datos sensibles propios de esta feature (no matrículas, no coordenadas PII).
- Secretos de integraciones en servicio de secretos.

### A03 — Injection
- Todas las consultas parametrizadas. **PROHIBIDO** concatenar strings.
- Denominación y nombre del pozo sanitizados (longitud, caracteres de control).
- No se ejecuta SQL dinámico para generación de UWI.

### A04 — Insecure Design
- **Amenaza:** Manipulación del UWI en el payload → UWI se genera server-side, cualquier valor del cliente se ignora.
- **Amenaza:** Manipulación de `operatorId` → se usa `User.OperatorId` del JWT.
- **Amenaza:** Creación masiva de pozos para agotar UWIs → rate limiting.
- Principio de menor privilegio en permisos de BD.

### A05 — Security Misconfiguration
- Headers: HSTS, X-Content-Type-Options, X-Frame-Options, CSP.
- CORS restrictivo. No exponer versiones en headers.
- Errores uniformes sin stack traces en producción.

### A06 — Vulnerable & Outdated Components
- Dependencias auditadas y bloqueadas.

### A07 — Identification & Authentication Failures
- JWT con expiración corta + refresh tokens (gestionado por `gop.identity`).
- Rate limiting: 30 req/min escritura, 120 req/min lectura.

### A08 — Software & Data Integrity Failures
- Optimistic concurrency vía ETag.
- UWI inmutable post-generación (RN-38).

### A09 — Security Logging & Monitoring Failures
- Logging estructurado: `Create`, `Update`, `Delete`, `StateTransition` (`NewWellCreation`), `AccessDenied`.
- `traceId` propagado. Nunca loguear tokens.

### A10 — SSRF
- No aplica directamente. Las validaciones contra Mapa de Tierras usan URLs configuradas, no inputs del usuario.

---

## 10. Edge Cases Backend

| # | Escenario | Respuesta | HTTP |
|---|-----------|-----------|------|
| EC-BE-01 | Token inválido/expirado | `UNAUTHORIZED` | `401` |
| EC-BE-02 | Sin rol para la acción | `FORBIDDEN` | `403` |
| EC-BE-03 | Acceso a pozo de otra operadora | `NOT_FOUND` | `404` |
| EC-BE-04 | Nombre duplicado | `DUPLICATE_WELL_NAME` | `409` |
| EC-BE-05 | UWI duplicado | `DUPLICATE_UWI` | `409` |
| EC-BE-06 | Editar/eliminar con F101 radicada | `WELL_LOCKED_BY_FORM101` | `409` |
| EC-BE-07 | Concurrencia (ETag mismatch) | `CONCURRENT_MODIFICATION` | `409` |
| EC-BE-08 | ANH intenta clasificación ≠ Estratigráfico | `INVALID_CLASSIFICATION_FOR_ANH` | `422` |
| EC-BE-09 | ST/PR/G sin pozo padre | `PARENT_WELL_REQUIRED` | `422` |
| EC-BE-10 | Denominación con sigla reservada | `FORBIDDEN_SIGLA_IN_NAME` | `422` |
| EC-BE-11 | Campos obligatorios faltantes al finalizar | `INCOMPLETE_WELL` | `422` |
| EC-BE-12 | Pozo padre de otro contrato | `BUSINESS_RULE_VIOLATION` | `422` |
| EC-BE-13 | Mapa de Tierras no disponible | Creación permitida con warning; `ProcedureSpatialValidation.Result = Unknown` | `200` con warnings |
| EC-BE-14 | Rate limiting | `RATE_LIMIT_EXCEEDED` | `429` |
| EC-BE-15 | Finalizar pozo ya finalizado | `INVALID_STATE_TRANSITION` | `409` |
| EC-BE-16 | Consecutivo con letras | `VALIDATION_ERROR` (RN-18) | `400` |

---

## 11. Criterios de Testing

### 11.1 Cobertura mínima
- **85%** en lógica de dominio (composición de nombre — todos los escenarios de RN-03 a RN-22, generación UWI — todos los escenarios de RN-27 a RN-39, validaciones).
- **80%** en capa de aplicación.

### 11.2 Escenarios de seguridad obligatorios

| # | Escenario | Verificación |
|---|-----------|-------------|
| ST-01 | Agente de Op A intenta GET pozo de Op B | `404` |
| ST-02 | Coordinador intenta POST crear | `403` |
| ST-03 | Coordinador intenta DELETE | `403` |
| ST-04 | ANH crea con clasificación Desarrollo | `422 INVALID_CLASSIFICATION_FOR_ANH` |
| ST-05 | Inyección SQL en denominación | Parametrizada; sin efecto |
| ST-06 | Manipulación de operatorId en payload | Ignorado; usa JWT |
| ST-07 | PUT con ETag aleatorio | `409` |
| ST-08 | PUT sobre pozo con F101 radicada | `409 WELL_LOCKED_BY_FORM101` |

---

## 12. Supuestos Confirmados

| ID | Supuesto |
|----|----------|
| S-01 | No crea Procedure ni Workflow; Well directo |
| S-02 | Nombre compuesto: denominación + consecutivo + sufijo |
| S-04 | Sigla ANH: "ANH MORICHE" → ANHMO |
| S-05 | RN-09: warning no bloqueante en V1 |
| S-06 | Datos del padre = atributos del Well |
| S-07 | "A3 asociado" = por contrato |
| S-08 | RN-20: pregunta interactiva |
| S-09 | Primer estratigráfico = por contrato |
| S-10 | Offshore: Dpto/Mpio opcionales |
| S-11 | Catálogos internos GOP |

---

## 13. Puntos de Decisión Pendientes

| # | Punto | Impacto |
|---|-------|---------|
| PD-BE-01 | PA-01: Fuente datos maestros. Se abstrae. | Bajo |
| PD-BE-02 | Estrategia de eliminación: soft/hard delete | Modelo |
| PD-BE-03 | Validación Mapa de Tierras: síncrona (bloquea) o asíncrona (warning) | Integración |

---

## 14. Anexos

- `api-contract.md` — Contrato API
- `openapi.yaml` — OpenAPI 3.1
- `spec-frontend.md` — Spec frontend
