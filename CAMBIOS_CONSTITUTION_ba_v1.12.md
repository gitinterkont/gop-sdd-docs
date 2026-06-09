# Cambios a CONSTITUTION-ba.md — v1.11 → v1.12
## Documento explicativo

**Documento modificado:** `CONSTITUTION-ba.md` (Constitución Arquitectónica del Backend GOP 360°)
**Versión:** 1.11 → **1.12** (Minor — aditivo, compatible hacia atrás)
**Fecha:** 2026-06-09
**Motivo:** Revisión de la constitución contra mejores prácticas de industria (.NET / Clean Architecture / DDD / microservicios) y alineación con la realidad del código (monolito modular).

---

## 1. Resumen ejecutivo

La constitución v1.11 es de **alta calidad** y mayoritariamente alineada con mejores prácticas (ProblemDetails, sin envelope, versionado, Result Pattern, CQRS, OWASP, NetArchTest, Token Exchange). La revisión encontró **3 puntos** donde la norma se apartaba de la práctica establecida o entraba en tensión con el estado real del backend:

| # | Problema en v1.11 | Cambio en v1.12 |
|---|---|---|
| **B1** | Microservicios-first obligatorio en V1, sin event bus, HTTP síncrono → riesgo de **monolito distribuido** | §6.1/§6.2/§6.3.1/§17.1 — reconocer **monolito modular** como topología válida de V1 |
| **B2** | Repository + UoW sobre EF Core sin acotar → riesgo de abstracción redundante | §11.3 — **MUST repos por agregado**, **PROHIBIDO `IRepository<T>` genérico** |
| **B3** | (Código real) `MigrateAsync()` en arranque → race conditions multi-instancia | §12.3 — **migrar en deploy, no en startup** en producción |

Todos los cambios son **aditivos**: los microservicios siguen siendo topología válida y objetivo. B1 y B2 no generan infracciones en código existente; **B3 sí requiere acción** (`MigrateAsync()` en startup de producción infringe §12.3 nuevo — ticket aparte). → versión **Minor** (clasificación por impacto arquitectónico, no por ausencia de trabajo pendiente).

---

## 2. Cambios en detalle

### 2.1 B1 — Monolito modular como topología de V1

**Secciones tocadas:** §6.1, §6.2, §6.3.1 (nueva), §17.1

**Antes (v1.11):**
> "Los bounded contexts se organizan como **microservicios independientes** […] El sistema V1 se despliega como los siguientes servicios independientes, cada uno con su propio repositorio y ciclo de release."

**Después (v1.12):**
- §6.1 reconoce el **monolito modular** como topología válida de V1: fronteras estrictas de bounded context, pero un solo proceso/repo, cada módulo = microservicio futuro. La descomposición a microservicios es **objetivo**, no prerrequisito.
- §6.2 reformula la tabla como **bounded contexts objetivo**; en monolito modular cada fila es un módulo.
- §6.3.1 (nueva) define la estructura `/src/Modules/{Module}/` con las mismas capas + tests por módulo + `Host` como composition root, y las reglas de aislamiento entre módulos (DbContext/esquema propio, sin acceso cruzado).
- §17.1 establece comunicación **in-process** (vía `Shared.Contracts`/puertos) como opción preferida en V1.

**Por qué (mejor práctica):**
- Descomponer en microservicios antes de tener señal real de escala/equipo/dominio produce un **monolito distribuido**: llegan los costos (latencia de red, fallos en cascada, transacciones distribuidas, complejidad operativa) sin los beneficios.
- Referencias: ***MonolithFirst*** (Martin Fowler), **"Don't start with microservices"** (Sam Newman). La práctica establecida recomienda monolito bien modularizado primero, extraer servicios cuando un límite lo justifique.
- v1.11 admitía además "V1 sin event bus, HTTP síncrono" (§17.1) — exactamente el patrón que materializa el riesgo. v1.12 lo evita con comunicación in-process.

**Impacto:**
- El backend actual (monolito modular `gop-backend`) deja de estar en divergencia con la norma — pasa a ser una topología **explícitamente contemplada**.
- Requiere **ADR-000** documentando la elección (la norma lo exige).
- La extracción futura a microservicios queda garantizada como **mecánica** por las fronteras de §6.3.1.

---

### 2.2 B2 — Repositorios por agregado, no genéricos

**Sección tocada:** §11.3

**Antes (v1.11):**
> "**MUST:** cada agregado tiene su repositorio en `Infrastructure`."  *(sin prohibir el genérico)*

