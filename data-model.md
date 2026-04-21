# Propuesta de Modelo de Datos — GOP 360°

## Diseño Orientado a Dominios (DDD) — Nivel Conceptual Consolidado

**Versión:** 0.5
**Fecha:** Abril 2026
**Motor:** SQL Server 2022 / SQL Azure
**Alcance:** Topología de esquemas, bounded contexts, entidades principales, patrones de modelado de coordenadas, ciclo de vida de pozos, y gobernanza evolutiva del modelo. No incluye aún atributos exhaustivos, constraints e índices a detalle — se derivan en fase posterior.

**Cambios respecto a v0.4:**
- **Alineación con `CONSTITUTION-ba.md` v1.5–v1.7** (OWASP hardening, Token Exchange federado + usuarios locales, permisos de BD).
- Nueva **§9.0 Integración con ASP.NET Core Identity**: declara que `User` extiende `IdentityUser<long>` y `Role` extiende `IdentityRole<int>`; uso nativo de `AspNetUserLogins` para federación OIDC (reemplaza columnas custom de IdP externo).
- **§9.1 `User`** — renombrado `AuthType` → `AuthenticationSource` (`External` | `Local`); eliminado `IsSuperuser` (contradecía §7.4 de CONSTITUTION).
- Nuevas tablas **§9.6 `RefreshToken`** (con `FamilyId` + rotation chain + detección de reuso) y **§9.7 `RevokedAccessToken`** (revocation list por `jti`).
- Eliminada **§9.6 anterior `UserSession`** — redundante con `RefreshToken`.
- **§10.1 `AuditLog`** — documentado el catálogo oficial de `EventType` (autenticación, autorización, dominio, datos sensibles, administración).
- Nota sobre **Always Encrypted** para columnas con `[DataClassification(Level ≥ 3)]` (§9, §10).
- Nueva **§14 Permisos de base de datos** — aislamiento lógico por esquema en instancia compartida V1 (alineado con CONSTITUTION §11.6.6).

**Cambios previos (v0.3 → v0.4):**
- Motor fijado: **SQL Server 2022 / SQL Azure** (por solicitud del cliente — reemplaza 2026 que era erróneo).
- **Alineación con `CONSTITUTION-ba.md` v1.1**: modelo unificado de alcance de datos en `UserRole`.
- Eliminada entidad `UserOperatorScope` — redundante con `UserRole`.
- `UserRole` (§9.4) ampliado con columnas `OperatorId`, `FieldId`, `AssignedBy`, `AssignedAt`, `Reason` y 7 reglas de interpretación.
- Formalizada invariante polimórfica de `Procedure` (§8.1) como CHECK constraint explícito.
- Agregados índices clave en `UserRole`.

**Cambios previos (v0.2 → v0.3):**
- Alineación completa con el *Documento de Estándares de Codificación* ANH (Camilo E. Gaviria, 05/02/2026) y los *Lineamientos OTI* (Feb/2026).
- Nomenclatura migrada a **PascalCase + inglés** por obligatoriedad normativa.
- Referencia explícita a compatibilidad **PPDM 3.9**.
- Nueva sección §17 — **Gobernanza Evolutiva del Modelo**.

---

## 1. Principios de diseño

**1.1. Alineación normativa obligatoria.** El modelo cumple con:
- *Estándares de Codificación ANH* v1.0 (nomenclatura, estructura, auditoría, tSQLt).
- *Guía de Desarrollo Seguro de Software* ANH-GTIC-GU-18 (segregación de privilegios, zero trust, cifrado, auditoría).
- *Manual del Ciclo de Vida de Herramientas Informáticas* ANH-GTIC-MA-02.
- *ISO/IEC 25010* (mantenibilidad y calidad).
- Compatibilidad **PPDM 3.9** para interoperabilidad con sistemas petroleros estándar.

**1.2. Separación estricta entre motor genérico y dominio regulatorio.** `Workflow` (motor) no conoce pozos, campos, formas ni firmas. `Procedure` (trámite regulatorio) es quien conecta el motor con los activos del negocio. Habilita reutilización del motor para cualquier flujo futuro sin tocar código.

**1.3. Dos agregados raíz operacionales, no uno.** Las formas Serie 100 escriben sobre `Well`. Las formas Serie 200 escriben sobre `ProducingFormation` (Field × Formation en su dimensión productiva). Bounded contexts separados con invariantes y patrones de acceso diferentes.

**1.4. Descomposición del `ControlForma` legado.** La tabla-Dios `GOP.ControlForma` (~40 columnas mezclando identidad, flujo, firmas y payload) se descompone en tres destinos: `Procedure` (identidad del expediente), `WorkflowInstance` + `TaskInstance` (estado del flujo), y `Form{NNN}Data` (payload específico).

**1.5. Dos capas de RBAC.** Capa tradicional sobre módulos y datos. Capa acoplada al motor sobre acciones y campos por paso/rol. Autenticación obligatoria via **SSO + Active Directory + MFA** (mandato OTI).

**1.6. Tres dimensiones para el estado del Well.** Etapa del ciclo de vida (FSM principal), fase operativa interna (subdivisión temporal), y situaciones operativas puntuales (flags simultáneos).

**1.7. Coordenadas con CRS canónico único.** Almacenamiento en MAGNA-SIRGAS geográficas (WGS84, EPSG:4326) mediante tipo nativo `geography` de SQL Server. Proyecciones a otros sistemas (Origen Nacional, Origen Regional legado) se derivan al reportar.

**1.8. Auditoría inmutable obligatoria.** Tabla `AuditLog` con triggers o EF Core interceptors sobre todas las entidades de dominio, conforme §2.4 del Documento de Estándares.

**1.9. Homologación con el legado sin sesgo.** Toda entidad raíz lleva `LegacyId` opcional preservando trazabilidad al origen sin contaminar el diseño.

**1.10. Nomenclatura estándar ANH:**
- **Tablas:** PascalCase en inglés (`Well`, `Procedure`, `ProducingFormation`).
- **Columnas:** PascalCase en inglés (`WellId`, `CreatedAt`, `IsActive`).
- **Primary Key:** `{Table}Id` (`WellId`, `ProcedureId`).
- **Foreign Key:** `{RelatedTable}Id` (FK de Well a Contract → `ContractId`).
- **Índices:** `IX_{Table}_{Columns}` (`IX_Wells_ContractId_CurrentStateId`).
- **Constraints:** `PK_{Table}`, `FK_{Table}_{RelatedTable}_{Column}`, `UQ_{Table}_{Column}`, `CK_{Table}_{Check}`.
- **Tipos de dato:** `datetime2(7)` para timestamps (no `datetime`), `nvarchar` para texto Unicode, `decimal(p,s)` explícito, `bit` para booleanos.

---

## 2. Topología de Esquemas

El modelo se organiza en **9 esquemas** de base de datos. Cada esquema tiene un criterio de existencia explícito: bounded context semántico, ciclo de vida del dato, o sensibilidad/permisos.

| # | Esquema | Criterio | Contenido principal | Volumen |
|---|---|---|---|---|
| 1 | `core` | Bounded Context — Entidades transversales | Operator, Contract, Field, Block, ClusterLocation | Bajo |
| 2 | `catalog` | Bounded Context — Catálogos estables | Basin, Formation, PoliticalDivision, Types, FormDefinition, AttachmentDefinition, DrillingRig, CoordinateReferenceSystem | Muy bajo |
| 3 | `ops` | Bounded Context — Operaciones de Pozo | Well, UWI, WellLifecycle, formas Serie 100, IDOP/IDOC | Medio-alto |
| 4 | `prod` | Bounded Context — Producción de Campo | ProducingFormation, Modality, MonthlyProduction, formas Serie 200 | Alto |
| 5 | `workflow` | Bounded Context — Motor genérico | WorkflowDefinition, TaskDefinition, Actions, Routing, Instances | Medio |
| 6 | `procedure` | Bounded Context — Trámites regulatorios | Procedure, Attachments, Signatures, PDFs, Notifications, SpatialValidations | Medio |
| 7 | `identity` | Seguridad | User, Role, ProfessionalLicense, RBAC, Sessions | Bajo-medio |
| 8 | `audit` | Ciclo de vida del dato | AuditLog, IntegrationLog, BulkLoadLog | Muy alto |
| 9 | `integration` | Ciclo de vida del dato | Staging y homologación con sistemas externos | Volátil |

> **Nota de traducción:** el esquema `tramite` de v0.2 pasó a llamarse `procedure` por la regla de inglés obligatorio del Documento de Estándares. Todas las entidades internas siguen la misma regla (ej: `Tramite` → `Procedure`, `TramiteAnexo` → `ProcedureAttachment`).

### 2.1 Criterios de asignación de tablas a esquemas

Cuando una entidad podría caber en dos esquemas, aplicamos en orden:

1. **Dueño conceptual.** Si tiene sentido sin el otro contexto, vive en su esquema natural.
2. **Escritor principal.** Va al esquema del escritor más frecuente.
3. **Ruptura conceptual.** Va al contexto que no puede operar sin ella.

### 2.2 `core` vs `catalog` — la distinción clave

| Criterio | `core` | `catalog` |
|---|---|---|
| Naturaleza | Transaccional transversal | Referencia estable |
| Nace de | Operación de negocio | Decisión administrativa |
| Cambio | Con operaciones (cesiones, nuevos contratos) | Con configuración (admin ANH agrega tipo, DANE publica municipio) |
| Ciclo de vida | Rico (estados, fechas, relaciones) | Pobre (listas cerradas) |
| Ejemplo | `Contract` nace cuando se firma | `PoliticalDivision.Municipality` nace por decreto DANE |
| Decisión | Operador/Contrato/Campo/Bloque | Cuenca/Formación/Tipos/CRS |

