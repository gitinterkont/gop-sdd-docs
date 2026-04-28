# Especificación Backend — Forma 202: Informe Mensual de Producción

**Feature:** RQF_GOP_13  
**Versión:** 1.0  
**Fecha:** 2026-04-28  
**Estado:** Propuesta — Pendiente aprobación del arquitecto humano  
**Stack:** .NET Core 10 (C#), MS SQL Server 2022  

---

## 1. Contexto y Objetivo

Implementar la lógica de negocio, persistencia, integraciones externas y controles de seguridad para la Forma 202 — Informe Mensual de Producción — Pozos de Petróleo y Gas.

La Forma 202 registra el detalle mensual de producción por pozo y formación productora, con precarga desde AVM, edición con trazabilidad, validación cruzada con Formas 204/205, firma electrónica, flujo de aprobación con plazos normativos, generación de PDF (SSRS) y enlace con ControlDoc.

---

## 2. Actores, Roles y Permisos

### 2.1 Matriz de Autorización

| Acción | Operador GOP | Supervisor ANH | Admin GOP |
|---|---|---|---|
| Listar formas (filtradas por alcance) | ✅ | ✅ | ✅ (global) |
| Crear forma | ❌ | ❌ | ✅ |
| Ver detalle de forma | ✅ (propias) | ✅ (asignadas) | ✅ |
| Actualizar encabezado | ✅ (en Registration) | ❌ | ❌ |
| Disparar precarga AVM | ✅ (en Registration) | ❌ | ❌ |
| Leer grilla de producción | ✅ | ✅ | ✅ |
| Agregar pozo manual | ✅ (en Registration) | ❌ | ❌ |
| Actualizar fila de producción | ✅ (en Registration) | ❌ | ❌ |
| Eliminar fila manual | ✅ (en Registration) | ❌ | ❌ |
| Cargue masivo Excel | ✅ (en Registration) | ❌ | ❌ |
| Leer totales | ✅ | ✅ | ✅ |
| Ejecutar validación cruzada | ✅ | ✅ | ✅ |
| Firmar y enviar | ✅ (en Registration) | ❌ | ❌ |
| Aprobar | ❌ | ✅ (en Submitted) | ❌ |
| Devolver | ❌ | ✅ (en Submitted, antes del día 19) | ❌ |
| Consultar log de cambios | ❌ | ✅ | ✅ |
| Descargar PDF | ✅ | ✅ | ✅ |
| Ver historial de versiones | ✅ | ✅ | ✅ |

### 2.2 Alcance de Datos
- **Operador GOP:** Solo formas de contratos asignados al usuario via `UserRole` (ver data-model.md §9.4).
- **Supervisor ANH:** Solo formas de contratos/campos/operadores asignados via `UserRole`.
- **Admin GOP:** Acceso global (sin restricción de tenant).
- El filtrado de datos se aplica **en toda consulta a BD**, nunca se confía en filtros del frontend.

---

## 3. Historias de Usuario y Criterios de Aceptación

Ver §3 y §4 de `spec-frontend.md` para las historias completas. Los criterios de aceptación aplican de forma transversal. A continuación se detallan los criterios **específicos del backend** en formato Given/When/Then:

### CA-BE-01 — Creación automática mensual
**Given** es el primer día del mes calendario  
**When** el job programado se ejecuta  
**Then** se crean instancias de Forma 202 para cada combinación vigente (campo + formación + modalidad) de cada operador activo  
**And** cada instancia se crea en estado `Initial` y transiciona a `Registration`  
**And** se genera el consecutivo único  
**And** si la instancia ya existe (idempotencia), no se duplica  

### CA-BE-02 — Precarga AVM con trazabilidad
**Given** una Forma 202 en estado `Registration`  
**When** se invoca el endpoint de precarga  
**Then** el backend consulta AVM para obtener pozos activos y datos de producción del mes  
**And** inserta las filas en `MonthlyWellProduction` con `DataOrigin = 'Avm'`  
**And** si la precarga se repite, actualiza solo las filas no modificadas por el operador  
**And** registra en `IntegrationLog` los detalles de la llamada a AVM  

### CA-BE-03 — Log automático de modificaciones
**Given** una fila de producción con `DataOrigin = 'Avm'`  
**When** el operador envía un PUT con valores distintos a los originales de AVM  
**Then** el backend registra en `MonthlyProductionChange`: valor original AVM, valor nuevo, campo modificado, pozo, usuario, timestamp  
**And** marca la fila como `IsModified = true`  

### CA-BE-04 — Validación de acumulados
**Given** un pozo con acumulado del mes anterior en la BD  
**When** el operador envía un valor de acumulado (petróleo, agua, gas, días) menor al del mes anterior  
**Then** el backend retorna 400 con código `CUMULATIVE_DECREASE`  
**And** el acumulado no se persiste  

### CA-BE-05 — Envío automático por incumplimiento
**Given** una Forma 202 en estado `Registration`  
**When** el job de plazos detecta que han transcurrido 7 días hábiles desde el 1.° del mes sin envío  
**Then** el backend cambia el estado a `Submitted` automáticamente  
**And** genera comunicación de incumplimiento vía SMTP  
**And** registra en `AuditLog` el evento de envío automático  

### CA-BE-06 — Envío automático por expiración de corrección
**Given** una Forma 202 devuelta (en estado `Registration` tras devolución)  
**When** transcurren 3 días hábiles sin reenvío  
**Then** el backend cambia el estado a `Submitted` automáticamente  
**And** registra el evento  

### CA-BE-07 — Generación de PDF vía SSRS
**Given** una Forma 202 que cambia a estado `Submitted` o `Approved`  
**When** la transición de estado se completa  
**Then** se encola un proceso para generar el PDF vía SSRS  
**And** el PDF generado se almacena y se registra en `ProcedurePdf`  
**And** se radica en ControlDoc y se guarda el número de radicado  

### CA-BE-08 — Persistencia de pozos entre meses
**Given** un pozo registrado en la Forma 202 de marzo/2026 para campo X, formación Y  
**When** se crea la Forma 202 de abril/2026 para la misma combinación  
**Then** el pozo aparece en la grilla de abril aunque su estado sea "Abandonado" o "Inactivo" (RN-13)  
**And** los acumulados del mes anterior se cargan como base para el cálculo automático  

---

## 4. Reglas de Negocio

### 4.1 Reglas de Encabezado

| ID | Regla | Validación |
|---|---|---|
| RN-01 | El mes de reporte es automáticamente el mes vencido (mes anterior al corriente) | Cálculo automático |
| RN-02 | Fuente mandante de operadores: BD SOLAR | Integración |
| RN-03 | Cada operador solo ve formas de sus contratos; cada usuario solo accede a contratos asignados | Filtrado por `UserRole` |
| RN-04 | Todas las formaciones con producción histórica deben estar en AVM | Validación de integridad |
| RN-05 | Se debe poder visualizar acumulados del campo completo (suma de todas las formaciones) | Cálculo |
| RN-06 | La modalidad de explotación debe ser concordante con lo aprobado en Forma 204 | Validación (dependencia externa pendiente — PA-06) |
| RN-07 | Debe existir combinación válida campo + formación + modalidad antes de habilitar consulta | Validación previa a precarga |

### 4.2 Reglas de Grilla de Producción

| ID | Regla | Validación |
|---|---|---|
| RN-08 | Datos AVM corresponden solo a pozos activos en dicho sistema | Integración |
| RN-09 | Todos los campos precargados son editables por el operador | Permiso de edición |
| RN-10 | Cada modificación sobre dato AVM genera log (original, nuevo, campo, pozo, usuario, timestamp) | Auditoría automática |
| RN-11 | Primer registro de acumulados (Versión 0) permite ingreso manual; posterior es automático | Cálculo condicional |
| RN-12 | Nombre de pozo se valida contra Forma 103 (GOP Operaciones); si no coincide, se permite con warning | Validación con warning |
| RN-13 | Pozo registrado persiste en meses subsiguientes independientemente de su estado | Regla de persistencia |
| RN-14 | Pozos de GOP Operaciones deben estar en AVM si presentaron producción en prueba inicial | Validación |
| RN-15 | Municipio mandante: GOP Operaciones (coordenadas verificadas por cartógrafa ANH) | Dato referencial |
| RN-16 | Días acumulados: primer registro manual, luego cálculo automático; nunca puede disminuir | Cálculo + validación |
| RN-17 | Sumatoria petróleo Forma 202 (todas formaciones) debe coincidir con Forma 204; alerta si no | Validación cruzada (warning) |
| RN-18 | Acumulado petróleo nunca menor al mes anterior; primer registro manual, luego automático | Validación |
| RN-19 | Sumatoria gas Forma 202 (todas formaciones) debe coincidir con Forma 205; alerta si no | Validación cruzada (warning) |
| RN-20 | BSW entre 0 y 100. Sin producción de agua = 0%; solo agua = 100% | Validación de rango |
| RN-21 | Gravedad API ≥ 0. Sin crudo = no se reporta. Solo agua = 10. Solo gas = no se reporta | Condicional |
| RN-22 | Estado del pozo al último día del período; pozo puede tener producción y terminar inactivo | Escenario válido |
| RN-23 | Homologar estados de pozo entre Forma 202 y AVM (pendiente PA-05) | Integración |
| RN-24 | Pozo tipo Monitor puede estar activo sin producción de fluidos | Escenario válido |
| RN-25 | Cuando AVM no esté disponible, permitir registro manual previa autorización ANH | Fallback |

### 4.3 Reglas de Flujo de Aprobación

| Regla | Detalle |
|---|---|
| Máquina de estados | `Initial` → `Registration` → `Submitted` → `Approved`. Devolución: `Submitted` → `Registration` → `Submitted`. |
| Plazo de envío | 7 días hábiles desde el 1.° del mes. Alerta el día 6 hábil (SMTP). Envío automático tras 7 días. |
| Plazo de corrección | 3 días hábiles tras devolución. Envío automático si se excede. |
| Límite de devolución | El supervisor solo puede devolver antes del día 19 del mes calendario. |
| Versionamiento | Cada envío incrementa la versión. Versión 0 = primer registro (puede incluir acumulados manuales). |
| Inmutabilidad post-aprobación | Una forma aprobada NO puede ser modificada. |
| No eliminación | La Forma 202 no puede ser eliminada (soft delete no aplica para esta entidad). |

### 4.4 Reglas de Cálculo de Fila TOTAL

| Campo | Fórmula |
|---|---|
| Días en Mes / Acumulados | Σ(filas) |
| Producción Diaria/Mensual/Acumulada (Petróleo, Agua, Gas) | Σ(filas) |
| Factores de Campo (Petróleo, Agua, Gas) | Promedio aritmético de filas con valor > 0 |
| BSW Total | `100 - ((ΣPetróleoMensual / (ΣPetróleoMensual + ΣAguaMensual)) × 100)`. Denominador = 0 → `null` |
| Gravedad API Total | `141.5 / (Σ(Vi × 141.5/(APIi + 131.5)) / ΣVi) - 131.5`. ΣVi = 0 → `null` |
| RGP Total | `(ΣGasMensual × 1000) / ΣPetróleoMensual`. Denominador = 0 → `null` |

---

## 5. Contratos de Entrada/Salida

Ver `api-contract.md` y `openapi.yaml` para la especificación completa de DTOs, payloads y respuestas.

**Particularidades backend:**
- Toda consulta lista filtra por el alcance de datos del usuario (`UserRole` + `User.OperatorId`). El filtro se inyecta automáticamente; nunca depende de parámetros del frontend.
- Los IDs son UUID v4 generados por el backend.
- Las fechas se almacenan en UTC (`datetime2(7)`); se retornan en ISO 8601.
- Los valores numéricos de producción se almacenan como `decimal(18,2)` (producción) y `decimal(18,5)` (factores de campo, BSW, API, RGP).

---

## 6. Modelo de Datos Impactado

### 6.1 Entidades Existentes Utilizadas (sin cambios)

| Entidad | Esquema | Uso |
|---|---|---|
| `Procedure` | `procedure` | Instancia del trámite de la Forma 202 (1:1 con Form202Data) |
| `ProcedureSignature` | `procedure` | Firmas electrónicas del operador y supervisor |
| `ProcedurePdf` | `procedure` | PDFs generados (post-envío y post-aprobación) |
| `ProcedureNotification` | `procedure` | Notificaciones SMTP de plazos e incumplimiento |
| `ProcedureObservation` | `procedure` | Observaciones del supervisor al devolver |
| `ProducingFormation` | `prod` | Formación productora del campo (ya tiene TrapType) |
| `ProductionModality` | `prod` | Modalidad de explotación vigente |
| `MonthlyProductionChange` | `prod` | Log de modificaciones sobre datos AVM |
| `WorkflowInstance` / `TaskInstance` | `workflow` | Motor de flujo genérico |
| `Field` | `core` | Campo petrolero |
| `Contract` | `core` | Contrato |
| `Operator` | `core` | Operadora |
| `Well` | `ops` | Pozo (referencia para nombre, UWI) |
| `Formation` | `catalog` | Formación geológica |
| `PoliticalDivision` | `catalog` | Municipio (código DANE) |
| `AuditLog` | `audit` | Auditoría inmutable |
| `IntegrationLog` | `audit` | Log de llamadas a AVM/SOLAR/ControlDoc |
| `BulkLoadLog` | `audit` | Log de cargues masivos |

### 6.2 Entidades Nuevas o Modificadas

#### `Form202Data` (NUEVA — esquema `prod`)

Payload específico de la Forma 202. Relación 1:1 con `Procedure`.

| Columna | Tipo Lógico | Tipo SQL | Nullable | Default | Constraints |
|---|---|---|---|---|---|
| `Form202DataId` | Identificador | `bigint IDENTITY` | No | — | PK |
| `ProcedureId` | FK → Procedure | `bigint` | No | — | FK, UNIQUE |
| `ProducingFormationId` | FK → ProducingFormation | `bigint` | No | — | FK |
| `ProductionModalityId` | FK → ProductionModality | `bigint` | No | — | FK |
| `ReportMonth` | Mes (1-12) | `tinyint` | No | — | CHECK (1-12) |
| `ReportYear` | Año | `smallint` | No | — | CHECK (≥ 2020) |
| `TrapType` | Enum textual | `nvarchar(20)` | No | — | CHECK (Structural/Stratigraphic/Mixed) |
| `Observations` | Texto libre | `nvarchar(2000)` | Sí | NULL | — |
| `Version` | Contador versión | `int` | No | 0 | — |
| `ReturnCount` | Devoluciones recibidas | `int` | No | 0 | — |
| `LastReturnReason` | Motivo última devolución | `nvarchar(2000)` | Sí | NULL | — |
| `CorrectionDeadlineAt` | Plazo de corrección | `datetime2(7)` | Sí | NULL | — |
| `CrossValOilStatus` | Estado validación petróleo | `nvarchar(20)` | Sí | NULL | CHECK (Passed/Warning/NotAvailable) |
| `CrossValOilForm202Total` | Total F202 petróleo | `decimal(18,2)` | Sí | NULL | — |
| `CrossValOilForm204Total` | Total F204 petróleo | `decimal(18,2)` | Sí | NULL | — |
| `CrossValGasStatus` | Estado validación gas | `nvarchar(20)` | Sí | NULL | — |
| `CrossValGasForm202Total` | Total F202 gas | `decimal(18,2)` | Sí | NULL | — |
| `CrossValGasForm205Total` | Total F205 gas | `decimal(18,2)` | Sí | NULL | — |
| `CrossValExecutedAt` | Fecha validación cruzada | `datetime2(7)` | Sí | NULL | — |
| `CreatedAt` | Timestamp creación | `datetime2(7)` | No | GETUTCDATE() | — |
| `CreatedBy` | FK → User | `bigint` | No | — | FK |
| `UpdatedAt` | Timestamp actualización | `datetime2(7)` | No | GETUTCDATE() | — |
| `UpdatedBy` | FK → User | `bigint` | No | — | FK |

**Claves e Índices:**
- PK: `Form202DataId`
- UK: `UQ_Form202Data_Procedure` (`ProcedureId`)
- UK: `UQ_Form202Data_NaturalKey` (`ProducingFormationId`, `ProductionModalityId`, `ReportMonth`, `ReportYear`)
- IX: `IX_Form202Data_ReportPeriod` (`ReportYear`, `ReportMonth`)

#### `MonthlyWellProduction` (MODIFICADA — esquema `prod`)

**Campo nuevo propuesto:**

| Columna | Tipo Lógico | Tipo SQL | Nullable | Justificación |
|---|---|---|---|---|
| `MunicipalityDaneCode` | Código DANE municipio | `char(5)` | No | RQF exige el código DANE por pozo en cada reporte. Viene de AVM o se ingresa manualmente para pozos históricos. |
| `IsModified` | Flag de modificación | `bit` | No, default 0 | Indica si el operador modificó algún valor precargado de AVM |

> **Nota:** Este cambio es **aditivo** (nueva columna NOT NULL con DEFAULT). No constituye un `[BREAKING CHANGE]` ya que no afecta features existentes. Las filas existentes (si las hubiera) recibirían un valor default.

---

## 7. Operaciones Transaccionales

| Operación | Atomicidad | Justificación |
|---|---|---|
| Creación de Forma 202 (POST `/`) | Atómica | Crea `Procedure` + `WorkflowInstance` + `Form202Data` en una transacción |
| Precarga desde AVM | Atómica por lote | Todas las filas de un pozo se insertan/actualizan en una transacción. Si una fila falla, se revierte todo el lote. |
| Cargue masivo Excel | Atómica por lote con rollback parcial | Filas válidas se insertan; filas inválidas se reportan. Se usa transacción por bloque (ej. 100 filas). Si un bloque falla completo, se reporta. |
| Actualización de fila (PUT) + log de cambio | Atómica | La fila y su `MonthlyProductionChange` se persisten en la misma transacción |
| Firma y envío (sign-and-submit) | Atómica | Transición de estado + creación de `ProcedureSignature` + encolado de generación PDF en la misma transacción. La generación de PDF y radicación ControlDoc son **eventual** (async). |
| Aprobación | Atómica | Transición de estado + firma + encolado PDF. ControlDoc es eventual. |
| Devolución | Atómica | Transición de estado + `ProcedureObservation` + notificación encolada |

**Generación de PDF y ControlDoc:** son operaciones asíncronas (eventual consistency). Si fallan, se reintentan con backoff. El estado de la forma ya cambió; el PDF se genera después. El `pdfGenerationStatus` trackea el estado de esta operación asíncrona.

---

## 8. Integraciones Externas

### 8.1 AVM (Administración Volumétrica)

| Aspecto | Detalle |
|---|---|
| Tipo | API REST (servicio externo) |
| Dirección | Entrada (GOP consume datos de AVM) |
| Datos consumidos | Pozos activos, producción diaria/mensual, factores de campo, BSW, API, RGP, municipio, días, método de producción |
| Trigger | Invocación explícita del operador (botón "Consultar") |
| Manejo de fallas | Si AVM no responde en 30s, retornar 502 al frontend. Registrar en `IntegrationLog`. Permitir registro manual (RN-25). |
| Reintentos | Hasta 3 reintentos con backoff exponencial (1s, 2s, 4s) antes de declarar falla |
| Contrato | Endpoint y formato pendiente de documentación por equipo AVM (PA-04) |

### 8.2 BD SOLAR

| Aspecto | Detalle |
|---|---|
| Tipo | Base de datos (lectura) |
| Dirección | Entrada |
| Datos consumidos | Operador, Contrato, Campo, Formaciones productoras |
| Uso | Precarga de encabezado y validación de combinaciones |
| Manejo de fallas | Datos replicados localmente (patrón de integración §11 del data-model). Si la réplica falla, se usa caché local. |

### 8.3 SSRS (SQL Server Reporting Services)

| Aspecto | Detalle |
|---|---|
| Tipo | Servicio de reportes |
| Dirección | Salida (GOP invoca SSRS) |
| Datos enviados | ID de la Forma 202, datos del encabezado y grilla |
| Resultado | PDF generado según formato Resolución 0545/2025 |
| Trigger | Post-envío (firma operador) y post-aprobación (firma supervisor) |
| Manejo de fallas | Reintentos con backoff. Si falla después de 3 intentos, marca `pdfGenerationStatus = Failed`. Job de reconciliación reintenta periódicamente. |

### 8.4 ControlDoc

| Aspecto | Detalle |
|---|---|
| Tipo | API de gestión documental |
| Dirección | Salida (GOP envía PDF a ControlDoc) |
| Datos enviados | PDF generado, metadatos de la forma |
| Resultado | Número de radicado |
| Trigger | Después de generar cada PDF (post-envío y post-aprobación) |
| Manejo de fallas | Reintentos con backoff. El radicado se asigna cuando ControlDoc responde exitosamente. |

### 8.5 SMTP

| Aspecto | Detalle |
|---|---|
| Tipo | Servicio de correo electrónico |
| Dirección | Salida |
| Notificaciones | Alerta de vencimiento próximo (día 6 hábil), comunicación de incumplimiento, notificación de devolución, notificación de aprobación |
| Manejo de fallas | Reintentos con backoff. Registrar en `ProcedureNotification` con `DeliveryStatus`. |

### 8.6 GOP Operaciones (Forma 103)

| Aspecto | Detalle |
|---|---|
| Tipo | Interna (mismo sistema GOP, esquema `ops`) |
| Dirección | Lectura |
| Datos consumidos | Nombre aprobado del pozo (RN-12) |
| Uso | Validación del nombre de pozo ingresado manualmente vs nombre aprobado en Forma 103 |

---

## 9. NFRs Backend

### 9.1 Performance

| Endpoint | Latencia p95 | Latencia p99 |
|---|---|---|
| GET `/` (listado) | < 300ms | < 500ms |
| GET `/{id}` (detalle) | < 200ms | < 400ms |
| GET `/{id}/well-productions` (50 filas) | < 300ms | < 500ms |
| GET `/{id}/well-productions` (500 filas) | < 800ms | < 1500ms |
| PUT `/{id}/well-productions/{wpId}` | < 200ms | < 400ms |
| POST `/{id}/preload` (40 pozos) | < 5s | < 10s |
| POST `/{id}/preload` (2000+ pozos) | < 30s | < 60s |
| POST `/{id}/well-productions/bulk-upload` (150 filas) | < 10s | < 20s |
| POST `/{id}/well-productions/bulk-upload` (2000+ filas) | < 60s | < 120s |
| POST `/{id}/actions/sign-and-submit` | < 500ms | < 1s |
| POST `/{id}/cross-validations` | < 2s | < 5s |
| GET `/{id}/totals` | < 500ms | < 1s |

### 9.2 Escalabilidad

| Métrica | Valor estimado |
|---|---|
| Operadores activos | ~200 |
| Formas 202 por mes | ~2,000 - 5,000 (depende de combinaciones campo/formación/modalidad) |
| Filas de producción por mes | ~50,000 - 200,000 |
| Crecimiento anual de filas | ~600,000 - 2,400,000 |
| Requests concurrentes pico (primer día del mes) | ~100-200 req/s |

### 9.3 Disponibilidad
- SLA objetivo: 99.5% durante horario laboral (L-V 7am-7pm COT)
- RTO: 4 horas
- RPO: 15 minutos (backups transaccionales)

### 9.4 Observabilidad

| Tipo | Detalle |
|---|---|
| Métricas | Latencia por endpoint (p50, p95, p99), requests/s, errores/s, tasa de éxito de precarga AVM, tasa de éxito de generación PDF, formas pendientes de envío (gauge) |
| Trazas | `traceId` (CorrelationId) propagado en todas las llamadas (HTTP headers, logs, IntegrationLog, AuditLog). Distributed tracing end-to-end: frontend → API → AVM/SSRS/ControlDoc. |
| Logs estructurados | Formato JSON. Campos obligatorios: `timestamp`, `level`, `message`, `traceId`, `userId`, `tenantId`, `endpoint`, `httpMethod`, `statusCode`, `durationMs`. **Nunca loguear:** tokens JWT, matrículas, nombres completos, contenido de payloads con PII, queries SQL con datos. |
| Alertas | Tasa de error > 5% en ventana de 5 min; latencia p95 > 2x del objetivo por más de 10 min; job de plazos no ejecutado en el día; falla de conexión AVM por más de 30 min. |

---

## 10. Seguridad Backend (OWASP Top 10)

### A01 — Broken Access Control
- **Autorización por endpoint y por recurso:** cada endpoint verifica el rol Y el alcance de datos del usuario. Un Operador GOP que intenta acceder a una forma de otro operador recibe 403 (no 404, para no filtrar existencia).
- **Filtrado de tenant en toda consulta:** la cláusula WHERE incluye siempre la restricción de `ContractId`/`OperatorId` derivada del `UserRole` del usuario autenticado. Nunca se confía en un `operatorId` enviado desde el frontend como filtro de seguridad.
- **Endpoints de acción verifican estado:** las acciones (firmar, aprobar, devolver) validan el estado actual de la forma además del rol. Ejemplo: un supervisor no puede aprobar una forma en estado `Registration`.

### A02 — Cryptographic Failures
- Datos en tránsito: TLS 1.2+ obligatorio.
- Datos en reposo: la columna `ProfessionalLicenseNumber` en `ProcedureSignature` se almacena como snapshot. Si la ANH clasifica este dato como nivel ≥ 3 en la matriz de clasificación, se aplica Always Encrypted.
- Hash del contenido del formulario al momento de la firma: se calcula un SHA-256 del estado de la forma al firmar y se almacena en `ProcedureSignature.DocumentHash` como evidencia de integridad.
- Secretos (connection strings, API keys AVM/ControlDoc): nunca en código fuente. Se obtienen de la configuración de secretos del entorno.

### A03 — Injection
- Todas las consultas a SQL Server son parametrizadas (via EF Core o parámetros explícitos en consultas raw). **Prohibido** concatenar strings para construir SQL.
- Los valores de filtros (`search`, `sort`, `wellStateFilter`) se validan contra listas blancas antes de usarse en queries dinámicas.
- El campo `Observations` y `returnReason` se almacenan como texto plano y se parametrizan. No se interpretan como HTML ni SQL.

### A04 — Insecure Design
- **Principio de menor privilegio:** el Operador GOP solo puede modificar formas en estado `Registration` de sus contratos. El Supervisor solo puede aprobar/devolver formas `Submitted` de sus campos asignados.
- **Defensa en profundidad:** la validación de estado se realiza tanto en la capa de aplicación como en la transición del workflow engine. Si el workflow no permite la transición, la operación falla independientemente de la capa de aplicación.
- **Threat modeling mínimo:**
  - Amenaza: Operador modifica acumulados históricos para inflar producción → Mitigación: log de cambios obligatorio (RN-10), validación de acumulados no decrecientes (RN-18), log consultable por Supervisor.
  - Amenaza: Operador borra pozos del listado para ocultar producción → Mitigación: RN-13 (pozo registrado persiste), DELETE restringido a pozos manuales pre-envío.
  - Amenaza: Suplantación de firma → Mitigación: firma vinculada al userId del JWT autenticado; hash del contenido al momento de la firma.

### A05 — Security Misconfiguration
- Headers de seguridad obligatorios en toda respuesta: `Strict-Transport-Security`, `Content-Security-Policy`, `X-Content-Type-Options: nosniff`, `X-Frame-Options: DENY`.
- Banners de servidor desactivados (no exponer versión de ASP.NET, IIS, Kestrel).
- CORS restrictivo: solo orígenes permitidos (dominio del frontend).
- No exponer swagger/openapi en producción sin autenticación.

### A06 — Vulnerable & Outdated Components
- Dependencias bloqueadas a versiones auditadas en `Directory.Packages.props`.
- Scan automático en CI con `dotnet list package --vulnerable` y Dependabot/Renovate.
- Actualización de dependencias críticas en ≤ 72 horas tras publicación de CVE.

### A07 — Identification & Authentication Failures
- JWT con expiración corta (conforme CONSTITUTION-ba.md §7.1).
- Refresh token con rotación obligatoria y detección de reuso (data-model.md §9.6).
- Rate limiting en endpoint de autenticación (`auth-strict` policy).
- El endpoint de firma (sign-and-submit, approve) verifica que el token no haya sido revocado (consulta a `RevokedAccessToken` para operaciones críticas).

### A08 — Software & Data Integrity Failures
- Hash SHA-256 del contenido de la forma al firmar → `ProcedureSignature.DocumentHash`. Verificable por auditoría.
- El cargue masivo valida formato del archivo Excel (estructura de columnas, tipos de datos) antes de procesar filas.
- Las transiciones de estado solo se ejecutan vía el workflow engine; no hay "atajo" directo en la BD.

### A09 — Security Logging & Monitoring Failures
- Todo evento de autenticación, autorización denegada, firma, aprobación, devolución, y modificación de datos se registra en `AuditLog` con `EventType` del catálogo oficial.
- El `traceId` se propaga end-to-end para correlacionar frontend → API → integraciones externas.
- **Nunca se loguean:** tokens JWT, matrículas profesionales, contraseñas, payloads con PII en claro.
- Los logs de seguridad son append-only (inmutables).

### A10 — SSRF
- Las integraciones salientes (AVM, ControlDoc, SSRS, SMTP) usan URLs configuradas como constantes de entorno. **No** se aceptan URLs dinámicas desde el request del usuario.
- Lista blanca de destinos para integraciones salientes.
- No hay funcionalidad que permita al usuario especificar una URL de destino.

---

## 11. Edge Cases

| # | Escenario | Comportamiento Esperado | HTTP |
|---|---|---|---|
| EC-BE-01 | Token JWT inválido o expirado | 401 Unauthorized. Sin información sobre la razón específica. | 401 |
| EC-BE-02 | Token válido pero usuario sin rol para Forma 202 | 403 Forbidden. | 403 |
| EC-BE-03 | Operador intenta acceder a forma de otro operador | 403 Forbidden (no 404 para no filtrar existencia). | 403 |
| EC-BE-04 | Operador intenta firmar forma que no está en `Registration` | 409 `INVALID_STATE`. | 409 |
| EC-BE-05 | Supervisor intenta devolver después del día 19 | 409 `RETURN_DEADLINE_EXCEEDED`. | 409 |
| EC-BE-06 | Creación de forma con combinación ya existente | 409 `DUPLICATE_FORM` con detalle del ID existente. | 409 |
| EC-BE-07 | Cargue masivo con pozo duplicado en el archivo | Upsert: si el pozo ya existe en la grilla, se actualiza. | 202 |
| EC-BE-08 | Cargue masivo con archivo no-Excel o corrupto | 400 `VALIDATION_ERROR` con mensaje: "Invalid file format." | 400 |
| EC-BE-09 | Cargue masivo excede 10 MB | 413 Payload Too Large. | 413 |
| EC-BE-10 | Acumulado propuesto menor al del mes anterior | 400 `CUMULATIVE_DECREASE` con el valor del mes anterior en `details`. | 400 |
| EC-BE-11 | Concurrencia: dos usuarios editan la misma fila | Optimistic concurrency via `ConcurrencyStamp`/`RowVersion`. El segundo PUT falla con 409 `CONCURRENCY_CONFLICT`. | 409 |
| EC-BE-12 | AVM no responde (timeout 30s) | 502 `AVM_CONNECTION_ERROR`. Se registra en `IntegrationLog`. | 502 |
| EC-BE-13 | AVM responde pero con datos parciales/malformados | Se procesan las filas válidas; se reportan warnings para las inválidas en la respuesta de preload. | 202 |
| EC-BE-14 | ControlDoc no responde al radicar PDF | PDF se genera y almacena localmente. Radicado queda `null`. Job de reconciliación reintenta. | (async) |
| EC-BE-15 | SSRS falla al generar PDF | `pdfGenerationStatus = Failed`. Job reintenta con backoff. Forma ya cambió de estado. | (async) |
| EC-BE-16 | Forma sin filas de producción intenta enviarse | 409 `EMPTY_GRID`. No se permite envío sin al menos una fila. | 409 |
| EC-BE-17 | Payload JSON excesivamente grande (>5 MB body) | 413 Payload Too Large (configuración del middleware). | 413 |
| EC-BE-18 | Rate limit excedido | 429 `RATE_LIMIT_EXCEEDED` con header `Retry-After`. | 429 |
| EC-BE-19 | Operador intenta eliminar un pozo precargado de AVM | 422 `CANNOT_DELETE_AVM_WELL`. | 422 |
| EC-BE-20 | Operador intenta eliminar un pozo con historial en meses anteriores | 422 `CANNOT_DELETE_HISTORICAL_WELL`. | 422 |
| EC-BE-21 | BSW enviado fuera de rango (ej. 105.5) | 400 `VALIDATION_ERROR` con detalle del campo y rango permitido. | 400 |
| EC-BE-22 | Job de plazos no encuentra días hábiles configurados | El job falla con alerta crítica. Se debe configurar el calendario de días hábiles antes del despliegue. | (job) |

---

## 12. Criterios de Testing y Cobertura

### 12.1 Niveles Requeridos

| Nivel | Alcance | Herramienta |
|---|---|---|
| Unitarias | Lógica de dominio (cálculos de totales, validación de acumulados, reglas de negocio), validadores FluentValidation | xUnit |
| Integración (BD real) | Repositorios, queries, transacciones atómicas, concurrencia optimista | xUnit + SQL Server contenedor |
| Contract tests | Validación de endpoints contra `openapi.yaml` | Testing HTTP contra el spec OpenAPI |
| E2E | Flujos completos con integraciones mockeadas | xUnit + WebApplicationFactory |
| BD | Procedures, triggers, constraints | tSQLt |

### 12.2 Cobertura Mínima
- **85%** en lógica de dominio (cálculos, validaciones, reglas de negocio)
- **75%** en capa de aplicación (handlers, servicios)
- **70%** en capa de infraestructura (repositorios, integraciones)

### 12.3 Escenarios de Seguridad con Test Obligatorio

| # | Escenario | Verificación |
|---|---|---|
| 1 | Operador GOP intenta acceder a forma de otro operador | Retorna 403 (no 404) |
| 2 | Operador intenta firmar forma en estado `Submitted` | Retorna 409 |
| 3 | Supervisor intenta devolver después del día 19 | Retorna 409 |
| 4 | Request con token expirado | Retorna 401 |
| 5 | Request sin token | Retorna 401 |
| 6 | Inyección SQL en campo `search` | Parametrizado; no afecta la query |
| 7 | Payload con campo `bswPercent: 999` | Retorna 400 con detalle de validación |
| 8 | Cargue masivo con archivo .exe renombrado a .xlsx | Retorna 400 (validación de contenido) |
| 9 | Doble submit de firma (idempotencia) | Segundo request retorna 200 con estado actual |
| 10 | Concurrencia en PUT de fila de producción | Segundo request retorna 409 `CONCURRENCY_CONFLICT` |

---

## 13. Anexos

- **API Contract:** `api-contract.md` (en este directorio)
- **OpenAPI Spec:** `openapi.yaml` (en este directorio)
- **Spec Frontend:** `spec-frontend.md` (en este directorio)
- **Modelo de Datos:** `data-model.md` (raíz del repositorio)
- **Fuente funcional:** RQF_GOP_13_Forma202 V1 + BP_GOP_13_Forma202 V1

---

## 14. Puntos de Decisión Pendientes

| # | Punto | Impacto | Responsable |
|---|---|---|---|
| PD-BE-01 | **Contrato de API de AVM:** el endpoint, formato de request/response y autenticación de AVM no están documentados. Se requiere documentación técnica del equipo AVM para implementar la integración. | Integración de precarga | Equipo AVM + RENATA |
| PD-BE-02 | **Calendario de días hábiles:** el cálculo de plazos normativos (7 días hábiles, 3 días hábiles) requiere un calendario de festivos colombianos. Se debe definir si es un catálogo administrable en GOP o una integración con un servicio externo. | Jobs de plazos | Equipo RENATA |
| PD-BE-03 | **Formato de reporte SSRS:** la plantilla del PDF según Resolución 0545/2025 debe ser diseñada y configurada en SSRS. Requiere coordinación con equipo funcional ANH para el diseño visual. | Generación PDF | ANH + RENATA |
| PD-BE-04 | **Interoperabilidad bidireccional GOP ↔ AVM (PA-04 del RQF):** si los datos cargados manualmente en GOP para pozos históricos deben alimentar retroactivamente a AVM, se requiere definir el mecanismo y la llave de sincronización. En este spec se asume dirección unidireccional (AVM → GOP) para V1. | Integración | ANH + Equipo AVM |
| PD-BE-05 | **Homologación de estados de pozo (PA-05 del RQF):** los 8 estados de pozo de la Forma 202 deben homologarse con los catálogos de AVM. Si no convergen, se requiere una tabla de mapeo en el esquema `integration`. | Catálogos | ANH (Producción) + AVM |
| PD-BE-06 | **Módulos de Pruebas Iniciales/Extensas/Producción Temprana (PA-06 del RQF):** la validación de concordancia de modalidad con Forma 204 depende de módulos no contemplados en este RQF. Para V1 se omite esta validación. | Validación de modalidad | ANH + RENATA |
| PD-BE-07 | **[BREAKING CHANGE] propuesto en data-model.md:** adición de columnas `MunicipalityDaneCode` y `IsModified` a `MonthlyWellProduction`. Cambio aditivo (NOT NULL con DEFAULT), no rompe compatibilidad, pero requiere migración de esquema. | Modelo de datos | Arquitecto de datos |
