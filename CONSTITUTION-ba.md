# CONSTITUTION-ba.md
## Constitución Arquitectónica y Técnica del Backend — GOP 360°

**Versión:** 1.10
**Fecha:** Abril 2026
**Estado:** Vigente para V1
**Autor responsable:** Arquitectura de Software — Modernización GOP
**Aprobación:** Pendiente — OTI / Dirección de Fiscalización ANH
**Ubicación:** `/backend/docs/CONSTITUTION-ba.md` (dentro del repositorio)

> **Naturaleza de este documento.** Este es un documento **normativo de principios, patrones y líneas rojas**. Los bloques de código, DTOs, entidades, contratos de API y estructuras de datos que aparecen son **ejemplos de referencia** para ilustrar los patrones descritos — no son la especificación vinculante del sistema. Los objetos de datos concretos, servicios, contratos OpenAPI, validadores y DTOs **MUST** definirse en la **spec de cada requerimiento** (historia de usuario, caso de uso o feature) que se desarrolle, y adherirse a los patrones de esta constitución. Si una spec necesita apartarse de un patrón, debe registrar un ADR.

---

## 0. Convenciones de este documento

Este documento usa las palabras clave **MUST**, **SHOULD** y **MAY** conforme a RFC 2119, traducidas así:

- **MUST (obligatorio):** la regla es vinculante. Un PR que no cumpla se bloquea en revisión.
- **SHOULD (recomendado):** la regla se cumple salvo justificación técnica razonable documentada en el PR (descripción o comentario en el código con el motivo). No exige ADR salvo que así lo indique el §específico.
- **MAY (permitido):** la regla describe una práctica válida pero no prescriptiva; el equipo decide caso por caso.

Los términos **PROHIBIDO** o **NUNCA** indican líneas rojas absolutas: ningún caso justifica transgredirlas sin modificar primero este documento.

### 0.1 Política unificada de ADR

Para evitar fricción innecesaria, un **ADR (Architecture Decision Record)** en `/backend/docs/adr/NNNN-titulo-corto.md` es **obligatorio** únicamente en los siguientes casos:

1. Modificar, añadir o eliminar una regla de este documento.
2. Transgredir una línea roja de **§18** (si se considera estrictamente inevitable, el ADR debe preceder al PR y contar con aprobación del arquitecto).
3. Cambiar un componente del **stack tecnológico** (§4) o introducir una dependencia NuGet fuera del stack aprobado.
4. Desactivar o relajar una regla de **§12** (seguridad / OWASP).
5. Introducir un patrón transversal nuevo que afecte a múltiples microservicios (ej. event bus, cache distribuido, nuevo middleware global).

Para el resto de desviaciones (excepciones puntuales a un **SHOULD**, umbrales numéricos configurables, decisiones tácticas de implementación) basta con **justificación en el PR**: descripción, comentario en el código donde aplique y, si la decisión se repetirá, apunte en la guía interna del servicio. No se requiere ADR.

> **Rationale.** Un documento con ADR obligatorio por cada desviación trivial vacía el significado de SHOULD y convierte al equipo en tramitador. Los ADR deben reservarse para decisiones que un nuevo miembro del equipo necesitaría conocer meses después.

---

## 1. Propósito y alcance

### 1.1 Propósito

Este documento define los principios, patrones, reglas y líneas rojas arquitectónicas del backend del sistema GOP 360°. Su propósito es garantizar consistencia técnica a lo largo del desarrollo, evitar decisiones ad-hoc que degraden el sistema, y reducir la carga cognitiva de los desarrolladores que se incorporan al proyecto.

### 1.2 Alcance

Aplica a todo código backend del proyecto GOP 360°: APIs REST, servicios de aplicación, acceso a datos, integraciones con sistemas externos, y componentes auxiliares (jobs, workers, tools). No aplica al frontend (ver `CONSTITUTION-fe.md`) ni al modelo de datos (ver `data-model.md`).

### 1.3 Relación con otros documentos

Este documento **MUST** leerse junto con:

- *Propuesta de Modelo de Datos GOP 360°* — define el esquema de datos y las entidades del dominio.
- *Documento de Estándares de Codificación ANH* — define nomenclatura, herramientas, y políticas de calidad de código.
- *Lineamientos OTI — Guía de Desarrollo Seguro de Software ANH-GTIC-GU-18* — define requisitos de seguridad.
- *Manual para la Gestión del Ciclo de Vida de Herramientas Informáticas ANH-GTIC-MA-02*.
- *CONSTITUTION-fe.md* — complemento frontend.

En caso de conflicto entre este documento y los anteriores, los documentos de ANH y OTI **MUST** prevalecer.

---

## 2. Gobernanza del documento

### 2.1 Ciclo de actualización

Este documento es un artefacto vivo. Los cambios se proponen vía Pull Request al repositorio donde vive.

- **Cambio mayor** (nueva sección, cambio de un patrón obligatorio, nueva línea roja) **MUST** ser aprobado por el arquitecto de software del proyecto y el líder técnico de backend, y **MUST** registrarse como ADR antes de mergear el PR que lo introduce (ver §0.1).
- **Cambio menor** (aclaraciones, ejemplos, correcciones tipográficas, reformulaciones que no alteran la semántica de una regla) **MAY** aprobarse por un solo revisor senior, **sin ADR**.

### 2.2 Versionado

Este documento sigue versionado `Major.Minor`:

- **Major:** cambios que obligan a refactorizar código existente.
- **Minor:** adiciones compatibles hacia atrás.

El código existente al momento de un cambio Major **SHOULD** adecuarse dentro de 2 sprints o declarar excepción explícita con ADR.

### 2.3 Excepciones

Las excepciones se manejan según la política unificada de **§0.1**:

- **Excepciones a reglas MUST fuera de §12 y §18**, o excepciones a cualquier SHOULD: se justifican en el PR (descripción y comentario en el código). No requieren ADR.
- **Excepciones a reglas MUST de §12 (seguridad) o a líneas rojas de §18**: requieren ADR previo + aprobación del arquitecto, conforme a §0.1.
- Toda excepción **MUST** dejar trazabilidad en el código donde se aplica (comentario con el motivo o enlace al ADR cuando aplique).
- Las excepciones a reglas MUST de §12/§18 se revisan en el siguiente refinamiento de arquitectura para decidir si se cierran o se elevan a cambio formal del documento.

---

## 3. Principios generales

El backend de GOP 360° **MUST** ser gobernado por los siguientes principios, en orden de prioridad ante conflictos:

1. **Correctitud regulatoria sobre elegancia técnica.** Cuando la normativa ANH entra en conflicto con un patrón técnico, la normativa gana.
2. **Seguridad por defecto.** Todo endpoint nace privado; se abre explícitamente. Toda consulta filtra por tenant; se documenta la excepción.
3. **Mantenibilidad sobre performance prematuro.** Optimizamos cuando tenemos medición; mientras, preferimos claridad.
4. **Consistencia sobre creatividad local.** Si una regla de este documento se aplica a un caso, se aplica siempre, aunque alguien "tenga una idea mejor".
5. **Separación estricta de responsabilidades.** El controlador coordina; el handler orquesta; el dominio decide; la infraestructura persiste.
6. **Fallos explícitos, no silenciosos.** Una excepción capturada sin acción es un bug. Un `catch { }` vacío es PROHIBIDO.
7. **Trazabilidad total.** Todo cambio de estado deja huella. Toda acción relevante produce evento de auditoría.
8. **Preparación para evolución.** Diseñamos asumiendo que V2 existirá y que V1 convivirá con ella durante meses.

---

## 4. Stack tecnológico

Este stack es vinculante. Cambios requieren ADR y aprobación de arquitectura.

| Componente | Tecnología | Versión | Observación |
|---|---|---|---|
| Lenguaje | C# | 12+ | Con .NET 10 |
| Runtime | .NET | 10 | Definido por Estándar de Codificación |
| Framework Web | ASP.NET Core | 10 | — |
| ORM | Entity Framework Core | 10 | Con proveedor SQL Server |
| Base de datos | SQL Server / SQL Azure | 2022 | Versión fijada por solicitud del cliente. Incluye compatibilidad on-premise |
| Dispatcher CQRS | .NET base + dispatchers propios | — | Implementación propia en `Anh.Gop.Shared.Cqrs`. Sin dependencia externa de mediador. Ver §10.1 y postura "lean base" en §10.1.1. |
| Validación | FluentValidation | 11+ | Obligatorio en DTOs |
| Logging | Serilog | 3+ | Structured logging |
| Documentación API | Swashbuckle (OpenAPI) | 6+ | Obligatorio |
| Migraciones BD | DbUp | 5+ | Sobre EF Core migrations |
| Testing unitario | xUnit | 2+ | — |
| Testing BD | tSQLt | — | Definido por Estándar de Codificación |
| Analyzers | StyleCop.Analyzers + Microsoft.CodeAnalysis.NetAnalyzers | — | Definido por Estándar de Codificación |
| Auditoría automática | Audit.NET + Audit.EntityFramework.Core | — | Obligatorio para escribir `audit.AuditLog`. Reemplaza interceptores EF manuales. Ver §11.6. |
| Cache | In-Process (MemoryCache) | — | V1 local. Redis queda para V2 si se requiere. |
| Identity framework | ASP.NET Core Identity | 10 | Base infraestructural de `gop.identity` (ver §5.2 y `data-model.md` §9). Provee `UserManager`, `RoleManager`, password hashing, lockout, tokens y stores EF Core. |
| Hardening de seguridad | `Anh.Gop.Shared.Security` | — | Middleware y helpers para cumplimiento automático de OWASP Top 10 (security headers, HSTS, antiforgery, rate limiting, SSRF handler, secrets validator, deserialización segura). Ver §12. |
| Rate limiting | `Microsoft.AspNetCore.RateLimiting` | 10 (built-in) | Policies predefinidas (`auth-strict`, `api-standard`, `export-heavy`, `read-standard`) en shared lib. Ver §12.5. |
| Analyzers de seguridad | `Microsoft.CodeAnalysis.NetAnalyzers` (categoría Security) + `SecurityCodeScan` | — | Warning-as-error en `Directory.Build.props`. Ver §12.13. |
| Supply chain scan | `dotnet list package --vulnerable` + Dependabot/Renovate + Trivy (Docker) + SBOM CycloneDX | — | Automatizado en CI. Ver §12.12. |
| Reverse Proxy | nginx | — | On-premise. Ver §17. |

**PROHIBIDO** introducir dependencias fuera de este stack sin ADR aprobado.

### 4.1 Política de licencias de dependencias NuGet

Toda dependencia NuGet **MUST** tener licencia compatible con uso en software del Estado:

- **Permitidas sin revisión:** MIT, Apache 2.0, BSD (2-Clause y 3-Clause), MS-PL, MS-RL.
- **Permitidas con evaluación:** LGPL (solo dynamic linking), MPL 2.0, EPL 2.0. Requieren ADR documentando cómo se consume.
- **PROHIBIDAS** salvo aprobación explícita de arquitectura + OTI: GPL v2, GPL v3, AGPL, SSPL, licencias comerciales no revisadas, licencias "source-available" con restricciones de uso.

**MUST:** el pipeline de CI ejecuta validación automática de licencias en cada build usando `dotnet-project-licenses` (o equivalente). El build **MUST** fallar si detecta una licencia no permitida. El CI es la única fuente de verdad para el cumplimiento — no se exige duplicar la verificación manualmente en el PR.

**MAY:** el desarrollador incluya en el PR una nota sobre la licencia de la dependencia añadida si considera que facilita la revisión (útil para dependencias nuevas o poco conocidas). No es obligatorio; CI lo cubre.

**Nota sobre mediadores:** este proyecto **no** usa MediatR ni equivalentes comerciales en V1. El patrón CQRS se implementa con dispatchers propios en `Anh.Gop.Shared.Cqrs` (ver §10.1 y §19.7). La adopción de un mediador externo se evalúa solo si se cumplen los criterios de §10.1.1.

---

## 5. Arquitectura de alto nivel

### 5.1 Patrón arquitectónico

El backend **MUST** seguir **Clean Architecture** con **CQRS** explícito, conforme a la sección 2.3 del *Documento de Estándares de Codificación*. Los bounded contexts se organizan como **microservicios independientes**, alineados con los esquemas del modelo de datos.

### 5.2 Microservicios planeados para V1

El sistema V1 se despliega como los siguientes servicios independientes, cada uno con su propio repositorio y ciclo de release:

| Servicio | Responsabilidad | Esquemas BD principales |
|---|---|---|
| `gop.identity` | Autenticación, usuarios, roles, permisos, sesiones. Construido sobre **ASP.NET Core Identity** (UserManager, RoleManager, password hashing, lockout, tokens) con stores EF Core sobre el esquema `identity`. | `identity` |
| `gop.core` | Operadores, contratos, campos, bloques, clusters, catálogos | `core`, `catalog` |
| `gop.operations` | Pozos, ciclo de vida, formas Serie 100, IDOP/IDOC | `ops` |
| `gop.production` | Producción, formas Serie 200, balances, inyección | `prod` |
| `gop.procedures` | Trámites, workflow, firmas, PDFs, validaciones espaciales | `procedure`, `workflow` |
| `gop.integration` | Staging, homologación, sincronización con externos | `integration` |
| `gop.audit` | **Consulta y gestión** del log de auditoría (búsqueda, exportación, retención, particionado). **No** es hub de escritura: cada microservicio escribe su propia auditoría localmente vía Audit.NET (ver §11.6). | `audit` |

**Nota:** `gop.procedures` incluye al motor de workflow porque conceptualmente son un binomio en V1. Si en el futuro el motor se reutiliza para flujos no-regulatorios, se extraerá a servicio propio.

### 5.3 Estructura de proyectos dentro de cada servicio

Cada microservicio **MUST** tener la siguiente estructura de proyectos .NET:

```
/src
 /Anh.Gop.{Service}.Api              → ASP.NET Core Web API (punto de entrada)
 /Anh.Gop.{Service}.Application       → Use cases, commands, queries, DTOs, validators
 /Anh.Gop.{Service}.Domain            → Entities, Value Objects, Domain Events, Aggregates
 /Anh.Gop.{Service}.Infrastructure    → EF DbContext, repos, external clients, migrations
 /Anh.Gop.{Service}.Tests.Unit        → Unit tests (Domain + Application)
 /Anh.Gop.{Service}.Tests.Integration → Integration tests (API + DB)
```

Dependencias entre proyectos (regla inviolable):

- `Api` → `Application`, `Infrastructure`
- `Application` → `Domain`
- `Infrastructure` → `Application`, `Domain`
- `Domain` → (nada — capa más interna)

**PROHIBIDO** que `Domain` dependa de `Application` o `Infrastructure`. **PROHIBIDO** que `Application` dependa de `Infrastructure` (usa interfaces).

### 5.4 Shared Libraries

Los componentes transversales viven en NuGet packages internos versionados:

| Package | Contenido | Consumido por |
|---|---|---|
| `Anh.Gop.Shared.Auth` | Middleware de autenticación, policies, requirements, clientes de Identity | Todos |
| `Anh.Gop.Shared.Security` | Hardening OWASP Top 10 automático: security headers, HSTS, cookies seguras, antiforgery, rate limiting, `SsrfProtectionHandler`, `StartupSecretsValidator`, `UseExceptionHandler` sanitizado, `JsonSerializerOptions` defaults, `LdapFilterEncoder`. Se activa con `AddGopSecurityHardening()` / `UseGopSecurityHardening()`. Ver §12. | Todos |
| `Anh.Gop.Shared.Cqrs` | Interfaces (`ICommand`, `IQuery`, handlers), dispatchers, decoradores y `AddGopCqrs()`. Sin dependencias externas de mediador. Ver §10.1 y §19.7. | Todos |
| `Anh.Gop.Shared.Contracts` | DTOs y eventos compartidos entre servicios | Todos |
| `Anh.Gop.Shared.Infrastructure` | Serilog config, health checks base, Swagger config, `AuditConfiguration` (Audit.NET) | Todos |
| `Anh.Gop.Shared.ResultPattern` | Tipo `Result<T>` y helpers | Todos |
| `Anh.Gop.Shared.Testing` | Utilidades de test (fixtures, builders) | Proyectos de test |

Las shared libraries **MUST** versionarse semánticamente. Cambios breaking requieren bump mayor. Los servicios adoptan nuevas versiones en su propio timing.

**PROHIBIDO** que las shared libraries contengan lógica de negocio de ningún dominio específico. Son puramente infraestructura transversal.

---

## 6. Arquitectura multitenant con host (ANH)

Esta sección es **central** para GOP y sus reglas son vinculantes en toda query, endpoint y servicio.

### 6.1 Modelo conceptual

GOP 360° implementa el patrón **Multitenancy with Regulator-as-Host**:

- **Tenants** son las empresas operadoras. Cada operadora es un tenant independiente con sus propios pozos, contratos, campos, trámites y usuarios.
- **Host** es la ANH. Opera "al otro lado de la ventanilla" — no es un tenant más, es el fiscalizador con visibilidad transversal controlada sobre todos los tenants.
- **Socios de contrato** (`ContractPartner`) son operadores que participan en contratos cuyo titular es otro operador. El socio es su propio tenant; su acceso al contrato del titular se modela mediante asignaciones explícitas de `UserRole` con `ContractId` del contrato externo.

### 6.2 Identificación de tenant

**TODO** usuario **MUST** pertenecer a exactamente uno de los siguientes contextos:

1. **Tenant operador**: `TenantType = "operator"`, `TenantId = OperatorId`.
2. **Host ANH**: `TenantType = "anh"`, `TenantId = null` (o valor convencional `0`).

El tenant se resuelve al autenticar y **MUST** incluirse en el JWT como claim. **PROHIBIDO** cambiar el tenant de un usuario existente — si cambia su relación laboral, se desactiva y se crea uno nuevo.

### 6.3 Segregación de datos

**MUST** para toda query que retorna datos de entidades de tenant:

- Usuarios operador ven **solo** datos de su `TenantId`.
- Usuarios ANH ven datos de **todos** los tenants, posiblemente filtrados por scope fino adicional (ver §7).
- La segregación se impone por una de estas dos vías:

**Vía A — Global Query Filter de EF Core (para tenant simple):**
Se configura un `HasQueryFilter` en el `DbContext` que aplica `WHERE TenantId = @currentTenantId` automáticamente a toda query sobre esa entidad. El `@currentTenantId` se lee del contexto de request.

**Vía B — Filtro explícito en handler (para casos complejos):**
El handler recibe `ICurrentUserContext` inyectado y aplica el filtro de forma explícita. Se usa cuando:
- La entidad pertenece a múltiples tenants indirectamente (ej. un contrato con socios).
- Se necesita lógica condicional (usuarios ANH tienen reglas distintas).
- La entidad no tiene columna `TenantId` directa (el tenant se resuelve por JOIN).

**PROHIBIDO** escribir una query sobre entidad de tenant sin aplicar alguna de las dos vías. El PR será bloqueado.

### 6.4 Bypass del filtro

Hay escenarios legítimos donde el host ANH debe ver todos los tenants (ej. dashboard nacional, listados consolidados). Para estos:

- **MUST** usar el método explícito `IgnoreTenantFilter()` en el repositorio.
- **MUST** validar en ese endpoint que el usuario es ANH (`[RequireTenantType(TenantType.Anh)]`).
- **MUST** dejar registro en auditoría con justificación del bypass.

**PROHIBIDO** bypass genérico sin validación de tipo de tenant.

### 6.5 Columna `TenantId` en el modelo

Todas las entidades de tenant **MUST** tener columna `OperatorId` o equivalente que cumple rol de `TenantId`. Esta columna:

- Es `NOT NULL` para entidades exclusivas del tenant.
- Puede ser `NULL` para entidades compartidas con ANH (ver caso por caso en el modelo).
- **MUST** estar indexada — es parte del índice compuesto de casi todas las queries.

---

## 7. Autenticación y autorización

### 7.1 Autenticación

**MUST:**
- SSO contra el IdP externo designado por ANH (Active Directory corporativo u otro que ANH confirme — ver §7.1.1), conforme mandato OTI.
- MFA obligatorio para todos los usuarios internos (ANH y operadoras) cuya identidad se gestione en el IdP externo.
- Tokens JWT firmados con algoritmo asimétrico (RS256 o ES256).
- Expiración corta del access token (30–60 minutos).
- Refresh tokens con rotación.
- Revocation list consultable en Identity Service.

**PROHIBIDO:**
- Passwords en código o config files.
- Tokens HMAC con secreto compartido (usa RS256).
- Expiración superior a 60 minutos para access tokens.
- Almacenar tokens en logs, auditoría o mensajes.

### 7.1.1 Modelo de identidad — IdP federado + JWT de aplicación (Token Exchange)

GOP aplica el patrón **Token Exchange** (RFC 8693 — *OAuth 2.0 Token Exchange*), que separa explícitamente **autenticación** (quién es el usuario) de **autorización de dominio** (qué puede hacer dentro de GOP):

1. **IdP externo** (AD corporativo ANH u otro proveedor que ANH confirme) autentica al usuario vía **OIDC**, aplica MFA si corresponde y emite un `id_token` estándar.
2. **`gop.identity`** actúa como **Relying Party (RP)**: valida el `id_token` (firma, `iss`, `aud`, `exp`, `nonce`), identifica al `User` correspondiente en el modelo de dominio GOP y **emite su propio JWT de aplicación** con los claims de dominio definidos en §7.2.
3. **Frontend y microservicios GOP** consumen **únicamente** el JWT interno emitido por `gop.identity`. El `id_token` o `access_token` del IdP externo no se propaga.

Este patrón es estándar de industria, está explícitamente contemplado en:

- **RFC 7519** — JSON Web Token (permite claims de autorización como `roles`, `scope`).
- **RFC 8693** — OAuth 2.0 Token Exchange (describe literalmente el intercambio de un token externo por un token interno con claims propios).
- **OWASP ASVS v4** — capítulos 2 (Authentication) y 3 (Session Management) admiten tokens auto-emitidos cumpliendo los controles criptográficos y de sesión.
- **Microsoft guidance** para ASP.NET Core Identity + `JwtBearer` — es el flujo recomendado para APIs de dominio con identidad federada.

**Separación de responsabilidades:**

| Responsabilidad | Dueño | Por qué |
|---|---|---|
| Autenticación primaria (credencial, MFA, SSO) | IdP externo (AD ANH) | Gobernanza central de identidades del Estado |
| Emisión de `id_token` OIDC | IdP externo | Prueba criptográfica de autenticación |
| Emisión de **JWT de aplicación GOP** | `gop.identity` | GOP es dueño del modelo de dominio (roles, scopes, tenants, contratos) |
| Claims de autorización (`roles`, `permissions`, `tenant_id`, `assigned_contract_ids`) | `gop.identity` | Son del dominio GOP, no del IdP |
| Revocación de sesión GOP | `gop.identity` | Evento de negocio propio |

**PROHIBIDO:**
- Propagar el `access_token` o `id_token` del IdP externo a microservicios GOP — sólo circula el JWT interno.
- Incluir claims de autorización específicos de GOP en el `id_token` del IdP externo (rompe la separación IdP ≠ Authorization Server de dominio).
- Que microservicios GOP validen contra el IdP externo — validan **sólo** contra la clave pública de `gop.identity`.

#### 7.1.1.1 Usuarios locales (modo transitorio y fallback operativo)

GOP usa **ASP.NET Core Identity** como capa infraestructural de `gop.identity` (ver §4 y §5.2). Esto habilita, **por configuración y por diseño**, la capacidad de gestionar **usuarios locales con credenciales propias** en paralelo al flujo federado OIDC. Esta capacidad está presente **intencionalmente** para cubrir tres escenarios:

1. **Testing funcional inicial (pre-federación):** mientras ANH confirma formalmente el IdP externo a utilizar, el equipo de desarrollo y los testers de aceptación operan con **usuarios locales administrados por ASP.NET Core Identity** para ejecutar pruebas funcionales, integración y aceptación. Cumplen las mismas políticas de password, lockout y MFA definidas en §12.6.
2. **Usuarios fuera del dominio de identidad de ANH:** operadores, contratistas asociados, socios de contrato u otros actores externos que **no** tengan cuenta en el IdP administrado por ANH. Para ellos `gop.identity` mantiene credenciales locales hasta que exista un mecanismo de federación alternativo (p.ej. Azure AD B2C, Gov.co, un IdP de operadoras).
3. **Cuentas técnicas y de servicio:** integraciones, jobs batch y cuentas administrativas de bootstrap (el primer `ANH_Admin` tiene que existir antes de cualquier federación).

**MUST:**
- Los usuarios locales cumplen **íntegramente** las políticas de §12.6 (password mínimo 12, complejidad, historial 5, expiración 90 días, lockout dual, MFA obligatoria para roles sensibles).
- Cada `User` en `identity` lleva columna `AuthenticationSource` (`External` | `Local`) que documenta el origen de su autenticación.
- El JWT de aplicación emitido por `gop.identity` es **idéntico en forma y claims** (§7.2) sea cual sea el origen de autenticación. Los microservicios consumidores **no** distinguen usuarios externos vs. locales — sólo ven el JWT interno.
- Las credenciales locales se almacenan con el hasher default de ASP.NET Core Identity (PBKDF2 HMAC-SHA512, ≥100k iteraciones — ver §12.8).

**SHOULD:**
- Preferir federación OIDC cuando el usuario **sí** exista en el IdP de ANH — los usuarios locales son excepción, no regla.
- Documentar en el ADR de despliegue qué porcentaje de usuarios opera en cada modo, para evaluar la evolución.

**PROHIBIDO:**
- Usuarios locales con rol `ANH_Admin` o roles equivalentes de privilegio **fuera** de:
  - La(s) cuenta(s) de bootstrap inicial (documentadas con ADR y rotación obligatoria tras primer login del admin real federado).
  - Cuentas técnicas cuyo uso no puede federarse (documentadas en ADR con revisión trimestral).
- Reutilización de passwords entre usuarios locales y credenciales de sistemas externos.

**Evolución esperada:**

La modalidad de usuarios locales es **transitoria y configurable**. Una vez que ANH confirme formalmente:
- Qué IdP externo se utilizará (AD, Azure AD, B2C, otro).
- Que **todos** los usuarios del sistema pueden federarse por ese IdP (ANH + operadoras + contratistas).

Entonces el flag `AllowLocalAuthentication` en configuración de `gop.identity` **MUST** desactivarse para todos los usuarios no-técnicos, y el ADR correspondiente registra la fecha de corte, el plan de migración de los usuarios locales existentes y el conjunto acotado de cuentas técnicas que permanecen locales con justificación.

Hasta que esa confirmación exista, la coexistencia federación + local es el modelo operativo por defecto.

### 7.2 Contenido del JWT

El JWT emitido por `gop.identity` **MUST** contener los siguientes claims:

| Claim | Tipo | Contenido |
|---|---|---|
| `sub` | string | UserId |
| `tenant_id` | string | OperatorId del tenant del usuario, o `"anh"` para host |
| `tenant_type` | string | `"operator"` \| `"anh"` |
| `roles` | string[] | Códigos de rol |
| `permissions` | string[] | Capabilities (ver §7.5) |
| `assigned_contract_ids` | long[] | Lista de IDs de contratos asignados (ver §7.4) |
| `contracts_truncated` | bool | `true` si la lista excede límite |
| `license_validated_until` | datetime? | Fecha hasta la que la matrícula profesional está validada (null si no aplica) |
| `exp`, `iat`, `nbf`, `iss`, `aud` | — | Estándar JWT |

**SHOULD:** el JWT total **no debe exceder 8 KB**. Si se aproxima, revisar estructura.

#### 7.2.1 Claims prohibidos en el JWT

El JWT es **firmado pero no cifrado** por defecto — cualquier portador del token puede decodificarlo en base64. Por tanto está **PROHIBIDO** incluir en el JWT:

| Categoría | Ejemplos | Razón |
|---|---|---|
| PII innecesaria | Email completo, nombre, dirección, teléfono, número de documento | Minimización (OWASP ASVS V8, principio de menor información) |
| Datos sensibles / especialmente protegidos | Datos de salud, financieros, biométricos, judiciales | Normativa de protección de datos + riesgo de exposición |
| Tokens de terceros | `access_token` / `id_token` del IdP externo, tokens de SGC/BMC/ContrAktor | Rompe aislamiento entre dominios de confianza |
| Secretos y credenciales | Passwords, hashes, claves API, client secrets | Nunca deben circular en claims legibles |
| Datos mutables de alta frecuencia | Contadores de operación, timestamps de última acción, saldos | Fuerzan refresh innecesario; el claim queda obsoleto entre emisiones |
| Estructuras grandes no acotadas | Listas sin límite superior, árboles profundos | Riesgo de JWT >8 KB y problemas de performance en headers |

**Regla general:** el JWT transporta **identidad + claims mínimos de autorización suficientes para decidir acceso en la capa de policies (§7.3 capa 3)**. Cualquier dato adicional se obtiene on-demand desde `gop.identity` o del servicio dueño.

**MUST:**
- Identificadores en claims: usar el `sub` (UserId) y los IDs estrictamente necesarios (`tenant_id`, `assigned_contract_ids`).
- El nombre visible del usuario se obtiene por endpoint `/users/me` de `gop.identity`, no por claim.

### 7.3 Arquitectura de autorización en 5 capas

Conforme al análisis de diseño validado, la autorización **MUST** implementarse en cinco capas, cada una con responsabilidad clara:

| Capa | Nivel | Implementación | Ejemplo |
|---|---|---|---|
| 1. Identity | Infra | ASP.NET Core Identity + SSO AD + MFA | Usuario se autentica, recibe JWT |
| 2. Tenancy | Middleware | Resuelve `TenantId` del JWT, lo inyecta en `ICurrentUserContext` | Bloquea request sin tenant válido |
| 3. Permissions | Policy-based (declarativo) | `[Authorize(Policy = "CanApproveForm103")]` | Atributo en endpoint |
| 4. Resource-based | Handler | `_authService.AuthorizeAsync(user, resource, "View")` | Después de cargar el recurso |
| 5. Domain Requirements | Handler | `HasValidProfessionalLicense` requirement | Antes de ejecutar firma |

### 7.4 Alcance de datos (Data Scope) — modelo unificado en `UserRole`

Todo alcance de datos del usuario (tenant, operadoras accesibles, contratos asignados, campos) se deriva de **dos fuentes y sólo dos**:

1. **`User.OperatorId`** — define el tenant "propio" del usuario (`NULL` para usuarios ANH).
2. **`UserRole(RoleId, ContractId?, OperatorId?, FieldId?, StartDate, EndDate)`** — define los accesos con sus dimensiones.

**PROHIBIDO** introducir entidades adicionales de scope (p.ej. `UserOperatorScope`, `UserFieldScope`) fuera de `UserRole`. Si aparece un nuevo eje de alcance, se modela como columna nullable en `UserRole`, no como tabla nueva.

Un usuario **MAY** tener múltiples filas de `UserRole` vigentes. Su **alcance efectivo es la unión** de todas las asignaciones vigentes (`StartDate ≤ now < EndDate` o `EndDate IS NULL`).

#### 7.4.1 Reglas de interpretación

Para un `UserRole` dado, el alcance resultante se resuelve así:

| Tipo rol | `ContractId` | `OperatorId` | `FieldId` | Alcance resultante |
|---|---|---|---|---|
| Operador | NULL | NULL | NULL | Todos los contratos del `User.OperatorId` |
| Operador | X | NULL | NULL | Solo el contrato X (de su operador) |
| ANH | NULL | NULL | NULL | Global — todos los operadores y contratos |
| ANH | X | NULL | NULL | Solo el contrato X |
| ANH | NULL | Y | NULL | Todos los contratos del operador Y |
| ANH | NULL | NULL | Z | Todos los contratos que contienen el campo Z |
| ANH | X | Y | NULL | Redundante — se **MUST** validar que X pertenezca a Y |

**Reglas adicionales:**

1. Usuarios operador **MUST NOT** tener filas con `OperatorId` distinto a `User.OperatorId` — la constraint lo valida.
2. Socios de contrato (operador cuyo operador es `ContractPartner.PartnerOperatorId`) **MUST** tener fila explícita `UserRole(ContractId = contrato_externo)` para ver el contrato del titular. Sin fila explícita, no lo ven.
3. El rol `OperatorAdmin` es un rol de operador que habilita gestionar usuarios y asignaciones dentro de su propia operadora.
4. Combinaciones contradictorias (p.ej. `ContractId` que no pertenece a `OperatorId`) **MUST** rechazarse en la capa de `gop.identity` al asignar el `UserRole`.

#### 7.4.2 Resolución al emitir el JWT

El claim `assigned_contract_ids` **no** es copia directa de la columna — es el resultado de **resolver** todas las filas vigentes de `UserRole` del usuario:

- `UserRole.ContractId = X` → agrega X.
- `UserRole.ContractId = NULL` y rol operador → expande a todos los contratos de `User.OperatorId`.
- `UserRole.OperatorId = Y` (ANH) → expande a todos los contratos del operador Y.
- `UserRole.FieldId = Z` (ANH) → expande a todos los contratos que contienen el campo Z (JOIN a `FieldContract`).
- `UserRole` ANH global (todo NULL) → la lista no se expande en JWT; se marca `contracts_truncated = true` y se consulta bajo demanda.

**Truncamiento para tamaño de JWT:**

- Si la lista resuelta tiene ≤100 elementos, se incluye completa y `contracts_truncated = false`.
- Si excede 100 o el rol es ANH global, `contracts_truncated = true`. El servicio consumidor **MUST** consultar `gop.identity` vía endpoint `/users/{id}/contracts` cuando necesite certeza.

**PROHIBIDO** que un servicio confíe en que `assigned_contract_ids` está completa sin verificar `contracts_truncated`.

### 7.5 Permission-Based Authorization

**MUST:** las policies de autorización se expresan en términos de **capabilities** (permisos finos), no de roles. El rol se convierte en un agrupador de capabilities.

**Ejemplo correcto:**

```csharp
[HttpPost("approve")]
[Authorize(Policy = "Forms.F103.CanApprove")]
public async Task<IActionResult> Approve([FromBody] ApproveForm103Command cmd)
{
    return Ok(await _commandDispatcher.DispatchAsync(cmd));
}
```

**PROHIBIDO:**

```csharp
[Authorize(Roles = "AnhGeologist")] // acoplamiento al nombre del rol
```

**MUST:** los permisos viven en la tabla `identity.Permission` (son datos, no código). Cambios de asignación rol-permiso son cambios de datos, no de despliegue.

La lista inicial de capabilities se mantiene en el apéndice §19.

### 7.6 Resource-Based Authorization

Cuando la autorización depende del recurso específico (ej. "¿este contrato pertenece al operador del usuario?"), **MUST** usarse `IAuthorizationService`:

```csharp
public async Task<Result<ContractDetailDto>> Handle(GetContractByIdQuery query, CancellationToken ct)
{
    var contract = await _repo.GetByIdAsync(query.ContractId, ct);
    if (contract is null) return Result.NotFound();

    var authResult = await _authService.AuthorizeAsync(
        _currentUser.Principal, contract, ContractOperations.View);
    if (!authResult.Succeeded) return Result.Forbidden();

    return _mapper.Map<ContractDetailDto>(contract);
}
```

Los `IAuthorizationHandler` viven en `Infrastructure` y consultan el contexto del usuario contra los atributos del recurso.

### 7.7 Domain Requirements

Autorizaciones específicas del dominio petrolero (ej. matrícula profesional vigente para firmar) se modelan como `IAuthorizationRequirement` personalizados:

```csharp
public class HasValidProfessionalLicenseRequirement : IAuthorizationRequirement
{
    public ProfessionType RequiredProfession { get; }
    // ...
}
```

Estos requirements consultan `ProfessionalLicense` en `identity` y validan vigencia. Si la matrícula expiró, la acción se deniega aunque el usuario tenga el permiso.

### 7.8 Autorización distribuida entre microservicios

Cada microservicio **MUST**:

1. Validar el JWT localmente (firma + expiración + issuer).
2. Extraer claims y construir `ICurrentUserContext` vía middleware de `Anh.Gop.Shared.Auth`.
3. Aplicar policies declarativas y resource-based authorization localmente.
4. Para data scope fino que requiera consulta de datos no presentes en JWT, usar el cliente `IIdentityServiceClient` de la shared library, que consulta `gop.identity` con cache en memoria (TTL 5 minutos).

**Criterio claro de cuándo consultar Identity Service:**

| Escenario | Fuente de verdad |
|---|---|
| Validar token (firma, expiración) | Local — clave pública en cada servicio |
| Tenant del usuario | JWT claim `tenant_id` |
| Roles del usuario | JWT claim `roles` |
| Permisos del usuario | JWT claim `permissions` |
| Contratos asignados (caso normal) | JWT claim `assigned_contract_ids` |
| Contratos asignados (truncado) | Identity Service con cache |
| Matrícula profesional vigente | JWT claim `license_validated_until` si existe; Identity Service si no |
| Verificar si usuario sigue activo | **Se asume activo durante la vida del token** — revocación se aplica en próximo refresh |

**PROHIBIDO:**
- Un microservicio **MUST NOT** hacer HTTP a Identity Service para verificar cada acción rutinaria. Eso invalida el diseño stateless del JWT.
- Un microservicio **MUST NOT** implementar su propia lógica de validación de token — **MUST** usar `Anh.Gop.Shared.Auth`.

### 7.9 Revocación de permisos

Dado que V1 no tiene event bus, la revocación tiene estas características (aceptadas como trade-off de diseño):

- Cambios de permisos o rol se aplican al siguiente refresh de token del usuario (máximo 60 min de demora).
- Para revocación inmediata de un usuario comprometido, `gop.identity` mantiene una **revocation list** consultable. Los servicios **MAY** consultarla en operaciones críticas (firmas, aprobaciones de abandono).
- Cuando se implemente event bus (V2+), la revocación se propagará por eventos y se invalidará cache local inmediatamente.

### 7.10 Clasificación de datos y auditoría reforzada

**MUST:** toda entidad que contenga **datos clasificados o reservados** (conforme a la normativa de protección de información del Estado y clasificación interna ANH) **MUST**:

1. Marcarse con el atributo `[DataClassification(Level = ...)]` en el DTO de exposición (Summary y Detail).
2. Exponerse únicamente por endpoints cuya policy exija capability de nivel igual o superior al del dato.
3. Registrar auditoría **adicional** de acceso: evento en `audit.AuditLog` con `EventType = 'DataAccess'` y, en `AdditionalContextJson`, los atributos `UserId`, `TenantId`, `Endpoint`, `ResourceType`, `ResourceId`, `AccessedFields`, `OccurredAt`, `Justification` (si el endpoint requiere justificación).
4. **PROHIBIDO** incluirlos en listados exportables sin trazabilidad del export en `audit.AuditLog` con `EventType = 'Export'` (atributos: volumen exportado, filtros aplicados, destino, justificación).

> **Nota de modelo:** `DataAccess` y `Export` son **tipos de evento especializados** dentro de la tabla única `audit.AuditLog` (ver `data-model.md` §10). No existen tablas físicas separadas `DataAccessLog` / `ExportLog`; el discriminador `EventType` y el payload en `AdditionalContextJson` cubren ambos casos sin fragmentar el esquema.

**Niveles de clasificación (marco de referencia):**

| Nivel | Código | Descripción | Auditoría extra |
|---|---|---|---|
| 0 | `Public` | Información publicable | Estándar |
| 1 | `Internal` | Uso interno, no confidencial | Estándar |
| 2 | `Confidential` | Requiere autorización expresa | `AuditLog` con `EventType='DataAccess'` |
| 3 | `Reserved` | Reservada por normativa | `AuditLog` con `EventType='DataAccess'` + justificación |
| 4 | `Restricted` | Alto impacto (ej. datos personales sensibles, coordenadas de pozos en zonas sensibles) | `AuditLog` con `EventType='DataAccess'` + justificación + notificación al responsable de datos |

**Decisión abierta — clasificación específica por entidad:**

La asignación concreta de nivel a cada entidad del modelo (p.ej. ¿`Well.FiscalizedUwi` es `Public` o `Internal`? ¿`ProcedureSignature.ProfessionalLicenseNumber` es `Confidential`?) **queda pendiente** de definición formal por parte de la Dirección de Fiscalización ANH y el responsable de datos. Hasta que ANH publique la matriz de clasificación:

- Todas las entidades operan por defecto como `Internal`.
- Entidades con datos personales (`User`, `ProfessionalLicense`) operan por defecto como `Confidential`.
- Se registrará ADR con la matriz final cuando ANH la emita.

**MUST:** ningún desarrollador asigna nivel `Confidential` o superior sin aprobación documentada del responsable de datos.

### 7.11 Líneas rojas de seguridad

**PROHIBIDO:**

- Log de tokens, passwords, hashes, datos sensibles del usuario.
- Exponer información de usuarios de otros tenants, aunque sea por error.
- Devolver stack traces en responses de producción.
- `[AllowAnonymous]` en endpoints de mutación — solo permitido en auth endpoints.
- Confiar en headers `X-User-*` salvo que vengan del gateway con conexión interna garantizada.

### 7.12 Cumplimiento normativo del modelo de identidad

Esta sección **mapea explícitamente** los controles de las normas y estándares aplicables al modelo de identidad de GOP. Su propósito es servir como **evidencia directa de cumplimiento** frente a auditorías internas, revisiones de ciberseguridad de ANH y cualquier stakeholder que requiera justificar el diseño.

#### 7.12.1 Ninguna norma aplicable prohíbe emitir JWT propios

Ni **OWASP Top 10**, ni **OWASP ASVS v4**, ni **ISO/IEC 27001/27002:2022**, ni los RFC aplicables **prohíben** que una aplicación emita sus propios JWT ni **exigen** un IdP externo como única fuente de tokens. Lo que exigen es que **los controles técnicos de emisión, firma, transporte, validación y revocación** cumplan buenas prácticas. El patrón **RFC 8693 — OAuth 2.0 Token Exchange** contempla literalmente el flujo "token externo → token interno con claims de dominio propios" que aplica GOP.

#### 7.12.2 Mapeo OWASP ASVS v4 (Authentication & Session Management)

| Control ASVS v4 | Requisito | Sección GOP |
|---|---|---|
| V2.1.1–V2.1.9 | Password strength, história, notificación de cambio | §12.6 |
| V2.2.1–V2.2.7 | Rate limiting, lockout, protección anti-automation | §12.5, §12.6 |
| V2.4.1–V2.4.5 | Credential storage (PBKDF2/Argon2/bcrypt, ≥100k iter) | §12.8 |
| V2.5.1–V2.5.7 | Recuperación de credenciales, reset seguro | §7.1, §12.6 |
| V2.7.1–V2.7.6 | MFA, factores robustos | §12.6 |
| V2.8.1–V2.8.7 | Credenciales de máquina (service accounts) | §7.1.1.1 |
| V3.2.1–V3.2.4 | Generación de tokens de sesión aleatorios, integridad | §7.1, §12.8 |
| V3.3.1–V3.3.4 | Terminación de sesión, idle/absolute timeout | §12.6 |
| V3.5.1–V3.5.3 | **Tokens auto-emitidos: firma fuerte, validación de claims** | §7.1, §7.2, §12.8 |
| V3.7.1 | Reautenticación para acciones sensibles | §7.7 (Domain Requirements) |

> **Nota ASVS V3.5:** este bloque de ASVS v4 aplica específicamente a **tokens auto-emitidos por la aplicación** (self-contained tokens / JWT). Su existencia en el estándar confirma que emitir JWT propios es un escenario reconocido y admitido, no una desviación.

#### 7.12.3 Mapeo OWASP JWT Cheat Sheet

| Recomendación | Cómo se cumple en GOP |
|---|---|
| Rechazar `alg: none` | `JwtBearerOptions.TokenValidationParameters.ValidateIssuerSigningKey = true`; algoritmos permitidos fijados explícitamente en shared lib |
| Algoritmo asimétrico (RS256/ES256) | **RS256** mandatorio (§7.1, §12.8); HS256 prohibido |
| Rotación de keys | Anual, keys en Key Vault (§12.8) |
| Expiración corta | 30–60 min (§7.1) |
| Validar `iss`, `aud`, `exp`, `nbf` | Aplicado en `Anh.Gop.Shared.Auth` (§16.3) |
| Refresh token rotation + detección de reuso | §12.6 |
| No datos sensibles en claims | §7.2.1 |
| Revocación | Revocation list (§7.9) |

#### 7.12.4 Mapeo OWASP Top 10 (2021)

| OWASP | Aplicación al modelo de identidad | Sección |
|---|---|---|
| A01 — Broken Access Control | Autorización en 5 capas, policies declarativas, tenant isolation, resource-based authz | §7.3–§7.8 |
| A02 — Cryptographic Failures | RS256 + keys en KeyVault, PBKDF2 para passwords, TLS, Always Encrypted para nivel ≥3 | §7.1, §12.3, §12.8 |
| A07 — Identification and Authentication Failures | MFA, lockout dual, password policy, refresh rotation, revocation list | §12.5, §12.6, §7.9 |
| A09 — Security Logging and Monitoring Failures | Audit.NET por mutación, eventos de autenticación registrados con justificación si aplica | §11.6, §7.10 |

Mapa global OWASP Top 10 → documento ya disponible en §12.15.

#### 7.12.5 Mapeo ISO/IEC 27001:2022 — Anexo A (controles ISO 27002:2022)

| Control | Descripción | Cómo se cumple |
|---|---|---|
| **A.5.15** Control de acceso | Política de acceso basada en roles y necesidad | §7.3 (5 capas), §7.5–§7.6 (RBAC + resource) |
| **A.5.16** Gestión de identidad | Ciclo de vida de la identidad (alta, modificación, baja) | §7.4 (`UserRole` con StartDate/EndDate), §7.9 (revocación), `data-model.md` §9 |
| **A.5.17** Información de autenticación | Asignación, uso y protección de credenciales | §7.1.1 (federación + local), §12.6 (políticas), §12.8 (hashing) |
| **A.5.18** Derechos de acceso | Provisión y revisión periódica | §7.4 (UserRole explícito, auditable), §11.6 (audit trail) |
| **A.8.2** Privilegios de acceso | Gestión restrictiva de privilegios elevados | §12.6 (MFA obligatoria para roles sensibles), §7.1.1.1 (prohibición de ANH_Admin local salvo bootstrap) |
| **A.8.3** Restricción de acceso a la información | Accesos según clasificación | §7.10 (data classification 0–4) |
| **A.8.5** Autenticación segura | Mecanismos robustos de autenticación | §7.1, §7.1.1 (OIDC federado), §12.6 (MFA) |
| **A.8.15** Logging | Registro de eventos de seguridad | §11.6 (Audit.NET), §13 (observabilidad) |
| **A.8.24** Uso de criptografía | Política y gestión de claves | §12.8 |
| **A.8.28** Desarrollo seguro | Prácticas de desarrollo seguro | §12 completo, §12.13 (analyzers) |

> **Nota ISO 27001:** los controles del Anexo A son **neutrales respecto a la tecnología**. Ninguno exige IdP externo específico, ni proveedor comercial, ni prohíbe emisión de tokens propios. Exigen que haya política, trazabilidad, control de privilegios y gestión del ciclo de vida — todos satisfechos por el diseño actual.

#### 7.12.6 Mapeo RFC aplicables