**Caso sutil:** `Formation` (Formación Mirador como concepto geológico) → `catalog`. `ProducingFormation` (Mirador produciendo en Campo Rubiales) → `prod`.

### 2.3 Patrón de dependencias entre esquemas

- **Permitidas:** `ops`, `prod`, `procedure` → `core`, `catalog`, `workflow`, `identity`. `core` y `catalog` no dependen de nada.
- **Prohibidas:** `workflow` no depende de ningún esquema de dominio (crítico para reutilización). `core` no depende de `ops` ni `prod`.
- **Logs independientes:** `audit` e `integration` reciben FKs desde todos, pero ningún esquema funcional depende de ellos para operar.

---

## 3. Esquema `core` — Entidades Transversales

### 3.1 `Operator`
Empresa operadora o ANH en calidad de operador (para pozos estratigráficos). Atributos: `OperatorId`, `TaxId`, `Name`, `Type` (Private / ANH), `Status`, metadatos de contacto. Equivalente legado: `dbo.Empresa` + `dbo.OperadoraMerge`.

### 3.2 `Contract`
Contrato de concesión. Atributos: `ContractId`, `Code`, `Name`, `Type` (EP / TEA / Convention / Association), `SignDate`, `TerminationDate`, `BasinId`, `Area`, `HolderOperatorId`, `AnhParticipationPercent`. Equivalente legado: `dbo.Contrato` + `dbo.ContratoBase`.

### 3.3 `ContractPartner`
Socios del contrato distintos al titular. Atributos: `ContractId`, `PartnerOperatorId`, `ParticipationPercent`, `Role`. Equivalente legado: `dbo.ContratoSocio` + `dbo.ContratoPart`.

### 3.4 `ContractPhase`
Fase contractual (Exploration / Evaluation / Production / Extension). Equivalente legado: `dbo.ContratoFase`.

### 3.5 `Field`
Área geográfica con yacimientos productivos. Atributos: `FieldId`, `Code`, `Name`, `DiscoveryDate`, `Status` (Exploratory / Commercial / Abandoned), `Area`, `Polygon` (tipo `geography`). Un campo se crea como "Exploratory" y se renombra cuando un pozo A3 aprueba su F103 con Lahee B3. Equivalente legado: `dbo.Campo`.

### 3.6 `FieldContract`
Asociación N:M entre campos y contratos (por cesiones a lo largo del tiempo). Equivalente legado: `dbo.CampoContrato`.

### 3.7 `Block`
Área operativa dentro de un contrato. Equivalente legado: `GOP.Bloque`.

### 3.8 `ClusterLocation`
Agrupamiento de pozos que comparten locación de superficie. Atributos: `ClusterLocationId`, `Name`, `ContractId`, `SitePoint` (tipo `geography`).

---

## 4. Esquema `catalog` — Catálogos de Referencia

### 4.1 `Basin`
Cuenca sedimentaria colombiana. Catálogo cerrado.

### 4.2 `Formation`
Formación geológica (Mirador, Barco, Guadalupe…). Catálogo administrado por ANH con proceso de alta para formaciones nuevas (RN-16 HU F101).

### 4.3 `PoliticalDivision`
Departamento y Municipio. Atributos: `DaneCode`, `Name`, `ParentId` (jerarquía dpto-mpio), `Level`. Sincronizado con DANE.

### 4.4 `CoordinateReferenceSystem` (CRS)
Catálogo de sistemas de referencia geoespaciales. Atributos: `CrsCode` (ej: `MAGNA_GEO`, `MAGNA_ORIG_NAC`), `EpsgCode` (4326, 9377…), `Name`, `Type` (Geographic / Projected), `Unit`, `IsCurrent`.

### 4.5 `Catalog` / `CatalogItem`
Patrón genérico para catálogos cerrados pequeños: Tipo de Pozo por Ángulo (H/V/D), por Objetivo (PH/I/M/D/C/GT/O), por Terminación, Trayectoria (ST/P/PR/ML/G/O), Lahee, Tipo de Trampa, Método de Producción. Equivalente legado: `dbo.Catalogo`.

### 4.6 `FormDefinition`
Define cada forma ministerial (F101, F102, F103, F202, F204…). Atributos: `FormCode`, `Name`, `Series` (100 / 200), `RegulatoryResolution`, `Version`, `Status` (Active / Deprecated), `FormKey` (identificador UI). Define **qué formas existen**, no cómo se tramitan.

### 4.7 `AttachmentDefinition`
Catálogo de tipos de documento anexo por forma. Atributos: `FormDefinitionId`, `Name`, `IsRequired`, `AllowedExtensions`, `MaxSizeBytes`, `NamePattern`. Configurable por Admin GOP ANH sin cambios de código (RN-17 HU F101).

### 4.8 `DrillingRig`
Catálogo de equipos de perforación. Atributos: `Brand`, `Model`, `HorsepowerHp`, `Type`, `OwnerOperatorId`, `Status`.

---

## 5. Esquema `ops` — Operaciones de Pozo

Agregado raíz: `Well`. Todas las formas Serie 100 escriben sobre este agregado. Ciclo de vida modelado como FSM con tres dimensiones ortogonales.

### 5.1 `Well` (agregado raíz)

Atributos principales:

- **Identificación:** `WellId`, `FiscalizedUwi` (único, inmutable), `Name`, `LegacyWellId`.
- **Contexto contractual:** `OperatorId`, `ContractId`, `FieldId`, `ClusterLocationId`.
- **Clasificación:** referencias a catálogos (Angle, Purpose, Completion, Location, InitialClassification, LaheeClassification).
- **Trayectoria:** `TrajectoryType` (ST/P/PR/ML/G/O), `OriginWellId` (autoreferencial).
- **Estado (cache):** `CurrentStateId`, `CurrentStageId` — derivados del historial.
- **Flags:** `IsLegacyWell`, `IsFinalized`.
- **Auditoría estándar:** `CreatedAt`, `CreatedBy`, `UpdatedAt`, `UpdatedBy` (obligatorio por Estándar §2.4).

**Invariantes:**
- UWI inmutable tras generación.
- No duplicidad en UWI ni en nombre.
- Pozo con F101 radicada no se edita ni elimina.
- Transiciones de estado solo vía eventos de dominio en `WellStateHistory`.

### 5.2 `FiscalizedUwiComponent`

Desagregados los 8 componentes del UWI: `DaneDeptCode`, `DaneMunCode`, `WellNameSigle`, `WellNumber`, `ClusterLocationCode`, `AngleCode`, `TrajectoryCode`, `PurposeCode`, `CompletionCode`. Relación 1:1 con `Well`. Persiste aunque derivable, para auditoría y reconstrucción si el algoritmo evoluciona.

### 5.3 `WellBranch`

Pozos multilaterales tienen N brazos con fondos distintos. Atributos: `WellBranchId`, `WellId`, `BranchNumber`, `PlannedStartDate`.

### 5.4 `WellFormation`

Relación N:M entre pozo (o brazo) y formaciones objetivo. Atributos: `IsPrimary`, `TopMdFeet`, `TopTvdFeet`, `TvdssFeet`.

### 5.5 `WellLocation` — modelado de coordenadas

Ubicaciones geoespaciales del pozo y brazos. Atributos:

- **Identificación:** `WellLocationId`, `WellId`, `WellBranchId` (nullable), `LocationType` (Surface / BottomHole / BranchBottomHole).
- **Punto canónico:** `Point` (`geography` POINT, WGS84 EPSG:4326) — única fuente de verdad.
- **Profundidades:** `MeasuredDepthFeet`, `TrueVerticalDepthFeet`, `TvdssFeet`.
- **Metadatos superficie** (null si tipo ≠ Surface): `GroundElevationFeet`, `KellyBushingElevationFeet`, `WaterDepthMeters` (solo offshore).
- **Trazabilidad:** `MeasurementDate`, `ReportSource` (Operator / AnhValidation / Correction / LegacyMigration), `ProcedureOriginId`, `CoordinateQuality` (High / Medium / Low / Unknown).
- **Historial:** `IsCurrent` (permite versiones con solo una vigente).
- **Preservación legado:** `OriginalReportJson` (JSON con CRS y coordenadas como llegaron), `LegacyRegionalE`, `LegacyRegionalN`.

**Patrón de uso:**
- Al recibir coordenadas, transformar al CRS canónico vía `geography::STTransform` (si procede).
- Preservar lo recibido en `OriginalReportJson` para trazabilidad.
- Proyecciones a otros CRS (Origen Nacional para PDF) se calculan al generar, no se almacenan duplicadas.
- Índice espacial sobre `Point` habilita consultas "pozos dentro del polígono X" en tiempo logarítmico.

### 5.6 `WellCasing`

Programa de tubería de revestimiento por fase. Atributos: `HoleDiameterInches`, `CasingDiameterInches`, `ShoeDepthFeet`, `CementTopFeet`. Para ML, se asocia al brazo.

### 5.7 `WellPermit`

Permisos especiales (perforación fuera de polígono, lindero <100m, producción conjunta). Atributos: `PermitType`, `AttachmentId`, `ApprovalDate`, `ValidUntil`, `Justification`. Equivalente legado: `GOP.Permiso`.

