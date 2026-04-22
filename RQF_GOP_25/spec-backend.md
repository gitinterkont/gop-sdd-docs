# Especificación Backend — RQF_GOP_25: Registro de Pozo Antiguo / Perforado

**Versión:** 1.0
**Fecha:** Abril 2026
**Feature:** RQF_GOP_25_RegistroPozoAntiguo
**Estado:** Propuesta — pendiente aprobación del arquitecto humano

---

## 1. Contexto y Objetivo

Implementar en el backend de GOP 360° la lógica de negocio, persistencia, seguridad y flujo de aprobación para el registro de pozos antiguos/perforados. A diferencia de la Creación de Pozo Nuevo (RQF_GOP_01), esta feature involucra:
- Un formulario extenso (~104 campos, 16 secciones, 6 tablas dinámicas de filas variables).
- Un flujo de aprobación ANH (Borrador → Enviado → Aprobado/Devuelto).
- Creación de la entidad `Well` en el dominio **solo tras aprobación ANH**.
- Generación de UWI fiscalizado, PDF condicional, y notificaciones.
- Tres conjuntos de coordenadas almacenados con transformación al CRS canónico.

---

## 2. Actores, Roles y Permisos

### 2.1 Matriz de autorización

| Endpoint / Acción | OperatorAgent | OperatorCoordinator | AnhOperationsEngineer | AnhGopAdministrator |
|--------------------|:---:|:---:|:---:|:---:|
| Crear registro | ✅ | ❌ | ❌ | ✅ |
| Listar registros | ✅ | ✅ | ✅ | ✅ |
| Ver detalle | ✅ | ✅ | ✅ | ✅ |
| Editar (Draft/Returned) | ✅ | ✅ | ❌ | ✅ |
| Eliminar (Draft/Returned) | ✅ | ❌ | ❌ | ✅ |
| Enviar a aprobación | ✅ | ❌ | ❌ | ✅ |
| Aprobar | ❌ | ❌ | ✅ | ✅ |
| Devolver | ❌ | ❌ | ✅ | ✅ |
| Descargar PDF | ✅ | ✅ | ✅ | ✅ |
| Subir adjuntos | ✅ | ✅ | ❌ | ✅ |
| Eliminar adjuntos | ✅ | ✅ | ❌ | ✅ |

### 2.2 Filtrado por tenant
- **Operadora:** todos los endpoints filtran implícitamente por `User.OperatorId`. Un usuario operadora **nunca** ve ni accede a registros de otra operadora.
- **ANH:** alcance determinado por `UserRole` — puede ser global, por operador, por contrato, por campo.
- **Admin GOP:** acceso total cross-operadora.

---

## 3. Historias de Usuario y Criterios de Aceptación

Referencia completa a secciones 3 y 4 de `spec-frontend.md`. Los criterios Given/When/Then del frontend aplican al backend en sus equivalentes de validación y lógica de negocio. A continuación, los criterios **específicos del backend** no cubiertos en el frontend.

### CA-BE-01 — Generación atómica de UWI
**Given** se crea un registro
**When** se procesan los datos de identificación
**Then** el sistema genera el UWI fiscalizado según RN-33 a RN-41, verifica unicidad en BD, y si colisiona, retorna `409 DUPLICATE_UWI` sin crear el registro.

### CA-BE-02 — Creación de Well al aprobar
**Given** un Ingeniero ANH aprueba un registro
**When** la aprobación se procesa
**Then** el sistema crea atómicamente: `Well` con `IsLegacyWell = true`, `WellLocation` (CRS canónico), `FiscalizedUwiComponent`, `WellFormation` (por cada formación del registro), `WellCasing` (por cada tubería), `WellStateHistory` con `TriggerEvent = LegacyWellRegistration` y `CurrentStateId` mapeado según tabla de equivalencia (sección 4.5), `ProcedureSignature` (operador + ANH), `ProcedurePdf` tipo `PostAnhApproval`.

### CA-BE-03 — Validación completa al enviar
**Given** un Agente envía a aprobación
**When** se procesan las validaciones
**Then** se validan los ~104 campos obligatorios (según condiciones: Continental/Offshore, hasForm103, Objetivo, etc.), se verifica coherencia cross-field, y se retorna `422 INCOMPLETE_REGISTRATION` con la lista completa de errores si falla.

### CA-BE-04 — Inmutabilidad post-aprobación
**Given** un registro en estado APPROVED
**When** se intenta editar o eliminar
**Then** se retorna `409 INVALID_STATE_TRANSITION`.

