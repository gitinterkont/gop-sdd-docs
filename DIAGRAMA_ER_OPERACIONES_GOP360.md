# Diagrama ER — Operaciones GOP360 (entidades y esquemas)

**Versión:** 0.3 (iterable) — `erDiagram` + ELK (aristas ortogonales 90°)
**Fecha:** 2026-07-01

**Alcance:** Propuesta Inicial de modelamiento de entidades de datos (sin atributos ni columnas), esta propuesta puede cambiar y es iterable conforme se identifiquen cambios en el levantamiento y aprobacion de los requerimientos funcionales por parte de ANH los cuales se encuentran en proceso de validacion e iteración.


> **Notación:** Mermaid `erDiagram` con **layout ELK** (`config: layout: elk`). Diagramas por dominio para legibilidad.
>
> ⚠️ **Requiere Mermaid v11+ con ELK** (GitHub y mermaid.live lo soportan; en VS Code exige la extensión con motor ELK). Si tu preview no lo soporta, el bloque puede no renderizar — en ese caso quita el frontmatter `config/layout: elk` o usa la v0.2 (flowchart recto).

---

## 1. Mapa de esquemas (alto nivel)

```mermaid
---
config:
  layout: elk
---
flowchart TB
  subgraph auth["🔐 auth — identidad/gobernanza"]
    AU["AspNetUsers/Roles/Claims · Permissions · UserProfiles · UserContracts · Tokens"]
  end
  subgraph catalog["📚 catalog — referencia"]
    CAT["DANE · lookups PPDM · catálogos documentales/yacimiento/tapón"]
  end
  subgraph core["🧱 core — maestros + pozo + estados"]
    CUENTA["Tenant · Operator · TenantOperator"]
    MAESTRO["Contract · Field · ContractField"]
    POZO["Wells + satélites · ClusterLocations"]
    ESTADO["StateMachine* · WellLifecycleHistorico · well_attribute_change"]
    DOCS["Documento · AnexoConfig"]
  end
  subgraph operations["⚙️ operations — Serie 100 + informes"]
    FORMAS["Forma101..112 + satélites"]
    INFORMES["IDOP · IDOC"]
    OTROS["Pruebas · WellService · Suspensión"]
  end
  subgraph production["🛢️ production — Serie 200"]
    PROD["Forma204 + satélites · PeriodoReporte · FacilidadProduccion"]
  end
  subgraph workflow["🔄 workflow — flujo (futuro)"]
    WF["WorkflowDefinition/Instance · TaskInstance · WorkflowStatus · TaskActionLog · FormSlaPolicy"]
  end
  subgraph integrations["🔌 integrations"]
    INT["ControlDocOutboxItem · LogIntegracion"]
  end
  subgraph audit["📝 audit"]
    AUD["OperationsEntityAudit"]
  end

  AU -->|"TenantId"| CUENTA
  CAT -.->|"lookups"| POZO
  CUENTA --> MAESTRO
  MAESTRO -->|"atribución"| POZO
  POZO --> FORMAS
  POZO --> INFORMES
  POZO --> OTROS
  FORMAS -.->|"polimórfico"| DOCS
  FORMAS -.->|"polimórfico"| WF
  FORMAS -->|"radicado"| INT
  ESTADO -->|"WellLifecycle"| POZO
```

---

## 2. Cuenta y datos maestros (`core`)

```mermaid
---
config:
  layout: elk
---
erDiagram
  Tenant   ||--o{ TenantOperator : asocia
  Operator ||--o{ TenantOperator : asocia
  Operator ||--o{ Contract       : posee
  Contract ||--o{ ContractField  : bridge
  Field    ||--o{ ContractField  : bridge
  Operator ||--o{ Wells          : opera
  Contract ||--o{ Wells          : cuelga
  Field    ||--o{ Wells          : pertenece
  Tenant   ||--o{ Wells          : tenancy
  Tenant   ||--o{ UserProfiles   : cuenta
```

---

## 3. Pozo y máquina de estados (`core`)

```mermaid
---
config:
  layout: elk
---
erDiagram
  ClusterLocations ||--o{ Wells : ubica
  Wells ||--o{ WellLocalizations        : tiene
  Wells ||--o{ WellPhysicalData         : tiene
  Wells ||--o{ WellFormations           : tiene
  Wells ||--o{ WellCasings              : tiene
  Wells ||--o{ WellTerminationIntervals : tiene
  Wells ||--o{ WellFreshWaterSands      : tiene
  Wells ||--o{ WellLogs                 : tiene
  Wells ||--o{ WellHistoricalContracts  : tiene
  Wells ||--o{ WellApprovals            : tiene
  Wells ||--o{ WellProductionTests      : prueba
  Wells ||--o{ WellInjectionTests       : prueba
  Wells ||--o{ WellMonitoringTests      : prueba
  Wells ||--o{ WellLifecycleHistorico   : historial
  Wells ||--o{ well_attribute_change    : cambios
  StateMachineState ||--o{ StateMachineTransition : define
  StateMachineState ||--o{ StateMachineHistory    : registra
```