| RFC | Relevancia | Aplicación en GOP |
|---|---|---|
| **RFC 7519** — JWT | Estructura y claims estándar | §7.2 (claims conformes); `sub`, `iss`, `aud`, `exp`, `nbf`, `iat` estándar + claims privados declarados |
| **RFC 7515** — JWS | Firma | RS256 (§7.1, §12.8) |
| **RFC 6749** — OAuth 2.0 | Framework de autorización | Flujo Authorization Code con PKCE para frontend SPA |
| **RFC 7636** — PKCE | Protección de Authorization Code | Requerido por `Anh.Gop.Shared.Auth` |
| **RFC 8693** — OAuth 2.0 Token Exchange | **Intercambio token externo → token interno** | §7.1.1 (patrón central del modelo federado) |
| **RFC 6750** — Bearer Token Usage | Transporte del token | Header `Authorization: Bearer` obligatorio; prohibida query string |
| **OIDC Core 1.0** | OpenID Connect | Integración con IdP externo (§7.1.1) |

#### 7.12.7 Controles residuales y excepciones documentadas

Los controles que **no** son automatizables y sí requieren decisión organizacional quedan fuera del alcance del backend y son responsabilidad del plan de seguridad del cliente (ANH):

- Confirmación formal del IdP externo a utilizar (input para desactivar autenticación local — ver §7.1.1.1).
- Matriz de clasificación concreta por entidad (input para `[DataClassification]` — ver §7.10).
- Política de revisión periódica de accesos (requerido por ISO A.5.18) — operativo, no de diseño.
- Plan de respuesta a incidentes (ISO A.5.24–A.5.30) — documentado fuera de esta constitución.

Cuando ANH emita la matriz de clasificación, los documentos de política correspondientes, o confirme el IdP definitivo, se emiten ADRs que actualizan esta sección sin romper el diseño base.

---

## 8. Patrón de exposición de datos — DTOs

Esta sección define cómo se expone la información al frontend y a otros consumidores. Es central para mantenibilidad.

### 8.1 Regla general

**MUST:** todo endpoint devuelve **DTOs**, nunca entidades de dominio ni modelos de EF. Los DTOs viven en `Application` y son clases inmutables (registros o `init` properties).

### 8.2 Clasificación de entidades (para decidir tratamiento)

Cada entidad del modelo de datos **MUST** clasificarse en uno de estos cuatro tipos:

| Tipo | Descripción | Tratamiento en API |
|---|---|---|
| **A. Catálogo global pequeño** | Lista cerrada <100 items, estable | Bootstrap endpoint + precarga en sesión |
| **B. Catálogo específico de feature** | Lista media, estable, usada en un módulo | Endpoint dedicado, lazy load |
| **C. Entidad transversal** | Miles de items, cambian con negocio | Endpoint de búsqueda paginada + por ID con cache |
| **D. Entidad transaccional de dominio** | Datos de operación frecuente | Lista resumida + detalle limpio |

La clasificación es decisión de arquitectura y **MUST** documentarse en el modelo de datos. Cambiar clasificación de una entidad ya productiva requiere ADR (impacta contratos públicos y caching del FE).

### 8.3 Patrón para entidades Tipo D — listado vs detalle

Esta es la regla más crítica del proyecto y aplica a la mayoría de endpoints.

#### 8.3.0 Principio rector — crudo con IDs; excepción calificada en listados

El backend emite **DTOs crudos con identificadores** (`OperatorId`, `ContractId`, `CurrentStateId`, etc.). **El frontend es quien hidrata** las etiquetas humanas resolviendo esos IDs contra sus servicios de catálogo locales (ver `CONSTITUTION-fe.md` §22 — *Normalized DTO with Client-Side Hydration*). Este principio aplica por defecto a todas las respuestas de la API.

**Justificación:**
- Una sola fuente de verdad para los catálogos (el FE los consume una vez y los reutiliza).
- Payloads más pequeños y cacheables.
- Si un catálogo cambia (renombra un estado, añade un campo), no hay que tocar endpoints transaccionales.
- Idioma/localización se aplican en cliente sin variar el contrato del back.

**Excepción calificada — listados (`SummaryDto`):** los endpoints de listado **MAY** incluir etiquetas precomputadas (resueltas por JOIN en SQL) de los catálogos referenciados. Esto se permite **exclusivamente** porque:
- Un listado puede devolver 50–500 filas por página; hidratar 500 × N IDs de catálogo en el cliente es costoso y ruidoso.
- La etiqueta se muestra literalmente en la columna de la tabla — es parte de la vista, no un dato derivado.
- El FE no puede conocer de antemano qué columnas van a necesitarse desde catálogos transversales (Tipo C, miles de items) sin prefetch que no vale la pena.

**Qué NO está cubierto por la excepción:**
- El DTO de **detalle** (`DetailDto`) nunca lleva labels precomputados — FE hidrata. La excepción del Summary no se extiende al Detail.
- Sub-agregados embebidos dentro del Summary — si se necesitan, el DTO no es un Summary; es una vista (§8.5).
- Texto largo, descripciones o campos no visibles en la tabla por defecto — esos viven en el Detail.

**Resumen en una línea:**
> **Regla general:** DTOs crudos con IDs → FE hidrata. **Excepción calificada:** `SummaryDto` de listados puede llevar labels precomputados de catálogos para evitar hidratación masiva en tablas.

#### 8.3.1 DTO de listado (Summary DTO)

**MUST** nombrar `{Entity}SummaryDto`.

**SHOULD** incluir solo campos que cumplan al menos una de estas condiciones:
- (a) Identificador (PK o código visible al usuario).
- (b) Se muestra como columna visible por defecto en la tabla de la vista.
- (c) Se usa para filtrar u ordenar el listado.
- (d) Es flag de estado que condiciona acciones en la fila (`isEditable`, `hasActiveAlerts`).

> El criterio es mantener el Summary liviano — si un caso legítimo requiere un campo adicional (ej. tooltip de uso frecuente), inclúyase con justificación en el PR. Lo que no cabe aquí se resuelve con un DTO especializado (§8.5).

**MAY** incluir labels precomputados de catálogos referenciados (resueltos vía JOIN en SQL), para evitar que el frontend tenga que hidratar 5,000 filas en un listado.

**PROHIBIDO** incluir:
- Sub-agregados o colecciones hijas.
- Texto largo (descripciones, observaciones, texto libre).
- Blobs, JSONs grandes, coordenadas en formato complejo.
- Historiales.
- Datos de auditoría (`CreatedBy`, `UpdatedBy` ok; histórico completo no).

**Ejemplo correcto:**

```csharp
public record WellSummaryDto
{
    public long WellId { get; init; }
    public string Name { get; init; }
    public string FiscalizedUwi { get; init; }
    public int CurrentStateId { get; init; }
    public string CurrentStateName { get; init; }    // precomputado con JOIN
    public long OperatorId { get; init; }
    public string OperatorName { get; init; }        // precomputado con JOIN
    public long ContractId { get; init; }
    public string ContractCode { get; init; }        // precomputado con JOIN
    public long FieldId { get; init; }
    public string FieldName { get; init; }           // precomputado con JOIN
    public bool IsEditable { get; init; }
    public bool HasActiveAlerts { get; init; }
    public DateTime CreatedAt { get; init; }
}
```

#### 8.3.2 DTO de detalle (Detail DTO)

**MUST** nombrar `{Entity}DetailDto`.

**MUST:**
- Devolver datos **crudos** (IDs), sin labels precomputados ni objetos hidratados.
- El frontend resuelve labels mediante servicios de catálogo locales.
- Sub-agregados del mismo bounded context se incluyen si son 1:1 o 1:pocos.
- Colecciones grandes se linkean vía endpoints separados bajo el recurso.

**Ejemplo correcto:**

```csharp
public record WellDetailDto
{
    public long WellId { get; init; }
    public string Name { get; init; }
    public string FiscalizedUwi { get; init; }
    public int CurrentStateId { get; init; }           // crudo, sin label
    public int AngleTypeId { get; init; }
    public int PurposeTypeId { get; init; }
    public int CompletionTypeId { get; init; }
    public long OperatorId { get; init; }
    public long ContractId { get; init; }
    public long FieldId { get; init; }
    public long? ClusterLocationId { get; init; }
    public string TrajectoryType { get; init; }
    public long? OriginWellId { get; init; }
    public bool IsLegacyWell { get; init; }
    public bool IsFinalized { get; init; }
    public UwiComponentsDto UwiComponents { get; init; }  // sub-agregado 1:1 embebido
    public WellLocationDto CurrentLocation { get; init; } // sub-agregado embebido
    public DateTime CreatedAt { get; init; }
    public string CreatedBy { get; init; }
    public DateTime UpdatedAt { get; init; }
    public string UpdatedBy { get; init; }
    // Colecciones se acceden por endpoints separados:
    // GET /wells/{id}/branches, /wells/{id}/procedures, /wells/{id}/state-history
}
```

### 8.4 Sub-recursos anidados

Colecciones hijas **MUST** exponerse como sub-recursos bajo el recurso padre:

```
GET /api/v1/wells/{wellId}/branches
GET /api/v1/wells/{wellId}/locations
GET /api/v1/wells/{wellId}/phases
GET /api/v1/wells/{wellId}/state-history
GET /api/v1/wells/{wellId}/procedures?status=active
```

### 8.5 Endpoints especializados (excepciones Tipo D)

**SHOULD** crearse cuando un caso de uso requiere forma distinta al listado o detalle estándar:

- Vistas de mapa (`/wells/map-markers`)
- Exportaciones (`/wells/export?format=xlsx`)
- Autocompletes (`/wells/search?q=xxx`)
- Dashboards (`/wells/dashboard-counts`)

**MUST:**
- Nombre del DTO refleja propósito: `WellMapMarkerDto`, `WellExportRowDto`, `WellAutocompleteDto`.
- Viven en folder separado: `Application/Wells/Views/` vs `Application/Wells/Queries/`.

**SHOULD:** documentar en el PR o en la propia vista por qué no bastan Summary/Detail. ADR solo si introduce un patrón recurrente nuevo (ver §0.1).

### 8.6 Patrón para entidades Tipo A — Catálogos globales

**MUST:** cada catálogo global se expone via:

```
GET /api/v1/catalogs/{catalog-name}
```

Donde `{catalog-name}` es el nombre del catálogo en kebab-case (`well-states`, `angle-types`, `purpose-types`).

**MAY:** ofrecerse un bootstrap agrupado en `gop.core`:

```
GET /api/v1/session/bootstrap
→ { wellStates: [...], angleTypes: [...], ... }
```

**MUST:** estructura uniforme de respuesta:

```json
{
  "items": [
    {
      "id": 42,
      "code": "DRILLING",
      "name": "En Perforación",
      "description": "...",
      "isActive": true,
      "displayOrder": 3,
      "metadata": { "dashboardColor": "#ff9900" }
    }
  ],
  "version": "2026-04-15T10:00:00Z"
}
```

**SHOULD:** soporte de ETag / If-None-Match para habilitar cache en frontend.

### 8.7 Patrón para entidades Tipo C — Entidades transversales

**MUST:**
- Endpoint de búsqueda paginada: `GET /api/v1/operators?search=xxx&pageSize=20`.
- Endpoint por ID: `GET /api/v1/operators/{id}`.
- Respuestas crudas (IDs, sin hidratación).

---

## 9. Contratos de API

### 9.1 Versionado

**MUST:** todo endpoint vive bajo `/api/v{N}/...`. Versión inicial: `v1`.

**SHOULD:** cuando se publica `v2`, `v1` se mantiene vivo al menos 6 meses tras la liberación de `v2` para dar margen de migración a los consumidores. El plazo puede acortarse si el inventario de consumidores es conocido y todos han migrado; se documenta la decisión (y el plan de migración confirmado) en el PR de retiro. Prolongarlo es siempre aceptable.

**PROHIBIDO** breaking changes en un endpoint una vez publicado en su versión.

### 9.2 Paginación uniforme

**MUST:** todos los endpoints de listado usan esta estructura:

**Query:**
```
?page=1&pageSize=50&sortBy=name&sortDirection=asc&filter.xxx=...
```

**Response:**
```json
{
  "items": [ /* WellSummaryDto[] */ ],
  "pagination": {
    "page": 1,
    "pageSize": 50,
    "totalItems": 247,
    "totalPages": 5
  }
}
```

**MUST:** todo endpoint de listado define un `pageSize` máximo configurable y rechaza con HTTP 400 cualquier solicitud que lo exceda. No se permiten listados sin tope superior.

**SHOULD:** valores por defecto del proyecto (pueden ajustarse por endpoint con justificación en el PR, sin ADR):
- Default: 50.
- Máximo en endpoints normales: 100.
- Máximo en endpoints de exportación: 500 por página (y exportación paginada internamente si excede).

Los valores viven en configuración del servicio, no hardcoded en código de contrato.

### 9.3 Filtros tipados

**MUST:** filtros son campos específicos, no free-form:

```
?filter.currentStateId=42&filter.operatorId=10&filter.namePartial=rubi
```

**MUST:** cada endpoint tiene un `GetXxxQuery` (implementa `IQuery<TResult>`) con propiedades tipadas validadas por FluentValidation. El binding de query string es automático.

**PROHIBIDO** filtros dinámicos tipo LINQ injection (`?filter=age > 5 and name like 'foo'`).

### 9.4 Ordenamiento

**MUST:** campo `sortBy` toma uno de los campos del `SummaryDto`. Campos no-listados **MUST** retornar HTTP 400.

**MUST:** `sortDirection` ∈ {`asc`, `desc`}. Default `asc`.

### 9.5 Respuestas de error — ProblemDetails

**MUST** usar RFC 7807 (ProblemDetails) para errores:

```json
{
  "type": "https://api.gop360.anh.gov.co/errors/validation",
  "title": "Validation failed",
  "status": 400,
  "detail": "One or more validation errors occurred",
  "instance": "/api/v1/wells",
  "traceId": "00-abc123-def456-01",
  "errors": {
    "Name": ["Name is required"],
    "OperatorId": ["Operator not found"]
  }
}
```

Códigos HTTP estándar:
- `400` — errores de validación o request malformed.
- `401` — token inválido o ausente.
- `403` — autenticado pero sin permiso.
- `404` — recurso no existe o el usuario no tiene alcance para verlo (no revelar existencia).
- `409` — conflicto (ej. concurrencia).
- `422` — regla de negocio violada (distinto a validación sintáctica).
- `500` — error no controlado (sin stack trace en producción).

**PROHIBIDO** devolver 500 por errores de validación o reglas de negocio.

### 9.6 Respuestas de éxito — POST / PUT / PATCH / DELETE

Las respuestas de error ya están estandarizadas por §9.5 (ProblemDetails RFC 7807). Esta sección fija el **contrato de éxito** para todas las mutaciones, garantizando uniformidad entre servicios y facilitando el consumo por el frontend.

#### 9.6.1 Principio — sin envelope universal

**PROHIBIDO** envolver respuestas exitosas en un wrapper custom tipo `{ status, message, data }` o `{ success: true, data: ... }`. Razones:

1. HTTP ya entrega `status` en el **status code** — redundarlo en el body rompe el protocolo.
2. Los errores viajan en **ProblemDetails** (§9.5) — un estándar con soporte nativo en librerías HTTP del ecosistema.
3. Un envelope universal obliga al cliente a hacer `res.data.xxx` en lugar de `res.xxx`, infla cada payload y ensucia los tipos generados desde OpenAPI.
4. Los **mensajes de UX** (toasts, confirmaciones) los produce el frontend con su i18n. El backend entrega **hechos** (recurso, estado), no texto de presentación.

La consistencia real del contrato viene de: **status code correcto + ProblemDetails en error + DTO tipado en éxito**.

#### 9.6.2 Tabla de convención — status, body y headers

| Verbo | Escenario | Status | Body | Headers |
|---|---|---|---|---|
| `POST /resources` | Crear recurso | **201 Created** | DTO Detail del recurso creado | `Location: /api/v1/resources/{id}` |
| `POST /resources/{id}/action` | Acción de dominio sincrónica (approve, sign, abandon, submit) | **200 OK** | DTO Detail del recurso con el nuevo estado | — |
| `POST /resources/{id}/action` | Acción asíncrona (integración SGC, export masivo, job largo) | **202 Accepted** | `{ operationId, statusUrl, estimatedCompletion? }` | `Location: /api/v1/operations/{operationId}` |
| `POST /resources/bulk-import` | Carga masiva | **200 OK** | `{ totalProcessed, successful, failed, errors: [{ row, field, message }] }` | — |
| `PUT /resources/{id}` | Reemplazo completo | **200 OK** | DTO Detail actualizado | `ETag` (si aplica §11.5) |
| `PATCH /resources/{id}` | Actualización parcial | **200 OK** | DTO Detail actualizado | `ETag` (si aplica §11.5) |
| `DELETE /resources/{id}` | Borrado físico (hard) | **204 No Content** | (sin body) | — |
| `DELETE /resources/{id}` | Inactivación lógica (soft) | **200 OK** | DTO Detail con el estado resultante | — |

#### 9.6.3 Reglas vinculantes

**MUST:**

1. **Devolver DTO Detail** en POST/PUT/PATCH. **PROHIBIDO** devolver sólo `{ id }` pelado — el frontend ya necesita mostrar el recurso y una respuesta con Detail elimina un GET subsecuente, reduce latencia y elimina una ruta de error.
2. **`Location` header obligatorio** en 201 (apunta al recurso) y en 202 (apunta al endpoint de tracking).
3. **204 sólo para DELETE hard** sin información útil de retorno. Ningún otro verbo devuelve 204 por defecto.
4. **Concurrencia optimista:** mutaciones sobre entidades con `RowVersion` (§11.5) exigen `If-Match: <ETag>`. Ausencia → **428 Precondition Required**. Mismatch → **412 Precondition Failed**. ETag se emite en la respuesta de GET, POST, PUT y PATCH.
5. **Idempotencia** (`Idempotency-Key`, §9.7) aplica a POST de creación y de acción crítica. Respuestas idempotentes replican status y body originales.
6. **Workflow actions** (`POST /procedures/{id}/approve`, `POST /wells/{id}/abandon`, `POST /forms/{id}/submit`) retornan **200 OK con el DTO Detail** del recurso afectado, no un `"ok"` ni un booleano. Los efectos colaterales (creación de `WellStateHistory`, notificaciones) se observan vía eventos de auditoría (§11.6) o endpoints específicos, no en este body.
7. **202 Accepted** incluye siempre `operationId` y `statusUrl` para polling. El endpoint `GET /operations/{operationId}` devuelve `{ status: 'Pending'|'InProgress'|'Completed'|'Failed', result?, error? }`.

**PROHIBIDO:**

- Devolver HTTP 200 con body `{ success: false, ... }` — un error disfrazado de éxito. Los errores son status ≥400.
- Incluir texto de UX en el body de éxito (`"message": "Pozo creado correctamente"`). Si existe necesidad real de comunicar algo contextual, va en un campo tipado del DTO (ej. `warnings: string[]`), nunca como mensaje de presentación.
- Mezclar formatos entre endpoints del mismo servicio. Todos los POST del servicio siguen esta tabla sin excepción.

#### 9.6.4 Helpers del Result Pattern

La shared library `Anh.Gop.Shared.ResultPattern` expone extensiones canónicas que cada controller usa para producir el status code correcto. Firmas de referencia:

```csharp
// POST crear
result.ToCreatedResult(controller: this, locationTemplate: "/api/v1/wells/{0}", dto => dto.Id);

// POST acción sincrónica / PUT / PATCH
result.ToOkResult();

// POST acción asíncrona
result.ToAcceptedResult(operationId, statusUrl);

// DELETE hard
result.ToNoContentResult();

// DELETE soft (devuelve Detail)
result.ToOkResult();
```

El desarrollador de feature **no** elige el status — la extensión lo determina por convención. Esto garantiza uniformidad sin disciplina manual por endpoint.

### 9.7 Idempotencia

**SHOULD:** endpoints POST críticos (crear pozo, radicar forma, aprobar, firmar) soportan header `Idempotency-Key`. El servicio mantiene una tabla de claves usadas por N horas y retorna la misma respuesta si la misma clave se repite.

### 9.8 Documentación

**MUST:** Swagger/OpenAPI en cada servicio, expuesto en `/swagger`. Incluye:
- Resúmenes XML de endpoints, parámetros y responses.
- Ejemplos de request y response.
- Códigos de error documentados.
- Esquemas de seguridad (Bearer Auth).

**MUST:** los **endpoints públicos de la API de negocio** (`/api/v{N}/...`) tienen documentación XML completa en producción (resumen, parámetros, responses, códigos de error esperados).

**SHOULD:** DTOs y propiedades con nombres autodocumentados tienen al menos un resumen corto; no se exige documentar cada propiedad trivial (`Id`, `Name`, `CreatedAt`).

Los endpoints de **infraestructura** (health checks, métricas Prometheus, `/swagger` mismo) están **exentos** de este requisito.

### 9.9 Convenciones adicionales de serialización

Reglas transversales que aplican a **todos** los payloads (request y response) de la API pública.

#### 9.9.1 Fechas, horas y zona horaria

**MUST:**
- Todas las fechas/horas se serializan en **ISO 8601** con offset explícito (`yyyy-MM-ddTHH:mm:ss.fffzzz`) o en UTC con sufijo `Z` (`yyyy-MM-ddTHH:mm:ss.fffZ`).
- El almacenamiento en BD es **UTC** (`datetime2(7)` + columna separada o convención `*_Utc`).
- La conversión a zona horaria del usuario (`America/Bogota`) es responsabilidad del **frontend**.
- Fechas sin hora (cumpleaños, fecha de vencimiento de forma) usan `yyyy-MM-dd` (tipo `DateOnly` en .NET 10).

**PROHIBIDO:** fechas en formato local, epoch como número, o strings ambiguos (`"21/04/2026"`).

#### 9.9.2 Serialización de enums

**MUST:** enums se serializan como **string** (nombre del valor), no como entero.

```csharp
// Program.cs (centralizado en Anh.Gop.Shared.Api)
builder.Services.ConfigureHttpJsonOptions(opt =>
{
    opt.SerializerOptions.Converters.Add(new JsonStringEnumConverter(
        namingPolicy: JsonNamingPolicy.CamelCase));
});
```

**Razón:** legibilidad en logs/debug, compatibilidad con evolución del enum (agregar valores no rompe clientes), alineación con OpenAPI.

**PROHIBIDO:** exponer enums como `int` en contratos públicos.

#### 9.9.3 Nomenclatura JSON

**MUST:** **camelCase** en todas las propiedades JSON (request y response). Es el default de `System.Text.Json` en .NET 10 y se mantiene explícito en `Anh.Gop.Shared.Api`:

```csharp
opt.SerializerOptions.PropertyNamingPolicy = JsonNamingPolicy.CamelCase;
opt.SerializerOptions.DictionaryKeyPolicy = JsonNamingPolicy.CamelCase;
```

**PROHIBIDO:** `snake_case`, `PascalCase`, o mezclas por endpoint.

#### 9.9.4 Null vs. omitido

**MUST:**
- En **responses**: propiedades con valor `null` se **omiten** del JSON (`DefaultIgnoreCondition = WhenWritingNull`), excepto cuando el `null` es semánticamente significativo (ej. "el campo fue limpiado"). En ese caso, documentar explícitamente en Swagger.
- En **requests (PATCH)**: un campo **omitido** significa "no cambiar"; un campo con valor `null` significa "limpiar a null". Se implementa con `JsonPatchDocument` o DTOs con `Optional<T>` cuando sea crítico.

#### 9.9.5 Uploads y downloads de archivos

**MUST — Uploads:**
- Contenido binario (PDFs, XLS, imágenes adjuntas a formas) usa `multipart/form-data`.
- Tamaño máximo por archivo: **50 MB** por defecto; configurable por servicio vía `appsettings` (`Uploads:MaxFileSizeBytes`).
- Validación **obligatoria** en el pipeline: MIME type real (magic bytes, no solo header), extensión permitida, antivirus scan (integración con servicio externo si Level ≥ 2).
- Los archivos se almacenan en **blob storage** (S3-compatible), **no** en la BD. En BD solo se guarda `BlobReference` (URI + hash SHA-256).