### CA-BE-05 — Transformación de coordenadas al CRS canónico
**Given** coordenadas geográficas del registro (Sección 7)
**When** se aprueba el registro
**Then** se transforman las coordenadas geográficas (lat/lon) al tipo `geography` POINT WGS84 (EPSG:4326) para almacenamiento en `WellLocation.Point`. Los tres conjuntos de coordenadas se preservan en `OriginalReportJson`.

---

## 4. Reglas de Negocio

### 4.1 Reglas de contrato e identidad

| ID | Regla | Validación | Error |
|----|-------|------------|-------|
| RN-01 | Operadora determinada automáticamente desde JWT claims | Backend inyecta `User.OperatorId`; ignora cualquier operatorId del payload | N/A |
| RN-02 | ANH puede seleccionar contratos de VAF + "Área Disponible" | Verificar que si `User.Role.Context = Anh`, se permiten contratos VAF y la opción especial | `422` si contrato inválido para el rol |
| RN-03 | "Área Disponible" habilita campo como texto libre | Si `contractId` corresponde al código especial "AREA_DISPONIBLE", `fieldId` puede ser null y `fieldNameFreeText` no vacío | `422` si ambos son null |
| RN-04 | Operador Inicial: "OTRO" habilita texto libre | Si `initialOperatorId = null`, `initialOperatorFreeText` es requerido y viceversa | `422` si ambos son null o ambos tienen valor |

### 4.2 Reglas de clasificación

| ID | Regla | Validación | Error |
|----|-------|------------|-------|
| RN-07 | Exclusión mutua: Lahee XOR Desarrollo/Estratigráfico | Exactamente uno de `laheeClassificationCode` o `finalClassificationCode` debe tener valor (al enviar) | `422 BUSINESS_RULE_VIOLATION` |
| RN-08 | Estratigráfico → Campo = N/A | Si `finalClassificationCode = STRATIGRAPHIC`, `fieldId` debe ser null y `fieldNameFreeText` ignorado (se almacena "N/A") | `422` si hay fieldId |

### 4.3 Reglas de ubicación y coordenadas

| ID | Regla | Validación | Error |
|----|-------|------------|-------|
| RN-09 | Offshore: habilita waterDepth, coastDistance; groundElevation = 0 | Si `locationType = OFFSHORE`, `waterDepthMeters` y `coastDistanceKm` son requeridos. `groundElevationFeet` se fuerza a 0 | `422` si campos offshore faltantes |
| RN-13 | Datum exclusivo (Magna Sirgas XOR Bogotá) | `datumType` debe ser uno de los dos valores válidos | `422` si inválido |
| RN-14 | Coordenadas geográficas: latitud ±gg.gggggg, longitud negativa | Latitud ∈ [-90, 90], Longitud ∈ [-180, 0], max 6 decimales | `422` |
| RN-32 | Offshore: Departamento/Municipio opcionales | Si `locationType = OFFSHORE`, `departmentDaneCode` y `municipalityDaneCode` son nullable | No error si omitidos |

### 4.4 Reglas de UWI (RN-33 a RN-41)

| ID | Regla |
|----|-------|
| RN-33 | Estructura: `[Dpto 2d][Mpio 3d][Sigla 4c][Num 4d][Cluster][Ángulo][Trayectoria][Objetivo]–[Terminación]` |
| RN-34 | Sigla: 4 chars uppercase. 1 palabra → primeras 4. 2+ palabras → primeras 2 de cada. Estratigráfico ANH → ANH + 2 chars |
| RN-35 | Numeración: 4 dígitos con zero-padding |
| RN-36 | Cluster: 2 alfa y/o 4 numéricos con zeros. Compuesto: primera letra de cada palabra. Mismo nombre = sufijo "C" |
| RN-37 | Ángulo: H/V/D |
| RN-38 | Trayectoria: ST(n)/P/PR/ML/G |
| RN-39 | Objetivo: PH/I/M/D/C/GT/O |
| RN-40 | Terminación: guión + CD/LC/LR/GP/CC/OH/O |
| RN-41 | UWI es único e inmutable una vez asignado. Validar unicidad en BD antes de persistir |

**Invariantes del UWI:**
- La generación es **determinista** dado el mismo set de inputs.
- Si un UWI generado colisiona, se retorna error (no se auto-incrementa ni modifica).
- El UWI se descompone y persiste en `FiscalizedUwiComponent` para trazabilidad.

### 4.5 Mapeo de Estado Actual del Pozo → FSM del data-model