**Después (v1.12):**
- **MUST** repos **por agregado** (`IWellRepository`, `IForma101Repository`) con métodos de dominio.
- **PROHIBIDO** repositorio genérico `IRepository<T>` / `Repository<T>` con CRUD indiscriminado.
- Rationale embebido: `DbContext` de EF Core **ya es** Repository + Unit of Work; una capa genérica encima es redundante.

**Por qué (mejor práctica):**
- Crítica conocida (guía de Microsoft, Jimmy Bogard): el repositorio genérico sobre EF Core duplica lo que el `DbContext` ya ofrece y oculta capacidades del ORM.
- El repositorio **por agregado** sí aporta valor: protege la frontera del agregado (DDD), encapsula queries de dominio y desacopla `Application` de EF.

**Impacto:**
- Aclaración de intención; no invalida el patrón, lo acota. Verificar que el código no introduzca un `IRepository<T>` genérico.

---

### 2.3 B3 — Migraciones en deploy, no en arranque

**Sección tocada:** §12.3

**Antes (v1.11):** definía el orden EF → DbUp en deploy, pero no prohibía la auto-migración al arranque. El código actual ejecuta `MigrateAsync()` en `Program.cs` al iniciar.

**Después (v1.12):**
- **MUST:** migrar en un **paso explícito del pipeline de deploy**, no en el arranque de la app.
- **PROHIBIDO:** `Database.Migrate()` / `MigrateAsync()` en arranque en ambientes productivos (race conditions multi-instancia, sin gate de revisión). En `Development` **MAY** usarse protegido por entorno.

**Por qué (mejor práctica):**
- Auto-migrar en startup mezcla el ciclo de vida del esquema con el del proceso. En despliegues horizontales, varias instancias migran a la vez → corrupción/locks.
- Separar la migración en un paso de deploy da control transaccional, revisión y rollback explícitos.

**Impacto:**
- El backend debe mover `MigrateAsync()` del startup a un job de deploy (`dotnet ef database update` o migrador). Ticket aparte; protegido por entorno en local.

---

## 3. Trazabilidad de ediciones

| Edición | Sección | Tipo |
|---|---|---|
| E1 | Header (versión 1.11→1.12, fecha) | Metadato |
| E2 | §6.1 Patrón arquitectónico | Adición (topología + rationale) |
| E3 | §6.2 intro | Reformulación |
| E4 | §6.3.1 Equivalencia en monolito modular | Nueva subsección |
| E5 | §11.3 Repository Pattern | Adición (MUST/PROHIBIDO + rationale) |
| E6 | §12.3 Migraciones | Adición (MUST/PROHIBIDO + rationale) |
| E7 | §17.1 Principios | Adición (comunicación in-process) |
| E8 | Control de cambios v1.11→v1.12 | Entrada de changelog |

---

## 4. Gobernanza (conforme a §0.1 y §2.1 de la constitución)

- **Tipo de cambio:** Minor (aditivo, compatible hacia atrás). Los microservicios siguen siendo válidos y objetivo.
- **ADR requerido:** sí — §0.1 regla 1 (modificar reglas del documento) y la propia §6.1 exigen ADR para adoptar la topología de monolito modular. **ADR-000** (en el plan de refactor) cubre esta decisión.
- **Aprobación:** estos cambios **MUST** ser aprobados por el arquitecto de software y el líder técnico de backend antes de mergear (§2.1, cambio mayor de patrón estructural aunque el versionado sea Minor).
- **Pendiente cliente:** la aprobación OTI / Dirección de Fiscalización ANH del documento sigue pendiente como en v1.11.

---

## 5. Relación con el refactor del backend

Estos cambios **cierran la brecha C1/C5** del documento `PROPUESTA_REFACTOR_BACKEND_CLEAN_ARCHITECTURE.md`:
- **C1** (monolito vs microservicios) — la norma ahora contempla la topología real. ADR-000 pasa de "desviación" a "aplicación de best-practice respaldada por la norma".
- **B3** se alinea con la épica de migraciones (mover `MigrateAsync` a deploy).
- Las divergencias **C2/C3/C4** (tests por módulo, shared libs, arch tests por módulo) ya estaban respaldadas por §6.3/§15.6 y ahora por §6.3.1.

---

*Documento explicativo. Las ediciones a `CONSTITUTION-ba.md` requieren aprobación de arquitectura antes de considerarse vigentes (§2.1).*