**MUST — Downloads:**
- Se sirven con `Content-Disposition: attachment; filename="..."` y `Content-Type` correcto.
- Archivos > 10 MB se transmiten con **streaming** (`FileStreamResult` o `PhysicalFileResult`), no cargados completos en memoria.
- Las URLs pre-firmadas (S3 presigned) son la opción preferida para downloads grandes — el cliente descarga directo del blob, no a través del servicio.

**PROHIBIDO:** enviar archivos como base64 embebido en JSON (salvo casos < 100 KB justificados y documentados).

#### 9.9.6 Exportes tabulares (CSV, Excel, PDF)

**MUST:**
- Endpoints de export se identifican por sufijo `/export` o query `?format=csv|xlsx|pdf`.
- **Content-Type correcto:**
  - CSV: `text/csv; charset=utf-8` (con BOM para compatibilidad Excel Windows).
  - Excel: `application/vnd.openxmlformats-officedocument.spreadsheetml.sheet` (.xlsx, vía `ClosedXML` o `EPPlus`).
  - PDF: `application/pdf` (vía `QuestPDF` — MIT/Community license).
- **Content-Disposition:** `attachment; filename="export_{resource}_{yyyyMMdd_HHmmss}.{ext}"`.
- Exports de > 10 000 filas: el endpoint debe ser **asíncrono** (devuelve `202 Accepted` + `operationId`; el usuario consulta el status y descarga cuando está listo).
- Exports nunca deben exceder límites de memoria — usar streaming para CSV y writers incrementales para Excel.

**PROHIBIDO:**
- Devolver exportes como JSON base64.
- Generar PDF/Excel con librerías de licencia comercial sin aprobación de arquitectura.

---

## 10. Patrones obligatorios

### 10.1 CQRS con dispatchers propios

**MUST:**
- Commands (mutaciones) separados de Queries (lecturas).
- Cada caso de uso implementa `ICommand<TResult>` o `IQuery<TResult>` (interfaces propias en `Anh.Gop.Shared.Cqrs`).
- Cada caso de uso tiene un handler que implementa `ICommandHandler<TCommand, TResult>` o `IQueryHandler<TQuery, TResult>`.
- Controllers **MUST** ser delgados: solo hacen `_commandDispatcher.DispatchAsync(command)` o `_queryDispatcher.DispatchAsync(query)`.
- Registro de handlers por **assembly scanning** vía método de extensión `AddGopCqrs(assembly)` sobre `IServiceCollection`.
- Cross-cutting concerns (validación, logging, autorización) se implementan como **decoradores** sobre los dispatchers.

**PROHIBIDO:**
- Lógica de dominio en controllers.
- Dependencias externas de mediador (MediatR, Mediator.SourceGenerator, etc.) en V1. Ver §10.1.1.

**Estructura por feature (sin cambios respecto al estándar previo):**

```
Application/Wells/
 Commands/
   CreateWell/
     CreateWellCommand.cs          // record : ICommand<Result<long>>
     CreateWellHandler.cs          // : ICommandHandler<CreateWellCommand, Result<long>>
     CreateWellValidator.cs        // : AbstractValidator<CreateWellCommand>
 Queries/
   GetWellById/
     GetWellByIdQuery.cs           // record : IQuery<Result<WellDetailDto>>
     GetWellByIdHandler.cs         // : IQueryHandler<GetWellByIdQuery, Result<WellDetailDto>>
     WellDetailDto.cs
```

El código de referencia de las interfaces, dispatchers y decoradores está en el apéndice §19.7.

### 10.1.1 Postura "lean base"

GOP 360° arranca usando únicamente las primitivas del .NET base para CQRS. Dependencias externas adicionales (mediador, decoradores avanzados, event bus) **se incorporan solo cuando un escenario concreto demuestre que el trabajo de implementarlo manualmente supera el costo de adoptar la librería**.

**Lo que usamos desde día 1:**

- `Microsoft.Extensions.DependencyInjection` (parte de .NET, licencia MIT).
- Interfaces propias `ICommand<T>`, `IQuery<T>`, `ICommandHandler<C,R>`, `IQueryHandler<Q,R>` en `Anh.Gop.Shared.Cqrs`.
- Dispatchers propios `ICommandDispatcher`, `IQueryDispatcher` en la misma shared library.
- Método de extensión `AddGopCqrs(assembly)` para registro automático.
- `FluentValidation` (Apache 2.0) para validadores.
- Decoradores manuales para pipeline (validación, logging, autorización).

**Lo que NO usamos en V1 (y lo que haría falta para incorporarlos):**

| Dependencia | Cuándo reconsiderar su adopción |
|---|---|
| Mediador externo (MediatR, etc.) | Cuando el número de cross-cutting concerns en el pipeline supere 5–6 y la gestión manual de decoradores se vuelva tediosa. |
| Pipeline condicional declarativo | Cuando aparezca necesidad de aplicar behaviors solo a ciertos commands/queries vía atributos. |
| Registro por source generator | Cuando el volumen de handlers (>200) haga que el registro manual o por scan cause problema de performance en arranque. |
| Publish-subscribe intra-servicio | Cuando aparezca necesidad de notifications con múltiples suscriptores dentro del mismo servicio. |
| `Scrutor` (helper de decoradores DI) | Cuando haya 3+ decoradores y la ergonomía manual sea fricción recurrente. MIT, aceptable como única dependencia de "comodidad DI". |

**Regla de decisión:** si ninguno de estos escenarios aparece en V1, nos quedamos con .NET base. Si aparece uno, se evalúa la librería correspondiente en ADR antes de adoptarla. La decisión por defecto es **no agregar**.

Esta postura aplica no solo al mediador — es el principio general que gobierna la expansión del stack de GOP 360°.

### 10.2 Result Pattern

**MUST:** los handlers retornan `Result<T>`, no lanzan excepciones para errores esperados.

```csharp
public async Task<Result<WellDetailDto>> Handle(GetWellByIdQuery query, CancellationToken ct)
{
    var well = await _repo.GetByIdAsync(query.WellId, ct);
    if (well is null) return Result.NotFound("Well not found");

    // authorize, map, return
    return Result.Success(_mapper.Map<WellDetailDto>(well));
}
```

**Razones:**
- Excepciones son caras.
- `Result<T>` hace explícito el contrato del handler.
- El controller puede mapear `Result` a códigos HTTP centralizadamente.

**PROHIBIDO** `throw new Exception` para errores de validación, autorización, o reglas de negocio.

**MAY** lanzar excepciones para errores técnicos irrecuperables (base de datos caída, configuración inválida).

### 10.3 Repository Pattern

**MUST:** cada agregado tiene su repositorio en `Infrastructure`. La interfaz vive en `Domain` o `Application`.

```csharp
// Domain
public interface IWellRepository
{
    Task<Well?> GetByIdAsync(WellId id, CancellationToken ct);
    Task<Well?> GetByUwiAsync(FiscalizedUwi uwi, CancellationToken ct);
    Task AddAsync(Well well, CancellationToken ct);
    // NO tiene Save — eso es responsabilidad del UnitOfWork
}
```

**PROHIBIDO:**
- `DbContext` inyectado en controllers o handlers directamente.
- Queries arbitrarias via LINQ en handlers — **MUST** estar encapsuladas en el repositorio o en queries dedicadas (Dapper/EF proyection).

### 10.4 Unit of Work

**MUST:** commands que modifican múltiples agregados se coordinan vía `IUnitOfWork`:

```csharp
await _wellRepo.UpdateAsync(well, ct);
await _procedureRepo.UpdateAsync(procedure, ct);
await _unitOfWork.SaveChangesAsync(ct);
```

### 10.5 Domain Events

**MUST:** cambios de estado relevantes del dominio emiten eventos:

```csharp
public class Well : AggregateRoot
{
    public void Approve(ApprovalContext context)
    {
        // ... cambia estado ...
        RaiseDomainEvent(new WellApprovedEvent(this.Id, DateTime.UtcNow));
    }
}
```

#### 10.5.1 Alcance en V1 — estrictamente intra-servicio

**MUST:** en V1 los domain events son **exclusivamente intra-servicio**:

- Se procesan síncronamente en el mismo `SaveChangesAsync` vía EF Core interceptor.
- **NUNCA** cruzan el límite de un microservicio por sí mismos (no hay event bus).
- El handler del evento vive en el mismo assembly/proceso que lo emite.

**PROHIBIDO** en V1:
- Publicar domain events a infraestructura compartida (colas, tópicos) esperando que otros servicios los consuman.
- Diseñar lógica inter-servicio asumiendo propagación asíncrona — no existe en V1.

#### 10.5.2 Comunicación inter-servicio en V1

Cuando un servicio A necesita informar o consultar a un servicio B, **MUST** usarse **HTTP síncrono** con:

- Cliente tipado vía `HttpClientFactory`.
- **Circuit breaker** (Polly) con umbrales documentados.
- **Retry** con backoff exponencial y límite explícito.
- **Timeout** explícito por operación.
- **Fallback** definido: qué pasa si el otro servicio no responde (regla por endpoint).
- Registro en `audit.IntegrationLog`.

**Escenarios conocidos en V1 que requieren comunicación inter-servicio:**

| Origen | Destino | Operación | Criticidad | Fallback |
|---|---|---|---|---|
| `gop.procedures` | `gop.operations` | Notificar aprobación de trámite Serie 100 para que aplique transición de estado al `Well` | Alta | Reintento con backoff; alerta operativa si persiste |
| `gop.operations` | `gop.core` | Consultar datos de `Operator`, `Contract`, `Field` para enriquecer vistas/validaciones | Media | Cache local TTL 10 min; 503 si cache vacío |
| `gop.production` | `gop.core` | Idem para contratos y campos | Media | Idem |
| `gop.procedures` | `gop.production` | Disparar creación/actualización de `ProducingFormation` al aprobar F103 | Alta | Reintento; compensación manual si falla |
| Cualquier servicio | `gop.identity` | Resolver `assigned_contract_ids` cuando `contracts_truncated = true`, revocation list en ops críticas, validar matrícula | Alta | Denegar operación crítica si Identity no responde |

> **Excepción — `gop.audit`:** no aparece como destino en la tabla anterior de forma intencional. `gop.audit` **no recibe escrituras inter-servicio**. Cada microservicio escribe su propia auditoría localmente a `audit.AuditLog` vía Audit.NET (ver §11.6). `gop.audit` solo **consume** esas tablas para consulta, búsqueda agregada, exportación y retención. Invocar a `gop.audit` para escribir está prohibido (ver §18.1).

**SHOULD:** minimizar estos caminos. Si una operación puede resolverse con datos cacheados o con lo que ya viene en el JWT, evitar la llamada HTTP.

#### 10.5.3 Evolución a V2

- **V2+:** se introduce event bus (RabbitMQ, Azure Service Bus o equivalente). Los domain events se promueven a **integration events** publicables entre servicios.
- La migración **SHOULD** ser gradual: los escenarios críticos se mueven primero, y el HTTP síncrono se preserva como fallback durante al menos un release.
- Cuando se introduzca event bus, los integration events **MUST** versionarse independientemente del schema de dominio (son contrato público entre servicios). Esta regla se activa al iniciar V2 y se desarrollará en una revisión futura del documento.

### 10.6 Validación de DTOs

**MUST:** todo DTO de command o query tiene su `AbstractValidator` en FluentValidation. Se registran automáticamente en el pipeline del dispatcher vía el decorador `ValidationBehavior` (ver §19.7).

```csharp
public class CreateWellValidator : AbstractValidator<CreateWellCommand>
{
    public CreateWellValidator()
    {
        RuleFor(x => x.Name).NotEmpty().MaximumLength(500);
        RuleFor(x => x.OperatorId).GreaterThan(0);
        // ...
    }
}
```

### 10.7 Specification Pattern

**MAY** usarse para queries complejas reutilizables. **SHOULD** usarse cuando la misma consulta aparece en 3+ lugares.

### 10.8 Desacoplamiento de reglas de formularios y lógica de negocio

#### 10.8.1 Propósito

Esta sección establece un enfoque base y pragmático para mantener las **reglas de formularios** y la **lógica de negocio** desacopladas del código transaccional (controllers, handlers de POST/PUT, servicios). El objetivo es que el equipo tenga un lugar claro donde colocar este tipo de reglas cuando sea razonable, sin forzar el patrón en escenarios donde genere más complejidad de la que evita.

**No es camisa de fuerza.** Este enfoque aplica a la mayoría de casos simples y medios. Casos complejos **MAY** quedar hardcodeados en código tipado cuando el intento de parametrizarlos genere más daño que beneficio, siempre documentando la decisión.

#### 10.8.2 Dos principios rectores

**Principio 1 — Reglas de formulario servidas desde el back, evaluadas en memoria por el front.**

Las reglas de comportamiento de formularios (campos condicionales, valores por defecto, habilitación/deshabilitación, filtros de catálogos, validaciones simples) **SHOULD** definirse en el backend y exponerse al frontend en un único request al cargar el formulario. El frontend las evalúa en memoria mientras el usuario diligencia, sin enviar peticiones al back por cada cambio de campo.

Esto evita que la lógica se duplique entre back y front, reduce chatter de red, y mejora la experiencia del usuario.

**Principio 2 — Reglas de negocio desacopladas del código transaccional del backend.**

La lógica de negocio consumida por endpoints POST, PUT, PATCH **SHOULD** mantenerse desacoplada de los handlers. Los handlers coordinan; las reglas viven en archivos o componentes dedicados dentro de la misma estructura del código fuente. Esto permite: versionamiento claro en Git, tests enfocados sobre las reglas, y evolución independiente del código transaccional.

#### 10.8.3 Ubicación dentro del código fuente

Cada microservicio organiza las reglas dentro de su proyecto `Application`, en una carpeta dedicada:

```
/src/Anh.Gop.{Service}.Application/
  /Rules/
    /forms/             ← reglas de formularios (consumidas por back y front)
    /business/          ← reglas de lógica de negocio transaccional
    /schemas/           ← JSON schemas para validar los archivos anteriores
```

**MUST:** las reglas viven en el repositorio del servicio (versionadas en Git), no en base de datos, salvo que un administrador no-técnico deba gestionarlas vía UI.

#### 10.8.4 Formato preferido

- **SHOULD:** usar archivos **JSON** como formato canónico para reglas declarativas. Razones: legibilidad, versionado limpio con diffs útiles, soporte nativo de herramientas, compatibilidad con el frontend.
- **MAY:** usar YAML si un caso particular lo justifica por legibilidad superior.
- **MAY:** usar clases C# planas (sin lógica, solo datos) cuando la regla sea específica de un único handler y no amerite archivo externo.

#### 10.8.5 Exposición al frontend

Cada formulario que siga el Principio 1 **SHOULD** exponer un endpoint estándar:

```
GET /api/v1/forms/{form-key}/rules
```

Que retorna las reglas del formulario en un formato consumible directamente por el frontend. El frontend las usa para renderizar condicionalmente sin más viajes al back.

**MUST:** el backend **también** aplica estas mismas reglas al validar el payload recibido en el POST/PUT correspondiente. No se confía únicamente en validación del cliente.

#### 10.8.6 Qué va y qué no va en los archivos de reglas

**Va bien en archivos de reglas:**
- Campos obligatorios condicionales (ej. "si pozo es offshore, lámina de agua es obligatoria").
- Valores por defecto según otro campo (ej. "si Lahee A3, el campo se asigna a exploratorio").
- Filtros de catálogos según contexto (ej. "mostrar solo contratos del operador actual").
- Plazos, umbrales, rangos numéricos.
- Habilitación/deshabilitación de campos según estado o rol.
- Listas de anexos requeridos por forma.

**No va en archivos de reglas (queda en código tipado):**
- Cálculos complejos (construcción del UWI, transformación de coordenadas).
- Reglas que requieren consultas a BD o servicios externos.
- Validaciones cruzadas con lógica condicional profunda (más de 2-3 niveles).
- Expresiones lógicas arbitrarias evaluadas dinámicamente.
- Invariantes de dominio (esas viven en el agregado).

**Regla heurística:** si la regla se puede expresar como "datos más referencias a códigos conocidos", va en archivo. Si requiere escribir una expresión para evaluar, va en código.

#### 10.8.7 Referencias a código desde los archivos

Cuando una regla necesita evaluar algo que no es un dato puro (por ejemplo, "aplicar el placeholder de campo exploratorio"), el archivo JSON **MUST** referenciar un **código simbólico** que tenga implementación C# tipada correspondiente.

**Ejemplo válido en JSON:**
```json
{ "defaultValueFrom": "EXPLORATORY_FIELD_PLACEHOLDER" }
```

**Con implementación en código:**
```csharp
public class ExploratoryFieldResolver : IPlaceholderResolver { ... }
```

**PROHIBIDO** expresiones evaluables dinámicamente en los archivos (ej. `"expression": "fieldA > 5 && fieldB == 'X'"`). Si se necesita lógica, se implementa como clase C# y se referencia por código simbólico.

#### 10.8.8 Validación al arranque

**MUST:** los archivos de reglas se cargan y validan al arrancar el servicio contra su JSON schema. Si un archivo es inválido, o si referencia un código simbólico sin implementación, el servicio **no arranca**. Los errores de configuración deben manifestarse inmediatamente, no en runtime impredecible.

#### 10.8.9 Relación con arquitectura Clean / hexagonal

Este patrón es compatible y refuerza el aislamiento del dominio:

- Los archivos JSON son **recursos de proyecto**.
- El motor que los carga (`Anh.Gop.Shared.RulesEngine`) vive en **Infrastructure**.
- Los tipos que representan las reglas parseadas viven en **Domain** o en la shared library.
- El código de dominio consume las reglas a través de un **puerto** (`IRulesProvider`), nunca leyendo archivos directamente.

**PROHIBIDO** leer archivos JSON, deserializar, o tocar el sistema de archivos desde código en Domain o desde handlers de Application.

#### 10.8.10 Cuándo NO aplicar este patrón

Este enfoque es ayuda, no obligación. Se **MAY** dejar una regla hardcodeada en código cuando:

- La regla es **invariante de dominio** (debe vivir en la entidad).
- La regla es **normativa estable con base legal citable** (queda en código con comentario de norma).
- La regla es tan específica a un caso de uso único que externalizarla solo agrega indirección.
- La regla tiene complejidad tal que expresarla declarativamente requeriría un mini-lenguaje propio.

En esos casos, documentar brevemente la decisión (en código o ADR corto) y seguir. La sección siguiente (§10.8.10.1) define **cómo** dejar esa regla hardcodeada de forma mantenible — el objetivo es que una regla en código no se convierta en deuda técnica ni en "magia escondida" dentro de un handler o componente.

#### 10.8.10.1 Cómo mantener una regla hardcodeada de forma sostenible

Cuando se decide dejar una regla en código (back o front) por complejidad o por alguno de los criterios anteriores, **SHOULD** aplicarse las siguientes prácticas para que la regla siga siendo localizable, testeable y auditable:

**1. Ubicación dedicada, nunca inline.**
La regla vive en una **clase o función con nombre propio** que refleje la regla de negocio, no dentro del handler, del controller o de un componente UI.

- Back: `Application/Rules/Business/{ContextName}/{RuleName}.cs` — aunque la regla no sea declarativa, ocupa el mismo árbol de carpetas que las reglas externalizadas (`Application/Rules/` de §10.8.3). Así, quien busque "reglas de negocio del servicio" encuentra todo en un solo lugar.
- Front: `features/{module}/business-rules/{rule-name}.ts` — clase o función pura, importada por los componentes.

Ejemplos de naming: `WellAbandonmentApprovalRule`, `F103ProducingFormationDerivationRule`, `ContractTransferEligibilityRule`. **Nombres prohibidos:** `Helper`, `Utils`, `Manager`, `Service` a secas.

**2. Interfaz o contrato tipado.**
La regla se expone detrás de una interfaz (`IAbandonmentApprovalRule`) o tipo de función (`type Evaluate = (input: FormData) => RuleResult`). Permite:
- Inyectarla en handlers (testing con double).
- Reemplazarla por una versión declarativa en el futuro sin tocar los consumidores (patrón Strategy).
- Documentar el contrato (input → output) sin leer la implementación.

**3. Test de comportamiento por cada rama significativa.**
Una regla hardcodeada **MUST** tener su test dedicado cubriendo cada rama de decisión relevante (no solo el happy path). Estructura Given-When-Then. Si la regla se reescribe a formato declarativo más adelante, los tests sirven como oráculo de equivalencia.

**4. Encabezado de procedencia.**
En el archivo de la regla, un bloque de comentario fijo:

```csharp
// Regla de negocio: Aprobación de abandono de pozo (F107).
// Base: Resolución ANH 40048 de 2015, art. 12 numerales 2–5.
// Contraparte FE: apps/gop-web/src/app/features/wells/business-rules/well-abandonment-approval.rule.ts
// Autor inicial: <equipo>  — Fecha: 2026-04
// Motivo de hardcodeo: la evaluación combina estado del pozo, tipo de formación y
// antigüedad del operador con ramas anidadas que un formato declarativo oscurecería.
// Revisión sugerida: cuando la resolución se actualice o si emergen >2 reglas similares.
```

El encabezado permite al próximo desarrollador entender en 30 segundos por qué la regla vive en código y qué hay que revisar si la normativa cambia.

**5. Pareja BE ↔ FE declarada explícitamente.**
Si la misma regla se evalúa en back y en front (caso típico de validación preventiva en la UI), **MUST** cumplirse:

- El back es la fuente de verdad (línea roja §10.8.11 — no duplicar lógica). El FE implementa una versión **equivalente** para mejorar UX, nunca para reemplazar la validación del back.
- Ambos archivos se referencian mutuamente en el encabezado (como en el ejemplo arriba).
- Si la regla declarativa (§10.8.2) aplica parcialmente, el FE **SHOULD** apoyarse en las porciones declarativas y dejar hardcodeado solo lo realmente complejo, documentando la delimitación.
- Cada cambio en una rama implica PR coordinado actualizando ambas implementaciones y sus tests. CI **SHOULD** tener un check (p. ej. test de contrato compartido) que falle si una versión evoluciona sin la otra.

**6. Registro en el catálogo interno del servicio.**
Cada microservicio **SHOULD** mantener un archivo `Application/Rules/BUSINESS-RULES.md` (o equivalente en el FE) que liste:

| Código | Nombre | Ubicación (BE) | Ubicación (FE) | Base normativa | Última revisión |
|---|---|---|---|---|---|

Funciona como índice. Sin este registro, las reglas hardcodeadas se vuelven invisibles — nadie recuerda que existen hasta que se rompen.

**7. Revisión periódica.**
En cada refinamiento de arquitectura (o al menos una vez por release mayor), el equipo revisa el catálogo de reglas hardcodeadas y evalúa si alguna cumple ya las condiciones para externalizarse al formato declarativo (§10.8.2). Las reglas hardcodeadas son **aceptables pero no permanentes por diseño** — se quedan mientras la complejidad lo justifique.

**Resumen:** una regla hardcodeada no deja de ser una regla de negocio. Aplican los mismos principios que a una regla declarativa — nombre propio, test, documentación, referencia cruzada BE/FE y trazabilidad normativa — solo que la **forma** de expresarla es código en lugar de JSON.

#### 10.8.11 Líneas rojas

- **NUNCA** construir un motor de reglas Turing-completo con evaluación dinámica de expresiones desde archivos o BD.
- **NUNCA** cargar reglas desde archivos sin validar schema al arranque.
- **NUNCA** duplicar lógica de validación entre back y front. La fuente de verdad es el back; el front consume.
- **NUNCA** leer archivos de reglas directamente desde código de Domain o desde handlers. Siempre vía puerto inyectado.

---

## 11. Acceso a datos y persistencia

### 11.1 EF Core y proyecciones

**MUST:**
- Queries de lectura usan `.Select()` explícito hacia el DTO, sin materializar la entidad completa.
- `AsNoTracking()` obligatorio en queries de lectura.

```csharp
var items = await _ctx.Wells
    .AsNoTracking()
    .Where(w => w.OperatorId == currentTenant)
    .Select(w => new WellSummaryDto {
        WellId = w.Id,
        Name = w.Name,
        // ... solo lo necesario
    })
    .ToPagedListAsync(query);
```