### 5.8 Ciclo de vida del Well — tres dimensiones

#### Dimensión 1 — Etapa (FSM principal)

Estados: `Registered` → `Planned` → `Drilling` → `Completing` → `ActiveTerminated` ↔ (`UnderIntervention` / `Suspended` / `AbandonmentInProgress`) → `EndOfCycle` (terminal).

**Entidades:**

- **`WellState`** — catálogo de estados. Atributos: `WellStateId`, `Code`, `Name`, `IsInitial`, `IsTerminal`, `IsReversible`, `Category`, `DashboardColor`, `VisualOrder`.
- **`WellStateTransition`** — transiciones válidas. Atributos: `FromStateId`, `ToStateId`, `TriggerEvent`, `RequiredFormType`, `IsAutomatic`, `PreconditionsJson`, `ValidationOrder`, `ValidFrom`, `ValidTo`.
- **`WellStateHistory`** — log inmutable. Atributos: `WellId`, `PreviousStateId`, `NewStateId`, `AppliedTransitionId`, `TriggerEvent`, `ProcedureOriginId`, `TransitionAt`, `UserId`, `Remarks`, `ContextDataJson`.

#### Dimensión 2 — Fase Operativa

Subdivisiones temporales dentro de `Drilling` (Conductor, Surface, Intermediate, Production).

**`WellPhase`:** `WellId`, `PhaseType`, `SequenceNumber`, `StartedAt`, `EndedAt`, profundidades MD inicio/fin, `HoleDiameterInches`, `HasCementation`, referencias al IDOP que la inició/cerró, `Status`.

#### Dimensión 3 — Situaciones Operativas Puntuales

Flags simultáneos que coexisten con cualquier etapa.

**`WellSituation`:** `WellId`, `SituationType` (DeadlineAlert / PermitBlocked / UnderAnhObservation / ProductionDiscrepancy), `Severity` (Info / Warning / Critical), `StartedAt`, `EndedAt`, `Reason`, `ProcedureOriginId`, `ResolvedByUserId`.

#### Acoplamiento con workflow vía eventos de dominio

1. Workflow del trámite completa última tarea → estado `Approved`.
2. Servicio `procedure` publica evento `ProcedureApproved` con tipo y pozo afectado.
3. Servicio `WellLifecycle` (suscriptor) consulta `WellStateTransition` por `TriggerEvent`.
4. Valida precondiciones, aplica transición, registra en `WellStateHistory`.
5. Actualiza cache `Well.CurrentStateId`.

### 5.9 Formas Serie 100 — entidades de payload

| Entidad | Forma |
|---|---|
| `Form101Data` | Drilling Permit |
| `Form102Data` | Biweekly Report |
| `Form103Data` | Official Completion |
| `Form104Data` | Pressure Test Report |
| `Form105Data` | Workover |
| `Form106Data` | Post-Completion Works |
| `Form107Data` | Monitor Well Permit |
| `Form108Data` | Abandonment Program |
| `Form109Data` | Plugging and Abandonment |
| `Form110Data` | Reactivation |
| `Form111Data` | Suspension Schedule |
| `Form112Data` | Temporary Abandonment Follow-up |
| `IdopData` | Daily Drilling Operations Report |
| `IdocData` | Daily Completion Operations Report |

Satélites (bits, mud, cementations, formations traversed, influxes, losses, logs) cuelgan de `IdopData` e `IdocData`.

---

## 6. Esquema `prod` — Producción de Campo

Agregado raíz: `ProducingFormation`.

### 6.1 `ProducingFormation`
Atributos: `FieldId`, `FormationId`, `FirstProductionDate`, `EndProductionDate`, `Status`, `TrapType`.

### 6.2 `ProductionModality`
Modalidad en período: `PrimaryTesting` / `ExtendedTesting` / `EarlyProduction` / `Exploitation`. Atributos: `ProducingFormationId`, `ModalityType`, `StartDate`, `EndDate`, `ApprovalProcedureId`.

### 6.3 `MonthlyWellProduction`
Agregado hijo. Atributos:

- **Llave natural:** `ProducingFormationId`, `WellId`, `Month`, `Year`.
- **Petróleo/Agua/Gas:** diaria, mensual, acumulada, factor de campo.
- **Calidad:** `BswPercent`, `ApiGravity`, `GasOilRatio`.
- **Estado/tipo** del pozo al cierre.
- **Operativos:** días activos/acumulados, método de producción.
- **Trazabilidad:** `DataOrigin` (AVM / Manual / BulkLoad), `Version`, `Form202ProcedureId`.

**Invariantes:**
- Acumulados no disminuyen.
- Desde versión 1, acumulados automáticos.
- Pozo registrado persiste en meses subsiguientes.

### 6.4 `MonthlyProductionChange`
Log de modificaciones sobre datos precargados de AVM. Atributos: `MonthlyWellProductionId`, `ModifiedField`, `AvmOriginalValue`, `NewValue`, `UserId`, `ChangedAt`, `Justification`.

### 6.5 `FieldBalance`, `PotentialTest`, `Injection`, `IncrementalProductionProject`, `EmissionEvent`, `Tank`, `TankMovement`
Ver v0.2 §6.5-6.10 con nomenclatura traducida.

### 6.6 Formas Serie 200 — entidades de payload

`Form201Data` a `Form217Data` siguiendo el mismo patrón. Referencia a `Procedure` (1:1), `ProducingFormation` o `Field` (según aplique).

---

## 7. Esquema `workflow` — Motor Genérico

**Crítico:** entidades de este esquema **no conocen del dominio**. No saben qué es pozo, forma, operador. Reutilizables para cualquier flujo futuro.

### 7.1 `WorkflowDefinition`
Atributos: `Code`, `Name`, `Version`, `Status` (Draft / Active / Inactive). Solo una versión activa por código.

### 7.2 `TaskDefinition`
Nodo del flujo. Atributos: `WorkflowDefinitionId`, `Code`, `Name`, `TaskType` (Human / System / Timer), `IsStart`, `IsEnd`, `FormKey`, `SlaHours`.

### 7.3 `ActionDefinition`
Catálogo: `Approve`, `Return`, `Reject`, `File`, `SignGeologist`, `SignEngineer`, `RequestCorrection`.

### 7.4 `TaskDefinitionAction`
N:M tarea-acción. Metadata UI.

### 7.5 `TaskRouting`
Central. Tarea origen + acción → tarea destino. Atributos: `TaskDefinitionId`, `ActionDefinitionId`, `NextTaskDefinitionId`, `ConditionType`, `ConditionValue`, `Priority`, `IsDefault`.

### 7.6 `WorkflowInstance`
Atributos: `WorkflowDefinitionId`, `BusinessKey` (string opaco — para procedures contiene `ProcedureId`), `Status`, timestamps, `CurrentTaskInstanceId`, `ContextDataJson`.

### 7.7 `TaskInstance`
Atributos: `WorkflowInstanceId`, `TaskDefinitionId`, `Status`, `AssignedUserId`, `AssignedRoleCode`, `SequenceNumber`, `DueAt`, `CompletionActionId`, `Comment`.

### 7.8 `TaskActionLog`
Inmutable. `TaskInstanceId`, `ActionDefinitionId`, `PerformedBy`, `PerformedAt`, `Comment`, `RoutingId`, `ResultNextTaskInstanceId`, `RequestPayloadJson`, `EvaluationTraceJson`.

### 7.9 `TaskDeadline` *(extensión)*
Plazos normativos con efecto en el flujo. `TaskDefinitionId`, `DeadlineType` (BusinessDays / Calendar / MonthFixedDay), `DeadlineValue`, `OnExpireActionId`, `NotificationBeforeHours`.

### 7.10 `TaskFieldPermission` *(extensión)*
Permisos de campo por paso. `TaskDefinitionId`, `FormFieldKey`, `PermissionType` (Editable / ReadOnly / Hidden / Required).

### 7.11 `TaskAssignmentRule` *(opcional V1)*
`TaskDefinitionId`, `AssignmentStrategy` (Role / ContractFiscalizer / AnhPool / SpecificUser), `ParametersJson`.

---

## 8. Esquema `procedure` — Trámites Regulatorios

Puente entre workflow genérico y activos del dominio.

### 8.1 `Procedure` (agregado raíz)

Atributos:

- **Identificación:** `ProcedureId`, `ConsecutiveNumber` (`F101-RUBI-0323-2026-001`), `ControlDocReference`.
- **Referencias:** `FormDefinitionId`, `WorkflowInstanceId` (1:1), `WellId` (nullable), `FieldId` (nullable), `ProducingFormationId` (nullable).
- **Versionamiento:** `Version`, `ParentProcedureId`.
- **Estado:** `DomainStatus` (Draft / Filed / UnderReview / Approved / Returned / Rejected / Expired) — cache del workflow.
- **Fechas:** `CreatedAt`, `FiledAt`, `ApprovedAt`, `ValidUntil`.
- **Migración:** `LegacyControlFormaId`.

**Invariantes:**
- Siempre `WorkflowInstance` (1:1).
- Referencia exactamente uno de: `WellId` (Serie 100), `ProducingFormationId` (mayoría Serie 200), `FieldId` (F204, F205).

**CHECK constraint** (formaliza la invariante polimórfica, conforme §17.4.3):

```sql
CONSTRAINT CK_Procedure_SingleTarget CHECK (
    (CASE WHEN WellId              IS NOT NULL THEN 1 ELSE 0 END) +
    (CASE WHEN ProducingFormationId IS NOT NULL THEN 1 ELSE 0 END) +
    (CASE WHEN FieldId             IS NOT NULL THEN 1 ELSE 0 END) = 1
)
```