> `StateMachineHistory`/`Transition` operan por `DomainName` (`WellLifecycle`, `Forma101`, `Forma204`) + `EntityId` — vínculo **polimórfico** (§9). `WellLifecycle` gobierna el estado del pozo; los dominios de forma son pre-workflow.

---

## 4. Apertura — F101, F102, F103 e informes diarios (`operations`)

```mermaid
---
config:
  layout: elk
---
erDiagram
  Wells ||--o{ Forma101 : ""
  Forma101 ||--o{ Forma101Formaciones        : tiene
  Forma101 ||--o{ Forma101Laterals           : tiene
  Forma101 ||--o{ Forma101Revestimientos     : tiene
  Forma101 ||--o{ Forma101PermisosEspeciales : tiene

  Wells ||--o{ Idop : ""
  Idop ||--o{ IdopFaseBroca           : tiene
  Idop ||--o{ IdopFormacionEncontrada : tiene
  Idop ||--o{ IdopInflujoPerdida      : tiene
  Idop ||--o{ IdopRevestimiento       : tiene
  Idop ||--o{ IdopRegistroElectrico   : tiene
  Idop ||--o{ IdopEventoHSE           : tiene

  Wells ||--o{ Idoc : ""
  Idoc ||--o{ IdocRegistroCorrido : tiene
  Idoc ||--o{ IdocIntervalo       : cañoneos
  Idoc ||--o{ IdocEstimulacion    : tiene

  Wells ||--o{ Forma102 : ""
  Forma102 ||--o{ Forma102Broca               : tiene
  Forma102 ||--o{ Forma102FormacionEncontrada : tiene
  Forma102 ||--o{ Forma102PropiedadLodo       : tiene
  Forma102 ||--o{ Forma102RegistroElectrico   : tiene
  Forma102 ||--o{ Forma102Revestimiento       : tiene
  Forma102 ||--o{ Forma102Cementacion         : tiene
  Forma102 ||--o{ Forma102PruebaIntegridad    : tiene

  Wells ||--o{ Forma103 : ""
  Forma103 ||--o{ Forma103FormacionRegistro : tiene
  Forma103 ||--o{ Forma103ArenaAguaDulce    : tiene
  Forma103 ||--o{ Forma103Muestra           : tiene
  Forma103 ||--o{ Forma103Intervalo         : INICIAL
  Forma103 ||--o{ WellProductionTests       : prueba
  Forma103 ||--o{ WellInjectionTests        : prueba
```

> IDOP→F102 e IDOC→F103 son **consolidación** (el diario es la fuente; DA-6).

---

## 5. Técnicas y pruebas — F104, Pruebas Iniciales (`operations` / `core`)

```mermaid
---
config:
  layout: elk
---
erDiagram
  Wells ||--o{ Forma104 : ""
  Forma104 ||--o{ Forma104Instrumento        : tiene
  Forma104 ||--o{ Forma104ParametroEntrada   : tiene
  Forma104 ||--o{ Forma104ParametroCalculado : tiene
  Forma104 ||--o{ Forma104Intervalo          : PRUEBA
  Wells ||--o{ PruebaInicialCronograma : ""
  PruebaInicialCronograma ||--o{ PruebaInicialModificacion : modifica
  Wells ||--o{ PruebaFormacion     : ""
  Wells ||--o{ WellProductionTests : contrato
  Wells ||--o{ WellInjectionTests  : contrato
  Wells ||--o{ WellMonitoringTests : contrato
```

---

## 6. Post-terminación — F105, F107, F108, Well Service (`operations`)

```mermaid
---
config:
  layout: elk
---
erDiagram
  Wells ||--o{ Forma105 : ""
  Forma105 ||--o{ Forma105Actualizacion : tiene
  Forma105 ||--o{ Forma105Intervalo     : "INI+ACT"
  Forma105 ||--o| Forma108              : cierra

  Wells ||--o{ Forma107 : ""
  Forma107 ||--o{ Forma107CondicionMonitoreo : tiene
  Forma107 ||--o{ Forma107Intervalo          : "INI+ACT"

  Wells ||--o{ Forma108 : ""
  Forma108 ||--o{ Forma108ResultadoTrabajo     : tiene
  Forma108 ||--o{ Forma108TipoTrabajoRealizado : tiene

  Wells ||--o{ AvisoWellService : ""
  AvisoWellService ||--o| InformeResultadosWellService : cierra
```

---

## 7. Abandono — F106, F109, F112 (`operations`)