**PROHIBIDO** `_ctx.Wells.ToList()` seguido de `.Select(...)` en memoria — trae todo, desperdicia SQL.

### 11.2 SQL crudo

**MAY** usarse Dapper o SQL crudo para:
- Queries geoespaciales complejas.
- Agregaciones pesadas (dashboards).
- Reportes de exportación.

**MUST:** SQL crudo vive en archivos `.sql` versionados en `Infrastructure/Sql/`, no como strings inline.

**PROHIBIDO** concatenación de strings SQL con inputs del usuario — solo parámetros. Siempre.

### 11.3 Migraciones

**MUST:** DbUp sobre archivos SQL numerados y versionados en Git:

```
Infrastructure/Migrations/
  0001_CreateWellTable.sql
  0002_AddUwiColumn.sql
  ...
```

**PROHIBIDO:**
- Editar una migración ya aplicada.
- Ejecutar SQL manual en ambientes superiores a desarrollo local.
- Deploys que omitan correr migraciones.

Cada migración se nombra con prefijo numérico secuencial y descripción breve en inglés.

### 11.4 Transacciones

**MUST:** commands que modifican múltiples agregados usan transacción explícita (via UnitOfWork o `IDbContextTransaction`).

**SHOULD:** transacciones cortas. Como referencia operativa, una transacción que permanezca abierta >5 segundos en producción es sospechosa y se monitorea con alerta (ver §16). La regla se hace cumplir por observabilidad y revisión de performance, no por bloqueo de PR — hay casos legítimos de batches o migraciones que la exceden puntualmente.

### 11.5 Concurrencia

**MUST:** entidades con alto riesgo de concurrencia (ej. `Procedure`, `Well`) tienen columna `RowVersion` / `Timestamp` para concurrency optimista.

### 11.6 Auditoría — escritura automática con Audit.NET

#### 11.6.1 Responsabilidad local, no centralizada

**MUST:** la **escritura** de registros de auditoría es responsabilidad de **cada microservicio** sobre su propio esquema `audit.*`. **PROHIBIDO** invocar al microservicio `gop.audit` para escribir registros de auditoría — eso introduce acoplamiento síncrono, latencia y un punto único de falla en el camino crítico de cada mutación.

El microservicio `gop.audit` existe únicamente para **consulta, búsqueda, exportación, retención y consolidación** de datos de auditoría. Es un "lector" especializado, no un "escritor" centralizado.

#### 11.6.2 Herramienta estándar

**MUST:** usar **Audit.NET** con el proveedor **Audit.EntityFramework.Core** para capturar mutaciones sobre entidades de dominio. Esta combinación:

- Intercepta `SaveChangesAsync` del `DbContext` automáticamente.
- Captura valores previos y nuevos por entidad y por columna.
- Permite enriquecer el registro con `UserId`, `TenantId`, `CorrelationId` vía `AuditScope.CurrentCustomFields`.
- Soporta providers personalizados para escribir a SQL Server, archivo, Elasticsearch, etc.
- Es open-source, licencia MIT, mantenido activamente.

**PROHIBIDO** escribir interceptores EF Core manuales para auditoría cuando Audit.NET cubre el caso. Si existe una necesidad que Audit.NET no resuelve, documentarla en ADR antes de salir del estándar.

#### 11.6.3 Configuración de referencia

La configuración base vive en `Anh.Gop.Shared.Infrastructure` y se consume en cada servicio en el arranque:

```csharp
// Anh.Gop.Shared.Infrastructure/AuditConfiguration.cs
public static class AuditConfiguration
{
    public static void ConfigureAudit(string connectionString)
    {
        Audit.Core.Configuration.Setup()
            .UseSqlServer(config => config
                .ConnectionString(connectionString)
                .Schema("audit")
                .TableName("AuditLog")
                .IdColumnName("AuditLogId")
                .JsonColumnName("AuditData")
                .LastUpdatedColumnName("LastUpdatedAt"));

        Audit.Core.Configuration.AddCustomAction(ActionType.OnScopeCreated, scope =>
        {
            var httpContext = ServiceLocator.GetService<IHttpContextAccessor>()?.HttpContext;
            var currentUser = ServiceLocator.GetService<ICurrentUserContext>();
            scope.SetCustomField("UserId", currentUser?.UserId);
            scope.SetCustomField("TenantId", currentUser?.TenantId);
            scope.SetCustomField("CorrelationId", httpContext?.TraceIdentifier);
        });
    }
}

// En cada servicio:
// program.cs → AuditConfiguration.ConfigureAudit(connectionString);
// DbContext → hereda de Audit.EntityFramework.AuditDbContext o se marca con [AuditInclude]
```

#### 11.6.4 Campos obligatorios del registro

Cada registro escrito por Audit.NET **MUST** incluir como mínimo:

- `EventType` (Create / Update / Delete).
- `AggregateType`, `AggregateId`.
- `UserId`, `TenantId`, `CorrelationId`, `OccurredAt`.
- `PreviousValueJson`, `NewValueJson` (solo campos modificados para `Update`).
- `MachineName`, `ServiceName` (qué microservicio emitió el registro).

#### 11.6.5 Entidades excluidas

**MUST NOT** auditarse (ruido / datos sensibles):
- Tablas de cache o staging en `integration`.
- Tabla de sesión `UserSession` (usar logs específicos de autenticación).
- Catálogos cuya mutación ya se audita a nivel administrativo.

La exclusión se declara con `[AuditIgnore]` en la entidad o via configuración global.

#### 11.6.6 Contrato de esquema y topología de BD en V1

**Topología de BD en V1:** los microservicios **comparten instancia física** de SQL Server 2022 pero cada uno opera sobre su **esquema dedicado** (`core`, `ops`, `prod`, `procedure`, `workflow`, `identity`, `integration`). El aislamiento es **lógico, no físico**, impuesto por permisos de base de datos:

- Cada servicio accede con un login/usuario propio.
- El usuario tiene `GRANT` completo únicamente sobre su esquema de dominio.
- Todos los usuarios de servicios tienen `INSERT` (y no más) sobre `audit.AuditLog` e `audit.IntegrationLog`.
- El usuario de `gop.audit` tiene `SELECT` sobre `audit.*` y sobre ninguna otra tabla de negocio.

Esta decisión es **pragmática para V1** on-premise: reduce operación (una sola instancia a respaldar, monitorear, parchear) sin sacrificar el contrato de aislamiento de dominios. **V2+ evaluará separación física** (instancia o BD por servicio) cuando el volumen o los requisitos de compliance lo justifiquen — registrado como ADR pendiente.

**Contrato de esquema `audit.*`:** es compartido por todos los microservicios escritores:

- Su definición vive en el repo de `gop.audit` y se expone como **migración DbUp compartida** (paquete NuGet interno versionado que todos los servicios consumen en bootstrap).
- Cambios al esquema `audit.*` son **breaking changes** coordinados entre todos los servicios (ADR + release tren).
- `gop.audit` es el único servicio que **crea y evoluciona** las tablas; los demás solo insertan con el permiso acotado descrito arriba.

---

## 12. Validación, sanitización y hardening de seguridad

Esta sección consolida los controles transversales de seguridad aplicables a **todos** los microservicios. El énfasis es **hardening automático** — los controles que OWASP Top 10 exige se materializan en middleware, analyzers y configuración declarativa, no en código repetido por feature.

### 12.0 Principio rector — "seguro por defecto, invisible al desarrollador"

**MUST:** todo control de OWASP Top 10 que sea automatizable se implementa en la shared library **`Anh.Gop.Shared.Security`** y se activa con una sola llamada (`AddGopSecurityHardening()` / `UseGopSecurityHardening()`) en el `Program.cs` de cada microservicio.

El desarrollador de feature **no** toma decisiones de seguridad transversales. Sólo:

1. Declara la sensibilidad del dato vía `[DataClassification(Level = ...)]` (ver §7.10).
2. Declara las policies de autorización por endpoint (ver §7.5–§7.7).
3. Usa los clientes HTTP, serializers y helpers provistos por la shared lib.

Los controles residuales que sí requieren decisión por spec (qué endpoints exigen MFA, qué recursos requieren rate-limit estricto, clasificación específica de entidades) se documentan en cada **spec funcional**, no en esta constitución.

### 12.1 Validación y sanitización de entrada *(A03)*

**MUST:**
- Todo input externo pasa por **FluentValidation**.
- Strings se validan en longitud máxima y patrón cuando aplique.
- IDs se validan positivos y existentes cuando relevante.
- Fechas se validan en rango razonable.
- La deserialización rechaza campos no mapeados (ver §12.9).

**PROHIBIDO:**
- Trust en inputs del cliente para lógica de autorización (ej. "el cliente me dice que es admin, le creo").
- Deserialización de tipos polimórficos sin `[JsonDerivedType]` / `KnownTypes` explícito.
- Concatenación de inputs en filtros LDAP (ver §12.11).

### 12.2 Security Headers por defecto *(A05 — Security Misconfiguration)*

**MUST:** el middleware `UseGopSecurityHardening()` fija en **toda** respuesta HTTP:

| Header | Valor | Razón |
|---|---|---|
| `Strict-Transport-Security` | `max-age=31536000; includeSubDomains` | Fuerza HTTPS en clientes |
| `X-Content-Type-Options` | `nosniff` | Previene MIME sniffing |
| `X-Frame-Options` | `DENY` | Anti clickjacking |
| `Referrer-Policy` | `strict-origin-when-cross-origin` | Limita leak de URLs |
| `Permissions-Policy` | `camera=(), microphone=(), geolocation=()` restrictivo | Desactiva APIs no usadas |
| `Content-Security-Policy` | Sólo en endpoints que sirven HTML (Swagger UI): `default-src 'self'; script-src 'self'` | Anti XSS en UI administrativa |

**MUST:** eliminar los headers `Server` y `X-Powered-By` vía configuración de Kestrel (`AddServerHeader = false`).

**Costo dev:** cero. Aplicado globalmente por la shared library.

### 12.3 HTTPS y cookies seguras *(A02, A05)*

**MUST:**
- `app.UseHttpsRedirection()` + `app.UseHsts()` habilitados en todos los servicios (no-Development).
- Toda cookie emitida (Antiforgery, sesiones administrativas de Swagger): `HttpOnly=true`, `Secure=true`, `SameSite=Strict`.
- `CookiePolicyOptions` configurado con defaults seguros por la shared library.

### 12.4 Antiforgery / CSRF *(A01, A03)*

**MUST:**
- Endpoints de **mutación** invocables desde navegador con cookie llevan filtro `[ValidateAntiForgeryToken]` (aplicado globalmente por la shared lib).
- APIs puras **JWT-Bearer** (sin cookie) están exentas de antiforgery — el modelo de token en header elimina la superficie CSRF. La shared lib distingue ambos modos vía convención de esquema de autenticación y documenta la exención.

### 12.5 Rate limiting y protección anti-abuso *(A04, A07)*

**MUST:** todo microservicio aplica rate limiting usando `Microsoft.AspNetCore.RateLimiting` (built-in .NET 10) con el conjunto de policies predefinidas en `Anh.Gop.Shared.Security`. No se admiten endpoints públicos sin policy asignada.

**SHOULD:** los valores iniciales del proyecto (ajustables por configuración en cada entorno, sin ADR):

| Policy | Límite de referencia | Aplica a |
|---|---|---|
| `auth-strict` | 5 intentos/min por IP + usuario | `POST /auth/login`, refresh de token, reset de password |
| `api-standard` | 60 req/min por usuario autenticado | Endpoints de mutación por defecto |
| `export-heavy` | 5 exports/hora por usuario | Endpoints de exportación masiva |
| `read-standard` | 300 req/min por usuario | Endpoints de consulta |

Los umbrales son guías iniciales; pueden endurecerse o relajarse por entorno a partir de métricas reales, documentando el ajuste en la configuración del servicio. Lo que no se admite es **desactivar** la policy sin ADR.

nginx mantiene rate limit por **IP** (defensa de borde); la app mantiene rate limit por **identidad** (JWT `sub`). Ambos son complementarios.

### 12.6 Políticas de password, lockout y MFA *(A07)*

Configuradas una sola vez en `gop.identity` vía `IdentityOptions`. Los **principios** son MUST; los **valores concretos** son SHOULD y viven en la política operativa del servicio (configurables por entorno), conforme a la normativa ANH vigente.

**MUST (principios):**
- Existe política de password con mínimo de longitud y complejidad (mayúscula, minúscula, dígito, símbolo).
- Existe historial de passwords para evitar reutilización inmediata.
- Existe mecanismo de lockout por intentos fallidos con contador dual (usuario + IP).
- Refresh token rotation con detección de reuso → invalidación de familia.
- MFA obligatorio para roles `ANH_Admin`, `ANH_Fiscalizador`, `Operator_Admin` y cualquier rol con capabilities de firma o aprobación. Shared policy `[RequireMfa]` se aplica por atributo en endpoints sensibles (firmas, aprobaciones de abandono, cambios de rol).

**SHOULD (valores iniciales del proyecto, revisables por política operativa):**

| Aspecto | Valor inicial |
|---|---|
| Longitud mínima de password | 12 caracteres |
| Historial | 5 últimas passwords |
| Expiración | 90 días (con notificación a los 14 y 7 días previos) |
| Lockout | 5 intentos fallidos → bloqueo 15 min |
| Idle timeout de sesión | 30 min |
| Absolute timeout de sesión | 8 h |

> Estos valores se alinean con la política vigente de ANH. Si la ANH adopta NIST SP 800-63B u otra guía (p. ej. sin expiración forzosa de password), la política se actualiza sin necesidad de cambiar este documento.

### 12.7 Gestión de secretos y configuración *(A02, A05)*

**PROHIBIDO:**
- Secretos en `appsettings*.json`, repositorio git o logs.
- Connection strings con `Password=...` en texto plano en config versionada.

**Fuentes aceptadas (orden de preferencia):**

1. Variables de entorno inyectadas por el orquestador.
2. **Azure Key Vault** o **HashiCorp Vault** (ADR V2).
3. DPAPI-protected config para on-premise V1 (transitorio, documentado en ADR).

**MUST:**
- `AddUserSecrets()` usado **sólo** en entorno `Development`.
- La shared library provee un `StartupSecretsValidator` que **falla el arranque** del servicio si detecta valores placeholder (`"CHANGEME"`, `"secret"`, `"password"`, connection strings con `Password=` literal). Esto previene despliegues con config por defecto.

### 12.8 Criptografía *(A02 — Cryptographic Failures)*

**MUST:**
- Hashing de passwords: ASP.NET Core Identity default (PBKDF2 HMAC-SHA512, ≥100k iteraciones).
- Datos marcados `[DataClassification(Level ≥ 3)]` se persisten con **Always Encrypted** de SQL Server 2022 (ver §7.10 y `data-model.md` §10).
- Firma de JWT: **RS256** con rotación anual de keys; keys privadas en Key Vault (nunca en repo).
- TLS 1.2 mínimo en todos los endpoints; TLS 1.3 preferido.

**PROHIBIDO** (bloqueado por analyzers CA5350/CA5351 como warning-as-error):
- MD5, SHA-1, DES, 3DES, RC4.
- Modo ECB para cifrado simétrico.
- Random no criptográfico (`System.Random`) para tokens, salts o ids de sesión — usar `RandomNumberGenerator`.

### 12.9 Deserialización segura *(A08 — Software and Data Integrity Failures)*

**MUST:**
- Serializer único: **`System.Text.Json`**.
- `JsonSerializerOptions` globales en shared lib:
  - `PropertyNameCaseInsensitive = false`.
  - `UnmappedMemberHandling = Disallow` — rechaza JSON con campos no declarados en el DTO (previene mass-assignment y campos fantasma).
  - `MaxDepth = 32` — previene stack overflow por payloads anidados.
- Polimorfismo sólo con `[JsonDerivedType]` explícito por tipo permitido.
- XML: si una integración externa lo impone, `XmlReaderSettings` MUST tener `DtdProcessing = Prohibit` y `XmlResolver = null` (previene XXE).

**PROHIBIDO:**
- Introducir `Newtonsoft.Json` como serializer primario de la aplicación (motivo: superficie histórica de vulnerabilidades en deserialización polimórfica con `TypeNameHandling`). Las dependencias transitivas que lo arrastren se toleran siempre que no se use directamente desde código propio; si un SDK de terceros lo exige para interoperar con el sistema externo, se documenta la excepción en el PR. Ver también §18.3.
- `BinaryFormatter` (deprecado y removido por Microsoft, superficie RCE).
- Deserialización de tipos no controlados (`object`, `dynamic`) desde entrada externa.

### 12.10 SSRF en clientes HTTP salientes *(A10 — Server-Side Request Forgery)*

GOP integra con sistemas externos (SGC, BMC/Helix, ContrAktor, Azure DevOps — ver §15) y con otros microservicios internos (§10.5). **Toda** llamada saliente es vector potencial de SSRF.

**MUST:**
- Toda llamada HTTP saliente usa `IHttpClientFactory` con un cliente nombrado **registrado en la shared library**.
- El pipeline del `HttpClient` incluye `SsrfProtectionHandler` que:
  - Valida host contra **allow-list declarativa** (`SsrfOptions:AllowedHosts` por nombre de cliente).
  - Bloquea IPs privadas (RFC 1918: `10.0.0.0/8`, `172.16.0.0/12`, `192.168.0.0/16`), loopback (`127.0.0.0/8`), link-local (`169.254.0.0/16`), metadata cloud (`169.254.169.254`), multicast y broadcast — **salvo** que el cliente nombrado esté marcado explícitamente como "internal-only" (comunicación intra-cluster) y el host esté en la allow-list interna.
  - Bloquea **redirecciones** que salten fuera de la allow-list (por defecto `MaxAutomaticRedirections = 0` o validación por redirect).
- Timeouts explícitos y `CancellationToken` propagado.

**PROHIBIDO:**
- `new HttpClient()` directo en código de dominio o aplicación.
- URLs de destino construidas con input del usuario sin pasar por validación del handler.
- `MaxAutomaticRedirections > 3` sin justificación.

### 12.11 Protección contra inyección LDAP/AD *(A03)*

Al integrar con Active Directory para SSO (ver §15.3):

**MUST:**
- Queries LDAP usan `System.DirectoryServices.Protocols` con **parámetros**, nunca concatenación.
- Shared helper `LdapFilterEncoder.Encode(input)` obligatorio para cualquier valor que provenga de input externo y se incluya en un filtro LDAP.
- Conexiones LDAP siempre con `SSL/TLS` (LDAPS) o `StartTLS`.

**PROHIBIDO:**
- Filtros LDAP construidos con interpolación de strings (`$"(cn={userInput})"`).
- Bind con credenciales del usuario final — el servicio usa una cuenta técnica y valida por separado.

### 12.12 Supply chain y componentes vulnerables *(A06, A08)*

Automatizado en CI — cero esfuerzo por desarrollador:

**MUST en pipeline:**
- `dotnet list package --vulnerable --include-transitive` → **falla el build** si aparece CVE de severidad **High** o **Critical**.
- `dotnet list package --deprecated` → warning visible en el reporte.
- Generación de **SBOM** (CycloneDX) adjunto al artefacto de release.
- Imágenes base de Docker escaneadas con **Trivy** o **Grype**; falla el build en CVE High/Critical.

**MUST:**
- **Dependabot** o **Renovate** habilitado por repo, con PR automático para actualizaciones de seguridad.
- `nuget.config` con `<signatureValidationMode>require</signatureValidationMode>` — sólo paquetes firmados.
- Restore con `--locked-mode` (requiere `packages.lock.json` commiteado).

**Escalation de CVE:** cualquier CVE High/Critical publicado contra una dependencia en uso activa un issue de tracking automático con SLA de 7 días para fix o mitigación documentada.

### 12.13 Analyzers de seguridad en build *(transversal)*

Activados **warning-as-error** en `Directory.Build.props` raíz — aplica a todos los proyectos sin configuración adicional:

- **Microsoft.CodeAnalysis.NetAnalyzers** — categoría Security (reglas `CA2100`, `CA2109`, `CA3001–CA3147`, `CA5350–CA5404`).
- **SecurityCodeScan** (o equivalente mantenido al momento de release) para reglas específicas de ASP.NET.
- **Roslynator** con subset de seguridad.

Los desarrolladores no configuran analyzers por proyecto — el `Directory.Build.props` heredado hace cumplir el estándar automáticamente.

### 12.14 Error handling que no filtra información *(A05, A09)*

**MUST:** middleware global `UseExceptionHandler` de la shared library:

- Loggea el error con stack trace completo + `correlation_id` (ver §13.2) en el log estructurado.
- Devuelve al cliente **ProblemDetails RFC 7807** con:
  - `type`, `title`, `status`, `correlation_id` únicamente.
  - **Sin** stack trace, **sin** detalle técnico interno, **sin** nombres de tabla/columna.
- En entorno `Development` puede incluir detalle extendido — nunca en producción.

**PROHIBIDO:**
- `try/catch` que devuelva `ex.ToString()` o `ex.Message` directamente al cliente en endpoints de producción.
- Páginas de error por defecto de ASP.NET Core (Developer Exception Page) habilitadas fuera de `Development`.

### 12.15 Mapa OWASP Top 10 → sección

| OWASP 2021 | Cubierto en |
|---|---|
| A01 — Broken Access Control | §7 (autorización 5 capas), §12.4 (CSRF), §18.3 |
| A02 — Cryptographic Failures | §12.3 (HTTPS), §12.7 (secretos), §12.8 (crypto), §7.10 (clasificación) |
| A03 — Injection | §12.1 (validación), §12.4 (CSRF), §12.11 (LDAP), §18.2 (SQL), §11 (EF Core) |
| A04 — Insecure Design | §3 (principios), §10 (patrones), §12.5 (rate limit), §12.0 (principio rector) |
| A05 — Security Misconfiguration | §12.2 (headers), §12.3 (HTTPS), §12.7 (secretos), §12.14 (errors) |
| A06 — Vulnerable Components | §4.1 (licencias), §12.12 (supply chain) |
| A07 — Auth Failures | §7.1 (auth), §12.5 (rate limit), §12.6 (password/MFA/lockout) |
| A08 — Integrity Failures | §12.9 (deserialización), §12.12 (signing, SBOM) |
| A09 — Logging/Monitoring | §11.6 (Audit.NET), §13 (observabilidad), §12.14 (errors) |
| A10 — SSRF | §12.10 (SSRF handler) |

---

## 13. Observabilidad

### 13.1 Logging estructurado

**MUST:** Serilog configurado con:
- Output a consola (desarrollo) y archivos/Seq/ApplicationInsights (producción).
- Formato JSON estructurado.
- Enrichers: `CorrelationId`, `UserId`, `TenantId`, `MachineName`.

**MUST:** nivel por defecto `Information`. `Debug` solo en desarrollo. `Trace` nunca en producción.

**Ejemplo:**

```csharp
_logger.LogInformation("Well {WellId} created by user {UserId} in tenant {TenantId}",
    well.Id, _currentUser.UserId, _currentUser.TenantId);
```

**PROHIBIDO:**
- `Console.WriteLine` en producción.
- Log de tokens, passwords, datos sensibles, payloads completos de request/response de endpoints que traen datos personales.
- Log de excepciones sin contexto (`_logger.LogError(ex)` sin mensaje).

### 13.2 Correlation ID

**MUST:** cada request entrante tiene un `X-Correlation-Id`:
- Si viene del cliente, se preserva.
- Si no, se genera GUID nuevo.
- Se propaga en llamadas salientes a otros servicios.
- Aparece en todos los logs de esa request.

### 13.3 Health Checks

**MUST:** cada servicio expone:
- `/health/live` — el proceso está vivo (siempre 200 si responde).
- `/health/ready` — está listo para recibir tráfico (verifica DB, dependencias críticas).

### 13.4 Métricas