### 8.2 `ProcedureAttachment`
Documentos adjuntos. `AttachmentDefinitionId`, `FileName`, `StoragePath`, `ContentHash`, `SizeBytes`, `UploadedByUserId`, `UploadedAt`, `Version`.

### 8.3 `ProcedureObservation`
Observaciones técnicas de revisores. `ProcedureId`, `TaskInstanceId`, `UserId`, `Role`, `Text`, `ObservationType` (Technical / Geological / Administrative), `CreatedAt`.

### 8.4 `ProcedureSignature`
Firmas digitales. `ProcedureId`, `UserId`, `SignatureType` (OperatorGeologist / OperatorEngineer / AnhGeologist / AnhEngineer / AnhSupervisor), `ProfessionalLicenseNumber` (snapshot), `DocumentHash`, `SignedAt`, `DigitalCertificateId`.

### 8.5 `ProcedurePdf`
`ProcedureId`, `PdfType` (DraftPresentation / PostOperatorSignature / PostAnhApproval), `StoragePath`, `ContentHash`, `GeneratedAt`, `ControlDocReference`.

### 8.6 `ProcedureNotification`
`ProcedureId`, `NotificationType`, `RecipientUserId`, `Channel` (Email / InApp), `SentAt`, `DeliveryStatus`.

### 8.7 `ProcedureSpatialValidation`
Validaciones contra Mapa de Tierras persistidas como hechos. `ProcedureId`, `WellLocationId`, `ValidationType` (WithinContractPolygon / WithinDepartment / WithinMunicipality / DistanceToBoundary / DistanceToNearestWell), `Result` (Passes / Fails / RequiresPermit), `MeasuredValue`, `ThresholdValue`, `ExpectedDepartment`, `DetectedDepartment`, `ExecutedAt`, `SpecialPermitAttachmentId`.

---

## 9. Esquema `identity` — Identidad y Seguridad

Cumple mandato OTI: SSO + IdP externo designado por ANH + MFA obligatorio. El modelo soporta **federación OIDC** y **usuarios locales** en paralelo, según el patrón Token Exchange documentado en `CONSTITUTION-ba.md` §7.1.1 y §7.1.1.1.

> **Nota sobre clasificación de datos (Always Encrypted):** columnas de este esquema marcadas con `[DataClassification(Level ≥ 3)]` — pendiente matriz ANH (ver `CONSTITUTION-ba.md` §7.10) — se marcarán con **Always Encrypted** de SQL Server 2022 en migración posterior a la publicación formal de la matriz. Candidatas probables: `ProfessionalLicense.LicenseNumber`, `User.PhoneNumber`, columnas de documento de identidad.

### 9.0 Integración con ASP.NET Core Identity

El esquema `identity` se materializa sobre **ASP.NET Core Identity** (ver `CONSTITUTION-ba.md` §4 y §5.2). Las entidades de dominio GOP **extienden** las entidades base de Identity en lugar de duplicarlas. Esto aporta nativamente todos los controles de autenticación exigidos por §12.6 de CONSTITUTION (password hashing PBKDF2, lockout, MFA, tokens de recuperación, email confirmation) sin código adicional.

**Herencia y mapeo:**

| Entidad GOP | Clase base Identity | Nombre de tabla | Observación |
|---|---|---|---|
| `User` | `IdentityUser<long>` | `AspNetUsers` | Renombrable vía `modelBuilder.Entity<User>().ToTable("User")` si se prefiere nombre de dominio |
| `Role` | `IdentityRole<int>` | `AspNetRoles` | Idem |
| — | `IdentityUserClaim<long>` | `AspNetUserClaims` | Uso puntual; los claims principales se proyectan al JWT desde `UserRole` + `RolePermission`, no desde esta tabla |
| — | `IdentityUserLogin<long>` | `AspNetUserLogins` | **Uso nativo para federación OIDC** — `LoginProvider` + `ProviderKey` mapean el `sub` del IdP externo al `UserId` local. Esto reemplaza cualquier columna `ExternalIdpProvider` / `ExternalSubjectId` custom. |
| — | `IdentityUserToken<long>` | `AspNetUserTokens` | Tokens de MFA, recuperación, confirmación de email |
| — | `IdentityRoleClaim<int>` | `AspNetRoleClaims` | No se usa — las capabilities viven en `RolePermission` (§9.5) |
| **`UserRole`** | **NO extiende `IdentityUserRole<long>`** | `UserRole` (custom) | La tabla `AspNetUserRoles` **no se usa**. `UserRole` (§9.4) es custom con alcance extendido (`ContractId`, `OperatorId`, `FieldId`, vigencias, auditoría de asignación). |

**Columnas heredadas automáticamente en `User` (desde `IdentityUser<long>`):**

`PasswordHash`, `SecurityStamp`, `ConcurrencyStamp`, `Email`, `NormalizedEmail`, `EmailConfirmed`, `UserName`, `NormalizedUserName`, `PhoneNumber`, `PhoneNumberConfirmed`, `TwoFactorEnabled`, `LockoutEnd`, `LockoutEnabled`, `AccessFailedCount`.

Estas columnas cubren íntegramente los requisitos de §12.6 de CONSTITUTION. **No** se redeclaran en §9.1.

**Federación OIDC vía `AspNetUserLogins`:**

Cuando un usuario se autentica vía el IdP externo (ver CONSTITUTION §7.1.1, §19.8.1), `gop.identity`:

1. Recibe el `id_token` con un claim `sub` = identificador del usuario en el IdP externo.
2. Consulta `AspNetUserLogins` buscando la fila `(LoginProvider = 'AzureAd' | 'Adfs' | 'Keycloak' | ..., ProviderKey = <sub del IdP>)`.
3. Si existe, recupera el `UserId` y emite el JWT interno con los claims de dominio.
4. Si no existe (primer login federado), se crea el `User` local (sin `PasswordHash`) y la fila de `AspNetUserLogins` que lo vincula al IdP.

Un mismo `User` **MAY** tener múltiples filas en `AspNetUserLogins` (p.ej. federado con AD corporativo y también con Gov.co) y/o `PasswordHash` local para fallback, según `AuthenticationSource` (§9.1).

### 9.1 `User`

Extiende `IdentityUser<long>`. Columnas adicionales de dominio GOP:

| Columna | Tipo | Null | Descripción |
|---|---|---|---|
| `FullName` | `nvarchar(200)` | No | Nombre visible del usuario |
| `OperatorId` | `bigint` FK → `Operator` | Sí | Tenant propio (NULL para usuarios ANH) |
| `FunctionaryId` | `bigint` FK → `Functionary` | Sí | Funcionario ANH asociado (NULL para usuarios operadora) |
| `AuthenticationSource` | `tinyint` (enum: `External=1`, `Local=2`) | No | Origen de autenticación — `External` si se autentica vía IdP federado, `Local` si usa credenciales propias de ASP.NET Core Identity. Ver `CONSTITUTION-ba.md` §7.1.1.1 |
| `Status` | `tinyint` (enum: `Active`, `Inactive`, `Suspended`) | No | Estado de dominio (ortogonal al lockout de Identity) |

**Eliminado respecto a v0.4:**
- `AuthType (Local / Sso / Ldap)` → reemplazado por `AuthenticationSource (External | Local)`; el IdP específico ya se modela en `AspNetUserLogins.LoginProvider`.
- `IsSuperuser` → eliminado. Los privilegios elevados se modelan exclusivamente por rol (`ANH_Admin` con capabilities). Ver `CONSTITUTION-ba.md` §7.4 — `UserRole` es la fuente única de alcance.
- `Login`, `Email` → ya provistos por `IdentityUser<long>` (`UserName`, `Email`), no se redeclaran.

### 9.2 `ProfessionalLicense`
`UserId`, `ProfessionType` (Geologist / PetroleumEngineer), `LicenseNumber`, `IssueDate`, `ValidUntil`, `LicenseStatus`, `ValidationSource` (Cpip / Manual).

### 9.3 `Role`
`RoleId`, `Code`, `Name`, `Context` (Anh / Operator). Roles: `AnhAdministrator`, `AnhGopAdministrator`, `AnhOperationsEngineer`, `AnhGeologist`, `AnhProductionSupervisor`, `OperatorAgent`, `OperatorCoordinator`, `OperatorSupervisor`, `OperatorGeologist`, `OperatorPetroleumEngineer`.

### 9.4 `UserRole` — fuente única de alcance

Entidad N:M extendida. Es la **única** fuente de verdad del alcance de datos del usuario (junto con `User.OperatorId`). Ver `CONSTITUTION-ba.md` §7.4 para las reglas vinculantes.

**Atributos:**

| Columna | Tipo | Null | Descripción |
|---|---|---|---|
| `UserRoleId` | `bigint` PK IDENTITY | No | — |
| `UserId` | `bigint` FK → `User` | No | — |
| `RoleId` | `int` FK → `Role` | No | — |
| `ContractId` | `bigint` FK → `Contract` | Sí | Si NULL y rol operador → todos los contratos del operador del usuario. Si NULL y rol ANH → según `OperatorId`/`FieldId`/global. |
| `OperatorId` | `bigint` FK → `Operator` | Sí | Solo para roles ANH — acota a todos los contratos de un operador específico. |
| `FieldId` | `bigint` FK → `Field` | Sí | Solo para roles ANH — acota a los contratos que contienen el campo. |
| `StartDate` | `datetime2(7)` | No | Vigencia desde. |
| `EndDate` | `datetime2(7)` | Sí | NULL = indefinida. Asignación vigente si `StartDate ≤ now < EndDate OR EndDate IS NULL`. |
| `AssignedBy` | `bigint` FK → `User` | No | Auditoría — quién hizo la asignación. |
| `AssignedAt` | `datetime2(7)` | No | Cuándo. |
| `Reason` | `nvarchar(500)` | Sí | Justificación de la asignación. |