| Estado HU (ítem 16) | Código | Estado FSM `WellState` | Observación |
|----------------------|--------|------------------------|-------------|
| Activo | `ACTIVE` | `ActiveTerminated` | Pozo en producción/operación |
| Inactivo | `INACTIVE` | `ActiveTerminated` + Situación `ProductionDiscrepancy` | Se registra situación operativa |
| Suspendido Temporalmente | `TEMPORARILY_SUSPENDED` | `Suspended` | |
| Abandonado Definitivamente | `PERMANENTLY_ABANDONED` | `EndOfCycle` | Estado terminal |
| Abandonado Temporalmente | `TEMPORARILY_ABANDONED` | `AbandonmentInProgress` | |
| Terminación Suspensión con plan | `SUSPENSION_TERMINATION_WITH_PLAN` | `Suspended` + Situación `DeadlineAlert` | |
| Terminación Abandono con plan | `ABANDONMENT_TERMINATION_WITH_PLAN` | `AbandonmentInProgress` + Situación `DeadlineAlert` | |

Al crear el `Well` tras aprobación:
1. Se inserta `WellStateHistory` con `NewStateId` según la tabla anterior.
2. Se actualiza `Well.CurrentStateId` al estado FSM mapeado.
3. Si aplica una `WellSituation` adicional, se crea el registro correspondiente.

### 4.6 Reglas de terminación e intervalos

| ID | Regla |
|----|-------|
| RN-17 | Al menos un intervalo de terminación obligatorio |
| RN-18 | Si tipo ≠ Cañoneo, `shotsPerFoot` = null (backend ignora/rechaza valor) |
| RN-19 | `thicknessFeet = toFeet - fromFeet` — calculado server-side, no confiado del cliente |
| RN-20 | `totalOpenFeet = SUM(thicknessFeet) WHERE status = OPEN` — calculado server-side |
| RN-31 | OH (Open Hole): solo aplica para estratigráficos o excepciones ANH. Backend lo valida si se requiere enforcement |

### 4.7 Reglas de formaciones

| ID | Regla |
|----|-------|
| RN-21 | Base de formación N = Tope de formación N+1. Última base = Profundidad Total MD |
| RN-22 | `thicknessMdFeet = baseMdFeet - topMdFeet` — calculado server-side |
| RN-23 | Redondeo de Tope MD y Tope TVD: ≥ 0.50 → ceiling, < 0.50 → floor |

### 4.8 Reglas de pruebas condicionales

| Objetivo F103 | Sección obligatoria | Sección ignorada |
|---------------|--------------------|--------------------|
| PH | Producción (10.1) | Inyectividad, Monitoreo |
| I | Inyectividad (10.2) | Producción, Monitoreo |
| M | Monitoreo (10.3) | Producción, Inyectividad |
| D (Disposal) | Inyectividad (10.2) — supuesto S-04 | Producción, Monitoreo |
| EST (Estratigráfico) | Ninguna | Todas |

### 4.9 Reglas de flujo de estados

| Transición | Precondición | Efecto |
|-----------|--------------|--------|
| → DRAFT | Usuario logueado con rol Agente/Admin | Registro creado, UWI generado |
| DRAFT → SUBMITTED | Validación completa exitosa + firma operador | WorkflowInstance creada, firma registrada, notificación ANH |
| SUBMITTED → APPROVED | Ing. ANH aprueba + firma ANH | Well creado, PDF generado, notificación operadora |
| SUBMITTED → RETURNED | Ing. ANH devuelve con observaciones | Observación registrada, notificación operadora |
| RETURNED → SUBMITTED | Agente reenvía tras corrección | Revalidación completa |
| DRAFT/RETURNED → DELETED | Agente/Admin elimina | Soft delete o hard delete según política |

**Transiciones prohibidas:**
- APPROVED → cualquier estado (terminal)
- SUBMITTED → DRAFT (solo ANH puede mover desde SUBMITTED)
- Cualquier edición en SUBMITTED o APPROVED

---

## 5. Contratos de Entrada/Salida

Referencia completa: `api-contract.md` y `openapi.yaml`.

Particularidades backend:
- El payload de request se persiste como entidad(es) de payload del formulario (análogo a `Form101Data` pero para registro de pozo antiguo).
- Los campos calculados (espesor, total pies abiertos, bases de formaciones, UWI) se calculan server-side y se persisten. No se confía en valores calculados enviados por el cliente.
- Las coordenadas de las 3 secciones se preservan como payload del formulario. Solo las geográficas (Sección 7) se transforman al CRS canónico para `WellLocation`.