**SHOULD:** exponer métricas vía endpoint `/metrics` en formato Prometheus, incluyendo:
- Counter de requests por endpoint y status.
- Histograma de latencia por endpoint.
- Métricas de negocio específicas (ej. trámites aprobados por hora).

---

## 14. Pruebas

### 14.1 Cobertura mínima

**MUST:** existe suite de tests automatizados ejecutada en CI que cubre los comportamientos críticos de §14.2–14.5.

**SHOULD:** cobertura de unit tests ≥ 80% (alineado con el Estándar de Codificación ANH). El porcentaje es una señal, no un fin — se prefiere **cobertura de comportamientos** (los listados en §14.2–14.5) sobre cobertura puramente numérica. Un PR por debajo del umbral se discute, no se bloquea automáticamente, si los comportamientos críticos están cubiertos.

### 14.2 Unit tests

**MUST** tests que cubran los siguientes comportamientos:
- Lógica de `Domain` (entidades, value objects, domain services).
- Command/Query handlers en `Application` (implementaciones de `ICommandHandler<,>` / `IQueryHandler<,>`) — al menos caminos happy y principales de error.
- Validators de FluentValidation — reglas no triviales.

**SHOULD** tests de comportamiento (Given-When-Then) más que de implementación.

### 14.3 Integration tests

**MUST** para:
- Endpoints críticos (crear pozo, radicar forma, aprobar).
- Flujos de autorización.
- Migraciones de BD.

**SHOULD:** usar `WebApplicationFactory` con base de datos SQL Server real en contenedor o LocalDb para tests que ejerciten EF Core y queries.

**MAY:** mockear `DbContext` o `IDbContextFactory` en unit tests puntuales cuando el test valida lógica que es independiente del ORM (ej. un guard clause en un handler antes de tocar el repositorio). En todo caso, los comportamientos que dependen de SQL real se cubren con integration tests.

### 14.4 tSQLt

**MUST** para:
- Stored procedures críticos.
- Triggers de auditoría.
- Vistas complejas.
- Lógica de migración de datos legado.

### 14.5 Migration tests

**MUST** crear una suite específica en `Tests.Integration/Migrations/` que:
- Carga dataset del GOP legado en BD de test.
- Ejecuta scripts de migración.
- Valida contadores, integridad referencial, y reglas de negocio críticas sobre los datos migrados.

---

## 15. Integraciones con sistemas externos

### 15.1 Sistemas a integrar

| Sistema | Propósito | Método | Criticidad |
|---|---|---|---|
| Active Directory (ANH) | SSO | LDAPS / OAuth2 | Alta |
| SOLAR | Operadores, contratos, campos | API REST | Alta |
| VAF | Contratos ANH (estratigráficos) | API REST | Media |
| AVM | Producción mensual | API / BD | Alta |
| Mapa de Tierras | Polígonos de contratos | API REST / WMS | Alta |
| DANE | Municipios, departamentos | Catálogo precargado | Baja |
| CPIP | Matrícula profesional | Por definir (web scraping o API) | Media |
| ControlDoc | Radicación documental | API | Alta |
| SMTP institucional | Notificaciones | SMTP | Media |

### 15.2 Patrón de integración

**MUST:**
- Toda integración vive en `Infrastructure` detrás de una interfaz en `Application`.
- Cliente HTTP usa `HttpClientFactory` con políticas de retry (Polly).
- Timeout explícito en cada llamada.
- Circuit breaker para sistemas poco confiables.
- Fallback definido: ¿qué pasa si el sistema externo no responde? — cada integración **MUST** documentarlo.

**MUST:** toda llamada se registra en `audit.IntegrationLog` con: sistema, endpoint, payload enviado, respuesta, status, duración, error.

### 15.3 Autenticación a sistemas externos

**PROHIBIDO** credenciales en código o `appsettings.json`. **MUST** usar:
- Azure Key Vault (en Azure).
- HashiCorp Vault o equivalente (on-premise).
- Configuración leída vía `IConfiguration` con provider seguro.

---

## 16. Concerns transversales entre microservicios

### 16.1 Principios

**MUST:**
- Cada servicio es dueño exclusivo de sus datos (escritura).
- Comunicación entre servicios **MUST** ser asíncrona cuando sea posible (V2+ con event bus).
- Para V1 sin event bus, comunicación síncrona HTTP **MAY** usarse pero **SHOULD** minimizarse.

**PROHIBIDO:**
- Un servicio escribir en la BD de otro servicio.
- Llamadas HTTP síncronas entre servicios para operaciones de autorización rutinaria.
- Replicación de lógica de negocio entre servicios.

### 16.2 Referencia de datos compartidos

**Reference Data (catálogos y entidades transversales):**

- **V1:** los servicios consultan `gop.core` vía API cuando necesitan datos de operadores, contratos, campos. Cache local en memoria con TTL 10 minutos.
- **V2+ (con event bus):** replicación por eventos. `gop.core` publica `OperatorUpdated`, `ContractCreated`, etc. Los servicios interesados mantienen proyecciones locales.

**Catálogos Tipo A:**

- **V1:** `gop.core` expone endpoint de bootstrap. Otros servicios consultan una vez al inicio (en health check de ready) y cachean en memoria hasta reinicio.
- **V2+:** igual, pero con invalidación por evento.

### 16.3 Shared Library `Anh.Gop.Shared.Auth`

**MUST** contener:
- `JwtValidationMiddleware` — valida firma y expiración.
- `CurrentUserContextMiddleware` — construye `ICurrentUserContext`.
- `ICurrentUserContext` interface con claims extraídos.
- Base de policies (`RequireTenantType`, `RequirePermission`, etc.).
- `IIdentityServiceClient` con cache para consultas no-triviales.
- Helpers de Resource-Based Authorization.

**Versionado semántico estricto.** Bump mayor obliga a migración coordinada.

### 16.4 Criterio claro de consulta a Identity Service

Ya documentado en §7.8. Se repite aquí por conveniencia:

- **Datos del JWT:** se leen directo, sin consulta.
- **`contracts_truncated = true`:** consulta a Identity con cache.
- **Matrícula profesional no en JWT:** consulta con cache.
- **Revocation list:** consulta opcional en operaciones críticas.

**PROHIBIDO** consulta rutinaria por cada request para datos que ya están en JWT.

---

## 17. Reverse Proxy y entrada al sistema (nginx)

Dado despliegue on-premise, nginx es el reverse proxy frente a los microservicios.

### 17.1 Responsabilidades de nginx (V1)

**MAY:**
- Terminación TLS.
- Routing por path a los microservicios correspondientes.
- **Rate limiting por IP** (defensa de borde). Complementa el rate limit por identidad que aplica la app (ver §12.5).
- Compresión gzip.
- Logging de acceso.
- **HSTS duplicado** como defensa redundante. El header canónico lo emite la app (ver §12.2); si nginx también lo emite, ambos deben coincidir.

> **Nota:** los **security headers** (`X-Content-Type-Options`, `X-Frame-Options`, `Referrer-Policy`, `Permissions-Policy`, `CSP`) los emite la **aplicación** vía `Anh.Gop.Shared.Security` (§12.2), no nginx. Esto garantiza que los headers sean consistentes independientemente de la topología de despliegue (reverse proxy en V1, API gateway en V2+) y que se prueben con tests de integración junto al resto del pipeline.

### 17.2 Funciones fuera del alcance de nginx

nginx en V1 **no** realiza las siguientes funciones — éstas quedan en otros componentes del sistema (no son prohibiciones sino descripción del reparto de responsabilidades):

- **Validación de JWT:** cada microservicio valida el token localmente vía `Anh.Gop.Shared.Auth`. V1 carece de gateway full-featured; se evaluará en V2+.
- **Transformación de tokens:** el JWT emitido por `gop.identity` llega intacto al servicio destino.
- **Circuit breaking avanzado:** resiliencia inter-servicio se implementa en el cliente HTTP de cada servicio (Polly, ver §16).

### 17.3 Network isolation

**MUST:** los microservicios no son accesibles directamente desde internet. Solo nginx tiene IP pública. Los servicios escuchan en red interna.

**SHOULD:** los microservicios validan que la conexión entrante viene de nginx (por IP o por red). Esto es defensa en profundidad — no reemplaza validar el JWT.

---

## 18. Líneas rojas consolidadas

Esta lista es vinculante. Cualquier PR que transgreda una línea roja **MUST** rechazarse. Transgredirla de forma excepcional requiere ADR previo con aprobación de arquitectura, conforme a §0.1. Las reglas de aquí son **invariantes arquitectónicos y de seguridad** — reglas operativas o de estilo viven en sus secciones respectivas.

### 18.1 Arquitectura

- **NUNCA** `Domain` depende de `Application` o `Infrastructure`.
- **NUNCA** `Application` depende de `Infrastructure`.
- **NUNCA** un servicio escribe en la BD de otro.
- **NUNCA** lógica de dominio en controllers o stored procedures (salvo migración legado).
- **NUNCA** referencias entre bounded contexts que violen el mapa de dependencias del modelo de datos.
- **NUNCA** un servicio llama a `gop.audit` para escribir auditoría. La escritura es **siempre local** vía Audit.NET sobre `audit.AuditLog` (ver §11.6).

### 18.2 Datos

- **NUNCA** concatenación SQL con inputs (SQL injection).
- **NUNCA** queries sobre entidades de tenant sin filtro de tenant.
- **NUNCA** editar migraciones ya aplicadas.
- **NUNCA** `DbContext` directo en controllers.

> **Nota:** `SELECT *` es un **anti-patrón a evitar** (SHOULD), no una línea roja — en queries internas o scripts puntuales puede ser aceptable. La regla arquitectónica vive en §11.1.

### 18.3 Seguridad

- **NUNCA** credenciales en código, config files sin cifrar, o logs.
- **NUNCA** stack traces en responses de producción.
- **NUNCA** `[AllowAnonymous]` fuera de endpoints de autenticación y endpoints de infraestructura explícitamente listados (health checks, métricas, OpenAPI en entornos no productivos). El listado vive en la configuración de la shared library y se revisa por arquitectura.
- **NUNCA** log de tokens, passwords, datos sensibles de usuario.
- **NUNCA** confiar en headers `X-User-*` sin validar JWT.
- **NUNCA** inserción manual a `audit.AuditLog` desde código de dominio o aplicación. La escritura de auditoría se realiza **exclusivamente** a través del pipeline de Audit.NET + `Audit.EntityFramework.Core` (ver §11.6).
- **NUNCA** `new HttpClient()` directo para llamadas salientes. Usar `IHttpClientFactory` con cliente nombrado registrado en `Anh.Gop.Shared.Security`, que incluye `SsrfProtectionHandler` (ver §12.10).
- **NUNCA** introducir `Newtonsoft.Json` como serializer primario de la aplicación. Serializer único: `System.Text.Json` con defaults endurecidos (ver §12.9). Si una dependencia transitiva lo arrastra, se tolera siempre que no se use directamente desde código propio; si un SDK de terceros lo exige para interoperar, se documenta la excepción (sin ADR salvo que se convierta en patrón).
- **NUNCA** algoritmos criptográficos débiles: MD5, SHA-1, DES, 3DES, RC4, modo ECB. `System.Random` no es aceptable para tokens/salts/ids de sesión — usar `RandomNumberGenerator` (ver §12.8).
- **NUNCA** secretos en `appsettings*.json`, repositorio git o logs. Usar variables de entorno o Key Vault (ver §12.7).
- **NUNCA** filtros LDAP construidos con interpolación de strings — usar `LdapFilterEncoder.Encode(input)` (ver §12.11).
- **NUNCA** `Developer Exception Page` habilitada fuera de entorno `Development` (ver §12.14).

### 18.4 API

- **NUNCA** breaking changes en versión publicada.
- **NUNCA** devolver entidades de dominio o modelos de EF directamente.
- **NUNCA** endpoints públicos de negocio sin documentación OpenAPI en producción (los endpoints de infraestructura están exentos; ver §9.8).
- **NUNCA** endpoints GET con efectos secundarios.
- **NUNCA** envelopes propietarios tipo `{ status, message, data }` / `{ success, result, error }` en responses de éxito. El contrato es: **body = DTO** (o `204 No Content`), **errores = ProblemDetails RFC 7807**, **metadatos = headers HTTP** (`Location`, `ETag`, `X-Pagination`). Ver §9.6.
- **NUNCA** fechas fuera de ISO 8601, enums como entero, o propiedades JSON que no sean `camelCase` en contratos públicos (ver §9.9).
- **NUNCA** archivos binarios embebidos como base64 en JSON salvo excepción documentada < 100 KB (ver §9.9.5).

### 18.5 Testing e higiene de CI

Las reglas de disciplina de CI (no pushear con tests rotos, no silenciar analyzers sin justificación, etc.) viven en la **política de CI y en los analyzers configurados como warning-as-error** (ver §12.13 y §14), no aquí. Se enforzan por pipeline, no por revisión manual de líneas rojas.

---

## 19. Apéndices

### 19.1 Capabilities iniciales (permisos)

Esta es la lista inicial sugerida. Se expande con cada módulo que se implemente.

**Pozos (ops):**
- `Wells.CanView`
- `Wells.CanCreate`
- `Wells.CanEdit`
- `Wells.CanDelete`
- `Wells.CanViewAllTenants` (solo ANH)

**Trámites Serie 100:**
- `Forms.F101.CanFile`
- `Forms.F101.CanSignGeologist`
- `Forms.F101.CanSignEngineer`
- `Forms.F101.CanReview`
- `Forms.F101.CanApprove`
- `Forms.F101.CanReturn`
- `Forms.F101.CanReject`
- *(análogo para F102-F112)*

**Trámites Serie 200:**
- `Forms.F202.CanFile`
- `Forms.F202.CanSign`
- `Forms.F202.CanReview`
- `Forms.F202.CanApprove`
- `Forms.F202.CanReturn`
- *(análogo para F201, F203-F217)*

**Administración:**
- `Admin.Users.CanManage`
- `Admin.Roles.CanManage`
- `Admin.Catalogs.CanManage`
- `Admin.Audit.CanView`

**Operadora:**
- `Operator.Users.CanManage` (OperatorAdmin)
- `Operator.ContractAccess.CanAssign`

### 19.2 Roles iniciales y agrupación de capabilities

| Rol | Contexto | Capabilities agrupadas |
|---|---|---|
| `AnhAdministrator` | ANH | Todas |
| `AnhGopAdministrator` | ANH | Admin.*, Catalog management, Audit |
| `AnhOperationsEngineer` | ANH | Wells.CanView*, Forms.F10x.CanReview/Approve/Return |
| `AnhGeologist` | ANH | Wells.CanView*, Forms.F10x.CanReview (geológica) |
| `AnhProductionSupervisor` | ANH | Production.*, Forms.F20x.CanReview/Approve/Return |
| `OperatorAdmin` | Operador | Operator.Users.*, Operator.ContractAccess.*, view own tenant |
| `OperatorAgent` | Operador | Wells.CanCreate/Edit (own), Forms.F10x.CanFile |
| `OperatorCoordinator` | Operador | Wells.CanView (own), review internal |
| `OperatorSupervisor` | Operador | View own tenant, internal approvals |
| `OperatorGeologist` | Operador | Forms.F10x.CanSignGeologist |
| `OperatorPetroleumEngineer` | Operador | Forms.F10x.CanSignEngineer, Forms.F20x.CanSign |

**Nota sobre alcance (Contract Scope) de roles ANH:** los roles `AnhOperationsEngineer`, `AnhGeologist` y `AnhProductionSupervisor` **típicamente** se asignan con `UserRole.ContractId` específico (por fiscalizador asignado a contratos concretos), no en modo global. Solo `AnhAdministrator` y `AnhGopAdministrator` reciben `UserRole` con `ContractId = NULL` (alcance global ANH). Ver §7.4.1 para la tabla de resolución de alcance.

### 19.3 Plantilla de Command Handler

```csharp
public record CreateWellCommand(
    string Name,
    long OperatorId,
    long ContractId,
    long FieldId
    // ... más campos
) : ICommand<Result<long>>;

public class CreateWellHandler : ICommandHandler<CreateWellCommand, Result<long>>
{
    private readonly IWellRepository _wellRepo;
    private readonly IUnitOfWork _uow;
    private readonly ICurrentUserContext _currentUser;
    private readonly ILogger<CreateWellHandler> _logger;

    public CreateWellHandler(
        IWellRepository wellRepo,
        IUnitOfWork uow,
        ICurrentUserContext currentUser,
        ILogger<CreateWellHandler> logger)
    {
        _wellRepo = wellRepo;
        _uow = uow;
        _currentUser = currentUser;
        _logger = logger;
    }

    public async Task<Result<long>> HandleAsync(CreateWellCommand cmd, CancellationToken ct)
    {
        // 1. Authorize (tenancy verified by middleware; here fine-grained)
        if (_currentUser.TenantType == TenantType.Operator
            && _currentUser.TenantId != cmd.OperatorId)
        {
            return Result.Forbidden("Cannot create well for another operator");
        }

        // 2. Check uniqueness (domain rule)
        var existing = await _wellRepo.GetByNameAsync(cmd.Name, ct);
        if (existing is not null)
            return Result.Conflict("Well name already exists");

        // 3. Create domain entity
        var well = Well.Create(cmd.Name, cmd.OperatorId, cmd.ContractId, cmd.FieldId);

        // 4. Persist
        await _wellRepo.AddAsync(well, ct);
        await _uow.SaveChangesAsync(ct);

        _logger.LogInformation("Well {WellId} created by user {UserId}", well.Id, _currentUser.UserId);

        return Result.Success(well.Id.Value);
    }
}
```

### 19.4 Plantilla de Query Handler con proyección

```csharp
public record GetWellsQuery(
    int Page = 1,
    int PageSize = 50,
    string? SortBy = "Name",
    string? SortDirection = "asc",
    int? FilterCurrentStateId = null,
    long? FilterOperatorId = null,
    string? FilterNamePartial = null
) : IQuery<Result<PagedResult<WellSummaryDto>>>;

public class GetWellsHandler : IQueryHandler<GetWellsQuery, Result<PagedResult<WellSummaryDto>>>
{
    private readonly GopDbContext _ctx;
    private readonly ICurrentUserContext _currentUser;

    public async Task<Result<PagedResult<WellSummaryDto>>> HandleAsync(GetWellsQuery q, CancellationToken ct)
    {
        var baseQuery = _ctx.Wells.AsNoTracking();

        // Tenant filter is automatic via Global Query Filter;
        // here, additional filters:
        if (q.FilterCurrentStateId.HasValue)
            baseQuery = baseQuery.Where(w => w.CurrentStateId == q.FilterCurrentStateId.Value);

        if (q.FilterOperatorId.HasValue)
            baseQuery = baseQuery.Where(w => w.OperatorId == q.FilterOperatorId.Value);

        if (!string.IsNullOrWhiteSpace(q.FilterNamePartial))
            baseQuery = baseQuery.Where(w => EF.Functions.Like(w.Name, $"%{q.FilterNamePartial}%"));

        var projected = baseQuery.Select(w => new WellSummaryDto
        {
            WellId = w.Id,
            Name = w.Name,
            FiscalizedUwi = w.FiscalizedUwi,
            CurrentStateId = w.CurrentStateId,
            CurrentStateName = w.CurrentState.Name,  // JOIN implícito
            OperatorId = w.OperatorId,
            OperatorName = w.Operator.Name,          // JOIN implícito
            ContractId = w.ContractId,
            ContractCode = w.Contract.Code,
            FieldId = w.FieldId,
            FieldName = w.Field.Name,
            IsEditable = !w.Procedures.Any(p => p.FormCode == "F101" && p.Status == "Filed"),
            HasActiveAlerts = w.Situations.Any(s => s.EndedAt == null && s.Severity == "Critical"),
            CreatedAt = w.CreatedAt
        });

        // Apply sorting (validated against allowlist in validator)
        projected = ApplySorting(projected, q.SortBy, q.SortDirection);

        var paged = await projected.ToPagedListAsync(q.Page, q.PageSize, ct);

        return Result.Success(paged);
    }
}
```

### 19.5 Clasificación de entidades para exposición (Tipo A/B/C/D)

Esta tabla aplica la regla de §8.2 a las entidades principales del modelo. El **Tipo** determina el **tratamiento de exposición** y por tanto el diseño de endpoints y DTOs.

**Recordatorio de tipos (§8.2):**
- **A** — Catálogo global pequeño (<100 items, estable). Bootstrap endpoint + precarga en sesión.
- **B** — Catálogo específico de feature (media, estable). Endpoint dedicado, lazy load.
- **C** — Entidad transversal (miles de items, cambian). Búsqueda paginada + byId con cache.
- **D** — Entidad transaccional de dominio. Listado resumen + detalle limpio (patrón Summary/Detail).

#### 19.5.1 Catálogos — esquema `catalog`

| Entidad | Tipo | Tratamiento |
|---|---|---|
| `WellState` | A | Bootstrap session |
| `WellStateTransition` | A | Bootstrap session (leída junto con `WellState`) |
| `AngleType` (Catalog) | A | Bootstrap session |
| `PurposeType` (Catalog) | A | Bootstrap session |
| `CompletionType` (Catalog) | A | Bootstrap session |
| `TrajectoryType` (Catalog) | A | Bootstrap session |
| `LaheeClassification` (Catalog) | A | Bootstrap session |
| `TrapType` (Catalog) | A | Bootstrap session |
| `ProductionMethod` (Catalog) | A | Bootstrap session |
| `CoordinateReferenceSystem` | A | Bootstrap session |
| `Basin` | A | Bootstrap session |
| `ActionDefinition` | A | Bootstrap session (motor workflow) |
| `Formation` | B | Lazy load por feature (puede crecer con altas) |
| `FormDefinition` | B | Lazy load por feature |
| `AttachmentDefinition` | B | Lazy load junto con `FormDefinition` |
| `DrillingRig` | B | Lazy load por feature |
| `PoliticalDivision` | B | Lazy load (~1.100 municipios + 33 deptos) |

> **Nota de seguimiento:** entidades marcadas Tipo **A** con volumen esperado >50 ítems (p.ej. `WellStateTransition`, que puede crecer con cambios normativos de la matriz de transiciones) se **reevaluarán tras el primer release**. Si superan ese umbral en producción, se migrarán a Tipo **B** (lazy load del feature de pozos) mediante ADR dedicado. No requiere cambio en V1.

#### 19.5.2 Entidades transversales — esquema `core`

| Entidad | Tipo | Tratamiento |
|---|---|---|
| `Operator` | C | Search paginada + byId con cache |
| `Contract` | C | Search paginada + byId con cache |
| `ContractPartner` | — | Sub-recurso de `Contract` (`/contracts/{id}/partners`) |
| `ContractPhase` | — | Sub-recurso de `Contract` |
| `Field` | C | Search paginada + byId con cache |
| `FieldContract` | — | Sub-recurso de `Field` o `Contract` |
| `Block` | C | Search paginada + byId con cache |
| `ClusterLocation` | C | Search paginada + byId con cache |

#### 19.5.3 Operaciones de pozo — esquema `ops`

| Entidad | Tipo | Tratamiento |
|---|---|---|
| `Well` | D | `WellSummaryDto` + `WellDetailDto` |
| `FiscalizedUwiComponent` | — | Embebido en `WellDetailDto` (sub-agregado 1:1) |
| `WellBranch` | — | Sub-recurso `/wells/{id}/branches` |
| `WellFormation` | — | Sub-recurso `/wells/{id}/formations` |
| `WellLocation` | — | Sub-recurso `/wells/{id}/locations` |
| `WellCasing` | — | Sub-recurso `/wells/{id}/casings` |
| `WellPermit` | — | Sub-recurso `/wells/{id}/permits` |
| `WellPhase` | — | Sub-recurso `/wells/{id}/phases` |
| `WellSituation` | — | Sub-recurso `/wells/{id}/situations` |
| `WellStateHistory` | — | Sub-recurso `/wells/{id}/state-history` (solo lectura) |
| `Form101Data` | D | `Form101SummaryDto` + `Form101DetailDto` |
| `Form102Data` — `Form112Data` | D | Patrón Summary/Detail análogo |
| `IdopData` | D | `IdopSummaryDto` + `IdopDetailDto` |
| `IdocData` | D | Análogo |