**Reglas de interpretación (7 escenarios)** — espejo de `CONSTITUTION-ba.md` §7.4.1:

| Tipo rol | `ContractId` | `OperatorId` | `FieldId` | Alcance resultante |
|---|---|---|---|---|
| Operador | NULL | NULL | NULL | Todos los contratos del `User.OperatorId` |
| Operador | X | NULL | NULL | Solo contrato X |
| ANH | NULL | NULL | NULL | Global ANH |
| ANH | X | NULL | NULL | Solo contrato X |
| ANH | NULL | Y | NULL | Todos los contratos del operador Y |
| ANH | NULL | NULL | Z | Todos los contratos que contienen el campo Z |
| ANH | X | Y | NULL | Redundante — validar que X pertenezca a Y |

**Constraints:**
- `CK_UserRole_OperatorScope_OnlyAnh` — si `OperatorId` o `FieldId` no son NULL, `Role.Context` **MUST** ser `Anh`.
- `CK_UserRole_Operator_NoCrossTenant` — si `Role.Context = Operator`, entonces `OperatorId IS NULL` (el tenant sale de `User.OperatorId`, no se re-asigna aquí).
- `CK_UserRole_ContractMatchesOperator` — si `ContractId` y `OperatorId` ambos no son NULL, validar que el contrato pertenezca al operador (trigger o validación aplicativa, el CHECK declarativo no puede cruzar tablas).

**Índices:**
- `IX_UserRole_UserId_EndDate` — para "roles vigentes de un usuario".
- `IX_UserRole_ContractId` (filtrado `WHERE ContractId IS NOT NULL`).
- `IX_UserRole_OperatorId` (filtrado `WHERE OperatorId IS NOT NULL`).
- `IX_UserRole_FieldId` (filtrado `WHERE FieldId IS NOT NULL`).

**Alcance efectivo** del usuario = unión de todas las filas `UserRole` vigentes. La resolución a lista concreta de contratos se ejecuta en `gop.identity` al emitir el JWT (ver `CONSTITUTION-ba.md` §7.4.2).

### 9.5 `Permission` / `RolePermission`
Permisos granulares por módulo y acción (capabilities). Ver `CONSTITUTION-ba.md` §19.1 para lista inicial.

### 9.6 `RefreshToken`

Modela los refresh tokens emitidos por `gop.identity` con **rotación obligatoria** y **detección de reuso** (ver `CONSTITUTION-ba.md` §7.1 y §12.6). Reemplaza `UserSession` de v0.4 (esa entidad se elimina; su información de última actividad se deriva del `RefreshToken` más reciente del usuario).

| Columna | Tipo | Null | Descripción |
|---|---|---|---|
| `RefreshTokenId` | `bigint` PK IDENTITY | No | — |
| `UserId` | `bigint` FK → `User` | No | Propietario del token |
| `FamilyId` | `uniqueidentifier` | No | Identifica la **cadena de rotación**. Todos los refresh tokens derivados de un mismo login inicial comparten `FamilyId`. Si se detecta reuso de un token ya rotado, **toda la familia** se revoca (defensa contra robo de token). |
| `TokenHash` | `varbinary(64)` | No | Hash SHA-512 del refresh token. **Nunca** se almacena el token en claro. |
| `IssuedAt` | `datetime2(7)` | No | — |
| `ExpiresAt` | `datetime2(7)` | No | Típico: `IssuedAt + 8h` (alineado con absolute timeout de §12.6). |
| `ReplacedByTokenId` | `bigint` FK → `RefreshToken` | Sí | Apunta al siguiente token de la cadena tras rotación. NULL = es el token vigente de la familia (o la familia está revocada). |
| `RevokedAt` | `datetime2(7)` | Sí | Marca de revocación. |
| `RevocationReason` | `nvarchar(50)` | Sí | `Rotated`, `Logout`, `Reuse`, `AdminRevoked`, `FamilyCompromised`, `PasswordChanged`. |
| `IpAddress` | `nvarchar(45)` | Sí | IPv4 o IPv6 desde donde se emitió. |
| `UserAgent` | `nvarchar(500)` | Sí | User-Agent del cliente en la emisión. |

**Constraints:**
- `CK_RefreshToken_RevokedImpliesReason` — si `RevokedAt IS NOT NULL` entonces `RevocationReason IS NOT NULL`.
- Unicidad implícita sobre `TokenHash` (índice único).

**Índices:**
- `UX_RefreshToken_TokenHash` (único) — lookup de validación.
- `IX_RefreshToken_UserId_RevokedAt` — búsqueda de tokens vigentes por usuario.
- `IX_RefreshToken_FamilyId` — revocación en cascada de familia.
- `IX_RefreshToken_ExpiresAt` — job de purga de tokens expirados.

**Job de mantenimiento:** tarea programada elimina filas con `ExpiresAt < now - 30 días` (configurable). La retención post-expiración permite investigación forense.

### 9.7 `RevokedAccessToken`

Revocation list para JWT de acceso, conforme `CONSTITUTION-ba.md` §7.9. Permite invalidación inmediata de un access token antes de su expiración natural (casos: logout forzado, compromiso de cuenta, cambio de password).

| Columna | Tipo | Null | Descripción |
|---|---|---|---|
| `Jti` | `char(36)` PK | No | Claim `jti` (JWT ID, UUID) del access token revocado |
| `UserId` | `bigint` FK → `User` | No | Propietario del token, para bulk-revoke por usuario |
| `RevokedAt` | `datetime2(7)` | No | — |
| `ExpiresAt` | `datetime2(7)` | No | Igual al claim `exp` del JWT. Permite purgar la fila tras vencimiento natural (ya no puede ser aceptado por ningún servicio). |
| `Reason` | `nvarchar(50)` | No | `UserLogout`, `AdminRevoked`, `CompromisedAccount`, `PasswordChanged`, `RoleChanged`. |

**Índices:**
- `IX_RevokedAccessToken_UserId` — revocación masiva por usuario.
- `IX_RevokedAccessToken_ExpiresAt` — job de purga (borra filas con `ExpiresAt < now`).

**Consulta por servicios:** el cliente `IIdentityServiceClient` de `Anh.Gop.Shared.Auth` consulta esta tabla **sólo** en operaciones críticas (firmas, aprobaciones — ver CONSTITUTION §7.9). La consulta por cada request invalidaría el diseño stateless del JWT.

---

## 10. Esquema `audit` — Auditoría y Trazabilidad

### 10.1 `AuditLog` (obligatorio por Estándar §2.4)

Atributos: `AuditLogId`, `EventType`, `AggregateType`, `AggregateId`, `UserId`, `TenantId`, `IpAddress`, `OccurredAt`, `PreviousValueJson`, `NewValueJson`, `AdditionalContextJson`, `CorrelationId`.

**Append-only.** Escritura **exclusivamente vía Audit.NET + `Audit.EntityFramework.Core`** sobre cada `DbContext` de servicio (ver `CONSTITUTION-ba.md` §11.6). **PROHIBIDO** inserción manual desde código de dominio. Particionable por `OccurredAt` para retención y performance.

> **Tabla única con discriminador:** `AuditLog` es la **única** tabla de auditoría. No existen `DataAccessLog` ni `ExportLog` separadas — son **valores especializados de `EventType`** con su detalle en `AdditionalContextJson`. Ver `CONSTITUTION-ba.md` §7.10.

**Catálogo oficial de `EventType`:**

| Categoría | Valores | Uso |
|---|---|---|
| Autenticación | `Login`, `LoginFailed`, `Logout`, `TokenIssued`, `TokenRefreshed`, `TokenRevoked`, `MfaChallenge`, `MfaFailed`, `PasswordChanged`, `PasswordResetRequested`, `AccountLocked`, `AccountUnlocked` | Todos los eventos del pipeline de autenticación — emitidos por `gop.identity` |
| Autorización | `AccessDenied`, `RoleAssigned`, `RoleRevoked`, `PermissionChanged` | Decisiones de autorización denegadas; cambios de rol/permiso en `UserRole` / `RolePermission` |
| Dominio | `Create`, `Update`, `Delete`, `StateTransition`, `Approve`, `Reject`, `Sign`, `Submit` | Cambios en entidades de dominio. Audit.NET los emite automáticamente por cambios detectados en el `DbContext` |
| Datos sensibles | `DataAccess`, `Export` | Acceso a entidades `[DataClassification(Level ≥ 2)]` y exportaciones masivas (ver CONSTITUTION §7.10) |
| Administración | `UserCreated`, `UserDeactivated`, `UserReactivated`, `TenantCreated` | Operaciones administrativas |
| Integración | `IntegrationCallSucceeded`, `IntegrationCallFailed` | Complemento a `IntegrationLog` (§10.2) para eventos de alto nivel |

**Convención:** valores de `EventType` se guardan como `nvarchar(40)` con el código exacto del catálogo (case-sensitive). Se valida en la shared library `Anh.Gop.Shared.Infrastructure` vía enum + mapping.