---

## 6. Modelo de Datos Impactado

### 6.1 Nuevas entidades propuestas (referencia a data-model.md actualizado)

**En `catalog.FormDefinition`:**
- Nueva entrada: `FormCode = 'FOPA'`, `Name = 'Registro de Pozo Antiguo'`, `Series = 100`, `Status = Active`.

**En esquema `ops`:**

| Entidad | Descripción | Relación |
|---------|-------------|----------|
| `LegacyWellRegistrationData` | Payload principal del registro (~80 campos escalares + referencias a catálogos + info de contrato histórico + coordenadas en 3 sistemas) | 1:1 con `Procedure` |
| `LwrCompletionInterval` | Fila de la tabla dinámica de intervalos de terminación | N:1 con `LegacyWellRegistrationData` |
| `LwrProductionTest` | Datos de prueba de producción (condicional) | 0..1:1 con `LegacyWellRegistrationData` |
| `LwrInjectivityTest` | Datos de prueba de inyectividad (condicional) | 0..1:1 con `LegacyWellRegistrationData` |
| `LwrMonitoringTest` | Datos de prueba de monitoreo (condicional) | 0..1:1 con `LegacyWellRegistrationData` |
| `LwrFormationFound` | Fila de formaciones encontradas | N:1 con `LegacyWellRegistrationData` |
| `LwrCasingPlaced` | Fila de tuberías de revestimiento | N:1 con `LegacyWellRegistrationData` |
| `LwrLogRun` | Fila de registros corridos (con referencia opcional a adjunto) | N:1 con `LegacyWellRegistrationData` |
| `LwrFreshwaterSand` | Fila de arenas de agua dulce | N:1 con `LegacyWellRegistrationData` |

> **Nota:** los prefijos `Lwr` (Legacy Well Registration) son propuesta de convención. El arquitecto siguiente decide la nomenclatura definitiva conforme al Estándar de Codificación ANH.

### 6.2 Entidades existentes afectadas (al aprobar)

| Entidad | Impacto |
|---------|---------|
| `ops.Well` | Se crea nueva fila con `IsLegacyWell = true` y todos los atributos relevantes del registro |
| `ops.FiscalizedUwiComponent` | Se crea con los componentes desagregados del UWI |
| `ops.WellLocation` | Se crean 2 registros: Surface y BottomHole, con `Point` transformado y `OriginalReportJson` con los 3 sistemas |
| `ops.WellFormation` | Se crea una fila por cada formación encontrada en el registro |
| `ops.WellCasing` | Se crea una fila por cada tubería de revestimiento del registro |
| `ops.WellStateHistory` | Se crea registro con `TriggerEvent = 'LegacyWellRegistration'` |
| `procedure.Procedure` | Se crea con `FormDefinitionId` = FOPA, `WellId` = nuevo well, `DomainStatus` según flujo |
| `procedure.ProcedureSignature` | Dos registros: firma operador y firma ANH |
| `procedure.ProcedurePdf` | PDF oficial post-aprobación |
| `procedure.ProcedureObservation` | Observaciones de devolución ANH |
| `procedure.ProcedureAttachment` | Adjuntos de registros eléctricos |

### 6.3 Breaking Changes
**No hay breaking changes.** Todas las entidades nuevas son aditivas. Las entidades existentes reciben nuevas filas, no cambios de esquema. El valor `IsLegacyWell = true` en `Well` es el discriminador para pozos registrados vía esta funcionalidad vs. creados vía F101.

---

## 7. Operaciones Transaccionales

### 7.1 Operaciones atómicas (todo o nada)

| Operación | Entidades involucradas | Justificación |
|-----------|----------------------|---------------|
| Crear borrador | `Procedure` + `LegacyWellRegistrationData` + satélites + `FiscalizedUwiComponent` | El registro debe ser consistente desde el inicio |
| Enviar a aprobación | `Procedure` (estado) + `ProcedureSignature` + `WorkflowInstance` + `TaskInstance` | La firma y el cambio de estado son indisociables |
| Aprobar | `Procedure` (estado) + `ProcedureSignature` + `Well` + `WellLocation` + `WellFormation` + `WellCasing` + `WellStateHistory` + `ProcedurePdf` | La creación del Well y la aprobación son atómicas |
| Devolver | `Procedure` (estado) + `ProcedureObservation` + `TaskInstance` | |
| Eliminar | `LegacyWellRegistrationData` + satélites + `ProcedureAttachment` + `Procedure` | Cascada completa |