#### 19.5.4 Producción — esquema `prod`

| Entidad | Tipo | Tratamiento |
|---|---|---|
| `ProducingFormation` | D | Summary + Detail |
| `ProductionModality` | — | Sub-recurso de `ProducingFormation` |
| `MonthlyWellProduction` | D | Summary + Detail (con paginación agresiva — volúmenes altos) |
| `MonthlyProductionChange` | — | Sub-recurso de `MonthlyWellProduction` |
| `FieldBalance` | D | Summary + Detail |
| `PotentialTest` | D | Summary + Detail |
| `Injection` | D | Summary + Detail |
| `IncrementalProductionProject` | D | Summary + Detail |
| `EmissionEvent` | D | Summary + Detail |
| `Tank` | C | Search paginada + byId |
| `TankMovement` | — | Sub-recurso de `Tank` |
| `Form201Data` — `Form217Data` | D | Patrón Summary/Detail |

#### 19.5.5 Workflow y trámites — esquemas `workflow`, `procedure`

| Entidad | Tipo | Tratamiento |
|---|---|---|
| `WorkflowDefinition` | B | Lazy load admin |
| `TaskDefinition` | B | Embebido en `WorkflowDefinition` detail |
| `TaskRouting` | — | Embebido en `TaskDefinition` detail |
| `TaskDeadline` | — | Embebido en `TaskDefinition` detail |
| `TaskFieldPermission` | — | Embebido en `TaskDefinition` detail |
| `WorkflowInstance` | — | Embebido en `ProcedureDetailDto` |
| `TaskInstance` | — | Sub-recurso `/procedures/{id}/tasks` |
| `TaskActionLog` | — | Sub-recurso `/procedures/{id}/tasks/{taskId}/log` (solo lectura) |
| `Procedure` | D | `ProcedureSummaryDto` + `ProcedureDetailDto` |
| `ProcedureAttachment` | — | Sub-recurso `/procedures/{id}/attachments` |
| `ProcedureObservation` | — | Sub-recurso `/procedures/{id}/observations` |
| `ProcedureSignature` | — | Sub-recurso `/procedures/{id}/signatures` |
| `ProcedurePdf` | — | Sub-recurso `/procedures/{id}/pdfs` |
| `ProcedureNotification` | — | Sub-recurso `/procedures/{id}/notifications` |
| `ProcedureSpatialValidation` | — | Sub-recurso `/procedures/{id}/spatial-validations` |

#### 19.5.6 Identidad — esquema `identity`

| Entidad | Tipo | Tratamiento |
|---|---|---|
| `User` | D | Summary + Detail (admin-only) |
| `Role` | A | Bootstrap session (lista cerrada) |
| `Permission` | A | Bootstrap session |
| `RolePermission` | — | Sub-recurso de `Role` |
| `UserRole` | — | Sub-recurso `/users/{id}/roles` |
| `ProfessionalLicense` | — | Sub-recurso `/users/{id}/licenses` |
| `UserSession` | — | Admin-only listing y revoke |

#### 19.5.7 Auditoría e integración — esquemas `audit`, `integration`

Estos esquemas no se exponen como recursos CRUD al frontend. Su acceso es:

| Entidad | Tratamiento |
|---|---|
| `AuditLog` | Endpoints de consulta (`/audit/search`) con filtros, solo para roles `Admin.Audit.CanView`. Paginación obligatoria. |
| `IntegrationLog` | Igual, solo admin. |
| `BulkLoadLog` | Igual, solo admin. |
| `Stg*`, `Homologation*`, `SyncQueue`, `CachedPolygon` | **No se exponen.** Uso interno del servicio `gop.integration`. |

**Reglas derivadas:**

- Una entidad marcada con `—` en "Tipo" es **sub-entidad** del agregado y **NUNCA** se expone como recurso raíz. Se accede únicamente vía el recurso padre.
- Cambiar el Tipo de una entidad requiere ADR (conforme §8.2).
- Entidades nuevas que se agreguen al modelo **MUST** clasificarse explícitamente en esta tabla al momento de incorporarse.

### 19.6 Plantilla de Controller

```csharp
[ApiController]
[Route("api/v1/wells")]
[Authorize]
public class WellsController : ControllerBase
{
    private readonly ICommandDispatcher _commands;
    private readonly IQueryDispatcher _queries;

    public WellsController(ICommandDispatcher commands, IQueryDispatcher queries)
    {
        _commands = commands;
        _queries = queries;
    }

    [HttpGet]
    [Authorize(Policy = "Wells.CanView")]
    public async Task<IActionResult> GetWells([FromQuery] GetWellsQuery query, CancellationToken ct)
    {
        var result = await _queries.DispatchAsync(query, ct);
        return result.ToActionResult();
    }

    [HttpGet("{wellId:long}")]
    [Authorize(Policy = "Wells.CanView")]
    public async Task<IActionResult> GetWellById(long wellId, CancellationToken ct)
    {
        var result = await _queries.DispatchAsync(new GetWellByIdQuery(wellId), ct);
        return result.ToActionResult();
    }

    [HttpPost]
    [Authorize(Policy = "Wells.CanCreate")]
    [ProducesResponseType(typeof(WellDetailDto), StatusCodes.Status201Created)]
    [ProducesResponseType(typeof(ProblemDetails), StatusCodes.Status400BadRequest)]
    public async Task<IActionResult> CreateWell([FromBody] CreateWellCommand cmd, CancellationToken ct)
    {
        // El handler devuelve Result<WellDetailDto> (no solo el Id).
        var result = await _commands.DispatchAsync(cmd, ct);
        return result.ToCreatedResult(
            controller: this,
            locationTemplate: "/api/v1/wells/{0}",
            idSelector: dto => dto.Id);
    }

    [HttpPut("{wellId:long}")]
    [Authorize(Policy = "Wells.CanEdit")]
    [ProducesResponseType(typeof(WellDetailDto), StatusCodes.Status200OK)]
    [ProducesResponseType(StatusCodes.Status409Conflict)]
    public async Task<IActionResult> ReplaceWell(long wellId, [FromBody] ReplaceWellCommand cmd, CancellationToken ct)
    {
        var result = await _commands.DispatchAsync(cmd with { WellId = wellId }, ct);
        return result.ToOkResult(); // incluye ETag automáticamente si el DTO expone RowVersion
    }

    [HttpDelete("{wellId:long}")]
    [Authorize(Policy = "Wells.CanDelete")]
    [ProducesResponseType(StatusCodes.Status204NoContent)]
    public async Task<IActionResult> DeleteWell(long wellId, CancellationToken ct)
    {
        var result = await _commands.DispatchAsync(new DeleteWellCommand(wellId), ct);
        return result.ToNoContentResult();
    }
}
```

### 19.7 CQRS sin mediador externo — código de referencia de `Anh.Gop.Shared.Cqrs`

Esta shared library encapsula todo el andamiaje CQRS del proyecto. No depende de MediatR ni de ningún otro mediador. Licencia: código propio.

#### 19.7.1 Interfaces marcador y de handler

```csharp
// Anh.Gop.Shared.Cqrs/ICommand.cs
namespace Anh.Gop.Shared.Cqrs;

/// <summary>Marcador de command con resultado tipado.</summary>
public interface ICommand<TResult> { }

/// <summary>Marcador de query con resultado tipado.</summary>
public interface IQuery<TResult> { }

public interface ICommandHandler<in TCommand, TResult>
    where TCommand : ICommand<TResult>
{
    Task<TResult> HandleAsync(TCommand command, CancellationToken ct);
}

public interface IQueryHandler<in TQuery, TResult>
    where TQuery : IQuery<TResult>
{
    Task<TResult> HandleAsync(TQuery query, CancellationToken ct);
}
```

#### 19.7.2 Dispatchers

```csharp
// Anh.Gop.Shared.Cqrs/ICommandDispatcher.cs
public interface ICommandDispatcher
{
    Task<TResult> DispatchAsync<TResult>(ICommand<TResult> command, CancellationToken ct = default);
}

public interface IQueryDispatcher
{
    Task<TResult> DispatchAsync<TResult>(IQuery<TResult> query, CancellationToken ct = default);
}

// Anh.Gop.Shared.Cqrs/CommandDispatcher.cs
internal sealed class CommandDispatcher : ICommandDispatcher
{
    private readonly IServiceProvider _sp;
    public CommandDispatcher(IServiceProvider sp) => _sp = sp;

    public Task<TResult> DispatchAsync<TResult>(ICommand<TResult> command, CancellationToken ct = default)
    {
        var handlerType = typeof(ICommandHandler<,>).MakeGenericType(command.GetType(), typeof(TResult));
        var handler = _sp.GetRequiredService(handlerType);
        // Invocación sin reflexión caliente: delegate cache.
        var invoker = HandlerInvokerCache.GetCommandInvoker<TResult>(command.GetType());
        return invoker(handler, command, ct);
    }
}

internal sealed class QueryDispatcher : IQueryDispatcher
{
    private readonly IServiceProvider _sp;
    public QueryDispatcher(IServiceProvider sp) => _sp = sp;

    public Task<TResult> DispatchAsync<TResult>(IQuery<TResult> query, CancellationToken ct = default)
    {
        var handlerType = typeof(IQueryHandler<,>).MakeGenericType(query.GetType(), typeof(TResult));
        var handler = _sp.GetRequiredService(handlerType);
        var invoker = HandlerInvokerCache.GetQueryInvoker<TResult>(query.GetType());
        return invoker(handler, query, ct);
    }
}
```

> **Nota sobre `HandlerInvokerCache`:** compila una vez por tipo un delegate `Func<object, object, CancellationToken, Task<TResult>>` vía `MethodInfo.CreateDelegate`, evitando el costo de `Invoke` por reflexión en caliente. Implementación en la shared library.

#### 19.7.3 Registro automático por assembly scan

```csharp
// Anh.Gop.Shared.Cqrs/ServiceCollectionExtensions.cs
public static class ServiceCollectionExtensions
{
    public static IServiceCollection AddGopCqrs(this IServiceCollection services, params Assembly[] assemblies)
    {
        services.AddSingleton<ICommandDispatcher, CommandDispatcher>();
        services.AddSingleton<IQueryDispatcher, QueryDispatcher>();

        foreach (var assembly in assemblies)
        {
            RegisterHandlers(services, assembly, typeof(ICommandHandler<,>));
            RegisterHandlers(services, assembly, typeof(IQueryHandler<,>));
        }
        return services;
    }

    private static void RegisterHandlers(IServiceCollection services, Assembly assembly, Type openHandler)
    {
        var implementations = assembly.GetTypes()
            .Where(t => !t.IsAbstract && !t.IsInterface)
            .SelectMany(t => t.GetInterfaces()
                .Where(i => i.IsGenericType && i.GetGenericTypeDefinition() == openHandler)
                .Select(i => (Interface: i, Implementation: t)));

        foreach (var (iface, impl) in implementations)
            services.AddScoped(iface, impl);
    }
}
```

Uso en `Program.cs` de cada microservicio:

```csharp
builder.Services.AddGopCqrs(typeof(CreateWellCommand).Assembly);
```

#### 19.7.4 Decoradores (cross-cutting concerns) — patrón manual

Los decoradores se escriben manualmente. Para V1 se proveen tres:

```csharp
// 1. Validación con FluentValidation
public sealed class ValidatingCommandDispatcher : ICommandDispatcher
{
    private readonly ICommandDispatcher _inner;
    private readonly IServiceProvider _sp;

    public ValidatingCommandDispatcher(ICommandDispatcher inner, IServiceProvider sp)
    {
        _inner = inner;
        _sp = sp;
    }

    public async Task<TResult> DispatchAsync<TResult>(ICommand<TResult> command, CancellationToken ct = default)
    {
        var validatorType = typeof(IValidator<>).MakeGenericType(command.GetType());
        var validator = _sp.GetService(validatorType) as IValidator;
        if (validator is not null)
        {
            var context = new ValidationContext<object>(command);
            var result = await validator.ValidateAsync(context, ct);
            if (!result.IsValid)
                throw new ValidationException(result.Errors);
        }
        return await _inner.DispatchAsync(command, ct);
    }
}

// 2. Logging estructurado
public sealed class LoggingCommandDispatcher : ICommandDispatcher
{
    private readonly ICommandDispatcher _inner;
    private readonly ILogger<LoggingCommandDispatcher> _logger;

    public LoggingCommandDispatcher(ICommandDispatcher inner, ILogger<LoggingCommandDispatcher> logger)
    {
        _inner = inner;
        _logger = logger;
    }

    public async Task<TResult> DispatchAsync<TResult>(ICommand<TResult> command, CancellationToken ct = default)
    {
        var name = command.GetType().Name;
        using var _ = _logger.BeginScope("Command {CommandName}", name);
        _logger.LogInformation("Dispatching {CommandName}", name);
        var sw = Stopwatch.StartNew();
        try
        {
            var result = await _inner.DispatchAsync(command, ct);
            _logger.LogInformation("Completed {CommandName} in {ElapsedMs}ms", name, sw.ElapsedMilliseconds);
            return result;
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Failed {CommandName} after {ElapsedMs}ms", name, sw.ElapsedMilliseconds);
            throw;
        }
    }
}

// 3. Autorización fina (si aplica un chequeo centralizado más allá del [Authorize])
// Análogo: envuelve al inner dispatcher y resuelve IAuthorizationService.
```

Composición en el arranque (decoración manual sin Scrutor):

```csharp
services.AddScoped<CommandDispatcher>();
services.AddScoped<ICommandDispatcher>(sp =>
{
    ICommandDispatcher dispatcher = sp.GetRequiredService<CommandDispatcher>();
    dispatcher = new ValidatingCommandDispatcher(dispatcher, sp);
    dispatcher = new LoggingCommandDispatcher(dispatcher, sp.GetRequiredService<ILogger<LoggingCommandDispatcher>>());
    return dispatcher;
});
```

Cuando haya ≥3 decoradores y la composición se vuelva tediosa, se evaluará la adopción de **Scrutor** (MIT) para declarar `services.Decorate<ICommandDispatcher, ValidatingCommandDispatcher>()` de forma fluida. Ver §10.1.1.

#### 19.7.5 Tests unitarios mínimos

La shared library **MUST** publicarse con al menos estos tests:

- `CommandDispatcher_resuelve_handler_por_tipo`.
- `CommandDispatcher_propaga_CancellationToken`.
- `QueryDispatcher_resuelve_handler_por_tipo`.
- `AddGopCqrs_registra_todos_los_handlers_del_assembly`.
- `ValidatingCommandDispatcher_lanza_ValidationException_si_invalid`.
- `LoggingCommandDispatcher_loggea_success_y_failure`.

### 19.8 Flujo de Token Exchange — IdP externo + JWT de aplicación

Este apéndice documenta, paso a paso, el flujo operativo del patrón de §7.1.1. Sirve como material de referencia para auditorías, onboarding y conversaciones con stakeholders de seguridad.

#### 19.8.1 Modo federado (producción, usuario con cuenta en IdP de ANH)

```
┌─────────────┐         ┌──────────────┐        ┌────────────────┐        ┌──────────────────┐
│   Browser   │         │  Frontend    │        │  IdP externo   │        │   gop.identity   │
│  (usuario)  │         │   SPA        │        │  (AD ANH/OIDC) │        │   (RP + AS)      │
└──────┬──────┘         └──────┬───────┘        └────────┬───────┘        └─────────┬────────┘
       │                       │                         │                          │
       │ 1. login              │                         │                          │
       │──────────────────────▶│                         │                          │
       │                       │ 2. redirect /authorize  │                          │
       │                       │    (client_id, PKCE,    │                          │
       │                       │     nonce, state)       │                          │
       │                       │────────────────────────▶│                          │
       │                       │                         │                          │
       │◀── 3. login form + MFA ─────────────────────────│                          │
       │                                                 │                          │
       │── 4. credencial + MFA ─────────────────────────▶│                          │
       │                                                 │                          │
       │◀── 5. redirect con code ────────────────────────│                          │
       │                       │                         │                          │
       │ 6. code ─────────────▶│                         │                          │
       │                       │                                                    │
       │                       │ 7. POST /auth/login-federated  (code + verifier)   │
       │                       │───────────────────────────────────────────────────▶│
       │                       │                                                    │
       │                       │                         │ 8. intercambio code      │
       │                       │                         │    por id_token + access │
       │                       │                         │◀─────────────────────────│
       │                       │                         │                          │
       │                       │                         │ 9. id_token OIDC ────────▶
       │                       │                                                    │
       │                       │                         │     10. gop.identity:    │
       │                       │                         │       - valida id_token  │
       │                       │                         │         (firma, iss, aud,│
       │                       │                         │          exp, nonce)     │
       │                       │                         │       - resuelve User    │
       │                       │                         │         local por mapping│
       │                       │                         │       - resuelve claims  │
       │                       │                         │         GOP (§7.2)       │
       │                       │                         │       - emite JWT interno│
       │                       │                         │         firmado RS256    │
       │                       │                         │       - emite refresh    │
       │                       │                         │         token rotativo   │
       │                       │                                                    │
       │                       │◀── 11. JWT GOP + refresh ──────────────────────────│
       │                       │                                                    │
       │                       │  ┌────────────────┐                                │
       │                       │  │  Llamadas API  │  Authorization: Bearer <JWT GOP>
       │                       │──┤  a servicios   │───────────────────────────────▶ (microservicios GOP)
       │                       │  │  GOP           │                                │
       │                       │  └────────────────┘                                │
```

**Notas:**
- Pasos 2–8: estándar OIDC Authorization Code + PKCE (RFC 6749 + RFC 7636).
- Paso 10: `gop.identity` realiza el **Token Exchange (RFC 8693)** — no propaga el `id_token` ni el `access_token` del IdP.
- Paso 11+: los microservicios GOP sólo ven el JWT interno; su clave de validación es la pública de `gop.identity`.

#### 19.8.2 Modo local (testing inicial, usuarios fuera del IdP de ANH, cuentas técnicas)

```
┌─────────────┐         ┌──────────────┐                                 ┌──────────────────┐
│   Browser   │         │  Frontend    │                                 │   gop.identity   │
│  (usuario)  │         │   SPA        │                                 │   (AS + Identity)│
└──────┬──────┘         └──────┬───────┘                                 └─────────┬────────┘
       │                       │                                                   │
       │ 1. login local        │                                                   │
       │──────────────────────▶│                                                   │
       │                       │ 2. POST /auth/login  (username, password)         │
       │                       │──────────────────────────────────────────────────▶│
       │                       │                                                   │
       │                       │                      3. ASP.NET Core Identity:    │
       │                       │                         - valida password (PBKDF2)│
       │                       │                         - aplica lockout policy   │
       │                       │                         - MFA challenge si aplica │
       │                       │                         - resuelve claims GOP     │
       │                       │                         - emite JWT interno RS256 │
       │                       │                         - emite refresh token     │
       │                       │                                                   │
       │                       │◀── 4. JWT GOP + refresh ──────────────────────────│
       │                       │                                                   │
       │                       │  Authorization: Bearer <JWT GOP>  ───▶  (microservicios)
```

**Identidad del JWT emitido:** los microservicios GOP consumen **el mismo formato de JWT** sea el flujo federado o local. No hay ramificación en el código del servicio consumidor.

#### 19.8.3 Endpoints de `gop.identity`

| Endpoint | Flujo | Condición |
|---|---|---|
| `POST /auth/login-federated` | Federado (OIDC Authorization Code + PKCE) | Siempre disponible si hay IdP externo configurado |
| `POST /auth/login` | Local (ASP.NET Core Identity) | Disponible mientras `AllowLocalAuthentication = true` en config |
| `POST /auth/refresh` | Ambos | Rotación de refresh token con detección de reuso |
| `POST /auth/logout` | Ambos | Revoca refresh token; agrega JWT a revocation list si sigue vigente |
| `GET /users/me` | Ambos | Devuelve perfil completo del usuario autenticado (nombre, email, roles) — datos que NO van en claims |
| `GET /users/{id}/contracts` | Ambos | Resolución on-demand cuando `contracts_truncated = true` (§7.4.2) |

#### 19.8.4 Configuración operativa (flags)

La shared library `Anh.Gop.Shared.Auth` expone las siguientes opciones de configuración (`GopIdentityOptions`):

| Flag | Default | Descripción |
|---|---|---|
| `AllowLocalAuthentication` | `true` (V1, hasta confirmación ANH) | Habilita `/auth/login` para credenciales locales. Al cerrarse, sólo queda flujo federado. |
| `AllowFederatedAuthentication` | `true` si hay IdP configurado | Habilita `/auth/login-federated`. |
| `ExternalIdp.Provider` | — | `"AzureAd"` \| `"Adfs"` \| `"Keycloak"` \| `"GovCo"` \| `"None"` |
| `ExternalIdp.Authority` | — | OIDC discovery endpoint |
| `ExternalIdp.ClientId` / `ClientSecret` | Vault | Credencial del RP |
| `Jwt.SigningKey` | Vault | Clave privada RS256 de `gop.identity` |
| `Jwt.AccessTokenLifetime` | `00:30:00` | 30 min default |
| `Jwt.RefreshTokenLifetime` | `08:00:00` | 8 h absolute |
| `TechnicalAccounts` | `[]` | Lista explícita de usernames locales que sobreviven a desactivación de local auth |

**Nota de despliegue:** los valores anteriores se inyectan por variables de entorno / Key Vault (ver §12.7). El `StartupSecretsValidator` falla el arranque si detecta placeholders en `SigningKey` o `ClientSecret`.

---

## 20. Glosario