**Detalle específico por `EventType`:** va en `AdditionalContextJson`. Ejemplos:

- `DataAccess` → `{ "endpoint": "/api/v1/wells/{id}", "resourceType": "Well", "resourceId": 1234, "accessedFields": ["fiscalizedUwi","coordinates"], "justification": "..." }`
- `Export` → `{ "entityType": "MonthlyWellProduction", "rowCount": 4580, "filters": { "year": 2025 }, "destination": "xlsx", "justification": "..." }`
- `StateTransition` → `{ "from": "Drilling", "to": "Producing", "trigger": "F103_Approved", "procedureId": 789 }`
- `TokenIssued` → `{ "tokenType": "access", "jti": "...", "expiresAt": "...", "authenticationSource": "External", "idpProvider": "AzureAd" }`

**Índices recomendados:**
- `IX_AuditLog_OccurredAt` (default de particionado).
- `IX_AuditLog_UserId_OccurredAt`.
- `IX_AuditLog_AggregateType_AggregateId` (histórico por entidad).
- `IX_AuditLog_EventType_OccurredAt` (reportes por tipo de evento).
- `IX_AuditLog_CorrelationId` (trazabilidad end-to-end).

### 10.2 `IntegrationLog`
Log de llamadas a sistemas externos (SOLAR, VAF, Mapa de Tierras, AVM, DANE, CPIP, ControlDoc, SMTP). Atributos: `TargetSystem`, `Endpoint`, `RequestPayload`, `ResponsePayload`, `StatusCode`, `ResponseTimeMs`, `Error`, `OccurredAt`, `ProcedureId`.

### 10.3 `BulkLoadLog`
Cargues masivos. `FileName`, `LoadType`, `UserId`, `LoadedAt`, `TotalRecords`, `SuccessfulRecords`, `FailedRecords`, `ErrorDetailsJson`.

---

## 11. Esquema `integration` — Staging y Homologación

### 11.1 Staging tables
Tablas espejo antes de validación: `StgSolarContract`, `StgSolarOperator`, `StgSolarField`, `StgAvmWellProduction`, `StgDaneMunicipality`, `StgMapaTierrasPolygon`.

### 11.2 Tablas de homologación
Mapeo externo ↔ GOP: `HomologationField`, `HomologationContract`, `HomologationOperator`, `HomologationWell`. Atributos: `SourceSystem`, `ExternalId`, `GopId`, `SyncedAt`, `Status`.

### 11.3 Colas de sincronización
`SyncQueue` con `EventType`, `PayloadJson`, `Status`, `RetryCount`, `LastError`, bidireccional GOP ↔ sistemas externos.

### 11.4 Polígonos cacheados
Copias locales de polígonos Mapa de Tierras (contratos, departamentos, municipios) con TTL, para validaciones espaciales sin dependencia en tiempo real.

---

## 12. Mapa de dependencias entre esquemas

```
        ┌──────────────┐
        │    audit     │  ← recibe FKs de todos; ninguno depende de él
        └──────────────┘

        ┌──────────────┐         ┌──────────────┐
        │  integration │─→ feeds │ core, catalog│
        └──────────────┘         └──────────────┘
                                         │
                         ┌───────────────┼───────────────┐
                         ▼               ▼               ▼
                   ┌──────────┐    ┌──────────┐    ┌──────────┐
                   │   ops    │    │   prod   │    │ identity │
                   └────┬─────┘    └────┬─────┘    └────┬─────┘
                        │               │               │
                        └───────────────┼───────────────┘
                                        ▼
                                 ┌──────────────┐
                                 │  procedure   │
                                 └──────┬───────┘
                                        │
                                        ▼
                                 ┌──────────────┐
                                 │   workflow   │  ← no depende de
                                 └──────────────┘    ningún otro
```

### 12.1 Topología de permisos de base de datos en V1

Alineado con `CONSTITUTION-ba.md` §11.6.6. En V1 los microservicios **comparten instancia física** de SQL Server 2022 pero cada uno opera sobre su **esquema dedicado**. El aislamiento es **lógico**, impuesto por permisos de BD.

**Principio:** cada microservicio tiene un **login SQL propio**. Los permisos se otorgan al nivel más estrecho posible para preservar los límites de bounded context aún cuando la instancia sea compartida.

**Matriz de permisos por microservicio:**

| Microservicio | Esquema propio (`GRANT` completo: SELECT/INSERT/UPDATE/DELETE/EXECUTE) | Escritura a auditoría | Lectura cross-schema | Observación |
|---|---|---|---|---|
| `gop.core` | `core` | `INSERT` en `audit.AuditLog`, `audit.IntegrationLog` | — | Dueño de entidades transversales |
| `gop.operations` | `ops` | `INSERT` en `audit.AuditLog`, `audit.IntegrationLog` | `SELECT` sobre `core`, `catalog` | Consume referencias |
| `gop.production` | `prod` | `INSERT` en `audit.AuditLog`, `audit.IntegrationLog` | `SELECT` sobre `core`, `catalog` | Consume referencias |
| `gop.procedures` | `procedure` | `INSERT` en `audit.AuditLog`, `audit.IntegrationLog` | `SELECT` sobre `core`, `catalog`, `ops`, `prod` (lecturas de contexto) | Orquesta trámites |
| `gop.workflow` | `workflow` | `INSERT` en `audit.AuditLog`, `audit.IntegrationLog` | — | Motor genérico, no conoce dominios |
| `gop.identity` | `identity` | `INSERT` en `audit.AuditLog` | — | Emisor de tokens y dueño del `User` |
| `gop.audit` | — (sin esquema propio de negocio) | — (no escribe — es consumidor) | `SELECT` sobre `audit.*` | Rol de consulta/exportación |
| `gop.integration` | `integration` | `INSERT` en `audit.AuditLog`, `audit.IntegrationLog` | `SELECT` sobre esquemas destino según cada homologación | Staging y sincronización |

**Reglas vinculantes:**

1. **Ningún microservicio tiene `SELECT` sobre el esquema de otro** excepto lo explícitamente declarado arriba (y documentado vía ADR si se amplía).
2. **Ningún microservicio (excepto `gop.audit`) tiene `SELECT` sobre `audit.*`** — la auditoría es write-only para los escritores.
3. **`gop.audit` no tiene permiso de escritura** sobre `audit.*` — es lector puro. Las migraciones DbUp del esquema `audit` las corre un login administrativo distinto durante el deployment.
4. **Ningún microservicio tiene `CONTROL` o `ALTER`** sobre su esquema en runtime. Las migraciones corren con un login administrativo separado y se ejecutan sólo durante deployments controlados (ver §11.3 de CONSTITUTION).

**Evolución V2+:** el ADR correspondiente evaluará separación física (instancia o BD por microservicio) según volumen, requisitos de compliance y operación. La matriz lógica de permisos se preserva en ese escenario — sólo cambia el aislamiento físico.

---

## 13. Mapeo de Homologación con el Legado

| Nuevo Schema.Entity | Tabla(s) Legado Principal |
|---|---|
| `core.Operator` | `dbo.Empresa`, `dbo.OperadoraMerge` |
| `core.Contract` | `dbo.Contrato`, `dbo.ContratoBase` |
| `core.Field` | `dbo.Campo`, `dbo.CampoContrato` |
| `core.Block` | `GOP.Bloque` |
| `catalog.Formation` | `GOP.Formacion`, `dbo.CampoFormacion` |
| `catalog.FormDefinition` | `dbo.Forma`, `GOP.FormaMinisterial` |
| `catalog.CoordinateReferenceSystem` | parte de `dbo.OrigenCoord` |
| `ops.Well` | `dbo.Well`, `dbo.PozoResp` |
| `ops.WellLocation` | columnas `Coor*` y `Longitud/Latitud` de `dbo.Well` |
| `ops.WellStateHistory` | `dbo.WellEstado` + `GOP.HistoricoEtapaWell` + `GOP.HistoricoWell` |
| `ops.WellPhase` | `dbo.WellFase` |
| `ops.Form101Data` + satélites | `GOP.Forma4CR` + satélites |
| `ops.Form103Data` + satélites | `GOP.Forma6CR`, `GOP.Forma6CRPOG`, `GOP.Forma6CRPOP` |
| `ops.IdopData` + satélites | `GOP.Idop` + ~10 satélites |
| `ops.IdocData` + satélites | `GOP.Idoc` + satélites |
| `prod.MonthlyWellProduction` | `IDP.ProduccionPozo` |
| `prod.FieldBalance` | `IDP.BalanceOil`, `IDP.BalanceGas`, `IDP.BalanceAgua` |
| `prod.PotentialTest` | `IDP.PruebasPotenciales` |
| `prod.Tank` | `IDP.Tanque`, `IDP.TanqueSaldo` |
| `procedure.Procedure` | `GOP.ControlForma` (descompuesta) |
| `identity.User` | `dbo.Usuario`, `dbo.UsuarioPwd` |
| `identity.Role`, `identity.UserRole` | `dbo.Rol`, `dbo.RolUsuario` |
| `audit.AuditLog` | `dbo.HistorialEstado` + `auditoria.*` |
| `audit.IntegrationLog` | `IDP.LogInterOperabilidad_*` |
| `integration.Stg*` | `IDP.T_*` |
| `integration.Homologation*` | `IDP.HomoCampo`, `HomoContrato`, `HomoOperador`, `HomoPozo` |

---