### 7.2 Operaciones que toleran eventual consistency
| Operación | Justificación |
|-----------|---------------|
| Envío de notificaciones (RN-29) | Si falla el envío, se encola para reintento. No bloquea la transición de estado |
| Generación de PDF | Si falla la generación, se marca como pendiente. Se puede regenerar bajo demanda |
| Registro de AuditLog | Audit.NET opera post-commit. Un fallo de auditoría no revierte la operación de negocio |

---

## 8. Integraciones Externas

| Sistema | Dato | Dirección | Manejo de fallas |
|---------|------|-----------|------------------|
| SOLAR / VCH / VPAA (vía `gop.core`) | Operadoras, Contratos, Tipos, Cuencas, Operadores históricos | Consumo (lectura) | Datos precargados en catálogos internos. Si catálogo desactualizado, se usa última versión cacheada |
| SOLAR (vía `gop.core`) | Campos por contrato/operadora | Consumo (lectura) | Idem |
| AVM (vía `gop.integration`) | Pozos sin F103 | Consumo (lectura) | Si no disponible: endpoint retorna `503`. Frontend muestra campo texto libre como fallback (PA-01) |
| SGC | UWI del SGC | No integrado en V1 | Campo de texto libre (supuesto A-07) |
| Mapa de Tierras (vía `gop.integration`) | Validación departamento/municipio por polígono | Consumo (lectura) | Si no disponible: se omite validación geográfica y se registra como `ProcedureSpatialValidation` con `Result = Unknown` |
| DANE | Códigos departamento/municipio | Catálogo precargado | Datos estáticos en `catalog.PoliticalDivision` |
| VAF (vía `gop.integration`) | Contratos para usuarios ANH | Consumo (lectura) | Si no disponible: se muestran solo contratos ya sincronizados en `core.Contract` |

---

## 9. NFRs Backend

### 9.1 Performance

| Métrica | Objetivo |
|---------|----------|
| Latencia p95 — GET detalle | < 400ms |
| Latencia p95 — POST crear/actualizar | < 800ms |
| Latencia p95 — POST submit | < 1.5s (incluye validación completa de ~104 campos) |
| Latencia p95 — POST approve | < 3s (incluye creación de Well + PDF) |
| Latencia p99 — cualquier endpoint | < 5s |
| Tamaño máximo del payload de request | < 500 KB (sin adjuntos) |
| Tamaño máximo de adjunto individual | 10 MB (configurable, PA-05 pendiente) |

### 9.2 Escalabilidad

| Métrica | Estimación |
|---------|------------|
| Registros de pozo antiguo esperados V1 | ~500-2000 (migración inicial + nuevos) |
| Registros por día en operación normal | 5-20 |
| Usuarios concurrentes editando registros | < 50 |
| Crecimiento anual estimado | ~500-1000 registros/año |

### 9.3 Disponibilidad
- SLA objetivo: 99.9% (alineado con Estándar ANH §1).
- RTO: 4 horas.
- RPO: 1 hora.

### 9.4 Observabilidad

| Tipo | Qué se registra |
|------|-----------------|
| Métricas | Conteo de registros por estado, latencia por endpoint, tasa de errores, tamaño promedio de payload |
| Trazas | `traceId` (CorrelationId) propagado en todos los logs y respuestas de error |
| Logs estructurados | Cada transición de estado, cada intento de acceso denegado, cada generación de UWI (con inputs), cada creación de Well post-aprobación |

---

## 10. Seguridad Backend — OWASP Top 10

### A01 — Broken Access Control
- **Autorización por endpoint:** verificada según matriz de la sección 2.1. No solo por ruta, sino por recurso concreto.
- **Tenant isolation:** todo query filtra por `User.OperatorId` para usuarios operadora. Un Agente de Operadora A **nunca** puede acceder a registros de Operadora B, incluso manipulando el `id` en la URL.
- **Principio de menor privilegio:** el Coordinador no puede crear ni enviar. El Agente no puede aprobar ni devolver.
- **Verificación del propietario:** `PUT`, `DELETE`, `submit` verifican que el registro pertenece al tenant del usuario. `approve`, `return` verifican que el usuario tiene alcance ANH sobre el contrato/operadora del registro.