- **Aggregate Root:** entidad raíz de un agregado DDD. Punto de entrada para modificar el agregado.
- **Always Encrypted:** feature de SQL Server 2022 que cifra columnas sensibles con claves que nunca viajan al motor de BD, sólo al cliente autorizado. Usado en GOP para datos `[DataClassification(Level ≥ 3)]` (ver §12.8).
- **Antiforgery / CSRF Token:** mecanismo que previene Cross-Site Request Forgery exigiendo un token adicional (no cookie) en requests de mutación desde navegadores con sesión basada en cookie. APIs JWT-Bearer están exentas (ver §12.4).
- **Audit.NET:** librería open-source (MIT) para captura automática de auditoría en .NET. En GOP se usa con el paquete `Audit.EntityFramework.Core` para interceptar cambios de `DbContext` y persistirlos en `audit.AuditLog` sin que el código de dominio deba conocer el log (ver §11.6).
- **Bounded Context:** contexto con su propio lenguaje, modelo y lógica. En GOP coincide con un microservicio y un esquema de BD.
- **Capability:** permiso fino asignable a un rol.
- **Clean Architecture:** estilo arquitectónico con capas concéntricas (Domain → Application → Infrastructure → Presentation) donde las dependencias apuntan siempre hacia adentro. Base estructural de todos los microservicios GOP (ver §5).
- **CQRS:** Command Query Responsibility Segregation. Separación de modelos de escritura (commands) y lectura (queries). En GOP se implementa con dispatchers propios en `Anh.Gop.Shared.Cqrs` (ver §10.1 y §19.7).
- **Data Classification (Niveles 0–4):** esquema de clasificación de sensibilidad del dato (`Public`, `Internal`, `Confidential`, `Reserved`, `Restricted`). Define auditoría extra y policies de exposición (ver §7.10).
- **DDD:** Domain-Driven Design.
- **Dispatcher:** componente que resuelve el handler correcto para un command/query y lo invoca. En GOP se implementa en `Anh.Gop.Shared.Cqrs` sin dependencias externas.
- **Decorador (pipeline behavior):** clase que envuelve un dispatcher o handler para aplicar cross-cutting concerns (validación, logging, autorización) sin modificar el handler.
- **FSM:** Finite State Machine.
- **IdP (Identity Provider):** sistema responsable de autenticar usuarios y emitir tokens que prueban su identidad. En GOP, el IdP externo es el designado por ANH (típicamente Active Directory corporativo). Ver §7.1.1.
- **Host Tenant / Regulator-as-Host:** en el patrón multitenant de GOP, el tenant privilegiado es **ANH** (el regulador), no un operador. ANH actúa como host — administra catálogos, define workflows, fiscaliza — y los operadores son tenants consumidores (ver §6.1).
- **ETag (Entity Tag):** identificador opaco asociado a la versión de un recurso, emitido en el header `ETag` de la response. El cliente lo reenvía en `If-Match` en un PUT/PATCH posterior para habilitar concurrencia optimista — si la versión en servidor ya cambió, la operación falla con `412 Precondition Failed`. En GOP se deriva de `RowVersion` / `xmin` (ver §9.6, §11.5).
- **Idempotencia:** propiedad de una operación por la cual aplicarla múltiples veces produce el mismo resultado.
- **Idempotency-Key:** header HTTP enviado por el cliente en requests POST críticos (crear pozo, radicar forma, firmar, aprobar) que permite al servicio detectar reintentos y retornar la misma respuesta sin re-ejecutar la operación. Se persiste en tabla de claves usadas por N horas (ver §9.7).
- **OIDC (OpenID Connect):** capa de identidad sobre OAuth 2.0 (RFC 6749) que permite a una aplicación (Relying Party) verificar la identidad de un usuario autenticado por un IdP y obtener información básica de perfil vía `id_token`. Protocolo usado en §7.1.1 para la federación con el IdP de ANH.
- **OWASP ASVS (Application Security Verification Standard):** estándar OWASP con controles verificables por nivel de assurance (L1/L2/L3). En GOP se usa como referencia normativa para autenticación y gestión de sesión (§7.12.2).
- **OWASP Top 10:** lista priorizada de las 10 categorías de riesgo de seguridad más críticas en aplicaciones web, mantenida por OWASP. GOP aplica OWASP 2021 (ver §12).
- **ProblemDetails:** formato estándar RFC 7807 para respuestas de error HTTP.
- **PKCE (Proof Key for Code Exchange):** extensión de OAuth 2.0 (RFC 7636) que protege el flujo Authorization Code contra interceptación, requerida en clientes públicos como SPAs. Aplicada en el flujo federado (§19.8.1).
- **Rate Limiting:** control que limita la frecuencia de requests por IP o identidad para prevenir abuso, brute-force y DoS. En GOP se aplica en dos capas: nginx por IP (§17.1) y app por usuario vía policies predefinidas (§12.5).
- **Relying Party (RP):** en el modelo OIDC, la aplicación que delega autenticación en un IdP externo y consume el `id_token` para verificar la identidad del usuario. En GOP, `gop.identity` actúa como RP frente al IdP de ANH (§7.1.1).
- **Result Pattern:** patrón donde los métodos devuelven un `Result<T>` (éxito con valor o fallo con error estructurado) en lugar de lanzar excepciones para flujos de error esperados (ver §10.2 y plantillas §19.3/§19.4). Las excepciones quedan reservadas para fallos técnicos inesperados.
- **SBOM (Software Bill of Materials):** inventario firmable de todas las dependencias de un artefacto, en formato estándar (CycloneDX en GOP). Generado en CI y adjunto al release (ver §12.12).
- **Security Headers:** conjunto de HTTP response headers que endurecen el comportamiento del navegador contra XSS, clickjacking, MIME sniffing y leaks (`HSTS`, `X-Content-Type-Options`, `X-Frame-Options`, `Referrer-Policy`, `Permissions-Policy`, `CSP`). Emitidos por la shared library `Anh.Gop.Shared.Security` (ver §12.2).
- **SSRF (Server-Side Request Forgery):** vulnerabilidad donde un atacante induce al servidor a hacer requests HTTP a destinos no intencionados (p.ej. metadata cloud `169.254.169.254`, IPs internas). Mitigado por `SsrfProtectionHandler` en todo cliente HTTP saliente (ver §12.10).
- **Token Exchange (RFC 8693):** patrón OAuth 2.0 que describe el intercambio de un token emitido por un Authorization Server externo por un token emitido por otro Authorization Server con claims de dominio propios. Patrón central del modelo de identidad GOP (§7.1.1, §19.8).
- **Specification Pattern:** patrón DDD para encapsular reglas de filtrado o invariantes de dominio en objetos componibles. En GOP se usa en repositorios para construir queries complejas sin filtrar en memoria (ver §10.7).
- **Tenant:** en GOP, cada operador es un tenant.

---

*Fin del CONSTITUTION-ba.md v1.10*

---

## Control de cambios v1.9 → v1.10

**Ajuste de agilidad — calibración de imperativos sin sacrificar el núcleo duro:**

El diagnóstico interno identificó que la ratio MUST/PROHIBIDO vs SHOULD/MAY (≈6:1) mezclaba invariantes arquitectónicos con decisiones operativas, generando fricción innecesaria. Esta versión recalibra sin modificar ningún invariante de seguridad, multitenant, Clean Architecture o contratos de API.

- **§0 — nueva §0.1 "Política unificada de ADR"** — ADR obligatorio solo para: modificar este documento, transgredir §18, cambiar stack (§4) o dependencia fuera del aprobado, relajar §12 (seguridad), introducir patrón transversal nuevo. El resto se justifica en el PR. Redefinido SHOULD: basta justificación en PR (no ADR) salvo que el § específico diga lo contrario.
- **§2.2** — ADR solo para cambios mayores del documento. Cambios menores (ejemplos, aclaraciones) sin ADR.
- **§2.3** — Excepciones a MUST fuera de §12/§18 y a cualquier SHOULD se justifican en el PR, sin ADR. ADR solo se mantiene para excepciones a §12/§18.
- **§4.1 Licencias** — eliminado el MUST del comentario manual de licencia en cada PR (CI ya lo valida). Degradado a MAY como cortesía para dependencias poco conocidas.
- **§8.2** — cambiar clasificación de entidades ya productivas requiere ADR (impacto real en contratos); la clasificación inicial solo se documenta en el modelo.
- **§8.3.0 — nueva sección "Principio rector — crudo con IDs; excepción calificada en listados"** — consagra explícitamente como principio rector que el backend emite DTOs crudos con IDs y el frontend hidrata (alineado con `CONSTITUTION-fe.md` §22). La inclusión de labels precomputados queda formalmente limitada a los `SummaryDto` de listados, como excepción justificada para evitar hidratación masiva en tablas. El Detail **nunca** lleva labels.
- **§10.8.10.1 — nueva sección "Cómo mantener una regla hardcodeada de forma sostenible"** — 7 prácticas aplicables cuando la complejidad obliga a dejar la regla en código (back y/o front) en lugar de externalizarla al formato declarativo: ubicación dedicada con naming propio, interfaz tipada (patrón Strategy-ready), test por rama, encabezado de procedencia normativa, pareja BE↔FE declarada con el backend como fuente de verdad, registro en catálogo interno (`BUSINESS-RULES.md`), revisión periódica para evaluar externalización futura. Convierte la excepción de §10.8.10 de un "documentar y seguir" genérico en una pauta concreta de mantenibilidad.
- **§8.3.1 SummaryDto** — composición de campos cambia de MUST a SHOULD (nombre y tipo siguen siendo MUST). Desviaciones puntuales se justifican en el PR.
- **§8.5 Endpoints especializados** — ADR solo cuando el DTO especializado introduce patrón recurrente nuevo; casos puntuales se documentan en el PR.
- **§9.1 Versionado** — v1 vivo 6 meses tras v2 degradado de MUST a SHOULD. Se puede acortar si los consumidores han migrado (documentado en el PR de retiro).
- **§9.3 Paginación** — MUST: existe tope superior de `pageSize`. Valores concretos (50/100/500) degradados a SHOULD configurables por entorno, sin ADR.
- **§9.8 Documentación** — MUST de XML docs acotado a **endpoints públicos de negocio**. DTOs con nombres autodocumentados → SHOULD. Endpoints de infraestructura (health, metrics, swagger) exentos.
- **§10.5.3 Evolución a V2** — versionado independiente de integration events se activa cuando se introduzca event bus; migración gradual degradada a SHOULD.
- **§11.4 Transacciones** — transacciones cortas degradado a SHOULD. Umbral de 5s pasa a ser referencia de monitoring/alerta, no bloqueante en PR.
- **§12.5 Rate limiting** — MUST: todo endpoint con policy asignada. Valores concretos de las 4 policies degradados a SHOULD configurables por entorno; desactivar la policy sigue requiriendo ADR.
- **§12.6 Password policy** — separada en principios MUST (existe política, historial, lockout, rotation, MFA) y valores SHOULD (12 chars, historial 5, 90 días, etc.) alineables con política operativa ANH sin tocar el documento.
- **§14.1 Cobertura** — 80% degradado a SHOULD; MUST es que existan tests para los comportamientos críticos de §14.2–14.5.
- **§14.3 Integration tests** — "nunca mockear EF" relajado: SHOULD usar SQL real; MAY mockear `DbContext` en unit tests puntuales cuyo comportamiento es independiente del ORM.
- **§18 Líneas rojas** — reformulada como invariantes arquitectónicos y de seguridad:
  - §18.2 Datos: `SELECT *` removido de líneas rojas (pasa a anti-patrón SHOULD en §11.1).
  - §18.3 Seguridad: `[AllowAnonymous]` admite excepción explícita para endpoints de infraestructura listados en shared lib.
  - §18.3 Seguridad: `Newtonsoft.Json` reformulado — se prohíbe introducirlo como serializer primario; dependencias transitivas y SDKs de terceros no activan el bloqueo.
  - §18.4 API: prohibición de endpoints sin OpenAPI acotada a endpoints públicos de negocio.
  - §18.5 Testing: sección reemplazada por referencia a políticas de CI/analyzers (las reglas de "no push con tests rotos" y "no `#pragma` sin comentario" viven en pipeline/analyzers, no como líneas rojas arquitectónicas).

**Impacto global:**

- MUST/PROHIBIDO/NUNCA pasan de ~33 standalone + 50 NUNCA a ~25 MUST + ~40 NUNCA. SHOULD sube de ~20 a ~35. Ratio baja de ≈6:1 a ≈2:1.
- Núcleo no negociable intacto: filtro de tenant, ProblemDetails/no-envelope, Result Pattern, Clean Architecture dependency rule, Audit.NET local, hardening OWASP (§12), líneas rojas de datos y seguridad.
- Decisiones tácticas (umbrales, plazos, coberturas, composición de DTOs) pasan a ser guías con valores iniciales configurables, liberando al equipo de tramitar ADRs triviales.

---

## Control de cambios v1.8 → v1.9

**Desacoplamiento de reglas de formularios y lógica de negocio:**

- **§10.8 — nueva sección "Desacoplamiento de reglas de formularios y lógica de negocio"** con 11 subsecciones:
  - §10.8.1 Propósito — enfoque base pragmático, no camisa de fuerza; casos complejos MAY quedar hardcodeados con decisión documentada.
  - §10.8.2 Dos principios rectores:
    - **P1** Reglas de formulario servidas desde el back, evaluadas en memoria por el front (single request al cargar, sin chatter).
    - **P2** Reglas de negocio desacopladas del código transaccional (handlers coordinan, reglas en componentes dedicados).
  - §10.8.3 Ubicación: `Application/Rules/{forms,business,schemas}` versionado en Git, no en BD (salvo admin no-técnico vía UI).
  - §10.8.4 Formato preferido: JSON (SHOULD); YAML/C# planas (MAY).
  - §10.8.5 Endpoint estándar `GET /api/v1/forms/{form-key}/rules`. El backend aplica las mismas reglas al validar payload — no se confía en el cliente.
  - §10.8.6 Qué va / qué no va en archivos de reglas. Heurística: "datos + referencias a códigos conocidos" → archivo; "expresión a evaluar" → código.
  - §10.8.7 Referencias a código por **código simbólico** (`EXPLORATORY_FIELD_PLACEHOLDER`) con implementación C# tipada (`IPlaceholderResolver`). PROHIBIDO expresiones dinámicas tipo `"fieldA > 5"`.
  - §10.8.8 Validación al arranque contra JSON schema; servicio no arranca si hay errores.
  - §10.8.9 Alineación Clean/hexagonal: archivos como recursos, motor en Infrastructure (`Anh.Gop.Shared.RulesEngine`), dominio consume vía puerto `IRulesProvider`.
  - §10.8.10 Cuándo NO aplicar (invariantes de dominio, norma legal estable, caso único, complejidad que requeriría mini-lenguaje).
  - §10.8.11 Líneas rojas: NUNCA motor Turing-completo, NUNCA cargar sin validar schema, NUNCA duplicar lógica back/front, NUNCA leer archivos desde Domain/handlers.

---

## Control de cambios v1.7 → v1.8

**Uniformidad de contratos de API — respuestas de éxito y serialización:**

- **§9.6 — nueva sección "Respuestas de éxito — POST / PUT / PATCH / DELETE"**. Define por convención vinculante:
  - Principio **sin envelope universal** (PROHIBIDO `{status, message, data}` y similares).
  - Tabla de convención status + body + headers por verbo y escenario (create, sync-action, async-action, bulk, replace, partial, hard-delete, soft-delete).
  - 7 reglas MUST vinculantes (201+Location al crear, 204 sin body al hard-delete, 200+DTO al soft-delete, ETag en PUT/PATCH, paginación en `X-Pagination`, ProblemDetails RFC 7807 para todo error).
  - Helpers del Result Pattern (`ToCreatedResult`, `ToOkResult`, `ToAcceptedResult`, `ToNoContentResult`) en `Anh.Gop.Shared.Api` — el desarrollador no elige status code manualmente.
- **§9.7 Idempotencia** y **§9.8 Documentación** — renumeradas (antes §9.6 y §9.7).
- **§9.9 — nueva sección "Convenciones adicionales de serialización"** con 6 subsecciones:
  - §9.9.1 Fechas y zona horaria (ISO 8601 + UTC en BD, conversión en frontend).
  - §9.9.2 Serialización de enums (string, no entero).
  - §9.9.3 Nomenclatura JSON (camelCase).
  - §9.9.4 Null vs. omitido (response omite `null`; PATCH distingue omitido = "no cambiar" de `null` = "limpiar").
  - §9.9.5 Uploads/downloads (`multipart/form-data`, 50 MB default, validación MIME real, blob storage con `BlobReference`, streaming en descargas grandes, presigned URLs preferidas).
  - §9.9.6 Exportes tabulares (CSV con BOM UTF-8, XLSX `ClosedXML`/`EPPlus`, PDF `QuestPDF`, async + `202` para > 10 000 filas).
- **§18.4 API** — añadidas 3 líneas rojas: prohibición de envelopes propietarios, prohibición de fechas/enums/naming no canónicos en contratos públicos, prohibición de base64 de binarios en JSON salvo excepción documentada.
- **§19.6 Plantilla de Controller** — actualizada: `CreateWell` ahora usa `ToCreatedResult` y devuelve `WellDetailDto` (no `{id}`); añadidos ejemplos de `ReplaceWell` (PUT 200+ETag) y `DeleteWell` (DELETE 204).
- **§20 Glosario** — añadidos: `ETag (Entity Tag)`, `Idempotency-Key`.

---

## Control de cambios v1.6 → v1.7

**Modelo de identidad formalizado — Token Exchange (RFC 8693) + usuarios locales (ASP.NET Core Identity):**

- **§7.1** — precisión del wording: SSO contra "IdP externo designado por ANH" (no exclusivamente AD corporativo); MFA obligatoria para usuarios gestionados en el IdP externo.
- **§7.1.1** — nueva sección **"Modelo de identidad — IdP federado + JWT de aplicación (Token Exchange)"**. Documenta explícitamente:
  - `gop.identity` actúa como Relying Party (RP) OIDC frente al IdP externo y como Authorization Server propio que emite JWT de aplicación con claims de dominio.
  - Separación autenticación (IdP externo) vs. autorización de dominio (`gop.identity`).
  - Referencia a RFC 7519, RFC 8693, OWASP ASVS v4.
  - Prohibición explícita de propagar `id_token` / `access_token` del IdP externo a microservicios GOP.
- **§7.1.1.1** — nueva subsección **"Usuarios locales (modo transitorio y fallback operativo)"**:
  - Habilitados por ASP.NET Core Identity para: testing funcional inicial (pre-federación), usuarios fuera del IdP de ANH (operadores/contratistas sin cuenta corporativa), cuentas técnicas y bootstrap admin.
  - Columna `AuthenticationSource` (`External` | `Local`) en `User` para trazabilidad.
  - JWT de aplicación **idéntico** en forma y claims sea el origen federado o local.
  - Flag `AllowLocalAuthentication` en configuración; evolución a desactivación cuando ANH confirme IdP definitivo y cobertura total.
  - `ANH_Admin` local prohibido salvo bootstrap / cuentas técnicas documentadas por ADR.
- **§7.2.1** — nueva **"Claims prohibidos en el JWT"**: PII innecesaria, datos sensibles, tokens de terceros, secretos, mutables de alta frecuencia, estructuras no acotadas. El nombre visible se obtiene por `/users/me`, no por claim.
- **§7.12** — nueva sección **"Cumplimiento normativo del modelo de identidad"** con 7 subsecciones y mapeo explícito a:
  - OWASP ASVS v4 (capítulos 2 y 3).
  - OWASP JWT Cheat Sheet.
  - OWASP Top 10 (2021) por categoría.
  - ISO/IEC 27001/27002:2022 — controles A.5.15, A.5.16, A.5.17, A.5.18, A.8.2, A.8.3, A.8.5, A.8.15, A.8.24, A.8.28.
  - RFCs aplicables: 7519, 7515, 6749, 7636, 8693, 6750 + OIDC Core 1.0.
  - Controles residuales fuera del alcance del backend (confirmación de IdP, matriz de clasificación, plan de respuesta a incidentes).
- **§19.8** — nuevo apéndice **"Flujo de Token Exchange — IdP externo + JWT de aplicación"** con:
  - Diagrama de secuencia en modo federado (Authorization Code + PKCE).
  - Diagrama de secuencia en modo local (ASP.NET Core Identity).
  - Tabla de endpoints de `gop.identity`.
  - Tabla de flags de configuración operativa (`GopIdentityOptions`).
- **§20 Glosario** — añadidos: `IdP`, `OIDC`, `OWASP ASVS`, `PKCE`, `Relying Party (RP)`, `Token Exchange (RFC 8693)`.

---

## Control de cambios v1.5 → v1.6

**Cumplimiento OWASP Top 10 (2021) — hardening nativo y automático:**

- **§12 renombrada** de "Validación y sanitización de entrada" a **"Validación, sanitización y hardening de seguridad"**. Contenido previo promovido a §12.1.
- **§12.0 — Principio rector** añadido: "seguro por defecto, invisible al desarrollador". Todos los controles OWASP automatizables residen en la nueva shared library `Anh.Gop.Shared.Security` y se activan con una sola línea en `Program.cs`.
- **§12.2 — Security Headers** (A05): HSTS, `X-Content-Type-Options`, `X-Frame-Options`, `Referrer-Policy`, `Permissions-Policy`, CSP (sólo HTML). Aplicados por middleware global.
- **§12.3 — HTTPS y cookies seguras** (A02, A05): `UseHttpsRedirection` + `UseHsts`, cookies con `HttpOnly`/`Secure`/`SameSite=Strict`.
- **§12.4 — Antiforgery / CSRF** (A01, A03): filtro global para mutaciones con cookie; APIs JWT exentas.
- **§12.5 — Rate limiting** (A04, A07): policies predefinidas (`auth-strict`, `api-standard`, `export-heavy`, `read-standard`) vía `Microsoft.AspNetCore.RateLimiting`.
- **§12.6 — Password, lockout y MFA** (A07): longitud 12, historial 5, expiración 90 días, lockout dual (usuario + IP), MFA obligatorio para roles sensibles.
- **§12.7 — Secretos** (A02, A05): variables de entorno / Key Vault / DPAPI; `StartupSecretsValidator` falla el arranque si detecta placeholders.
- **§12.8 — Criptografía** (A02): PBKDF2 default, Always Encrypted para nivel ≥3, RS256 para JWT, prohibición de algoritmos débiles enforced por analyzers.
- **§12.9 — Deserialización segura** (A08): `System.Text.Json` único (prohibido `Newtonsoft.Json` sin ADR), `UnmappedMemberHandling=Disallow`, `MaxDepth=32`, polimorfismo sólo con `[JsonDerivedType]`, XXE blindado, `BinaryFormatter` prohibido.
- **§12.10 — SSRF** (A10): `SsrfProtectionHandler` en toda `HttpClient` — allow-list declarativa, bloqueo de IPs privadas/loopback/metadata, validación de redirects. Prohibido `new HttpClient()` directo.
- **§12.11 — Inyección LDAP** (A03): `LdapFilterEncoder` obligatorio; LDAPS/StartTLS obligatorio.
- **§12.12 — Supply chain** (A06, A08): CI falla con CVE High/Critical, SBOM CycloneDX, Dependabot/Renovate, signature validation, Trivy para Docker.
- **§12.13 — Analyzers** de seguridad warning-as-error en `Directory.Build.props` raíz (CA3xxx, CA5xxx, SecurityCodeScan, Roslynator).
- **§12.14 — Error handling** (A05, A09): ProblemDetails sanitizado, stack y detalle técnico sólo en `Development`.
- **§12.15 — Mapa** explícito OWASP Top 10 → sección del documento.

**Cambios colaterales:**

- **§4 Stack** — nuevas filas: hardening de seguridad, rate limiting, analyzers de seguridad, supply chain scan.
- **§5.4 Shared Libraries** — nueva fila `Anh.Gop.Shared.Security`.
- **§17.1 nginx** — aclarado que los security headers los emite la app (no nginx); HSTS puede duplicarse como defensa redundante; rate limit de IP en nginx es complementario al rate limit por identidad en la app.
- **§18.3 Seguridad** — 7 líneas rojas nuevas: `new HttpClient()` directo, `Newtonsoft.Json` sin ADR, algoritmos débiles, secretos en config/logs, filtros LDAP interpolados, Developer Exception Page fuera de Development.
- **§20 Glosario** — añadidos: `Always Encrypted`, `Antiforgery / CSRF Token`, `OWASP Top 10`, `Rate Limiting`, `SBOM`, `Security Headers`, `SSRF`.

---

## Control de cambios v1.4 → v1.5

**Correcciones de wording y consistencia:**
- §4 — "Obligatorio por Estándar" → "Definido por Estándar de Codificación" (Runtime .NET, tSQLt, Analyzers).
- §7.3 — ".NET Identity" → "ASP.NET Core Identity" (alineación con §4 y §5.2).
- §17.2 — reescrita como prosa descriptiva ("Funciones fuera del alcance de nginx"); eliminado el encabezado "NO:" que se leía como prohibición.

**Ambigüedades resueltas:**
- §7.10 — `audit.DataAccessLog` / `audit.ExportLog` sustituidos por eventos especializados en `audit.AuditLog` (`EventType='DataAccess'` / `'Export'` + payload en `AdditionalContextJson`). Añadida nota de modelo.
- §11.6.6 — añadida topología de BD V1 (instancia física compartida, aislamiento por permisos de esquema + `INSERT` acotado a `audit.AuditLog`). V2+ como ADR pendiente.
- §10.5.2 — la fila `gop.audit | (consumidor)` se movió fuera de la tabla como nota de excepción explícita.
- §19.5.1 — añadida nota sobre reevaluación Tipo A → B para `WellStateTransition` y entidades A con >50 ítems.

**Líneas rojas (§18) ampliadas:**
- §18.1 — "NUNCA un servicio llama a `gop.audit` para escribir auditoría".
- §18.3 — "NUNCA inserción manual a `audit.AuditLog` desde código; siempre vía Audit.NET".

**Glosario (§20):** añadidos `Audit.NET`, `Clean Architecture`, `Data Classification`, `Host Tenant / Regulator-as-Host` (alineado con §6.1), `Result Pattern`, `Specification Pattern`.