## 14. Decisiones fijadas por esta versión

1. **9 esquemas.**
2. **Motor:** SQL Server 2022 / SQL Azure.
3. **Nomenclatura:** PascalCase + inglés (obligatorio por Estándar).
4. **`core` separado de `catalog`.**
5. **Dos agregados raíz operacionales** (`Well`, `ProducingFormation`).
6. **Workflow genérico desacoplado.**
7. **`Procedure` puente entre workflow y dominio.**
8. **FSM del Well con tres dimensiones ortogonales.**
9. **FSM híbrida** (catálogo + servicio).
10. **Coordenadas con CRS canónico único** (WGS84 via tipo `geography`).
11. **Validaciones espaciales como hechos persistidos.**
12. **Formas como entidades tipadas.**
13. **`AuditLog` unificado, obligatorio.**
14. **`integration` separado.**
15. **Autenticación SSO + AD + MFA** (mandato OTI).
16. **Compatibilidad PPDM 3.9.**

---

## 15. Decisiones abiertas para validación posterior

1. **Una sola BD con 9 esquemas vs múltiples BDs.** V1 recomienda una BD; abrir separación de `audit` e `integration` si volúmenes lo exigen.
2. **Replicación de catálogos SOLAR vs federación.** Recomendación: replicación (disponibilidad + performance).
3. **`WellState` en `catalog` u `ops`.** Recomendación: `ops` (reglas de transición son del dominio).
4. **Precondiciones como JSON vs subtabla.** V1 JSON; migrar si crece complejidad.
5. **Particionamiento de `MonthlyWellProduction` y `AuditLog`:** por año/mes.
6. **Nomenclatura de brazos multilateral** (pendiente HU F101 PA-02).
7. **Tabla `reporting` / vistas materializadas.** Fuera de V1.

---

## 16. Próximos pasos

1. **Atributos completos y constraints** por entidad: tipos, NOT NULL, UNIQUE, FK, CHECK.
2. **Índices** por agregado raíz, FKs, columnas de filtro.
3. **Script DDL inicial** alineado con nomenclatura del Estándar.
4. **Diccionario de catálogos poblado** (estados, transiciones, CRS, roles).
5. **Definiciones de workflow por forma** (F101, F103, F202…).
6. **Definiciones de transiciones del Well** con eventos y precondiciones.
7. **Plan de migración detallado** desde legado.
8. **Versionado de esquema** con **DbUp** o **Flyway** (integrable con el flujo CI/CD del Estándar).
9. **tSQLt tests** para procedimientos críticos de transición y validaciones.

---

## 17. Gobernanza Evolutiva del Modelo

Esta sección es la hoja de ruta para que el modelo siga siendo sano a lo largo del desarrollo y no se degrade como el legado. Responde a cuatro preguntas prácticas:

### 17.1 ¿Cómo evoluciona este documento?

**17.1.1 Estado del documento.** Este documento no es un entregable congelado. Es un **artefacto vivo** que acompaña el desarrollo del GOP 360° durante toda su vida. Su fuente de verdad vive en el repositorio de código (no en Drive) para versionarlo junto al schema.

**17.1.2 Estructura del versionado.**

- **Versión mayor (0.x → 1.0, 1.x → 2.0):** cambio estructural — nuevo bounded context, división de esquema, cambio de motor, ruptura de compatibilidad con el legado. Requiere aprobación de arquitectura OTI + equipo funcional.
- **Versión menor (0.2 → 0.3):** refinamientos significativos — agregar entidades, cambiar nomenclatura, redefinir un patrón. Requiere aprobación del líder técnico.
- **Versión patch (0.3.1):** correcciones menores, aclaraciones, ejemplos nuevos. Cualquier desarrollador puede proponer; revisión por PR.

**17.1.3 Ciclo de actualización.**

1. **Detección** de un gap: un desarrollador encuentra que su historia de usuario exige una entidad/campo no contemplado.
2. **Propuesta** vía PR al documento (en el repo), con justificación y análisis de impacto.
3. **Revisión** por el arquitecto de datos (síncrona si es mayor, asíncrona si es menor).
4. **Decisión** registrada como ADR (Architecture Decision Record) en `docs/adr/` del repo.
5. **Merge** del PR y comunicación al equipo.
6. **Implementación** del cambio en el schema via migración versionada.

**17.1.4 ADRs (Architecture Decision Records).** Cada decisión significativa se captura en un archivo `docs/adr/NNNN-titulo-corto.md` con estructura:
- Contexto: qué problema resolvemos.
- Decisión: qué elegimos.
- Alternativas consideradas: qué descartamos y por qué.
- Consecuencias: qué efectos (positivos y negativos) tiene.
- Estado: Proposed / Accepted / Deprecated / Superseded.

Los ADRs son inmutables. Si una decisión cambia, se crea uno nuevo que *supersede* el anterior; el antiguo no se borra — queda como registro histórico.

**17.1.5 Secciones de este documento y responsables.**

| Sección | Responsable | Cadencia de revisión |
|---|---|---|
| §1 Principios | Arquitecto de datos | Por versión mayor |
| §2 Topología de esquemas | Arquitecto de datos | Por versión mayor |
| §3-§11 Entidades por esquema | Líder técnico del esquema | Continua |
| §12 Dependencias | Arquitecto de datos | Por versión mayor |
| §13 Homologación legado | Líder de migración | Continua |
| §14-§15 Decisiones | Arquitecto de datos | Continua |
| §16 Próximos pasos | PMO | Por sprint |
| §17 Gobernanza | Arquitecto de datos | Semestral |

### 17.2 ¿Cómo decidir en qué esquema va una tabla nueva?

Cuando aparece una entidad nueva en el desarrollo, aplicar el siguiente árbol de decisión:

**Paso 1 — ¿Es un log de evento/cambio?**
→ Sí: va en `audit` (`AuditLog`, `IntegrationLog`, `BulkLoadLog`).
→ No: continuar.

**Paso 2 — ¿Es staging o homologación con sistema externo?**
→ Sí: va en `integration` (`Stg*`, `Homologation*`, `SyncQueue`, `CachedPolygon`).
→ No: continuar.

**Paso 3 — ¿Involucra autenticación, usuarios, roles, permisos, sesiones?**
→ Sí: va en `identity`.
→ No: continuar.

**Paso 4 — ¿Es un nodo/instancia de flujo genérico sin conocer el dominio?**
→ Sí: va en `workflow`. **Prueba:** ¿podrías reutilizar esta tabla para un flujo que no sea una forma regulatoria? Si la respuesta es no, NO va en `workflow`.
→ No: continuar.

**Paso 5 — ¿Es un expediente, anexo, firma, PDF, notificación o validación de un trámite regulatorio?**
→ Sí: va en `procedure`.
→ No: continuar.

**Paso 6 — ¿Es payload estructurado de una forma Serie 100 o Serie 200?**
→ Sí Serie 100: va en `ops`.
→ Sí Serie 200: va en `prod`.
→ No: continuar.

**Paso 7 — ¿Pertenece al ciclo de vida del Well (estados, fases, situaciones, revestimiento, brazos, ubicaciones)?**
→ Sí: va en `ops`.
→ No: continuar.

**Paso 8 — ¿Pertenece a producción de campo (modalidad, producción mensual, balance, pruebas, inyección, emisiones, tanques)?**
→ Sí: va en `prod`.
→ No: continuar.

**Paso 9 — ¿Es una lista cerrada estable de referencia, administrada por configuración?**
→ Sí: va en `catalog`.
→ No: continuar.

**Paso 10 — ¿Es una entidad transaccional transversal (Operator, Contract, Field, Block, ClusterLocation)?**
→ Sí: va en `core`.
→ No: **STOP** — la entidad probablemente esté mal definida o revele un bounded context nuevo no contemplado. Escalar al arquitecto.

**Caso borderline — reglas de desempate:**
- Si duda entre dos esquemas, preferir el más específico (el bounded context sobre `core`).
- Si la entidad es leída por varios pero escrita por uno, va donde vive el escritor.
- Si se hace pivote entre varios (tabla relacional pura), va donde vive el agregado raíz al que "cuelga".

### 17.3 ¿Cómo cambiar la decisión de esquema de una tabla existente?

Mover una tabla entre esquemas en SQL Server es operación costosa (cambia FKs, vistas, procedimientos, permisos, ORMs). La prescripción es:

**17.3.1 Umbral de cambio.** Solo se mueve una tabla si:
- El equipo detecta que el esquema inicial fue incorrecto y genera consultas cross-schema muy frecuentes.
- El bounded context al que pertenece cambió (ej: V1 la pusimos en `ops`, después se determinó que es transversal y debe ir a `core`).
- Hay impacto de rendimiento demostrable (queries cross-schema con penalidad medible).

**17.3.2 Proceso de cambio:**
1. **ADR obligatorio** documentando razón.
2. **Análisis de impacto:** qué queries, qué servicios, qué vistas, qué procs.
3. **Migración en dos pasos** para minimizar downtime:
   - Paso A: crear tabla en nuevo esquema, sincronizar datos via trigger o schema-bound view.
   - Paso B: actualizar código aplicativo para leer del nuevo lugar.
   - Paso C: dejar la tabla vieja como vista apuntando a la nueva durante 1-2 sprints.
   - Paso D: eliminar la vista antigua cuando todo el código apunte al nuevo lugar.
4. **Comunicación** al equipo con fechas y ventana de cambio.
5. **Auditoría:** revisar que todos los `AuditLog` sigan apuntando correctamente (cambia el `AggregateType`).