### A02 — Cryptographic Failures
- **TLS 1.2+** obligatorio en todas las comunicaciones.
- **Matrículas profesionales:** almacenadas cifradas en reposo si clasificadas como `DataClassification Level ≥ 3` (pendiente matriz ANH). En `ProcedureSignature`, el snapshot de la matrícula se cifra.
- **Hashing de contraseñas:** no aplica directamente a esta feature (gestionado por `gop.identity`).
- **Secretos:** API keys de integraciones (AVM, Mapa de Tierras) gestionados por el servicio de secretos (nunca en código ni config plano).

### A03 — Injection
- **Todas las consultas a SQL Server** deben usar parámetros (EF Core o parametrización explícita). **PROHIBIDO** concatenar strings en queries.
- **Validación de inputs:** todos los campos de texto libre (`wellName`, `observations`, `initialContractName`, etc.) validados en longitud máxima y sanitizados contra caracteres de control.
- **Nombres de archivo de adjuntos:** sanitizados (sin path traversal, sin caracteres especiales). Se genera nombre interno con UUID; el nombre original se preserva solo como metadato.

### A04 — Insecure Design
- **Threat modeling mínimo para esta feature:**
  - *Amenaza:* Un usuario operadora intenta acceder a registros de otra operadora manipulando el UUID en la URL. *Mitigación:* Filtrado por tenant en toda consulta.
  - *Amenaza:* Un Agente intenta aprobar su propio registro simulando ser ANH. *Mitigación:* Verificación de rol en el token JWT; el rol no viene del request.
  - *Amenaza:* Manipulación del UWI en el payload. *Mitigación:* UWI se genera server-side; cualquier valor enviado por el cliente se ignora.
  - *Amenaza:* Envío de archivo malicioso como adjunto de registro eléctrico. *Mitigación:* Validación de content-type contra whitelist, escaneo de malware (si disponible), tamaño máximo.
- **Defensa en profundidad:** validaciones en frontend + backend + constraints de BD.
- **Principio de menor privilegio:** login SQL del microservicio solo tiene permisos sobre sus esquemas.

### A05 — Security Misconfiguration
- **Headers de seguridad:** `Strict-Transport-Security`, `X-Content-Type-Options: nosniff`, `X-Frame-Options: DENY`, `Content-Security-Policy` — configurados a nivel de middleware/gateway.
- **CORS restrictivo:** solo orígenes autorizados (dominio de la SPA).
- **Banners/versiones:** no exponer versión de ASP.NET, SQL Server ni otras dependencias en headers de respuesta.
- **Modo de error:** en producción, los errores retornan estructura uniforme (sección 5 de api-contract.md) sin stack traces ni nombres de tablas.

### A06 — Vulnerable & Outdated Components
- **Dependencias bloqueadas** a versiones específicas auditadas.
- **Escaneo de vulnerabilidades** en pipeline CI/CD.
- Actualización de dependencias según cadencia del proyecto.

### A07 — Identification & Authentication Failures
- **JWT con expiración corta** (típico 15-30 min) + refresh tokens con rotación (gestionado por `gop.identity`).
- **Protección contra fuerza bruta:** rate limiting por usuario (sección 6.3 de api-contract.md).
- **MFA:** obligatorio según mandato OTI (gestionado por `gop.identity`).
- **Matrícula profesional:** verificada al momento de la firma (submit/approve). Si la matrícula no existe o no está vigente en `ProfessionalLicense`, se rechaza con `422`.

### A08 — Software & Data Integrity Failures
- **Firma de artefactos de deployment:** según pipeline CI/CD del estándar.
- **Validación de integridad de adjuntos:** `ContentHash` (SHA-256) se calcula al upload y se verifica al download.
- **Optimistic concurrency:** `ETag` basado en `ConcurrencyStamp` o `UpdatedAt` para prevenir overwrites silenciosos.

### A09 — Security Logging & Monitoring Failures
- **Logging estructurado** de eventos de seguridad:
  - `AccessDenied` — cada intento de acceso denegado (403).
  - `StateTransition` — cada cambio de estado del registro.
  - `Sign` — cada firma registrada (submit/approve).
  - `Create`, `Update`, `Delete` — operaciones CRUD sobre el registro.
- **`traceId`** propagado en toda la cadena.
- **NUNCA loguear:** tokens JWT, matrículas profesionales completas, coordenadas exactas (solo contexto suficiente para debug: ej. "coordenadas validadas para WellId=X").

### A10 — SSRF
- **Integraciones salientes** (AVM, Mapa de Tierras, SGC): URLs destino configuradas vía servicio de secretos, no aceptadas desde inputs de usuario.
- **Lista blanca de destinos:** solo los sistemas declarados en la sección 8 de este spec.
- **No se procesan URLs** provenientes del payload del usuario en ningún endpoint de esta feature.