```mermaid
---
config:
  layout: elk
---
erDiagram
  Wells ||--o{ Forma106 : ""
  Forma106 ||--o{ Forma106CementacionRemedial : tiene
  Forma106 ||--o{ Forma106RetiroTuberia       : tiene
  Forma106 ||--o{ Forma106Monumento           : tiene
  Forma106 ||--o{ Forma106CambioPrograma      : tiene
  Forma106 ||--o{ Forma106AmpliacionPlazo     : tiene
  Forma106 ||--o{ Forma106Intervalo           : "INI+ACT"
  Forma106 ||--o{ TaponCemento106             : tiene
  Forma106 ||--o{ FluidoEntreTapones106       : tiene
  Forma106 ||--o{ Forma109 : habilita
  Forma106 ||--o{ Forma112 : habilita

  Wells ||--o{ Forma109 : ""
  Forma109 ||--o{ Forma109PlacaAbandono : tiene
  Forma109 ||--o{ TaponCemento109       : tiene
  Forma109 ||--o{ TaponMecanico109      : tiene
  Forma109 ||--o{ FluidoEntreTapones109 : tiene

  Wells ||--o{ Forma112 : ""
  Forma112 ||--o{ Forma112PeriodoAbandonoTemporal : tiene
  Forma112 ||--o{ Forma112ProrrogaAT              : tiene
  Forma112 ||--o{ Forma112InformeSeguimientoAT    : tiene
  Forma112 ||--o{ TaponCemento112       : tiene
  Forma112 ||--o{ TaponMecanico112      : tiene
  Forma112 ||--o{ FluidoEntreTapones112 : tiene
```

> Los `Tapon*`/`FluidoEntreTapones*` comparten **contrato de columnas** (§8.2) pero son **tablas por-forma** (P-8); el intervalo va **inline** en `TaponCemento*`.

---

## 8. Inyección y Suspensión (`operations`)

```mermaid
---
config:
  layout: elk
---
erDiagram
  Wells ||--o{ Forma110 : ""
  Forma110 ||--o{ Forma110FluidoLiquido     : tiene
  Forma110 ||--o{ Forma110ProyectoInyeccion : tiene
  Forma110 ||--o{ ParametroYacimiento110    : tiene
  Forma110 ||--o{ PozoInyector110           : inline

  Wells ||--o{ Forma111 : ""
  Forma111 ||--o{ Forma111FluidoGaseoso     : tiene
  Forma111 ||--o{ Forma111ProyectoInyeccion : tiene
  Forma111 ||--o{ ParametroYacimiento111    : tiene
  Forma111 ||--o{ PozoInyector111           : inline

  Wells ||--o{ SuspensionTemporal : ""
  SuspensionTemporal ||--o{ SuspensionBarreraIntegridad    : tiene
  SuspensionTemporal ||--o{ SuspensionUmbralPresion        : tiene
  SuspensionTemporal ||--o{ SuspensionInformeSeguimiento   : tiene
  SuspensionTemporal ||--o{ CronogramaReactivacionAbandono : tiene
  Wells ||--o{ ReinicioPeriodoInactividad : ""
```

---

## 9. Transversales y vínculos POLIMÓRFICOS

Se relacionan con **cualquier forma/evento** por par polimórfico (no FK). Hub central (flowchart + ELK, aristas punteadas = no-FK).

```mermaid
---
config:
  layout: elk
---
flowchart LR
  FORMA["Cualquier evento operacional (FormaTipo, FormaId)"]

  FORMA -.->|"(FormaTipo,FormaId)"| Documento
  Documento --> AnexoConfig
  FORMA -.->|"firma"| FormaNNNFirmas
  FORMA -.->|"(business_type_code, business_key)"| WorkflowInstance
  WorkflowDefinition --> WorkflowInstance
  WorkflowInstance --> TaskInstance
  WorkflowInstance --> TaskActionLog
  WorkflowStatus --> WorkflowInstance
  FormSlaPolicy -.->|"por business_type_code"| WorkflowInstance
  FORMA -.->|"(DomainName,EntityId)"| StateMachineHistory
  FORMA -.->|"radicado"| ControlDocOutboxItem
  FORMA -.->|"traza técnica"| OperationsEntityAudit
```

> - `Documento` = **única tabla polimórfica** de datos de negocio (PD-2); `AnexoConfig` se referencia por FK tipada.
> - El **flujo de la forma** vive en `workflow` (futuro); vínculo `(business_type_code, business_key)`.
> - `StateMachineHistory` (dominios de forma) es **pre-workflow** → migrará al Track B.

---

## 10. Producción (Serie 200 — contexto, `production`)

```mermaid
---
config:
  layout: elk
---
erDiagram
  PeriodoReporte      ||--o{ Forma204 : periodo
  FacilidadProduccion ||--o{ Forma204 : facilidad
  Forma204 ||--o{ Forma204BalanceFluidos       : tiene
  Forma204 ||--o{ Forma204CalidadCrudo         : tiene
  Forma204 ||--o{ Forma204DistribucionMunicipio: tiene
  Forma204 ||--o{ Forma204Entrega              : tiene
  Forma204 ||--o{ Forma204Perdida              : tiene
  Forma204 ||--o{ Forma204VolumenMuerto        : tiene
  Forma204 ||--o{ Forma204HistoricoEstado      : tiene
  Forma204 ||--o{ Forma204AlertaCambioAvm      : tiene
  Forma204 ||--o{ Forma204AuditoriaCrossTenant : tiene
  Forma204 ||--o| PermisosExtemporaneos        : extemporánea
  Forma204 ||--o{ Notificaciones               : notifica
```

---

*v0.3 — Vista ER a nivel de entidades/esquemas de todo el modelo de Operaciones. Modelo Iterable. La definicion tecnica de las columnas se desarrollaroan conforme al desarrollo de cada especificación una vez aprobada*