**17.3.3 Opción más barata — alias via vista.** En lugar de mover la tabla físicamente, crear una vista en el esquema correcto que la apunta. Costo cero de migración. El desarrollador puede usar `newSchema.Entity` y la implementación real sigue en `oldSchema.Entity`. Solo se mueve físicamente cuando hay razón de rendimiento.

### 17.4 ¿Cuándo conviene una FK y cuándo no?

Este es uno de los temas donde más se degrada un modelo de largo plazo. El legado de GOP está lleno de columnas `*_Id` sin FK declarada (ej: `Well_CampoId INT NULL` sin `FOREIGN KEY`), lo que produce datos huérfanos. Pero también se cometen errores al poner FKs excesivas que dificultan migraciones e integración.

**17.4.1 Siempre declarar FK cuando:**
- La relación es **intra-esquema** y obligatoria para la integridad del agregado (ej: `Well.OperatorId` → `core.Operator.OperatorId`).
- La relación es **1:1 o 1:N estricta** dentro de un agregado (ej: `WellLocation.WellId` → `ops.Well.WellId`).
- Se espera **cascada en delete** intencional (ej: eliminar un trámite elimina sus anexos, firmas, PDFs).
- La columna es **obligatoria** (`NOT NULL`) y el dato referenciado debe existir.

**17.4.2 Considerar FK pero evaluar:**
- Referencias a catálogos (`catalog.*`). Se declaran FK en general pero se permite `ON DELETE NO ACTION` y el catálogo no se borra físicamente — solo se desactiva (`IsActive = 0`).
- Referencias cross-schema a `core` — igual, se declaran FK pero el catálogo es apend-only operativamente.

**17.4.3 NO declarar FK (solo mantener columna lógica) cuando:**
- La referencia apunta a un **sistema externo** (la columna es semánticamente un "identificador externo", no una FK real). Ej: `SolarId`, `AvmRecordId`, `ControlDocReference`.
- La tabla está en `integration` (staging). Los datos pueden llegar en orden variable y las FKs bloquearían el cargue.
- La referencia es a un **evento histórico** que puede haber sido borrado (auditoría). El `AuditLog.AggregateId` no tiene FK real a la entidad porque puede referenciar entidades ya eliminadas.
- La tabla registra un **snapshot** que debe sobrevivir cambios/eliminación del origen. Ej: `ProcedureSignature.ProfessionalLicenseNumber` guarda el número como texto (no FK) porque la matrícula puede cambiar y la firma debe preservarse con el valor del momento.
- Existe una relación **N:M polimórfica** (una columna puede apuntar a varios tipos distintos). Ej: `Procedure` referencia alternativamente `WellId` / `FieldId` / `ProducingFormationId`. Se declaran las tres FKs como nullable con CHECK constraint que garantiza exactamente una no nula, **no** se usa columna polimórfica.
- La columna es `LegacyXxxId` — referencia al sistema legado inexistente en el nuevo modelo. Nunca FK.

**17.4.4 Regla de oro:** si dudas entre FK o no FK, **declarar la FK**. Es más fácil quitar una FK después que agregarla sobre datos ya sucios. La integridad referencial es barata cuando se establece temprano y caótica cuando se establece tarde.

**17.4.5 Cascadas:**
- `ON DELETE CASCADE` solo dentro de un agregado (Procedure → ProcedureAttachment sí; Well → MonthlyWellProduction **no**, la producción sobrevive al pozo).
- `ON DELETE NO ACTION` por defecto para todo lo demás.
- `ON UPDATE` nunca — las PKs no se actualizan en este modelo.
- Considerar **soft delete** (`IsDeleted`, `DeletedAt`) en lugar de DELETE físico para entidades críticas.

### 17.5 ¿Cómo minimizar el impacto de cambios en el schema?

**17.5.1 Versionado de schema con migraciones.** Obligatorio usar **DbUp** o **Flyway** (compatible con el flujo CI/CD del Estándar §2.7). Cada cambio es una migración numerada y firmada. Nunca editar el schema manualmente en ambientes superiores a desarrollo local.

**17.5.2 Categorías de cambio por impacto:**

| Tipo | Impacto | Ejemplo | Ventana |
|---|---|---|---|
| **Aditivo** | Cero | Agregar columna NULLABLE, agregar tabla, agregar índice CONCURRENTLY | Sin downtime |
| **Compatible** | Bajo | Agregar columna con DEFAULT, ampliar tipo (VARCHAR(50) → VARCHAR(100)) | Sin downtime, backfill opcional |
| **Ruptor** | Medio-alto | Renombrar columna, cambiar tipo incompatible, NOT NULL sobre existente, eliminar columna | Requiere proceso Expand-Contract |
| **Estructural** | Alto | Mover tabla entre esquemas, dividir tabla en dos, cambiar PK | Requiere ADR, plan de migración explícito, ventana de mantenimiento |

**17.5.3 Patrón Expand-Contract para cambios ruptores:**

1. **Expand:** agregar lo nuevo SIN quitar lo viejo. Ambos coexisten.
2. **Migrate:** backfill de datos del esquema viejo al nuevo (job en batch).
3. **Dual-write:** aplicación escribe a ambos lados temporalmente.
4. **Dual-read-transition:** aplicación lee del nuevo, con fallback al viejo.
5. **Contract:** una vez verificado, se elimina el esquema viejo.

Este patrón permite despliegues **zero-downtime** exigidos por el Estándar §1 (SLA >99.9%).

**17.5.4 Reglas preventivas para facilitar cambios futuros:**
- Nunca usar `SELECT *` en procedimientos o servicios — obliga a revisar todo cambio de columna.
- ORMs configurados con mapeo explícito de columnas (no convenciones implícitas).
- Nombres de constraints explícitos (`PK_Well` no `PK__Well__E2CDE1B2A4B3C6F2` autogenerado) para poder scriptarlas.
- Vistas como contrato de lectura para otros servicios — los cambios internos al schema no rompen consumidores si la vista se mantiene estable.

**17.5.5 Compatibilidad hacia atrás del API.** Los cambios de schema NO deben forzar cambios de contrato de API simultáneos. El backend puede introducir la columna nueva y exponer el endpoint viejo hasta que los consumidores (Angular, integraciones) migren. Versionado de API `/api/v1/...` y `/api/v2/...` habilita esto (conforme Estándar §2.3).

### 17.6 ¿Cómo mantener compatibilidad con migración del GOP legado?

**17.6.1 Principio de preservación.** Todo registro migrado conserva:
- Su identificador original en campo `LegacyXxxId` (siempre NULLABLE, nunca FK).
- Una marca temporal de migración (`MigratedAt`).
- Una marca de fuente (`DataSource = 'LegacyMigration'`).
- El payload original en `OriginalPayloadJson` cuando la transformación es lossy (ej: coordenadas regionales → WGS84).

**17.6.2 No romper la migración.** Durante el período de migración (Fases 4-5 del plan ANH), las siguientes reglas son inviolables:
- **No agregar NOT NULL** a columnas nuevas sin DEFAULT — datos legados pueden no tenerla.
- **No eliminar `LegacyXxxId`** hasta que se declare oficial el cierre de la migración.
- **No cambiar la lógica de transformación** sin re-migrar los registros afectados.
- **No borrar** tablas de `integration.Homologation*` — son la llave para re-migrar o reconciliar.

**17.6.3 Tabla de equivalencias.** Mantener vigente la §13 de este documento. Cuando se agrega una entidad nueva que tiene correspondiente en el legado, documentarlo aquí inmediatamente.

**17.6.4 Dobles vías durante cutover.** Durante la fase de piloto (Fase 4), el GOP nuevo y el SOLAR-VORP coexisten. Esto exige:
- Identificadores disjuntos: la nueva BD **nunca** usa el mismo rango de PKs que el legado. Si el legado usa `IDENTITY(1,1)` empezando en 1, el nuevo empieza en `IDENTITY(10000000,1)` o usa `bigint` para evitar colisiones.
- Sync bidireccional temporal: cambios en GOP nuevo propagan a SOLAR para mantener consistencia mientras la migración termina.
- `integration.SyncQueue` registra cada evento de sincronización para reproceso ante fallos.

**17.6.5 Validación post-migración.** Cada batch migrado se valida con:
- Conteo de registros origen vs destino (≠ 0% de pérdida o justificada).
- Validación de integridad referencial (no FKs huérfanas).
- Validación de negocio (UWIs únicos, estados coherentes, coordenadas transformadas correctamente).
- Validación espacial (puntos dentro de Colombia con tolerancia razonable).

Estas validaciones viven en `tests/migration/*` como tSQLt tests que corren en CI antes de promover la migración.

**17.6.6 Reversibilidad.** Cada migración debe tener un script de rollback. Durante las primeras 48h post-cutover de cada batch, el rollback debe ser viable. Después de ese período, el rollback pasa a ser un proceso excepcional con aprobación gerencial.

**17.6.7 Datos "incorregibles" del legado.** El legado contiene registros imposibles de migrar automáticamente (coordenadas sin datum identificable, pozos con referencias rotas, formas con estados inconsistentes). Para estos:
- Se migran con flag `RequiresManualReview = 1`.
- Se asigna un estado especial en el pozo (`DataQualityIssue`).
- Se bloquea la creación de nuevos trámites hasta que el operador regularice vía formulario específico.
- Se genera reporte para el equipo de migración con el backlog de revisión.

---

*Fin del documento — versión 0.3*