---

## 11. Edge Cases Backend

| # | Escenario | Respuesta | HTTP |
|---|-----------|-----------|------|
| EC-BE-01 | Token inválido, expirado o revocado | `UNAUTHORIZED` | `401` |
| EC-BE-02 | Usuario autenticado sin rol para la acción | `FORBIDDEN` sin revelar si el recurso existe | `403` |
| EC-BE-03 | Intento de acceder a registro de otra operadora | `NOT_FOUND` (no `403`, para no filtrar existencia) | `404` |
| EC-BE-04 | Violación de exclusión mutua de clasificación | `BUSINESS_RULE_VIOLATION` con detalle de RN-07 | `422` |
| EC-BE-05 | UWI duplicado | `DUPLICATE_UWI` con el UWI colisionante | `409` |
| EC-BE-06 | Nombre de pozo duplicado | `DUPLICATE_WELL_NAME` | `409` |
| EC-BE-07 | Editar registro en estado SUBMITTED o APPROVED | `INVALID_STATE_TRANSITION` | `409` |
| EC-BE-08 | Eliminar registro en estado SUBMITTED o APPROVED | `INVALID_STATE_TRANSITION` | `409` |
| EC-BE-09 | Enviar registro incompleto | `INCOMPLETE_REGISTRATION` con lista de campos faltantes | `422` |
| EC-BE-10 | Matrícula profesional inválida o vencida | `INVALID_PROFESSIONAL_LICENSE` | `422` |
| EC-BE-11 | Concurrencia: PUT con ETag desactualizado | `CONCURRENT_MODIFICATION` | `409` |
| EC-BE-12 | Payload > 500 KB (sin adjuntos) | `PAYLOAD_TOO_LARGE` | `413` |
| EC-BE-13 | Adjunto > tamaño máximo | `PAYLOAD_TOO_LARGE` | `413` |
| EC-BE-14 | Tipo de archivo de adjunto no permitido | `BUSINESS_RULE_VIOLATION` | `422` |
| EC-BE-15 | AVM no disponible al buscar pozos | `503 Service Unavailable` con retry header | `503` |
| EC-BE-16 | Rate limiting excedido | `RATE_LIMIT_EXCEEDED` con header `Retry-After` | `429` |
| EC-BE-17 | Intento de aprobar cuando no es SUBMITTED | `INVALID_STATE_TRANSITION` | `409` |
| EC-BE-18 | Devolución con observaciones vacías | `EMPTY_OBSERVATIONS` | `422` |
| EC-BE-19 | Trayectoria ST/PR/G sin pozo padre | `BUSINESS_RULE_VIOLATION` (RN-05) | `422` |
| EC-BE-20 | Contrato no pertenece a la operadora del usuario | `BUSINESS_RULE_VIOLATION` | `422` |
| EC-BE-21 | Objetivo F103 = PH pero falta productionTest | `INCOMPLETE_REGISTRATION` | `422` |
| EC-BE-22 | Coordenadas geográficas fuera de Colombia (validación laxa) | Warning en respuesta, no bloquea. Se persiste `ProcedureSpatialValidation` con `Result = Fails` | `200` con warnings |
| EC-BE-23 | Formaciones con topes no en orden ascendente | `BUSINESS_RULE_VIOLATION` | `422` |

---

## 12. Criterios de Testing y Cobertura

### 12.1 Niveles requeridos

| Nivel | Alcance |
|-------|---------|
| **Unitarias** | Generación de UWI (todos los escenarios de RN-33 a RN-41), cálculos (espesor, total pies abiertos, base de formaciones, redondeo RN-23), validaciones de reglas de negocio (exclusión mutua, condicionales offshore/F103/objetivo), mapeo de estados HU → FSM |
| **Integración (con BD real/contenedor)** | Creación completa de registro con satélites, flujo Borrador→Envío→Aprobación→Well creado, concurrencia (ETag), unicidad de UWI y nombre, eliminación en cascada, filtrado por tenant |
| **Contract tests** | Validación de request/response contra `openapi.yaml` para cada endpoint |
| **E2E** | Flujo completo end-to-end con frontend contra backend real |

### 12.2 Cobertura mínima sugerida
- **85%** en lógica de dominio (generación UWI, validaciones de reglas de negocio, cálculos, mapeo de estados).
- **80%** en capa de aplicación (handlers/services).
- **70%** en capa de infraestructura (repositorios, integraciones).

### 12.3 Escenarios de seguridad con test obligatorio

| # | Escenario | Verificación |
|---|-----------|-------------|
| ST-01 | Agente de Operadora A intenta GET registro de Operadora B | Retorna `404` (no `403`) |
| ST-02 | Coordinador intenta POST submit | Retorna `403` |
| ST-03 | Agente intenta POST approve | Retorna `403` |
| ST-04 | Token expirado en cualquier endpoint | Retorna `401` |
| ST-05 | Intento de inyección SQL en campo `wellName` | Query parametrizada; no hay SQL injection |
| ST-06 | Adjunto con extensión `.exe` disfrazado | Retorna `422` (content-type no permitido) |
| ST-07 | Intento de path traversal en nombre de archivo | Nombre sanitizado; archivo almacenado con UUID |
| ST-08 | PUT con ETag manipulado (valor aleatorio) | Retorna `409` |
| ST-09 | Intento de modificar `operatorId` en el payload | Backend ignora el campo; usa `User.OperatorId` del JWT |
| ST-10 | Intento de enviar UWI custom en el payload | Backend ignora; genera UWI server-side |

---

## 13. Supuestos Confirmados Documentados

| ID | Supuesto | Origen |
|----|----------|--------|
| S-A02 | El operador ingresa 3 sistemas de coordenadas. Backend transforma geográficas al CRS canónico. Los 3 se preservan en `OriginalReportJson` | A-02, confirmado |
| S-A03 | Se crea nuevo `FormDefinition` con código `FOPA` en catálogo. Se crea `Procedure` con workflow de aprobación propio | A-03, confirmado |
| S-A04 | Estado del Well al aprobarse = mapeado según tabla §4.5 desde el campo "Estado Actual" seleccionado por el operador | A-04, confirmado |
| S-A06 | Datos maestros disponibles en catálogos internos de GOP (no consulta directa a SOLAR en cada request) | A-06, confirmado |
| S-A07 | UWI SGC: texto libre en V1 sin integración | A-07, confirmado |
| S-I01 | Ítems 14 (Objetivo F103) y 17 (Objetivo Actual) son campos distintos en el modelo | I-01, confirmado |
| S-I02 | Eliminación bloqueada en Enviado Y Aprobado | I-02, confirmado |
| S-I03 | Fecha Aprobación F103: obligatoria solo si hasForm103 = SÍ | I-03, confirmado |
| S-I04 | Offshore: Departamento/Municipio opcionales (RN-32 adoptada) | I-04, confirmado |
| S-S01 | Catálogos de tuberías como `CatalogItem` precargados (pendientes de poblar) | S-01, confirmado |
| S-S02 | Máximo configurable de filas en tablas dinámicas | S-02, confirmado |
| S-S03 | PDF generado server-side | S-03, confirmado |
| S-S04 | Disposal habilita Inyectividad | S-04, confirmado |

---

## 14. Puntos de Decisión Pendientes

| # | Punto | Impacto |
|---|-------|---------|
| PD-BE-01 | PA-01: API de AVM. Si no disponible en V1, el endpoint de búsqueda retorna `503` y el frontend usa texto libre como fallback | Integración |
| PD-BE-02 | PA-05: Formatos y tamaño de adjuntos. Propuesta: PDF/TIFF/LAS, max 10 MB. Requiere confirmación ANH/OTI | Validación de upload |
| PD-BE-03 | PA-06: Flujo de estados. Se adopta el flujo de §4.2 de la HU como definitivo. Si ANH requiere estados intermedios (ej: "En Revisión" explícito), el workflow se ajusta | Workflow |
| PD-BE-04 | Estrategia de eliminación: soft delete vs hard delete para registros en Draft/Returned | Modelo de datos |
| PD-BE-05 | Catálogos de coordenadas: lista exacta de orígenes MAGNA-SIRGAS y zonas Datum Bogotá con códigos EPSG (Resolución IGAC 370/2021) | Catálogo `CoordinateReferenceSystem` |
| PD-BE-06 | Notificaciones: canal exacto (Email, InApp, ambos) y templates de contenido | Infraestructura de notificaciones |

---

## 15. Anexos

- **API Contract:** `api-contract.md` — Contrato de interfaz HTTP.
- **OpenAPI Spec:** `openapi.yaml` — Especificación validable.
- **Spec Frontend:** `spec-frontend.md` — Especificación funcional del frontend.
- **Data Model:** `data-model.md` — Modelo de datos actualizado con changelog.
