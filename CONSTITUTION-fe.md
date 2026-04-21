# Principios Arquitectónicos para Angular Empresarial — GOP 360°

**Versión:** 1.1
**Fecha:** Abril 2026
**Estado:** Vigente para V1
**Alineación:** CONSTITUTION-ba.md v1.8 (contratos de API, identidad, uploads)

> **Naturaleza de este documento.** Es un documento **normativo de principios, patrones y líneas rojas** para el frontend de GOP 360°. Los bloques de código, DTOs, dominios y componentes son **ejemplos de referencia**; los objetos concretos se definen en la spec de cada feature (SDD §15). Si una spec necesita apartarse de un patrón, debe registrar un ADR.

---

## 1. Conceptos Core
* **Domain-Driven Design (DDD):** La aplicación se divide en "Dominios" de negocio (Pozos, Producción, Operaciones, Admin) en lugar de capas técnicas. Cada dominio es independiente y encapsula sus propios modelos, servicios, estado y componentes.
* **Standalone Components:** Eliminación de `NgModules` para reducir la complejidad y mejorar el tree-shaking (carga más rápida).
* **Smart vs. Dumb Components:** Separación estricta entre componentes de presentación (UI reutilizable, *Dumb*) y componentes contenedores (Páginas conectadas a servicios/estado, *Smart*).
* **Lazy Loading Extremo:** Cada dominio de negocio se carga bajo demanda a través del enrutador, reduciendo el tamaño del bundle inicial.
* **Gestión de Estado Centralizada/Local:** Uso de **NgRx** (para estado global complejo como catálogos o sesión) y **Signals** (para estado local reactivo en componentes como el Wizard de creación de pozos).

---

## 2. Ejemplo de Scaffolding (Estructura de Carpetas)

```text
/                                        # Raíz del repositorio
├── CONSTITUTION.md                      # Reglas arquitectónicas globales e inmutables (este archivo)
├── CLAUDE.md                            # Ancla de contexto SDD para Claude Code
├── GEMINI.md                            # Ancla de contexto SDD para Gemini
├── blueprint.md                         # Mapa funcional del sistema (rutas, dominios, flujos)
│
├── specs/                               # ── DOCUMENTACIÓN SDD (ver §15) ──────────────────────────
│   └── features/                        # Un directorio por cada feature a desarrollar
│       ├── 001-auth/                    # NNN = secuencial │ nombre = slug de la feature
│       │   ├── spec.md                  #   El "QUÉ": historias de usuario y criterios de aceptación
│       │   ├── plan.md                  #   El "CÓMO": árbol de archivos, componentes, estado
│       │   └── tasks.md                 #   El "CUÁNDO": tareas atómicas con checkboxes
│       └── NNN-nombre-feature/
│           └── ...
│
└── src/                                 # ── CÓDIGO FUENTE ────────────────────────────────────────
    ├── app/
    │   ├── core/                        # 1. CORE: Singleton, configuraciones y seguridad
    │   │   ├── auth/                    # Lógica de autenticación, interceptores de token
    │   │   ├── guards/                  # Protecciones de rutas (RBAC)
    │   │   ├── http/                    # Interceptores de errores globales, manejo de API
    │   │   └── layout/                  # Shell del layout autenticado + config de nav items (Sidebar/Header vienen del UIKit)
    │   │
    │   ├── shared/                      # 2. SHARED: UI genérica, modelos y servicios transversales
    │   │   ├── models/                  # Interfaces/tipos usados por 2+ dominios (Well, Operator, UserRole)
    │   │   │   ├── well.model.ts        #   → model + dto + mapper por entidad compartida
    │   │   │   ├── well.dto.ts
    │   │   │   ├── well.mapper.ts
    │   │   │   └── index.ts             #   → barrel export del grupo
    │   │   ├── services/                # Servicios transversales sin lógica de negocio propia
    │   │   │   ├── catalog.service.ts   #   → catálogos read-only (tipos de pozo, operadoras)
    │   │   │   └── index.ts
    │   │   ├── locale/                  # Textos globales (Layout, Sidebar, Errores genéricos) -> locale.ts
    │   │   ├── ui/                      # Componentes "Dumb" (Botones, Modales, Tooltips, Tablas)
    │   │   ├── directives/              # Directivas estructurales o de atributos
    │   │   ├── pipes/                   # Transformadores de datos (Fechas, Monedas, Formatos)
    │   │   └── utils/                   # Funciones puras (ej. generador de UWI genérico)
    │   │
    │   ├── domains/                     # 3. DOMAINS: Lógica de negocio agrupada por contexto
    │   │   │
    │   │   ├── wells/                   # Dominio: Gestión de Pozos
    │   │   │   ├── components/          # Componentes compartidos del dominio (ej. WellStatusBadge)
    │   │   │   ├── features/            # Funcionalidades completas (Smart Components)
    │   │   │   │   ├── well-create/     # Feature: Wizard de creación (F101)
    │   │   │   │   │   ├── components/  # Componentes internos exclusivos de esta feature
    │   │   │   │   │   ├── locale.ts    # Textos/Constantes locales de la feature
    │   │   │   │   │   └── well-create.component.ts
    │   │   │   │   ├── well-manage/     # Feature: Explorador jerárquico
    │   │   │   │   └── well-info/       # Feature: Infografía del pozo
    │   │   │   ├── models/              # Solo modelos PRIVADOS del dominio (WellHistory, Trajectory)
    │   │   │   ├── services/            # Solo servicios PRIVADOS del dominio (wells-api.service.ts)
    │   │   │   ├── store/               # Estado del dominio (NgRx Feature State o Signals)
    │   │   │   └── wells.routes.ts      # Rutas lazy-loaded del dominio
    │   │   │
    │   │   ├── operations/              # Dominio: Operaciones y Formas 100
    │   │   │   ├── features/
    │   │   │   │   ├── idop/            # Informe Diario de Perforación
    │   │   │   │   └── forms-100/       # Bandeja de Formas 100
    │   │   │   ├── models/              # Solo modelos PRIVADOS: IDOP, Forma100
    │   │   │   ├── services/            # Solo servicios PRIVADOS: operations-api.service.ts
    │   │   │   └── operations.routes.ts
    │   │   │
    │   │   ├── production/              # Dominio: Producción y Formas 200
    │   │   │   ├── features/
    │   │   │   │   └── fiscalization/   # Fiscalización Volumétrica
    │   │   │   ├── models/              # Solo modelos PRIVADOS: FiscalizationReport
    │   │   │   ├── services/            # Solo servicios PRIVADOS: production-api.service.ts
    │   │   │   └── production.routes.ts
    │   │   │
    │   │   └── admin/                   # Dominio: Administración y Seguridad
    │   │       ├── features/
    │   │       │   ├── users/           # Gestión de usuarios y roles
    │   │       │   └── audit-logs/      # Trazabilidad
    │   │       ├── models/              # Solo modelos PRIVADOS de admin
    │   │       ├── services/            # Solo servicios PRIVADOS: admin-api.service.ts
    │   │       └── admin.routes.ts
    │   │
    │   ├── app.component.ts             # Componente raíz
    │   ├── app.routes.ts                # Enrutador principal (Carga los dominios por Lazy Loading)
    │   └── app.config.ts                # Proveedores globales (HttpClient, Router, Store)
    │
    ├── environments/                    # Variables de entorno (dev, qa, prod)
    ├── assets/                          # Imágenes, iconos
    └── styles/                          # Estilos globales, variables CSS, configuración de Tailwind
```

> **Referencia:** Los dominios, features, modelos y servicios mostrados en esta estructura son ilustrativos. La estructura real de cada dominio se define y expande conforme a las especificaciones funcionales de cada elemento a desarrollar. No crear carpetas ni archivos anticipándose a funcionalidades no especificadas.
>
> La carpeta `specs/` y su contenido (`spec.md`, `plan.md`, `tasks.md`) siguen la metodología SDD descrita en el **§15 — Spec-Driven Development**.

---

## 3. Descripción de las Capas (Por qué esta estructura)

> Esta sección describe las capas del **código fuente** (`src/`). Para la estructura de documentación SDD (`specs/features/`), ver **§15 — Spec-Driven Development**.

### 1. `core/` (El Motor)
Aquí reside todo lo que la aplicación necesita instanciar **una sola vez** al arrancar.
* **Ejemplo en tu app:** El `AuthContext` actual pasaría a ser un `AuthService` inyectado en el `core/`. Los interceptores HTTP manejarían automáticamente la inyección del token JWT y la captura de errores globales (ej. mostrar un toast si la API de la ANH falla).

### 2. `shared/` (La Caja de Herramientas)
Contiene todo lo que es transversal a más de un dominio pero no pertenece al ciclo de vida del arranque de la app. Esto incluye UI reutilizable, **modelos e interfaces compartidas**, **servicios de catálogo o utilidad**, textos globales y funciones puras.
* **Regla de oro de `shared/`:** Un archivo en `shared/` no debe saber qué es un "IDOP" o una "Forma 101" — solo conoce abstracciones generales como `Well`, `Operator` o `UserRole` que múltiples dominios necesitan.
* **`shared/models/`:** Interfaces y DTOs que 2 o más dominios consumen. Incluyen su mapper y un `index.ts` de barrel export.
* **`shared/services/`:** Servicios sin lógica de negocio propia: catálogos read-only, utilidades HTTP genéricas. **No** incluye servicios que escriben datos de un dominio específico.
* **Ejemplo en tu app:** `Well` y `WellStatus` van en `shared/models/` porque los usan `wells/`, `operations/` y `production/`. Pero `WellHistory` va en `domains/wells/models/` porque solo lo usa el dominio de pozos.

### 3. `domains/` (El Corazón del Negocio)
Esta es la mejora más grande frente a tu arquitectura actual. En lugar de tener una carpeta `pages/` gigante con archivos de diferentes módulos mezclados, cada dominio es un ecosistema cerrado compuesto por **Features**.
* **Features (Funcionalidades):** Reemplazan el concepto tradicional de "Páginas". Una *feature* encapsula una funcionalidad completa (ej. `well-create`). Cada feature contiene sus propios componentes internos, su lógica específica y, crucialmente, **su propio archivo de constantes/textos locales**.
* **Cada feature de código tiene su artefacto SDD correspondiente** en `specs/features/NNN-nombre/` (ver **§15**). El `plan.md` de la feature es el documento de referencia que define exactamente qué archivos viven dentro de ella.
* **Ejemplo en tu app:** Si un desarrollador necesita arreglar un bug en la creación del UWI o en la validación de la Forma 101, no tiene que buscar en `src/utils`, luego en `src/pages`, y luego en `src/store`. Todo lo relacionado con pozos vive exclusivamente dentro de `src/app/domains/wells/features/well-create/`.
* **Escalabilidad:** Si el día de mañana el módulo de "Producción" crece demasiado, esta arquitectura permite extraer la carpeta `production/` a un micro-frontend o a una librería separada en un monorepo (usando herramientas como Nx).

---

## 4. Gestión de Estado: Cuándo usar Signals vs. NgRx

La regla es clara y no admite interpretación libre: **el scope del estado determina la herramienta**.

| Criterio | Usar |
|---|---|
| Estado vive en un solo componente o feature | Signals |
| Estado debe sobrevivir al destroy del componente | NgRx |
| Estado es compartido entre múltiples dominios | NgRx |
| Estado de sesión (usuario autenticado, permisos) | NgRx |
| Catálogos globales (tipos de pozo, operadoras) | NgRx |
| UI state (loading, modal abierto, paso del wizard) | Signals |

**Regla de oro:** Si un desarrollador tiene duda, usa Signals. Solo escala a NgRx cuando el estado cruce los límites de la feature. NgRx tiene un costo de boilerplate que no se justifica para estado local.

```typescript
// ✅ Signals: estado local de un wizard (vive y muere con el componente)
currentStep = signal(1);
isLoading = signal(false);
formData = signal<Partial<WellForm>>({});

// ✅ NgRx: catálogo de operadoras usado en múltiples dominios
// store/catalogs/catalogs.actions.ts → loadOperators
// store/catalogs/catalogs.selectors.ts → selectOperators
```

### 4.1. Navegación post-acción: exclusivamente en Effects

**Regla:** Los servicios (`AuthService`, `WellsApiService`, etc.) **nunca** inyectan `Router` ni `Store`. Su responsabilidad se limita a llamadas HTTP/mock y transformación de datos. Toda navegación y orquestación posterior a una acción (redirect tras login, redirect tras logout, navegación tras crear un recurso) es responsabilidad exclusiva de los **NgRx Effects**.

```typescript
// ❌ Prohibido: el servicio navega directamente
@Injectable({ providedIn: 'root' })
export class AuthService {
  private router = inject(Router); // ← NUNCA
  logout(): void { this.router.navigate(['/login']); }
}

// ✅ Correcto: el Effect orquesta navegación + side effects
logout$ = createEffect((actions$ = inject(Actions), router = inject(Router)) =>
  actions$.pipe(
    ofType(AuthActions.logout),
    tap(() => router.navigate(['/login'])),
    map(() => AuthActions.logoutSuccess()),
  )
);
```

**Por qué:** Centralizar la navegación en Effects hace el flujo predecible, testeable y trazable desde NgRx DevTools. Si el servicio navega, el componente pierde control sobre cuándo y hacia dónde ocurre el redirect.

---

## 5. Manejo de Errores HTTP (`core/http/`)

Todo el manejo de errores de red es responsabilidad exclusiva de `core/http/`. Los servicios de dominio **nunca** manejan errores HTTP directamente — solo transforman la respuesta exitosa.

> Para desarrollo sin backend disponible, ver **Sección 13 — Mocks HTTP mediante Interceptor**.
> Para el contrato completo de errores del backend GOP, ver **§21.3 ProblemDetails RFC 7807**.

### Formato de error canónico: ProblemDetails (RFC 7807)

**Todo** error que retorna el backend GOP sigue el formato RFC 7807 (ver CONSTITUTION-ba §9.5). El interceptor debe parsear esta estructura como primera opción:

```typescript
// core/http/problem-details.ts
export interface ProblemDetails {
  type?: string;        // URI identificador del tipo de error
  title: string;        // Resumen corto legible (constante por tipo)
  status: number;       // Código HTTP
  detail?: string;      // Descripción específica del caso
  instance?: string;    // URI de la ocurrencia (opcional)
  traceId?: string;     // Correlation ID para soporte
  errors?: Record<string, string[]>; // 422 — validación por campo (RFC 7807 §3.1 ext.)
  // Extensiones propias de GOP (documentadas en CONSTITUTION-ba §9.5):
  code?: string;        // Código de error de dominio (ej. 'WELL_ALREADY_EXISTS')
}

export function isProblemDetails(body: unknown): body is ProblemDetails {
  return !!body && typeof body === 'object'
    && 'title' in body && 'status' in body;
}
```

### Patrón del Interceptor

```typescript
// core/http/error.interceptor.ts

// Mensajes de fallback cuando el backend no retorna ProblemDetails (ej. nginx, proxy, gateway)
const HTTP_ERROR_MESSAGES: Record<number, string> = {
  400: 'Solicitud inválida.',
  403: 'No tienes permisos para realizar esta acción.',
  404: 'El recurso solicitado no fue encontrado.',
  409: 'El recurso fue modificado por otro proceso. Recarga e intenta de nuevo.',
  412: 'El recurso cambió desde que lo cargaste. Recarga para ver la versión actual.',
  422: 'Los datos enviados no son válidos.',
  429: 'Demasiadas solicitudes. Espera un momento e intenta de nuevo.',
  500: 'Error interno del servidor.',
  503: 'Servicio no disponible. Intenta más tarde.',
};
const FALLBACK_MESSAGE = 'Ocurrió un error inesperado.';
const NO_CONNECTION_MESSAGE = 'Sin conexión. Verifica tu red e intenta de nuevo.';

// Prioridad de extracción del mensaje visible:
// 1) ProblemDetails.detail (explicación específica del caso)
// 2) ProblemDetails.title  (resumen del tipo de error)
// 3) Fallback local por código HTTP
// 4) Fallback genérico
function resolveErrorMessage(error: HttpErrorResponse): { summary: string; detail: string } {
  const body = error.error;
  if (isProblemDetails(body)) {
    return {
      summary: body.title ?? 'Error',
      detail:  body.detail ?? HTTP_ERROR_MESSAGES[error.status] ?? FALLBACK_MESSAGE,
    };
  }
  return {
    summary: 'Error',
    detail:  HTTP_ERROR_MESSAGES[error.status] ?? FALLBACK_MESSAGE,
  };
}

// Nota: el interceptor usa inject(ToastService) de appgop-web en lugar de MessageService directo.
// MessageService es la API interna de PrimeNG — ToastService es el wrapper del UIKit que
// garantiza consistencia con la instancia de <app-toast> registrada en app.html.
export const errorInterceptor: HttpInterceptorFn = (req, next) => {
  return next(req).pipe(
    catchError((error: HttpErrorResponse) => {
      const toastService = inject(ToastService);
      const store = inject(Store);

      switch (error.status) {
        case 0:
          // Sin conexión: el backend no responde, no hay mensaje posible
          toastService.error(NO_CONNECTION_MESSAGE, { summary: 'Sin conexión' });
          break;

        case 401:
          // Token expirado / inválido → despacha acción NgRx. El Effect maneja limpieza + navegación.
          store.dispatch(AuthActions.sessionExpired());
          break;

        case 412: {
          // Concurrencia optimista: el recurso cambió (ETag no coincide con If-Match).
          // El componente puede interceptar el error para ofrecer "Recargar" antes de mostrar toast.
          // El interceptor solo muestra el toast si nadie más maneja el error.
          const { summary, detail } = resolveErrorMessage(error);
          toastService.warn(detail, { summary });
          break;
        }

        case 422: {
          // Errores de validación por campo — el formulario los consume via error.error.errors{}
          // El interceptor muestra el título/resumen; el detalle por campo lo pinta el formulario.
          const { summary, detail } = resolveErrorMessage(error);
          toastService.error(detail, { summary });
          break;
        }

        default: {
          const { summary, detail } = resolveErrorMessage(error);
          toastService.error(detail, { summary });
          break;
        }
      }

      return throwError(() => error);
    })
  );
};
```

**Consumo del error en formularios (422):**

```typescript
// El shell/feature hace catchError local para bindear errores por campo al form control
createWell(payload: CreateWellDTO): void {
  this.api.createWell(payload).pipe(
    catchError((err: HttpErrorResponse) => {
      if (err.status === 422 && isProblemDetails(err.error) && err.error.errors) {
        // errors = { "wellName": ["ya existe un pozo con ese nombre"], "uwi": [...] }
        Object.entries(err.error.errors).forEach(([field, messages]) => {
          this.form.get(field)?.setErrors({ server: messages[0] });
        });
      }
      return EMPTY; // el toast global ya se mostró; el formulario ya tiene los errores
    })
  ).subscribe(well => this.store.dispatch(WellActions.createSuccess({ well })));
}
```

### Patrón del Interceptor de Token (Auth)

```typescript
// core/http/auth.interceptor.ts
export const authInterceptor: HttpInterceptorFn = (req, next) => {
  const token = inject(AuthService).getToken();
  if (!token) return next(req);

  const authReq = req.clone({
    setHeaders: { Authorization: `Bearer ${token}` }
  });
  return next(authReq);
};
```

### Regla de los Servicios de Dominio

```typescript
// ✅ Correcto: el servicio solo transforma la respuesta exitosa
getWell(id: string): Observable<Well> {
  return this.http.get<WellDTO>(`/api/wells/${id}`).pipe(
    map(dto => mapWellDTOToModel(dto)) // Transforma DTO → Modelo de dominio
  );
}

// ❌ Prohibido: el servicio NO maneja errores HTTP
getWell(id: string): Observable<Well> {
  return this.http.get<Well>(`/api/wells/${id}`).pipe(
    catchError(err => { console.error(err); return EMPTY; }) // ← NUNCA
  );
}
```

---

## 6. Modelos, Servicios y Barrel Exports: Reglas de Ubicación

### 6.1. La regla de decisión (una sola pregunta)

> **¿Lo necesita más de un dominio?**
> - **Sí** → va en `shared/models/` o `shared/services/`
> - **No** → va en `domains/[dominio]/models/` o `domains/[dominio]/services/`

No hay excepciones. Si un modelo que estaba en `domains/wells/models/` pasa a ser necesario en `operations/`, se mueve a `shared/models/` en ese momento — no antes.

### 6.2. Tabla de referencia rápida

| Elemento | ¿Dónde vive? | Condición |
|---|---|---|
| `Well`, `WellStatus`, `Operator` | `shared/models/` | Usado en 2+ dominios |
| `UserRole`, `User` | `shared/models/` | Usado en auth + admin |
| `WellHistory`, `Trajectory` | `domains/wells/models/` | Solo lo usa wells |
| `IDOP`, `Forma100` | `domains/operations/models/` | Solo lo usa operations |
| `FiscalizationReport` | `domains/production/models/` | Solo lo usa production |
| Catálogos read-only (tipos de pozo, operadoras) | `shared/services/` | Llamadas GET usadas en 2+ dominios |
| `WellsApiService` (CRUD de pozos) | `domains/wells/services/` | Solo lo llama wells |
| `OperationsApiService` | `domains/operations/services/` | Solo lo llama operations |
| `AuthService` | `core/auth/` | Singleton de sesión, arranca con la app |
| Interceptores HTTP | `core/http/` | Se registran una vez en `app.config.ts` |

> **Referencia:** Las entidades, modelos y servicios listados en esta tabla son ejemplos ilustrativos basados en los dominios conocidos. Los nombres reales, su alcance y su ubicación se determinan al especificar y desarrollar cada funcionalidad.

### 6.3. Estructura interna de un grupo de modelos

Cada entidad en `models/` (ya sea en `shared/` o en un dominio) sigue la misma estructura de 3 archivos + barrel:

```
models/
├── well.model.ts       # Interfaz del dominio → lo que usa el frontend
├── well.dto.ts         # Interfaz del contrato de la API → lo que devuelve el backend
├── well.mapper.ts      # Función pura: DTO → Model
└── index.ts            # Barrel export del grupo
```

**Alineación con el backend GOP (CONSTITUTION-ba §9.9):** el backend emite JSON en **camelCase**, con **fechas ISO 8601 UTC**, **enums como string** y **datos crudos con IDs** (sin hidratación server-side — ver §22). Por lo tanto los DTOs del FE **no** renombran propiedades; la clase DTO y la del modelo son similares en nombres y el mapper cumple otras funciones:

**Responsabilidades del mapper (tras alineación camelCase):**
1. Parsear fechas ISO string → `Date` o `DateTime` de Luxon (ver §21.2).
2. Tipar enums string → union literal del dominio (`string` crudo → `WellStatus`).
3. Incluir campos derivados del cliente (ej. `displayName` calculado, `isActive`).
4. Omitir campos que el FE no usa (shaping).
5. Dejar los **IDs crudos** que vienen del backend — la hidratación (resolver `statusId` → `statusLabel`) ocurre en selectors/computed (§22).

```typescript
// well.dto.ts → refleja exactamente la respuesta de la API (camelCase, datos crudos)
export interface WellDTO {
  id: string;
  name: string;
  operatorId: number;       // ID crudo → se hidrata en selector
  statusId: string;         // ID crudo → se hidrata en selector
  fieldId: number | null;
  spudDateUtc: string | null;   // ISO 8601 UTC string
  depthMeters: number | null;
  rowVersion: string;       // base64 — sirve como ETag para concurrencia optimista (§21.4)
}

// well.model.ts → modelo del dominio en el FE. Mantiene IDs crudos; las fechas son objetos Date.
export interface Well {
  id: string;
  name: string;
  operatorId: number;
  statusId: WellStatusId;   // union literal, sin label precomputado
  fieldId: number | null;
  spudDate: Date | null;
  depthMeters: number | null;
  rowVersion: string;
}

// well.mapper.ts → función pura, sin efectos secundarios, testeable de forma aislada
export function mapWellDTOToModel(dto: WellDTO): Well {
  return {
    id: dto.id,
    name: dto.name,
    operatorId: dto.operatorId,
    statusId: dto.statusId as WellStatusId,
    fieldId: dto.fieldId,
    spudDate: dto.spudDateUtc ? new Date(dto.spudDateUtc) : null,
    depthMeters: dto.depthMeters,
    rowVersion: dto.rowVersion,
  };
}

// index.ts → barrel export: un solo punto de entrada para todo el grupo
export * from './well.model';
export * from './well.dto';
export * from './well.mapper';
```

> **Por qué mantener DTO separado del Model aunque ambos sean camelCase:** el DTO representa el **contrato externo** (puede cambiar si el backend actualiza el contrato); el Model es la **forma interna** que el FE consume. Mantenerlos separados protege templates, selectors y componentes de cambios del contrato — el único lugar que se rompe al cambiar un campo es el mapper.

**Regla de `export type` en barrels:** TypeScript con `isolatedModules` (habilitado por defecto en Angular) requiere que las re-exportaciones de tipos usen `export type`. Si un archivo solo exporta interfaces o types, el barrel debe usar `export type` para esas re-exportaciones:

```typescript
// ✅ Correcto: export type para re-exportar tipos/interfaces
export type { Well } from './well.model';
export type { WellDTO } from './well.dto';
export { mapWellDTOToModel } from './well.mapper'; // funciones usan export normal

// ❌ Error de compilación con isolatedModules
export { Well } from './well.model'; // ← falla: Well es solo un tipo
```

**Por qué el mapper:** Si el backend cambia `well_id` por `id`, se actualiza solo el mapper — ningún template, componente ni selector se rompe.

> **Referencia:** El modelo `Well` mostrado es un ejemplo del patrón. Los campos, tipos y estructura real de cada entidad se definen según el contrato de API establecido en la especificación funcional correspondiente.

### 6.4. Servicios: tipos y responsabilidades

```typescript
// shared/services/catalog.service.ts → catálogos read-only, sin lógica de negocio
// Solo hace GET, no escribe datos, no conoce reglas de negocio
@Injectable({ providedIn: 'root' })
export class CatalogService {
  getWellTypes(): Observable<WellType[]> { ... }
  getOperators(): Observable<Operator[]> { ... }
}

// domains/wells/services/wells-api.service.ts → CRUD específico del dominio
// Conoce los endpoints de pozos, aplica mappers, devuelve modelos limpios
@Injectable({ providedIn: 'root' })
export class WellsApiService {
  getWell(id: string): Observable<Well> {
    return this.http.get<WellDTO>(`/api/wells/${id}`).pipe(
      map(mapWellDTOToModel)
    );
  }
  createWell(payload: CreateWellDTO): Observable<Well> { ... }
}
```

**Regla:** Los servicios de dominio solo hacen llamadas HTTP y aplican mappers. Nunca manejan errores HTTP — eso es responsabilidad exclusiva de `core/http/` (ver sección 5).

> **Referencia:** `CatalogService` y `WellsApiService` son ejemplos del patrón. Los servicios reales, sus métodos y los endpoints que consumen se definen al implementar cada feature según su especificación funcional.

### 6.5. Path Aliases: solución al problema de discoverability

Tener modelos y servicios en dos niveles (`shared/` y `domains/`) no implica rutas de importación largas. Se configuran aliases en `tsconfig.json` para que cada import sea predecible y corto:

```json
// tsconfig.json
{
  "compilerOptions": {
    "paths": {
      "@core/*":       ["src/app/core/*"],
      "@shared/*":     ["src/app/shared/*"],
      "@wells/*":      ["src/app/domains/wells/*"],
      "@operations/*": ["src/app/domains/operations/*"],
      "@production/*": ["src/app/domains/production/*"],
      "@admin/*":      ["src/app/domains/admin/*"],
      "@env/*":        ["src/environments/*"]
    }
  }
}
```

**Resultado:** imports siempre predecibles sin importar desde dónde se escriban:

```typescript
// ✅ Con aliases — claro, corto, no depende de la ubicación del archivo
import { Well }            from '@shared/models';
import { WellsApiService } from '@wells/services';
import { APP_LOCALE }      from '@shared/locale/locale';
import { environment }     from '@env/environment';

// ❌ Sin aliases — frágil, se rompe si se mueve el archivo
import { Well } from '../../../shared/models/well.model';
import { WellsApiService } from '../../domains/wells/services/wells-api.service';
```

**Regla:** Toda importación entre capas usa el alias correspondiente. Las importaciones dentro de la misma feature pueden usar rutas relativas cortas.

---

## 7. Patrón de Locale: Desacoplamiento de Textos

**Está estrictamente prohibido "quemar" (hardcode) textos, etiquetas o mensajes directamente en los archivos HTML o TS.** En lugar de librerías de i18n con pipes asíncronos y archivos JSON, se usa un enfoque ágil basado en objetos TypeScript constantes (`locale.ts`), sin dependencias externas.

Los textos se dividen en tres niveles:
- **Global** (`shared/locale/locale.ts`): navegación, errores genéricos, botones comunes, branding de la app.
- **Por Feature de dominio** (`domains/.../features/.../locale.ts`): textos exclusivos de esa funcionalidad. Si la feature se elimina, sus textos desaparecen con ella — sin textos huérfanos.
- **Por Feature de core** (`core/.../features/.../locale.ts`): textos de funcionalidades del sistema (auth, onboarding). Misma regla: el locale vive junto a la feature que lo consume.

**Por qué este enfoque sobre i18n tradicional:**
- Sin pipes asíncronos ni archivos JSON que sincronizar
- Tipado fuerte: el IDE detecta si una clave de texto desaparece o cambia
- Si el cliente cambia "Taladro" por "Equipo de Perforación", se cambia en un solo `locale.ts` y se refleja en toda la feature
- Los templates quedan limpios y enfocados en estructura, no en texto

---

> **Distinción crítica entre tipos de locale:**
> - `shared/locale/locale.ts` (APP_LOCALE) → se define localmente. Contiene textos del layout, errores genéricos y acciones comunes de **toda la app**.
> - `locale.ts` de features que usan **páginas del UIKit** → solo re-exporta desde `'appgop-web'`. Ver §20.4. Nunca definir ni copiar estos locales.
> - `locale.ts` de features **propias** (sin página UIKit equivalente) → se define localmente con el patrón de esta sección.

### Locale Global (`shared/locale/locale.ts`)

Contiene todos los textos del wrapper de la aplicación: navegación, errores genéricos y acciones comunes.

```typescript
// src/app/shared/locale/locale.ts
// REFERENCIA: las claves y textos se ajustan según el diseño final del layout y los flujos definidos
export const APP_LOCALE = {
  nav: {
    wells: 'Pozos',
    operations: 'Operaciones',
    production: 'Producción',
    admin: 'Administración',
  },
  actions: {
    save: 'Guardar',
    cancel: 'Cancelar',
    confirm: 'Confirmar',
    delete: 'Eliminar',
    back: 'Volver',
  },
  errors: {
    generic: 'Ocurrió un error inesperado.',
    noConnection: 'Sin conexión. Verifica tu red.',
    forbidden: 'No tienes permisos para realizar esta acción.',
  },
} as const; // "as const" garantiza tipos literales y evita mutación accidental
```

### Locale de Feature (`domains/wells/features/well-create/locale.ts`)

```typescript
// src/app/domains/wells/features/well-create/locale.ts
// REFERENCIA: las claves, secciones y textos se definen al especificar y desarrollar cada feature
export const WELL_CREATE_LOCALE = {
  title: 'Creación de Pozo',
  steps: {
    general: 'Información General',
    technical: 'Datos Técnicos',
    trajectory: 'Trayectoria',
  },
  fields: {
    wellName: 'Nombre del Pozo',
    uwi: 'UWI',
    operator: 'Operadora',
  },
  actions: {
    submit: 'Radicar Solicitud',
    saveDraft: 'Guardar Borrador',
  },
  messages: {
    success: 'Pozo creado exitosamente.',
    draftSaved: 'Borrador guardado.',
  },
} as const;
```

> **Referencia:** Las claves y textos de ambos locales son ilustrativos. Cada feature define su propio `locale.ts` con las claves que realmente necesita según su especificación funcional. El `APP_LOCALE` global crece únicamente con textos del layout y acciones verdaderamente comunes.

### Cómo se expone en el componente (patrón estándar)

```typescript
// well-create.component.ts
@Component({ ... })
export class WellCreateComponent {
  // "protected readonly" → accesible en template, no modificable, no parte de la API pública
  protected readonly locale = WELL_CREATE_LOCALE;
}
```

```html
<!-- well-create.component.html -->
<h1>{{ locale.title }}</h1>
<button>{{ locale.actions.submit }}</button>
```

**Regla:** Siempre `protected readonly locale = NOMBRE_LOCALE`. Nunca importar la constante directamente en el HTML ni exponer como `public`.

---

## 8. Convenciones de Nomenclatura

Reglas no negociables para garantizar consistencia en el equipo:

| Elemento | Convención | Ejemplo |
|---|---|---|
| Archivos | `kebab-case` | `well-create.component.ts` |
| Clases y Componentes | `PascalCase` | `WellCreateComponent` |
| Interfaces y Modelos | `PascalCase` | `Well`, `WellDTO` |
| Constantes de locale | `UPPER_SNAKE_CASE` | `WELL_CREATE_LOCALE` |
| Signals | `camelCase` | `currentStep`, `isLoading`, `wells` |
| Selectores CSS | prefijo `app-` | `app-well-status-badge` |
| Rutas de archivo | `kebab-case` | `well-status-badge.component.ts` |
| Variables de entorno | `camelCase` en objeto | `environment.apiUrl` |
| Funciones mapper | `mapXDTOToModel` | `mapWellDTOToModel` |
| Archivos de store NgRx | sufijo por tipo | `wells.actions.ts`, `wells.reducer.ts` |

### Convenciones de carpetas dentro de una feature

```
well-create/
├── components/         # Componentes "Dumb" internos de la feature (PascalCase en clase)
├── locale.ts           # Siempre "locale.ts" — nunca "strings.ts", "labels.ts", etc.
├── well-create.component.ts
├── well-create.component.html
└── well-create.component.spec.ts
```

---

## 9. Reglas de Dependencias entre Capas

Esta regla es la más crítica para evitar el acoplamiento que hace colapsar la arquitectura DDD con el tiempo.

```
┌─────────────────────────────────────────────┐
│                  domains/                    │  ← Puede importar de core/ y shared/
│  (wells, operations, production, admin)      │  ← NUNCA importa de otro dominio
└───────────────────┬─────────────────────────┘
                    │ puede importar de ↓
        ┌───────────┴────────────┐
        │        shared/         │  ← Puede importar de utils/ propios
        │  (ui, pipes, locale,   │  ← NUNCA importa de core/ ni domains/
        │   directives, utils)   │
        └───────────┬────────────┘
                    │
        ┌───────────┴────────────┐
        │         core/          │  ← No importa de shared/ui/ ni domains/
        │  (auth, guards, http,  │  ← Puede importar de shared/models/,
        │       layout)          │    shared/locale/, shared/utils/
        └────────────────────────┘
```

**Reglas explícitas:**

1. `domains/wells/` **NO puede** importar de `domains/operations/`. Si dos dominios comparten lógica, esa lógica sube a `shared/`.
2. `shared/` **NO puede** importar de ningún dominio — si lo hace, ya no es "shared", es parte del dominio.
3. `core/` **NO puede** importar componentes de `shared/ui/` — puede importar de `shared/models/`, `shared/locale/` y `shared/utils/`.
4. Toda comunicación entre dominios pasa por el **Store (NgRx)** o por un **servicio en `core/`**, nunca por importación directa.

---

## 10. Seguridad (OWASP Top 10 — Aplicación Frontend)

La seguridad no es opcional. Estas reglas son de cumplimiento obligatorio desde el primer commit.

### A01 — Control de Acceso Roto
- **Route Guards obligatorios** en todas las rutas protegidas (nunca dejar una ruta sin guard si requiere autenticación).
- **RBAC en guards:** verificar rol + autenticación. No asumir que "si está logueado, puede todo".
- **Nunca** ocultar rutas solo en el sidebar — el guard en el router es la única defensa real.

```typescript
// core/guards/auth.guard.ts — lee el estado NgRx, no el servicio directamente
export const authGuard: CanActivateFn = () => {
  const store = inject(Store);
  const router = inject(Router);
  return store.select(selectIsAuthenticated).pipe(
    take(1),
    map(isAuth => isAuth ? true : router.createUrlTree(['/login'])),
  );
};

// core/guards/role.guard.ts
export const roleGuard = (roles: UserRole[]): CanActivateFn => () => {
  const store = inject(Store);
  const router = inject(Router);
  return store.select(selectCurrentUser).pipe(
    take(1),
    map(user => user && roles.includes(user.role) ? true : router.createUrlTree(['/forbidden'])),
  );
};
```

### A03 — Inyección (XSS)
- **Prohibido** usar `[innerHTML]` sin sanitización. Si es absolutamente necesario, usar `DomSanitizer.sanitizeHtml()`.
- **Prohibido** usar `bypassSecurityTrustHtml()` salvo excepciones documentadas y revisadas.
- Angular escapa automáticamente la interpolación `{{ }}` — usar siempre interpolación, nunca concatenación en atributos.

```html
<!-- ✅ Seguro: Angular escapa el valor -->
<p>{{ userInput }}</p>

<!-- ❌ Peligroso: XSS directo -->
<p [innerHTML]="userInput"></p>

<!-- ✅ Si se necesita HTML dinámico (caso excepcional documentado) -->
<p [innerHTML]="sanitizer.sanitize(SecurityContext.HTML, userInput)"></p>
```

### A02 — Fallos Criptográficos / Exposición de Datos
- **Prohibido** almacenar el **access token** en `localStorage` o `sessionStorage`. El access token vive **en memoria** (`AuthService`) — ver §24.5 y §24.7.
- **Preferido** que el **refresh token** viaje en cookie `HttpOnly; Secure; SameSite=Strict` emitida por `gop.identity`. Si se usa `sessionStorage` como fallback, requiere ADR.
- **Prohibido** loggear datos sensibles en `console.log` — remover todos los logs antes de merge a `main`. Esto incluye claims del JWT, passwords, datos clasificados `Confidential`/`Reserved`/`Restricted` (CONSTITUTION-ba §7.10).
- **Prohibido** incluir credenciales, API keys o URLs de producción en el código fuente. Usar `environments/`.

```typescript
// ✅ Correcto: access token solo en memoria del AuthService
@Injectable({ providedIn: 'root' })
export class AuthService {
  private readonly _accessToken = signal<string | null>(null);
  getToken(): string | null { return this._accessToken(); }
  hydrateFromToken(token: string): void { this._accessToken.set(token); /* + claims */ }
  clearSession(): void { this._accessToken.set(null); }
}

// ❌ Prohibido: persistir el access token en ningún storage del navegador
sessionStorage.setItem('token', token);  // ← NUNCA
localStorage.setItem('token', token);    // ← NUNCA
```

### A07 — Fallos de Autenticación
- El `AuthService` es el **único** punto de acceso al token — ningún componente ni servicio de dominio lo lee directamente.
- Implementar refresh automático del token antes de su expiración.
- El `AuthService` **NO inyecta** `Router` ni `Store`. La navegación y el despacho de acciones son responsabilidad exclusiva de los **NgRx Effects**.
- El logout se inicia despachando una acción (`logout()` o `sessionExpired()`); el Effect correspondiente llama a `authService.clearSession()`, navega y despacha `logoutSuccess()`.

```typescript
// core/auth/auth.service.ts — access token en memoria, nunca storage del navegador
// Ver §23 para permissions/claims y §24 para el flujo completo de autenticación.
@Injectable({ providedIn: 'root' })
export class AuthService {
  private readonly _accessToken = signal<string | null>(null);
  private readonly _permissions = signal<ReadonlySet<string>>(new Set());

  hydrateFromToken(token: string): void {
    this._accessToken.set(token);
    const claims = decodeJwt(token);
    this._permissions.set(new Set(claims.permissions ?? []));
  }
  clearSession(): void {
    this._accessToken.set(null);
    this._permissions.set(new Set());
  }
  getToken(): string | null { return this._accessToken(); }
  hasPermission(p: string | undefined): boolean { return !p || this._permissions().has(p); }
}

// core/auth/store/auth.effects.ts — Effect maneja la orquestación completa
logout$ = createEffect((actions$ = inject(Actions), authService = inject(AuthService), router = inject(Router)) =>
  actions$.pipe(
    ofType(AuthActions.logout),
    tap(() => { authService.clearSession(); router.navigate(['/login']); }),
    map(() => AuthActions.logoutSuccess()),
  )
);
```

### A05 — Configuración Insegura
- Los `environments/` **nunca** se suben al repositorio si contienen datos sensibles — usar `.gitignore` y variables de CI/CD.
- El archivo `environments/environment.ts` solo expone URLs y flags de feature — nunca secrets.
- Configurar **Content Security Policy (CSP)** en el servidor (coordinar con backend/infraestructura).

```typescript
// environments/environment.ts (solo datos públicos, nunca secrets)
export const environment = {
  production: false,
  apiUrl: 'https://api-dev.gop.internal',
  featureFlags: {
    enableBetaWizard: true,
  },
};
```

### A06 — Componentes Vulnerables
- Ejecutar `npm audit` antes de cada release y corregir vulnerabilidades `high` y `critical` obligatoriamente.
- Mantener las dependencias de Angular y PrimeNG actualizadas en cada sprint.
- No agregar librerías npm sin revisión previa del equipo (evitar supply chain attacks).

### A09 — Registro y Monitoreo
- El dominio `admin/audit-logs/` debe registrar todas las acciones sensibles: creación/eliminación de pozos, cambios de roles, aprobaciones de formas. La auditoría real vive en el backend (CONSTITUTION-ba §11.6); el FE solo lee.
- Los errores `5xx` capturados en el interceptor deben preservar el `traceId` del ProblemDetails y mostrarlo al usuario para soporte (_"Comparte este código: {{ traceId }}"_). Ver §21.3.
- El FE **no** envía telemetría propia de auditoría al backend — la fuente de verdad es la capa de dominio (EF Core + Audit.NET).

---

## 11. Librería de Componentes y Estilos CSS

Esta arquitectura se basa en un enfoque de Diseño Atómico simplificado, optimizando PrimeNG para funcionalidad y Tailwind CSS para diseño visual.

> Los colores, tipografía y tamaños usados en clases Tailwind y componentes PrimeNG deben provenir siempre de los tokens CSS definidos en la **Sección 14 — Sistema de Design Tokens**. Nunca usar valores hardcodeados.

### 11.1. Estrategia de Composición

**Principio rector: "PrimeNG para la lógica compleja, Tailwind para la estructura y estética".**

- **PrimeNG:** exclusivamente para componentes con lógica de estado compleja (tablas con filtros, modales con animaciones, selectores de fecha, drag & drop).
- **Tailwind CSS:** layouting (Grids/Flexbox), espaciado, tipografía y personalización de PrimeNG mediante Pass Through (PT).

### 11.2. Definición de Uso por Tipo de Componente

**A. Contenedores Básicos (Layout)**
- Regla: Prohibido usar PrimeNG para estructuras simples.
- Uso: Tailwind CSS puro (`div`, `section`, `main` con clases utilitarias).
- Por qué: Evita sobrecarga de selectores CSS y facilita cambios de diseño sin tocar el TS.

**B. Tablas y Grids de Datos**
- Regla: Usar `p-table` de PrimeNG.
- Personalización: Aplicar estilos de Tailwind mediante `styleClass` o Pass Through (`pt: { header: 'bg-slate-100 text-sm' }`).
- Por qué: Reinventar paginación, filtrado y ordenación es ineficiente y propenso a errores.

**C. Modales y Overlays**
- Regla: Usar `DialogService` de PrimeNG disparado desde un servicio centralizado.
- Personalización: Tailwind para el contenido interno del modal.
- Por qué: PrimeNG gestiona z-index, foco del teclado (accesibilidad) y bloqueo del scroll de forma nativa.

### 11.3. Cuándo Crear Componentes Personalizados (Wrappers)

**Por defecto: usar PrimeNG directamente.** Los wrappers no son la norma — son la excepción justificada. Crearlos sin criterio genera indirección innecesaria y duplica el trabajo de mantener la API del componente original.

El sistema de **Pass Through (PT) global** en `providePrimeNG({ pt: {...} })` ya resuelve la consistencia visual sin necesidad de wrappers. Úsalo siempre primero.

Un wrapper se crea **únicamente** cuando se cumple al menos uno de estos tres criterios:

#### Criterio 1 — Abstracción de Negocio (Smart Component repetido)
El mismo componente + lógica de API aparece en 2 o más features distintas.

```typescript
// ✅ Justificado: WellStatusDropdown siempre carga el mismo catálogo y aplica la misma lógica
// Se repite en well-create, well-manage y el filtro del listado → tiene sentido abstraerlo
@Component({ selector: 'app-well-status-dropdown', ... })
export class WellStatusDropdownComponent {
  options = toSignal(inject(CatalogService).getWellStatuses());
}

// ❌ No justificado: un p-dropdown que solo aparece en un formulario → usar p-dropdown directo
```

#### Criterio 2 — Comportamiento Forzado (lo que PT no puede hacer)
Se necesita añadir lógica de negocio transversal que el sistema PT de PrimeNG no puede manejar: verificación de permisos, eventos de auditoría, estados de carga estandarizados.

```typescript
// ✅ Justificado: AppButton agrega verificación de permiso y estado loading uniforme
// PT puede cambiar estilos, pero no puede inyectar lógica de autorización
@Component({ selector: 'app-button', ... })
export class AppButtonComponent {
  @Input() permission?: string;           // Oculta el botón si no tiene el permiso
  @Input() loading = false;               // Estado de carga estandarizado
  protected canRender = inject(AuthService).hasPermission(this.permission);
}

// ❌ No justificado: un p-button con solo un color diferente → usar styleClass o PT
```

#### Criterio 3 — UI Altamente Específica (PrimeNG como base, no como fin)
El componente de PrimeNG requiere tantos overrides de CSS o configuración que un componente propio con Tailwind resulta más simple y mantenible.

```
Señal de alerta: si el PT de un componente supera ~10 claves de configuración
para lograr el diseño requerido, considerar construirlo desde cero con Tailwind.
```

#### Tabla de decisión rápida

| Caso | Solución |
|---|---|
| Mismo estilo en todos los botones del proyecto | PT global en `providePrimeNG` |
| `p-dropdown` que siempre carga el mismo catálogo y aparece en 3+ features | Wrapper (Criterio 1) |
| `p-button` con validación de permiso RBAC | Wrapper (Criterio 2) |
| `p-tag` con un color distinto | `styleClass` directo, sin wrapper |
| `p-calendar` usado una sola vez | `p-calendar` directo, sin wrapper |
| Diseño de card totalmente personalizado sin lógica compleja | Tailwind puro, sin PrimeNG |

#### Dónde vive cada wrapper una vez creado

La ubicación depende de una sola pregunta: **¿el wrapper conoce conceptos del negocio (pozos, operaciones, producción)?**

| ¿Conoce el dominio? | ¿Cuántas features lo usan? | Ubicación |
|---|---|---|
| No — agnóstico al negocio | Cualquiera | `shared/ui/` |
| Sí — conoce el dominio | 2+ features del mismo dominio | `domains/[dominio]/components/` |
| Sí — conoce el dominio | Solo una feature | `domains/[dominio]/features/[feature]/components/` |

**Ejemplos:**

```
shared/ui/
├── app-button/             # p-button + RBAC + loading → no sabe qué es un "Pozo"
├── app-confirm-button/     # p-button + confirmación integrada → agnóstico
├── app-modal/              # DialogService + estructura base de contenido
└── app-data-table/         # p-table + paginación + skeleton → agnóstico

domains/wells/components/
├── well-status-badge/      # p-tag con colores de WellStatus → sabe del dominio wells
└── well-status-dropdown/   # p-dropdown que carga catálogo de estados → sabe del dominio wells

domains/wells/features/well-create/components/
└── uwi-preview/            # Solo lo usa el wizard de creación → vive dentro de la feature
```

> **Referencia:** Los componentes y wrappers listados son ejemplos del patrón. Los wrappers reales se crean únicamente cuando una feature los requiere y se cumplen los criterios definidos en este mismo punto. No crear wrappers anticipados sin una necesidad concreta.

> `shared/utils/` es exclusivamente para funciones puras (sin componentes, sin estado):
> `formatUWI()`, `parseCoordinates()`, `calculateDepth()`.
> Nunca colocar componentes o servicios ahí.

### 11.4. Mejores Prácticas

- **Signals & Standalone:** Usar Angular Signals para estado de UI de componentes (abrir/cerrar modales, loading) para detección de cambios ultra rápida.
- **Configuración Global:** Definir un preset de Tailwind alineado con los colores de PrimeNG para evitar inconsistencias visuales.
- **Pass Through Global:** Configurar estilos por defecto en `providePrimeNG({ pt: {...} })` para que todos los componentes compartan la misma estética sin necesidad de repetir clases.
- **PrimeNG + Tailwind: override en formularios nativos.** El plugin `tailwindcss-primeui` y el tema Aura de PrimeNG inyectan estilos oscuros (`background`, `color`) en elementos nativos de formulario (`input`, `select`, `textarea`). Cuando se usen inputs nativos (no componentes `p-inputtext` de PrimeNG), agregar clases explícitas de Tailwind para garantizar el fondo y texto esperados: `bg-white text-gray-900 placeholder-gray-400`. Sin esto, los inputs heredan fondos oscuros del tema.

---

## 12. Ambientes y Variables de Entorno (DEV / QA / PROD)

### 12.1. Principio de funcionamiento

Angular utiliza **file replacement** en tiempo de compilación: todo el código importa siempre `environment.ts`, y el compilador lo reemplaza físicamente por el archivo del perfil indicado en el flag `--configuration`. No hay lógica condicional en runtime.

```
src/environments/
├── environment.interface.ts   ← Contrato TypeScript compartido por los 3 perfiles
├── environment.ts             ← DEV  (default — el que importa todo el código)
├── environment.qa.ts          ← QA   (reemplaza environment.ts en build QA)
└── environment.prod.ts        ← PROD (reemplaza environment.ts en build PROD)
```

**Regla de seguridad:** Los archivos `environment.qa.ts` y `environment.prod.ts` **nunca se suben al repositorio**. Se generan en el pipeline de CI/CD mediante variables de entorno del servidor. Solo `environment.ts` (DEV) se versiona como referencia de estructura.

```
# .gitignore
src/environments/environment.qa.ts
src/environments/environment.prod.ts
```

### 12.2. Interfaz tipada (`environment.interface.ts`)

La interfaz garantiza que los 3 archivos de ambiente tengan siempre la misma forma. TypeScript obliga a declarar todos los hosts en cada perfil — no es posible olvidar uno.

```typescript
// src/environments/environment.interface.ts
export interface AppEnvironment {
  name: 'development' | 'qa' | 'production';
  production: boolean;
  hosts: {
    gopApi: string; // API principal del sistema GOP
    // Al integrar un nuevo servicio externo, agregar su clave aquí
    // y declarar su URL en los 3 archivos de ambiente
  };
  featureFlags: {
    enableBetaFeatures: boolean;
  };
}
```

### 12.3. Archivos por perfil

```typescript
// src/environments/environment.ts — DEV
import { AppEnvironment } from './environment.interface';

export const environment: AppEnvironment = {
  name: 'development',
  production: false,
  hosts: {
    gopApi: 'http://localhost:3000',
  },
  featureFlags: { enableBetaFeatures: true },
};
```

```typescript
// src/environments/environment.qa.ts — QA
import { AppEnvironment } from './environment.interface';

export const environment: AppEnvironment = {
  name: 'qa',
  production: false,
  hosts: {
    gopApi: 'https://api-qa.gop.internal',
  },
  featureFlags: { enableBetaFeatures: true },
};
```

```typescript
// src/environments/environment.prod.ts — PROD
import { AppEnvironment } from './environment.interface';

export const environment: AppEnvironment = {
  name: 'production',
  production: true,
  hosts: {
    gopApi: 'https://api.gop.internal',
  },
  featureFlags: { enableBetaFeatures: false },
};
```

> Los hosts y URLs mostrados son de referencia. Las URLs reales de cada ambiente se definen según la infraestructura del proyecto. Al integrar un nuevo servicio externo, se agrega su clave en `AppEnvironment.hosts` y su URL en los 3 archivos.

### 12.4. Centralización de endpoints (`api-endpoints.ts`)

Los paths de la API **no cambian entre ambientes** — solo cambia el host. Por eso viven en un archivo de constantes separado en `core/http/`. Los servicios importan `API`, nunca `environment` directamente.

```typescript
// src/app/core/http/api-endpoints.ts
// REFERENCIA: los grupos y paths se definen según las especificaciones funcionales de cada dominio
import { environment } from '@env/environment';

const { hosts } = environment;

export const API = {
  wells: {
    base:    `${hosts.gopApi}/api/v1/wells`,
    byId:    (id: string) => `${hosts.gopApi}/api/v1/wells/${id}`,
  },
  operations: {
    base: `${hosts.gopApi}/api/v1/operations`,
  },
  production: {
    base: `${hosts.gopApi}/api/v1/production`,
  },
  admin: {
    base: `${hosts.gopApi}/api/v1/admin`,
  },
  catalogs: {
    base: `${hosts.gopApi}/api/v1/catalogs`,
  },
} as const;
```

> Los grupos de endpoints y sus paths son de referencia. Cada endpoint específico se agrega al momento de implementar la feature que lo requiere, según el contrato definido en la especificación funcional.

**Uso en servicios:**
```typescript
// ✅ El servicio importa API, no environment
import { API } from '@core/http/api-endpoints';

getWell(id: string): Observable<Well> {
  return this.http.get<WellDTO>(API.wells.byId(id)).pipe(map(mapWellDTOToModel));
}

// ❌ Prohibido: el servicio no construye URLs directamente
getWell(id: string): Observable<Well> {
  return this.http.get<WellDTO>(`${environment.hosts.gopApi}/api/v1/wells/${id}`);
}
```

### 12.5. Configuración en `angular.json`

```json
"configurations": {
  "production": {
    "fileReplacements": [
      { "replace": "src/environments/environment.ts",
        "with":    "src/environments/environment.prod.ts" }
    ],
    "optimization": true,
    "sourceMap": false,
    "outputHashing": "all",
    "budgets": [
      { "type": "initial", "maximumWarning": "500kB", "maximumError": "1MB" },
      { "type": "anyComponentStyle", "maximumWarning": "4kB", "maximumError": "8kB" }
    ]
  },
  "qa": {
    "fileReplacements": [
      { "replace": "src/environments/environment.ts",
        "with":    "src/environments/environment.qa.ts" }
    ],
    "optimization": true,
    "sourceMap": true,
    "outputHashing": "all",
    "budgets": [
      { "type": "initial", "maximumWarning": "500kB", "maximumError": "1MB" },
      { "type": "anyComponentStyle", "maximumWarning": "4kB", "maximumError": "8kB" }
    ]
  },
  "development": {
    "optimization": false,
    "sourceMap": true,
    "extractLicenses": false,
    "namedChunks": true
  }
}
```

### 12.6. Scripts de compilación (`package.json`)

| Script npm | Comando Angular | Perfil | Optimizado | Source map |
|---|---|---|---|---|
| `npm start` | `ng serve` | DEV | No | Sí |
| `npm run start:qa` | `ng serve --configuration=qa` | QA | Sí | Sí |
| `npm run build:dev` | `ng build --configuration=development` | DEV | No | Sí |
| `npm run build:qa` | `ng build --configuration=qa` | QA | Sí | Sí |
| `npm run build:prod` | `ng build --configuration=production` | PROD | Sí | No |

### 12.7. Por qué `hosts` en lugar de un solo `baseUrl`

| Escenario | `baseUrl` único | `hosts` por servicio |
|---|---|---|
| Un solo backend en todos los ambientes | Funciona | Funciona |
| Servicios externos con URL diferente por ambiente | No soportado | Funciona |
| Agregar un nuevo host sin tocar servicios existentes | Requiere refactor | Solo agregar clave en `hosts` e interfaz |
| TypeScript detecta host faltante en un perfil | No | Sí — error en compilación |

---

## 13. Mocks HTTP mediante Interceptor

### 13.1. Propósito y cuándo aplicar

Los mocks HTTP son una herramienta **temporal y transitoria**. Su único propósito es permitir que el desarrollo y maquetado del frontend avance de forma desacoplada mientras el API real no está disponible o un endpoint específico aún no está implementado. El interceptor simula la respuesta del backend de forma transparente — el servicio de dominio, el store y el componente no distinguen si la respuesta proviene de un mock o de la red real.

Este enfoque garantiza que la integración con el API real sea inmediata y sin fricción desde el primer día: como el servicio ya consume la interfaz correcta (DTO → Model → Mapper), reemplazar el mock por el endpoint real no requiere cambios en ninguna capa del frontend.

**Cuándo usar:**
- Desarrollo paralelo frontend/backend — el equipo de frontend no debe bloquearse esperando que el backend entregue un endpoint
- Maquetado y validación de UI con datos representativos y controlados
- Demos o revisiones de producto sin infraestructura de backend activa

**Cuándo NO usar:**
- En QA o PROD — los mocks están estrictamente limitados al perfil DEV
- Como solución definitiva — un mock que sobrevive a la entrega del endpoint real es deuda técnica inmediata
- Para cubrir errores de integración — si el API real está disponible, usarlo directamente

### 13.2. Control por ambiente

El mock se habilita exclusivamente mediante un flag en el archivo de ambiente. Nunca se activa por lógica condicional en el código de la app.

```typescript
// environment.interface.ts — agregar el flag al contrato
export interface AppEnvironment {
  // ...hosts, production, name...
  featureFlags: {
    enableBetaFeatures: boolean;
    useMocks: boolean; // true solo en DEV cuando el backend no está disponible
  };
}

// environment.ts (DEV) — activar según necesidad
featureFlags: { enableBetaFeatures: true, useMocks: true }

// environment.qa.ts / environment.prod.ts — siempre false
featureFlags: { enableBetaFeatures: true, useMocks: false }
```

### 13.3. Estructura de archivos

Los datos mock viven **junto al dominio** al que pertenecen, no en una carpeta global. Al eliminar una feature, sus mocks se eliminan con ella.

```
domains/wells/
├── features/
├── models/
├── services/
└── mocks/                          ← carpeta de mocks del dominio
    ├── wells.mock.ts               ← datos y handlers de pozos
    └── index.ts                    ← barrel export de todos los handlers del dominio
```

### 13.4. Anatomía de un mock handler

Cada handler es una función pura que recibe la petición y retorna un `HttpResponse` tipado:

```typescript
// domains/wells/mocks/wells.mock.ts
// REFERENCIA: los datos y endpoints mockeados se definen según la especificación funcional de cada feature

import { HttpRequest, HttpResponse } from '@angular/common/http';

export interface MockHandler {
  // Patrón de URL que este handler intercepta (string exacto o RegExp)
  urlPattern: string | RegExp;
  method: 'GET' | 'POST' | 'PUT' | 'PATCH' | 'DELETE';
  // Retorna la respuesta simulada o null si no aplica a esta petición
  handle: (req: HttpRequest<unknown>) => HttpResponse<unknown> | null;
}

export const wellsMockHandlers: MockHandler[] = [
  {
    urlPattern: /\/api\/v1\/wells$/,
    method: 'GET',
    handle: () =>
      new HttpResponse({
        status: 200,
        body: [
          { well_id: 'MOCK-001', well_name: 'Pozo Alpha', status_code: 'ACTIVE' },
          { well_id: 'MOCK-002', well_name: 'Pozo Beta',  status_code: 'INACTIVE' },
        ],
      }),
  },
];
```

### 13.5. Registro central de handlers

Un archivo central agrega todos los handlers de todos los dominios que tengan mocks activos:

```typescript
// core/http/mock.registry.ts
// REFERENCIA: se agregan handlers conforme se desarrollan las features que los requieren

import { MockHandler } from './mock.interceptor';
import { wellsMockHandlers } from '@wells/mocks';

export const MOCK_HANDLERS: MockHandler[] = [
  ...wellsMockHandlers,
  // ...operationsMockHandlers,  ← se agrega cuando operations lo necesite
];
```

### 13.6. El interceptor mock

Se registra **antes** que `authInterceptor` y `errorInterceptor` en la cadena, de modo que las peticiones mockeadas nunca llegan a la red:

```typescript
// core/http/mock.interceptor.ts
import { HttpInterceptorFn, HttpResponse } from '@angular/common/http';
import { of } from 'rxjs';
import { environment } from '@env/environment';
import { MOCK_HANDLERS } from './mock.registry';

export const mockInterceptor: HttpInterceptorFn = (req, next) => {
  // Si los mocks están desactivados, pasar la petición sin tocarla
  if (!environment.featureFlags.useMocks) return next(req);

  const handler = MOCK_HANDLERS.find(h => {
    const urlMatch =
      typeof h.urlPattern === 'string'
        ? req.url.includes(h.urlPattern)
        : h.urlPattern.test(req.url);
    return urlMatch && h.method === req.method;
  });

  if (handler) {
    const response = handler.handle(req);
    if (response) return of(response); // Retorna el mock como Observable
  }

  // Si no hay handler para esta petición, continúa hacia la red
  return next(req);
};
```

```typescript
// app.config.ts — orden de interceptores (mock siempre primero)
provideHttpClient(
  withInterceptors([mockInterceptor, authInterceptor, errorInterceptor])
)
```

### 13.7. Ciclo de vida de un mock

Un mock nace con una feature y muere con la entrega de su endpoint real. Nunca debe cruzar ese umbral.

```
[INICIO DE FEATURE]
  1. Endpoint no disponible → activar useMocks: true en environment.ts
  2. Definir contrato de datos (DTO) según especificación funcional
  3. Crear handler en domains/[dominio]/mocks/[dominio].mock.ts
  4. Registrar handler en core/http/mock.registry.ts
  5. Desarrollar, maquetar y validar la feature con datos controlados

[ENTREGA DEL ENDPOINT REAL]
  6. Conectar el servicio al API real — sin cambios en componentes ni store
  7. Eliminar handler del registry (mock.registry.ts)
  8. Eliminar archivo de mock si el dominio no tiene más handlers activos
  9. Desactivar useMocks: false en environment.ts si no quedan mocks activos
```

**Señal de alerta:** si un mock permanece activo después del merge de su feature, es deuda técnica — debe eliminarse en el mismo sprint.

> **Referencia:** Los handlers, estructuras de datos mock y endpoints mostrados son ilustrativos. Los datos reales de cada mock se definen según el contrato de API establecido en la especificación funcional de cada feature.

---

## 14. Sistema de Design Tokens (Variables CSS)

### 14.1. Principio y por qué

**Está estrictamente prohibido hardcodear colores, familias tipográficas, tamaños de fuente o cualquier valor visual directamente** en templates HTML, archivos TypeScript o estilos de componente.

Todo valor visual del sistema proviene de **CSS custom properties (variables CSS)** definidas globalmente. Este enfoque garantiza:

- **Un solo punto de cambio:** modificar el color primario del sistema implica cambiar una variable, no buscar en 300 archivos.
- **Consistencia garantizada:** Tailwind y PrimeNG apuntan a los mismos tokens — es imposible que diverjan.
- **Preparación para theming:** dark mode, white-labeling o ajustes de marca se implementan cambiando el bloque `:root`, no el código de los componentes.
- **Colaboración UI/UX ágil:** el equipo de diseño trabaja sobre los tokens; el equipo de desarrollo aplica clases semánticas sin negociar valores.

### 14.2. Estructura de archivos

Los tokens viven en la carpeta `src/styles/`, separados del CSS global de la app:

```
src/
├── styles/
│   ├── _tokens.css       ← Design tokens: paleta primitiva + tokens semánticos
│   ├── _typography.css   ← Escala tipográfica y familias de fuente
│   └── _overrides.css    ← Override de variables CSS de PrimeNG y otras librerías
└── styles.css            ← Punto de entrada: importa Tailwind + archivos de styles/
```

```css
/* src/styles.css */
@tailwind base;
@tailwind components;
@tailwind utilities;

@import './styles/tokens';
@import './styles/typography';
@import './styles/overrides';
```

### 14.3. Tokens: dos niveles

Los tokens se organizan en **dos capas** para separar los valores brutos de su significado semántico:

**Capa 1 — Primitivos:** los colores, tamaños y valores concretos. Nunca se usan directamente en componentes.

```css
/* src/styles/_tokens.css — REFERENCIA: valores reales según identidad visual del proyecto */
:root {
  /* Paleta de color — valores primitivos */
  --primitive-blue-50:   #eff6ff;
  --primitive-blue-100:  #dbeafe;
  --primitive-blue-500:  #3b82f6;
  --primitive-blue-600:  #2563eb;
  --primitive-blue-700:  #1d4ed8;
  --primitive-gray-50:   #f9fafb;
  --primitive-gray-100:  #f3f4f6;
  --primitive-gray-300:  #d1d5db;
  --primitive-gray-500:  #6b7280;
  --primitive-gray-700:  #374151;
  --primitive-gray-900:  #111827;
  --primitive-red-500:   #ef4444;
  --primitive-green-500: #22c55e;
  --primitive-yellow-500:#eab308;
  --primitive-white:     #ffffff;
}
```

**Capa 2 — Semánticos:** asignan significado a los primitivos. Estos son los que usan los componentes.

```css
:root {
  /* Colores semánticos de marca */
  --color-primary:         var(--primitive-blue-600);
  --color-primary-hover:   var(--primitive-blue-700);
  --color-primary-light:   var(--primitive-blue-50);
  --color-primary-contrast:var(--primitive-white);

  /* Superficies y fondos */
  --color-surface:         var(--primitive-white);
  --color-surface-alt:     var(--primitive-gray-50);
  --color-border:          var(--primitive-gray-300);

  /* Texto */
  --color-text-primary:    var(--primitive-gray-900);
  --color-text-secondary:  var(--primitive-gray-500);
  --color-text-disabled:   var(--primitive-gray-300);

  /* Estados de feedback */
  --color-error:           var(--primitive-red-500);
  --color-success:         var(--primitive-green-500);
  --color-warning:         var(--primitive-yellow-500);

  /* Espaciado base (referencia para escala) */
  --space-unit: 0.25rem; /* 4px — base de la escala de Tailwind */
}
```

### 14.4. Tokens tipográficos

```css
/* src/styles/_typography.css — REFERENCIA: fuentes según identidad visual del proyecto */
:root {
  /* Familias */
  --font-family-sans:  'Inter', system-ui, sans-serif;
  --font-family-mono:  'JetBrains Mono', monospace;

  /* Escala de tamaños */
  --font-size-xs:   0.75rem;   /* 12px */
  --font-size-sm:   0.875rem;  /* 14px */
  --font-size-base: 1rem;      /* 16px */
  --font-size-lg:   1.125rem;  /* 18px */
  --font-size-xl:   1.25rem;   /* 20px */
  --font-size-2xl:  1.5rem;    /* 24px */
  --font-size-3xl:  1.875rem;  /* 30px */

  /* Pesos */
  --font-weight-normal:   400;
  --font-weight-medium:   500;
  --font-weight-semibold: 600;
  --font-weight-bold:     700;

  /* Interlineado */
  --line-height-tight:  1.25;
  --line-height-normal: 1.5;
  --line-height-relaxed:1.75;
}
```

### 14.5. Override de librerías (`_overrides.css`)

PrimeNG expone sus propias variables CSS con prefijo `--p-`. Este archivo las remapea a nuestros tokens semánticos. Así, si cambia el color primario del sistema, PrimeNG lo refleja automáticamente.

```css
/* src/styles/_overrides.css */

/* PrimeNG — remap de variables internas a tokens del sistema */
:root {
  --p-primary-color:           var(--color-primary);
  --p-primary-hover-color:     var(--color-primary-hover);
  --p-primary-contrast-color:  var(--color-primary-contrast);
  --p-surface-0:               var(--color-surface);
  --p-surface-ground:          var(--color-surface-alt);
  --p-text-color:              var(--color-text-primary);
  --p-text-muted-color:        var(--color-text-secondary);
  --p-content-border-color:    var(--color-border);

  /* Al integrar nuevas librerías, agregar sus overrides aquí siguiendo el mismo patrón */
}
```

### 14.6. Integración con Tailwind

`tailwind.config.js` extiende la paleta para que las clases de Tailwind (`bg-primary`, `text-error`, etc.) apunten a los mismos tokens semánticos:

```javascript
// tailwind.config.js
module.exports = {
  content: ['./src/**/*.{html,ts}'],
  theme: {
    extend: {
      colors: {
        primary:   'var(--color-primary)',
        'primary-hover':  'var(--color-primary-hover)',
        'primary-light':  'var(--color-primary-light)',
        surface:   'var(--color-surface)',
        'surface-alt':    'var(--color-surface-alt)',
        border:    'var(--color-border)',
        error:     'var(--color-error)',
        success:   'var(--color-success)',
        warning:   'var(--color-warning)',
        'text-primary':   'var(--color-text-primary)',
        'text-secondary': 'var(--color-text-secondary)',
      },
      fontFamily: {
        sans: 'var(--font-family-sans)',
        mono: 'var(--font-family-mono)',
      },
      fontSize: {
        xs:   'var(--font-size-xs)',
        sm:   'var(--font-size-sm)',
        base: 'var(--font-size-base)',
        lg:   'var(--font-size-lg)',
        xl:   'var(--font-size-xl)',
        '2xl':'var(--font-size-2xl)',
        '3xl':'var(--font-size-3xl)',
      },
    },
  },
  plugins: [require('tailwindcss-primeui')],
};
```

**Resultado:** en templates, solo se usan clases semánticas:
```html
<!-- ✅ Correcto: clases semánticas que apuntan a tokens -->
<button class="bg-primary text-white hover:bg-primary-hover font-semibold text-sm">
  Guardar
</button>

<!-- ❌ Prohibido: valores hardcodeados o clases arbitrarias de Tailwind -->
<button class="bg-[#2563eb] text-white text-[14px]">
  Guardar
</button>
```

### 14.7. Reglas de uso en componentes

| Regla | Correcto | Prohibido |
|---|---|---|
| Colores de fondo | `bg-primary`, `bg-surface` | `bg-blue-600`, `bg-[#fff]` |
| Colores de texto | `text-text-primary`, `text-error` | `text-gray-900`, `text-[#111]` |
| Colores de borde | `border-border` | `border-gray-300` |
| Tipografía | `font-sans`, `text-sm`, `font-semibold` | `font-['Inter']`, `text-[14px]` |
| Variables en TS/CSS | `var(--color-primary)` | `#2563eb` |
| Colores en estilos inline | Nunca inline, siempre clase | `[style]="'color:#111'"` |

### 14.8. Flujo para cambiar el tema visual

Cambiar toda la estética del sistema — colores, tipografía — se reduce a editar `_tokens.css` y `_typography.css`:

```
1. Identificar el token semántico a cambiar (ej: --color-primary)
2. Cambiar el primitivo al que apunta (ej: --primitive-green-600)
   o agregar un nuevo primitivo y apuntar el semántico a él
3. Tailwind y PrimeNG reflejan el cambio automáticamente
4. Ningún componente ni template requiere modificación
```

> **Referencia:** Los valores de color, tipografía y escala mostrados son ilustrativos. Los tokens reales — primitivos y semánticos — se definen según la identidad visual y el sistema de diseño establecido para el proyecto.

---

## 15. Metodología de Desarrollo: Spec-Driven Development (SDD)

Este proyecto adopta la metodología **SDD (Spec-Driven Development)** de INTERKONT. La especificación precede siempre a la implementación: **ningún archivo de código puede crearse sin que existan los artefactos documentales aprobados.**

### 15.1. Estructura de Documentación

```text
/                                        # Raíz del repositorio
├── CONSTITUTION.md                      # GLOBAL: reglas inmutables de todo el proyecto (este archivo)
├── CLAUDE.md                            # Ancla de contexto para Claude Code
├── GEMINI.md                            # Ancla de contexto para Gemini
├── blueprint.md                         # Mapa funcional del sistema (rutas, dominios, flujos)
└── specs/
    └── features/                        # Un directorio por cada feature a desarrollar
        ├── 001-auth/                    # NNN = número secuencial │ nombre = slug de la feature
        │   ├── spec.md                  #   El "QUÉ": historias de usuario, criterios de aceptación
        │   ├── plan.md                  #   El "CÓMO": árbol de archivos, componentes, estado
        │   └── tasks.md                 #   El "CUÁNDO": tareas atómicas con checkboxes
        └── NNN-nombre-feature/
            └── ...
```

**Regla absoluta:** `CONSTITUTION.md` y `blueprint.md` son los únicos artefactos globales. Los archivos `spec.md`, `plan.md` y `tasks.md` son **exclusivos de cada feature** y viven dentro de su carpeta `specs/features/NNN-nombre/`. Nunca se comparten entre features.

### 15.2. Responsabilidad de cada Artefacto

| Artefacto | Alcance | Propósito | Quién lo valida |
|---|---|---|---|
| `CONSTITUTION.md` | Global | Reglas arquitectónicas inmutables | Líder técnico — consenso del equipo |
| `blueprint.md` | Global | Mapa funcional del sistema | Equipo — se actualiza si cambia el sistema |
| `spec.md` | Por feature | *Qué* construir: historias, criterios Given/When/Then, edge cases | Desarrollador — antes de generar `plan.md` |
| `plan.md` | Por feature | *Cómo* construirlo: árbol de archivos, estrategia de estado, dependencias | Desarrollador — debe respetar `CONSTITUTION.md` |
| `tasks.md` | Por feature | Desglose atómico en checkboxes, ordenado por dependencias | Desarrollador — granularidad: 1 tarea = 1 archivo |

### 15.3. Flujo Obligatorio por Feature

```
1. Abrir rama de trabajo (git)
2. Crear directorio  specs/features/NNN-nombre/
3. Redactar spec.md  → Revisar edge cases y criterios medibles. Aprobar antes de continuar.
4. Redactar plan.md  → Validar que respeta CONSTITUTION.md. Aprobar antes de continuar.
5. Redactar tasks.md → Asegurar granularidad atómica (1 archivo por tarea). Aprobar antes de continuar.
6. Implementar       → Tarea por tarea. Marcar [x] solo cuando cumple estándares de calidad.
7. Actualizar        → Sección "Feature Activa" en CLAUDE.md y GEMINI.md al cambiar de feature.
```

> **Regla de gobernanza SDD:** La IA genera propuestas; el desarrollador es el responsable final. Nunca avanzar al siguiente artefacto sin revisar e iterar el anterior.

---

## 16. Patrón App Shell: Main Layout

> **Implementación actual:** `AuthLayoutComponent` y `MainLayoutComponent` (incluidos Sidebar y TopHeader) provienen de `appgop-web`. No existen implementaciones locales de estos componentes de UI — todo el layout visual vive en el UIKit. Lo que existe localmente es únicamente un **shell inteligente** que conecta el Store con el UIKit.

### 16.1. Dos layouts, dos contextos

La aplicación tiene exactamente **dos layouts raíz** que nunca coexisten:

| Layout | Rutas | Sidebar | TopHeader | Guard | Origen |
|---|---|---|---|---|---|
| `AuthLayoutComponent` | `/login`, `/forgot-password` | No | No | `noAuthGuard` | `appgop-web` |
| `MainLayoutComponent` | Todas las rutas autenticadas | Sí | Sí | `authGuard` | `appgop-web` (envuelto por shell local) |

### 16.2. Composición del Main Layout

El UIKit provee el layout visual completo. El shell local (`core/layout/main-layout/main-layout.component.ts`) actúa como **smart parent**: lee el Store, computa los nav items filtrados por RBAC y los pasa al UIKit:

```
MainLayoutComponent (UIKit — appgop-web)     ← layout visual: sidebar + header + outlet
    ↑ inputs: navItemsOverride, user, company, etc.
    ↑ outputs: logout

core/layout/main-layout/main-layout.component.ts  ← Smart shell local
    - Lee Store → currentUser
    - Computed → filteredNavItems (RBAC)
    - Computed → uiNavItems (formato UIKit)
    - Despacha AuthActions.logout()
```

**Regla de la separación:**
- La UI (sidebar, header, íconos, estilos) vive **solo** en `appgop-web`
- La lógica de negocio (RBAC, sesión, Store) vive **solo** en el shell local
- El shell no tiene HTML propio ni estilos — solo un `<app-main-layout ... />` en su template

### 16.3. Registro en rutas

El `MainLayoutComponent` actúa como **shell padre** de todas las rutas autenticadas. Los dominios se cargan como `children`. Los guards específicos por rol se definen en la especificación funcional de cada feature:

```typescript
// app.routes.ts — patrón estructural
{
  path: '',
  component: MainLayoutComponent,
  canActivate: [authGuard],
  children: [
    { path: '', pathMatch: 'full', loadComponent: () => DashboardComponent },
    { path: 'wells',       loadChildren: () => wellsRoutes },
    { path: 'operations',  loadChildren: () => operationsRoutes },
    { path: 'production',  loadChildren: () => productionRoutes },
    { path: 'admin',       loadChildren: () => adminRoutes },
  ],
},
```

> **Regla:** El `MainLayoutComponent` se renderiza una sola vez y persiste durante toda la navegación entre dominios. El `<router-outlet>` interno cambia solo el contenido — Sidebar y TopHeader nunca se destruyen ni re-renderizan al cambiar de ruta.

### 16.4. Estado del sidebar

El estado collapsed/expanded es **UI local** del `MainLayoutComponent` → se gestiona con **Signals**, no con NgRx.

```typescript
// main-layout.component.ts
sidebarCollapsed = signal(false);

onToggleSidebar(): void {
  this.sidebarCollapsed.update(v => !v);
}
```

En **mobile** (< `lg`), el sidebar inicia colapsado por defecto y se abre como overlay.

---

## 17. Navegación y RBAC en Sidebar

> **Implementación actual:** El `SidebarComponent` visual proviene del UIKit (`appgop-web`). No existe una implementación local de este componente. El RBAC se implementa en el shell de `MainLayoutComponent` computando los nav items antes de pasarlos al UIKit.

### 17.1. Interface NavItem local

La definición local de nav items vive en `core/layout/sidebar/nav-items.ts`. Es la **única fuente de verdad** del menú de navegación de este proyecto:

```typescript
// core/layout/sidebar/nav-items.ts

import { UserRole } from '@shared/models';

export interface NavItem {
  key:      string;       // Clave para APP_LOCALE.sidebar[key]
  icon?:    string;       // Clase de ícono (PrimeNG icon class)
  route?:   string;       // Ruta absoluta de navegación
  roles:    UserRole[];   // Roles que pueden VER este item
  children?: NavItem[];   // Sub-items
}

export const NAV_ITEMS: NavItem[] = [ /* ... definidos por feature spec */ ];
```

### 17.2. Flujo RBAC en el shell local

El shell `MainLayoutComponent` (local) filtra `NAV_ITEMS` por rol del usuario autenticado antes de pasarlos al UIKit:

```
Store → selectCurrentUser
          ↓
computed filteredNavItems()   ← filtra por user.role
          ↓
computed uiNavItems()         ← convierte NavItem → NavItem del UIKit (id, label, icon, route)
          ↓
<app-main-layout [navItemsOverride]="uiNavItems()" />   ← UIKit renderiza el sidebar
```

**Conversión de formato:** Los labels se obtienen de `APP_LOCALE.sidebar[item.key]` — nunca se hardcodean en el array.

### 17.3. Defensa en profundidad: Sidebar + Guard

El sidebar oculta items por rol (**primera capa** — UX, filtrado en el shell). El `roleGuard` en la ruta bloquea el acceso por URL directa (**segunda capa** — seguridad). Ambas capas deben estar sincronizadas con la misma fuente de roles de `NAV_ITEMS`.

**Nunca confiar solo en ocultar el link en el sidebar** — el guard es la defensa real (§10.A01).

### 17.4. Responsabilidades

| Componente | Tipo | Ubicación | Responsabilidad |
|---|---|---|---|
| `MainLayoutComponent` (local) | **Smart shell** | `core/layout/main-layout/` | Inyecta Store, computa nav items filtrados por RBAC, pasa al UIKit, despacha logout |
| `MainLayoutComponent` (UIKit) | **Dumb UI** | `appgop-web` | Renderiza sidebar, header, outlet — no conoce NgRx ni negocio |
| `nav-items.ts` | Config | `core/layout/sidebar/` | Fuente de verdad de rutas, iconos y roles |

> **Regla:** El UIKit nunca inyecta Store ni AuthService. Toda la lógica de obtención de datos vive en el smart shell local.

---

## 18. TopHeader

> **Implementación actual:** El `HeaderComponent` (top header) proviene del UIKit (`appgop-web`) y está embebido dentro del `MainLayoutComponent` del UIKit. No existe un `TopHeaderComponent` local — fue eliminado al adoptar el UIKit.

### 18.1. Responsabilidades del Header (UIKit)

El header del UIKit recibe datos del usuario desde el shell local a través del `MainLayoutComponent`:

| Dato | Flujo |
|---|---|
| Nombre del usuario | `store.select(selectCurrentUser)` → shell → UIKit `[userName]` |
| Rol del usuario | `store.select(selectCurrentUser)` → shell → UIKit `[userRole]` |
| Empresa/tenant | `store.select(selectCurrentUser)` → shell → UIKit `[company]` |
| Logout | UIKit emite `(logoutClicked)` → shell despacha `AuthActions.logout()` |

### 18.2. Textos del Header

Los textos visibles del header (tooltip de logout, aria-labels, etc.) son responsabilidad del UIKit. Si se requiere cambiar un texto, se modifica en `appgop-web` y se reconstruye la librería (§19.5).

El `APP_LOCALE` global puede incluir textos de `topHeader` para el shell local si son necesarios (ej. mensajes de confirmación de logout), pero los labels internos del header son del UIKit.

### 18.3. Ubicación de componentes — estado actual

| Componente | Origen | Ubicación | Tipo |
|---|---|---|---|
| `MainLayoutComponent` (smart) | Local | `core/layout/main-layout/` | Smart shell — conecta Store al UIKit |
| `MainLayoutComponent` (UI) | UIKit | `appgop-web` | Layout visual completo (sidebar + header + outlet) |
| `AuthLayoutComponent` | UIKit | `appgop-web` | Layout de autenticación |
| `nav-items.ts` | Local | `core/layout/sidebar/` | Config de rutas, iconos y roles |
| ~~`SidebarComponent`~~ | ~~Local~~ | ~~eliminado~~ | Supersedido por UIKit |
| ~~`TopHeaderComponent`~~ | ~~Local~~ | ~~eliminado~~ | Supersedido por UIKit (embedded en MainLayout) |

---

## 19. UIKit `appgop-web`: Integración de Páginas y Layouts

`appgop-web` es el UIKit institucional de GOP 360°. Vive en un repositorio separado (`appgop-web`) y se integra en este proyecto como librería local mediante `libs/appgop-web/` (versionado en git). **Ningún estilo, componente de UI ni layout se escribe directamente en este proyecto** — toda la presentación proviene del UIKit.

### 19.1. Qué provee el UIKit

| Tipo | Ejemplos | Importación |
|---|---|---|
| Páginas completas | `LoginPage`, `ForgotPasswordPage`, `WellCreatePage`, `WellOldCreatePage`, `UsersRolesPage` | `import { LoginPage } from 'appgop-web'` |
| Layouts raíz | `AuthLayoutComponent`, `MainLayoutComponent` | `import { AuthLayoutComponent } from 'appgop-web'` |
| Organismos | `WellFormComponent`, `UsersTableComponent`, `UserFormModalComponent` | `import { ... } from 'appgop-web'` |
| Locales por defecto | `LOGIN_LOCALE`, `WELL_CREATE_LOCALE`, etc. | `import { LOGIN_LOCALE } from 'appgop-web'` |
| Modelos/interfaces | `LoginCredentials`, `RecoveryRequest`, `WellFormData` | `import type { ... } from 'appgop-web'` |

### 19.2. Patrón Shell (regla obligatoria)

Cada página del UIKit se consume mediante un **shell de feature** — un componente Angular delgado que:

1. Importa la página del UIKit
2. Conecta el `locale` del proyecto (ver §19.3)
3. Conecta servicios reales / NgRx (inputs/outputs)
4. **No contiene ningún HTML propio ni estilos**

```typescript
// domains/wells/features/well-create/well-create.component.ts
@Component({
  selector: 'app-well-create-feature',
  template: `<app-well-create [locale]="LOCALE" />`,
  imports: [WellCreatePage],
  changeDetection: ChangeDetectionStrategy.OnPush,
})
export class WellCreateComponent {
  protected readonly LOCALE = WELL_CREATE_LOCALE;
}
```

**Reglas del shell:**
- Selector: siempre `app-[feature-name]-feature` (evita colisión con el selector del UIKit)
- `template` inline si no hay más de una línea; `templateUrl` si hay múltiples bindings
- No agrega `styles` ni `styleUrl` — todo el CSS viene del UIKit
- Usa `ChangeDetectionStrategy.OnPush` siempre

### 19.3. Patrón Locale de Shell (regla obligatoria)

Cada shell de feature que usa una página UIKit **debe** tener un archivo `locale.ts`. Su contenido es una **re-exportación directa** desde `'appgop-web'` — nunca una copia ni una definición propia:

```
domains/wells/features/well-create/
├── locale.ts                  ← Re-export desde 'appgop-web' (obligatorio, nunca copiar)
└── well-create.component.ts   ← Shell que pasa [locale]="LOCALE"
```

**Contenido del locale.ts de un shell UIKit:**

```typescript
// domains/wells/features/well-create/locale.ts
// ✅ Correcto: re-export limpio. El locale vive y se versiona en appgop-web.
export { WELL_CREATE_LOCALE } from 'appgop-web';
```

**En el shell component:**

```typescript
// well-create.component.ts
import { WellCreatePage, WELL_CREATE_LOCALE } from 'appgop-web';

protected readonly LOCALE = WELL_CREATE_LOCALE;
```

```html
<!-- template del shell -->
<app-well-create [locale]="LOCALE" />
```

**¿Por qué definir aquí y no en el UIKit?**
El proyecto es la fuente de verdad de sus propios textos. Si mañana se cambia de idioma, se ajusta terminología de negocio o se adapta para otro cliente, todos los cambios ocurren en los `locale.ts` locales sin tocar el UIKit ni recompilar la librería. El UIKit solo garantiza la **estructura** (qué claves existen); el proyecto decide el **contenido**.

Ver §20.4 para el patrón completo de implementación.

### 19.4. Separación de responsabilidades UIKit / Proyecto

| Responsabilidad | UIKit (`appgop-web`) | Proyecto (`appgop-base-web`) |
|---|---|---|
| Estructura HTML y estilos CSS | ✅ | ❌ Prohibido |
| Textos por defecto (defaults del Storybook) | ✅ | — |
| Textos del proyecto (todos los labels visibles) | — | ✅ `locale.ts` del shell |
| Conexión a servicios reales / NgRx | — | ✅ Shell component |
| Navegación post-acción | — | ✅ NgRx Effects |
| Validación de inputs | UIKit valida formato | Shell valida negocio |

### 19.5. Ciclo de actualización de la librería

```
[CAMBIO EN UIKit]
  1. Modificar en appgop-web (nuevo componente, fix visual, etc.)
  2. Ejecutar: npm run build:lib   (en el repo appgop-web)
  3. Copiar dist: dist/lib/ → libs/appgop-web/   (en este repo)
  4. Verificar que no hay errores de compilación: ng build
  5. Commitear libs/appgop-web/ junto con los cambios del shell

[CAMBIO SOLO DE TEXTOS]
  → Editar locale.ts del shell correspondiente. No requiere rebuild del UIKit.

[CAMBIO DE LÓGICA DE NEGOCIO]
  → Editar el shell component (.ts). No requiere rebuild del UIKit.
```

**Regla:** `libs/appgop-web/browser/` y `libs/appgop-web/prerendered-routes.json` están en `.gitignore` — son artefactos del build de la app del Storybook, no de la librería.

### 19.6. Imports: valor vs. tipo

```typescript
// ✅ Correcto: separar imports de valor y de tipo
import { LoginPage, LOGIN_LOCALE } from 'appgop-web';
import type { LoginCredentials }   from 'appgop-web';

// ❌ Error: mezclar en un solo import puede romper el analizador estático de Angular
import { LoginPage, LoginCredentials } from 'appgop-web';
```

**Regla:** Las interfaces y tipos del UIKit siempre se importan con `import type`. Los componentes, locales y constantes usan `import` normal.

### 19.7. Sincronización de Estilos Globales (regla obligatoria)

Los componentes del UIKit dependen de estilos globales que **no están incluidos en el bundle de la librería** — solo existen en `appgop-web/src/styles.scss`. Cuando estos estilos faltan en la consuming app, los componentes importados se ven vacíos o rotos.

**Archivos a mantener sincronizados:**

| Archivo en `appgop-web` | Archivo equivalente en este proyecto |
|---|---|
| `src/styles/_tokens.scss` → tokens semánticos y primitivos | `src/styles/_tokens.css` — FUENTE DE VERDAD, debe ser superset del storybook |
| `src/styles.scss` → reglas globales (`lucide-icon`, `.sr-only`, body) | `src/styles.css` — sección "SYNC con appgop-web" |
| `src/styles/_primeng-overrides.scss` | `src/styles/_overrides.css` — bloques de PrimeNG |

**Síntoma de estilos de sync faltantes:** El componente importado se renderiza con áreas vacías o íconos invisibles, aunque el código compila sin errores.

**Causa más frecuente:** El fix global de `lucide-icon` — sin él todos los íconos colapsan a 0×0 y los componentes que dependen del tamaño del ícono para su layout se ven vacíos.

**Regla de actualización:** Cada vez que se agregue un estilo global nuevo en `appgop-web/src/styles.scss` o `src/styles/_primeng-overrides.scss`, ese cambio debe replicarse en `src/styles.css` o `src/styles/_overrides.css` **en el mismo commit** que el rebuild del dist.

**Checklist al importar un componente nuevo del UIKit:**
- [ ] ¿Los tokens de `_tokens.scss` del storybook están todos en `_tokens.css`? → Comparar y agregar los faltantes
- [ ] ¿Usa `lucide-icon` / `lucide-angular`? → `styles.css` debe tener el fix `display: inline-flex`
- [ ] ¿Usa `p-drawer` o `p-dialog`? → `_overrides.css` debe tener `.p-drawer` y `.p-overlay-mask-*`
- [ ] ¿Usa clases utilitarias globales (`.sr-only`, `.visually-hidden`)? → Verificar que están en `styles.css`
- [ ] ¿Hay variables CSS nuevas en el UIKit (`--px-*`, `--color-*` nuevos)? → Agregar a `_tokens.css` o `_overrides.css`
- [ ] ¿El ícono lucide que usa el componente está registrado en `LucideAngularModule.pick()` en `app.config.ts`?

---

## §20 — Integración con appgop-web (UIKit GOP 360°)

> **Objetivo de esta sección:** cualquier developer que clone el repo debe poder importar un componente nuevo de la librería en menos de 10 minutos leyendo solo este §.

---

### 20.1 Setup global — verificar una sola vez, no repetir por pantalla

#### angular.json — estilos

```json
"styles": [
  "src/styles.css",
  "node_modules/primeicons/primeicons.css"
]
```

`src/styles.css` importa `_tokens.css`, `_typography.css` y `_overrides.css`, aplica los fixes globales de Tailwind, lucide-icon y átomos del UIKit. **No agregar más imports de la librería en styles.**

#### app.config.ts — providers obligatorios

```typescript
import { provideAnimationsAsync } from '@angular/platform-browser/animations/async';
import { MessageService } from 'primeng/api';

providers: [
  provideAnimationsAsync(),   // ← async, no provideAnimations()
  MessageService,             // ← requerido internamente por PrimeNG Toast
  // ...resto de providers
]
```

> `ToastService` de `appgop-web` es `providedIn: 'root'` — no necesita declararse aquí.

#### app.html — layout raíz

```html
<app-toast key="gop" position="top-right" [life]="4000" />
<router-outlet />
```

#### app.ts — imports del componente raíz

```typescript
import { ToastComponent } from 'appgop-web';

@Component({
  imports: [RouterOutlet, ToastComponent],
  // ...
})
```

---

### 20.2 Catálogo completo de exports de appgop-web

Importar **siempre** de `'appgop-web'` (nunca rutas relativas a `libs/`):

```typescript
// LAYOUTS
import { AuthLayoutComponent, MainLayoutComponent } from 'appgop-web';

// PÁGINAS  — siempre vienen en par con su locale
import { LoginPage,          LOGIN_LOCALE          } from 'appgop-web';
import { ForgotPasswordPage, FORGOT_PASSWORD_LOCALE } from 'appgop-web';
import { WellCreatePage,     WELL_CREATE_LOCALE     } from 'appgop-web';
import { WellOldCreatePage,  WELL_OLD_CREATE_LOCALE } from 'appgop-web';
import { WellManagementPage, WELL_MANAGEMENT_LOCALE } from 'appgop-web';
import { UsersRolesPage,     USERS_ROLES_LOCALE     } from 'appgop-web';

// ORGANISMOS
import {
  HeaderComponent, SidebarComponent,
  WellFormComponent, WellOldFormComponent,
  WellExplorerComponent, WellTechnicalCardComponent,
  WellLifeCycleComponent, ActiveFormsTableComponent,
  ContractDetailsCardComponent, NotificationsPanelComponent,
  UserFormModalComponent, UsersTableComponent, WellActionsPanelComponent,
} from 'appgop-web';

// MOLÉCULAS
import {
  FormFieldComponent, NavComponent, UserProfileComponent,
  UwiBannerComponent, WellStepsNavComponent, WellSummaryCardComponent,
  PaginatorComponent, SignatureUploadComponent,
} from 'appgop-web';

// ÁTOMOS
import {
  ButtonComponent, InputComponent, SelectComponent,
  AvatarComponent, IconButtonComponent, BadgeComponent,
  LogoComponent, BreadcrumbComponent, TypographyComponent,
  SkeletonComponent, SpinnerComponent, ToastComponent,
  CheckboxComponent, RadioComponent,
  ChipComponent, StatusPillComponent, TooltipComponent, LinkComponent,
} from 'appgop-web';

// SERVICIO UI (providedIn: 'root' — solo inyectar, no declarar en providers)
import { ToastService } from 'appgop-web';
// inject(ToastService) → .success(summary, detail?) .error() .warn() .info()
```

**Inputs de loading por página:**

| Página | Input | Tipo |
|--------|-------|------|
| `WellManagementPage` | `[loading]` | `boolean` |
| `WellCreatePage` | `[isLoading]` | `boolean` |
| `WellOldCreatePage` | `[isLoading]` | `boolean` |
| `UsersRolesPage` | `[loading]` | `boolean` |

---

### 20.3 Patrón shell component — la única forma válida

Cada pantalla es un **shell component** en `domains/<dominio>/features/<nombre>/`:

```typescript
// domains/wells/features/well-create/well-create.component.ts
@Component({
  selector: 'app-well-create-feature',
  standalone: true,
  changeDetection: ChangeDetectionStrategy.OnPush,
  imports: [WellCreatePage],
  template: `
    <app-well-create
      [locale]="LOCALE"
      [isLoading]="loading()"
      [operadorasOptions]="operadoras()"
      (formSubmitted)="onSubmit($event)"
    />
  `,
})
export class WellCreateComponent implements OnInit {
  protected readonly LOCALE = WELL_CREATE_LOCALE;   // locale sin modificar

  private  readonly store    = inject(Store);
  protected readonly loading  = signal(true);
  protected readonly operadoras = this.store.selectSignal(selectOperadoras);

  ngOnInit(): void {
    // Mientras no haya store real: simular carga con timeout
    setTimeout(() => this.loading.set(false), 1500);
    // Con store real:
    // this.store.dispatch(WellActions.loadOptions());
  }

  onSubmit(data: unknown): void {
    this.store.dispatch(WellActions.create({ data }));
  }
}
```

**Reglas del shell:**
- `changeDetection: ChangeDetectionStrategy.OnPush` — obligatorio
- `protected readonly LOCALE = WELL_CREATE_LOCALE` — locale tal como viene de la librería, sin copiar ni modificar
- `loading` inicia en `true`, se pone en `false` cuando los datos están listos (store selector o timeout temporal)
- El shell NO contiene HTML ni lógica visual — solo conecta inputs/outputs al store

---

### 20.4 Locales — regla de oro

**El `locale.ts` de cada feature es la fuente de verdad de todos los textos del proyecto.** El UIKit solo provee la estructura (qué claves existen); el contenido real — títulos, labels, placeholders, mensajes, aria-labels — se define aquí.

**Por qué:** Si el proyecto cambia de idioma, ajusta un término de negocio o adapta textos para un cliente, todo ocurre en `locale.ts` sin tocar el UIKit ni recompilar la librería.

**Patrón obligatorio:**

```typescript
// domains/wells/features/well-create/locale.ts
import { WELL_CREATE_LOCALE as _UIK } from 'appgop-web';  // ← solo para el tipo

export const WELL_CREATE_LOCALE = {
  page: {
    title: 'Crear Pozo Nuevo',
    overline: 'Módulo de Registro y Fiscalización de Activos',
  },
  actionBar: { ... },
  // ← TODOS los campos, sin omitir ninguno
} as unknown as typeof _UIK;
// El cast es necesario porque el UIKit usa 'as const' con tipos literales estrictos.
// Si el UIKit agrega campos nuevos, el build fallará en tiempo de ejecución (el componente
// recibirá undefined) — revisar el bundle al hacer rebuild de la librería.
```

**El shell importa siempre desde `'./locale'`**, nunca directamente de `'appgop-web'`:

```typescript
// well-create.component.ts
import { WellCreatePage } from 'appgop-web';
import { WELL_CREATE_LOCALE } from './locale';   // ← fuente de verdad del proyecto
```

**Reglas:**
- Incluir **todos** los campos que el UIKit espera — no omitir ninguno
- El `_UIK` importado se usa solo como referencia de tipo, nunca en runtime
- Si el UIKit agrega un campo nuevo → copiarlo al `locale.ts` local en el mismo commit del rebuild
- El `as unknown as typeof _UIK` es el cast estándar; no usar `as any`

---

### 20.5 Modo oscuro

```typescript
// Activar
document.documentElement.setAttribute('data-theme', 'modern');

// Desactivar
document.documentElement.removeAttribute('data-theme');
```

---

### 20.6 Skeleton integrado

Todas las páginas tienen un skeleton animado que se activa pasando `[loading]="true"` o `[isLoading]="true"`. **No crear skeletons propios.**

```html
<!-- ✅ Correcto -->
<app-well-create [isLoading]="loading()" />

<!-- ❌ Prohibido -->
<app-skeleton *ngIf="loading()" />
<div class="skeleton-custom" *ngIf="loading()"></div>
```

---

### 20.7 Sincronización de estilos — fixes globales en styles.css

> **Ver §19.7** para el detalle completo. Resumen operativo:

Los componentes de `appgop-web` dependen de estilos globales que NO vienen en el bundle de la librería. Están en `src/styles.css`, `src/styles/_tokens.css` y `src/styles/_overrides.css`.

**Checklist al importar un componente nuevo:**

| Qué revisar | Archivo |
|-------------|---------|
| Nuevo token CSS (`--color-*`, `--px-*`) | `src/styles/_tokens.css` |
| Nuevo componente PrimeNG (Dialog, Tabs, DatePicker…) | `src/styles/_overrides.css` |
| Nuevo ícono Lucide | `LucideAngularModule.pick({...})` en `app.config.ts` |
| Fix display de nuevo átomo (`app-x`) | `src/styles.css` |

**Síntoma de token faltante:** labels invisibles, icon boxes vacíos, colores incorrectos.  
**Síntoma de ícono no registrado:** espacio vacío donde debería estar el ícono.

---

### 20.8 Reglas absolutas

| ✅ Correcto | ❌ Prohibido |
|------------|-------------|
| Shell component conecta inputs/outputs al store | Lógica NgRx dentro de componentes de librería |
| `[loading]="true"` activa el skeleton integrado | Crear skeletons o spinners propios |
| `locale.ts` re-exporta desde `'appgop-web'` (una línea) | Copiar, extender o recrear locale localmente |
| `inject(ToastService)` para notificaciones en features | Usar `MessageService` directamente en features |
| `<app-toast>` una sola vez en `app.html` | Múltiples instancias de toast |
| `provideAnimationsAsync()` | `provideAnimations()` |
| Importar de `'appgop-web'` | Importar de rutas relativas a `libs/` |
| Fix visual de un componente UIKit → modificar en `appgop-web`, rebuild | Agregar estilos ad-hoc en templates del consuming app |
| Override de variables PrimeNG (`--p-*`) en `_overrides.css` | Override en `styles` del `@Component` |

**Sobre `!important` en `_overrides.css`:**

El archivo `src/styles/_overrides.css` puede usar `!important` **únicamente** para corregir estilos inyectados en runtime por `@primeuix/styles` que no pueden ganarse con especificidad normal. Cada bloque con `!important` debe incluir un comentario explicando qué bug de PrimeNG está corrigiendo:

```css
/* _overrides.css */
/* Fix: @primeuix/styles inyecta .p-inputicon después del component style, ganando la cascada.
   Se necesita !important porque el orden de inyección runtime no es controlable. */
.gop-select-panel .p-select-header .p-inputicon {
  margin-top: 0 !important;
  inset-inline-start: 0.5rem !important;
}
```

**Prohibido:** `!important` en templates de componentes, en `styleUrl` de shells, o para sobreescribir el diseño del UIKit en lugar de reportarlo como bug.

---

## §21 — Contrato de consumo del backend GOP

> **Fuente normativa:** CONSTITUTION-ba.md v1.8, secciones §9.1–§9.9, §11.5. Esta sección traduce ese contrato a las reglas operativas del FE. Si hay contradicción, manda el backend.

### 21.1 Principio — sin envelope en respuestas de éxito

El backend GOP **no envuelve** las respuestas de éxito. El cuerpo de la respuesta **es el DTO** (o está vacío en `204 No Content`). Los errores siempre usan **ProblemDetails RFC 7807**. Los metadatos (paginación, versión, localización) viajan en **headers HTTP**.

| Verbo / escenario | Status | Body | Header relevante |
|---|---|---|---|
| `GET /resources` | 200 | `TResourceListItemDto[]` | `X-Pagination` |
| `GET /resources/{id}` | 200 | `TResourceDetailDto` | `ETag` |
| `POST /resources` (create) | 201 | `TResourceDetailDto` | `Location`, `ETag` |
| `POST /resources/{id}/action` (sync) | 200 | `TResultDto` | — |
| `POST /resources/{id}/action` (async) | 202 | `{ operationId, statusUrl }` | `Location` |
| `POST /resources/bulk-import` | 200 | `{ totalProcessed, successful, failed, errors[] }` | — |
| `PUT /resources/{id}` | 200 | `TResourceDetailDto` | `ETag` |
| `PATCH /resources/{id}` | 200 | `TResourceDetailDto` | `ETag` |
| `DELETE /resources/{id}` (hard) | 204 | — | — |
| `DELETE /resources/{id}` (soft) | 200 | `TResourceDetailDto` | — |
| Cualquier error | 4xx/5xx | `ProblemDetails` (RFC 7807) | — |

**Reglas:**
- **PROHIBIDO** servicios que asuman envelopes tipo `{ status, data, message }`, `{ success, result }` u otros.
- El tipado en `HttpClient.get<T>` es siempre `T = DTO directo`, no `ApiResponse<T>`.

```typescript
// ✅ Correcto
this.http.get<WellDetailDto>(API.wells.byId(id));

// ❌ Prohibido
this.http.get<ApiResponse<WellDetailDto>>(API.wells.byId(id));
```

### 21.2 Fechas, zona horaria y serialización

El backend emite **ISO 8601 UTC** (`yyyy-MM-ddTHH:mm:ss.fffZ`) o **`yyyy-MM-dd`** para fechas sin hora. El FE es responsable de convertir a la zona horaria del usuario (`America/Bogota` por defecto).

**Reglas:**
- Parseo: el **mapper** convierte `string` ISO → `Date` (o Luxon `DateTime` si se adopta). Los componentes nunca reciben strings sin parsear.
- Render: pipes del FE (`| date:'dd/MM/yyyy HH:mm':'America/Bogota'`) o un pipe wrapper (`| appDateTime`) con la TZ leída desde configuración/preferencia del usuario.
- Envío al backend: los `DateTime` se serializan en **UTC** antes del submit. Nunca mandar fecha local sin offset.
- Enums del backend llegan como **string** — el FE los tipa como union literal (`'Active' | 'Inactive' | ...`), nunca como `number`.

**Prohibido:** formatos locales (`"21/04/2026"`), epoch numérico, strings ambiguos en payloads hacia el backend.

### 21.3 Errores: ProblemDetails RFC 7807

Todos los errores del backend llegan con este formato (ver §5 para el interceptor):

```json
{
  "type": "https://gop.anh.gov.co/errors/well-already-exists",
  "title": "Pozo duplicado",
  "status": 409,
  "detail": "Ya existe un pozo con UWI AB-123 para el operador 42.",
  "instance": "/api/v1/wells",
  "traceId": "0HMVA7Q9PLJ9P:00000001",
  "code": "WELL_ALREADY_EXISTS",
  "errors": {
    "uwi": ["el UWI ya está registrado"]
  }
}
```

- `errors{}` solo aparece en validaciones (típicamente `422`).
- `code` (extensión GOP) sirve al FE para ramificar comportamiento sin parsear strings: ej. mostrar un CTA específico para `WELL_ALREADY_EXISTS`.
- `traceId` se muestra al usuario en errores `500` (_"Error interno. Comparte este código con soporte: {{ traceId }}"_).

### 21.4 Concurrencia optimista: `ETag` + `If-Match`

Recursos que soportan `PUT`/`PATCH` devuelven un header `ETag` derivado de `rowVersion` (ver §6.3). El FE debe:

1. Guardar el `ETag` recibido en el GET/POST/PUT/PATCH anterior.
2. Enviarlo como `If-Match: "<etag>"` en el siguiente PUT/PATCH.
3. Si el backend responde `412 Precondition Failed`, el FE debe:
   - Mostrar toast/dialog: _"El recurso cambió desde que lo cargaste. ¿Quieres recargar?"_
   - Al confirmar: refetch del recurso y refresh del formulario.
   - No reintentar automáticamente — la decisión es del usuario.

**Patrón recomendado:** un helper en `core/http/etag.utils.ts` que persiste el ETag en el store del recurso junto al modelo, y un interceptor específico (`ifMatchInterceptor`) que lo adjunta automáticamente en mutaciones si el store lo tiene.

```typescript
// core/http/if-match.interceptor.ts (esbozo)
export const ifMatchInterceptor: HttpInterceptorFn = (req, next) => {
  if (!['PUT', 'PATCH', 'DELETE'].includes(req.method)) return next(req);
  const etag = inject(EtagRegistry).get(req.url);
  if (!etag) return next(req);
  return next(req.clone({ setHeaders: { 'If-Match': etag } }));
};
```

### 21.5 Idempotencia: `Idempotency-Key`

**POSTs críticos** (crear pozo, radicar forma, firmar, aprobar) deben enviar un header `Idempotency-Key` con un UUID v4 generado por el FE. El backend garantiza que si la misma clave se recibe dentro de la ventana de tiempo, devuelve la misma respuesta sin re-ejecutar la operación.

**Implementación recomendada:** un helper por comando NgRx o un decorador de método en el servicio:

```typescript
// shared/utils/idempotency.ts
export function generateIdempotencyKey(): string {
  return crypto.randomUUID();
}

// domains/wells/services/wells-api.service.ts
createWell(payload: CreateWellDTO): Observable<Well> {
  return this.http.post<WellDetailDto>(API.wells.base, payload, {
    headers: { 'Idempotency-Key': generateIdempotencyKey() },
    observe: 'response',
  }).pipe(
    map(resp => ({ ...mapWellDTOToModel(resp.body!), etag: resp.headers.get('ETag') ?? '' }))
  );
}
```

**Regla:** la clave se genera **una vez** cuando el usuario confirma la acción; si el FE reintenta tras un timeout/red caída, debe reusar la misma clave (no generar una nueva). Esto garantiza que el reintento no produce duplicados.

### 21.6 Paginación: header `X-Pagination`

El backend (CONSTITUTION-ba §9.2) devuelve la lista como `T[]` en el body y los metadatos de paginación en el header `X-Pagination` (JSON):

```
X-Pagination: {"page":1,"pageSize":20,"totalItems":137,"totalPages":7,"hasNext":true}
```

**Patrón del servicio del FE:**

```typescript
// shared/models/paged-result.model.ts
export interface PagedResult<T> {
  items: T[];
  page: number;
  pageSize: number;
  totalItems: number;
  totalPages: number;
  hasNext: boolean;
}

// Helper genérico en core/http/paged-response.helper.ts
export function toPagedResult<TDto, TModel>(
  resp: HttpResponse<TDto[]>,
  mapper: (dto: TDto) => TModel,
): PagedResult<TModel> {
  const header = resp.headers.get('X-Pagination');
  const meta = header ? JSON.parse(header) : { page: 1, pageSize: resp.body?.length ?? 0 };
  return {
    items: (resp.body ?? []).map(mapper),
    page: meta.page,
    pageSize: meta.pageSize,
    totalItems: meta.totalItems ?? resp.body?.length ?? 0,
    totalPages: meta.totalPages ?? 1,
    hasNext: meta.hasNext ?? false,
  };
}

// Uso:
getWells(query: WellsQuery): Observable<PagedResult<Well>> {
  return this.http.get<WellDTO[]>(API.wells.base, {
    params: toHttpParams(query),
    observe: 'response',
  }).pipe(map(resp => toPagedResult(resp, mapWellDTOToModel)));
}
```

### 21.7 Filtros, ordenamiento y query params

Convenciones alineadas con CONSTITUTION-ba §9.3–§9.4:

| Query param | Formato | Ejemplo |
|---|---|---|
| Paginación | `?page=1&pageSize=20` | — |
| Ordenamiento | `?sort=field` asc · `?sort=-field` desc · múltiples: `?sort=-createdAt,name` | `?sort=-spudDate,name` |
| Filtro de igualdad | `?{field}={value}` | `?statusId=ACTIVE` |
| Filtro de rango | `?{field}Gte=...&{field}Lte=...` | `?depthMetersGte=1000` |
| Filtro de texto libre | `?q={term}` | `?q=campo%20rubiales` |
| Filtro multivalor | `?{field}=a&{field}=b` (repetido) | `?operatorId=10&operatorId=20` |

**Regla:** los servicios nunca construyen query strings a mano con `?` — usan `HttpParams` + helper `toHttpParams(query: object)` que omite nulls, convierte fechas a ISO y serializa arrays según el contrato.

### 21.8 Tabla resumen: qué debe implementar el FE para cumplir el contrato BE v1.8

| Regla | Dónde vive en el FE | Estado |
|---|---|---|
| Sin envelope en éxito | Servicios — tipar `HttpClient.get<DTO>` directo | Ver §6.3 |
| ProblemDetails RFC 7807 | `errorInterceptor` + `isProblemDetails()` | §5 |
| Validación 422 por campo | `catchError` local en el shell + `form.setErrors` | §5 |
| Fechas UTC ↔ TZ usuario | Mapper parsea → pipe de FE renderiza | §21.2 |
| Enums como string | Tipo union literal en el modelo | §6.3 |
| ETag / If-Match | `EtagRegistry` + `ifMatchInterceptor` | §21.4 |
| `412` → reload dialog | Shell del feature maneja el error antes del toast | §21.4 |
| `Idempotency-Key` | Helper por comando crítico | §21.5 |
| Paginación header | `toPagedResult<T>()` en `core/http/` | §21.6 |
| Query params tipados | `toHttpParams()` + `WellsQuery` interface | §21.7 |

---

## §22 — Modelo de datos normalizado e hidratación en cliente

> **Patrón formal:** *Normalized DTO with Client-Side Hydration* — en jerga Angular/NgRx moderna, *Normalized Entity State con Selectors Composables*.

### 22.1 Principio rector

**Todo dato se expone normalizado desde el backend (crudo con IDs, sin labels precomputados). La hidratación ocurre en el frontend mediante:**

1. **Servicios de catálogo** cargados en sesión (datos pequeños y estables: tipos de pozo, estados, operadoras, unidades, roles).
2. **Servicios de entidades transversales** con cache bajo demanda (datos medianos/grandes y dinámicos: contratos, campos, usuarios).
3. **Selectors composables** (NgRx selectors / `computed()` de Signals) que componen entidad + catálogo + referencias.

**Solo se desvía de esta regla por excepciones explícitamente documentadas** (ver §22.7).

**Por qué este patrón:**
- **Consistencia con el backend:** un repositorio por entidad en BE (§2.3 de CONSTITUTION-ba) se traduce a un servicio por entidad en FE — Clean Architecture aplicada a ambas capas.
- **Evolución barata del schema:** si ANH agrega `colorHex` a un catálogo de estados, basta actualizar el catálogo — todos los componentes lo ven.
- **Cache-friendly:** catálogos cacheados en sesión reducen llamadas al backend y mejoran tiempo de respuesta.
- **Angular 21 con Signals + `computed()` hace la hidratación trivial** — 5 líneas para un pozo hidratado con estado + operador + contrato, invalidado automáticamente al cambiar cualquier dependencia.
- **Reducción del zoológico de DTOs:** una sola regla ("todo normalizado salvo excepciones documentadas") es más barata intelectualmente que DTOs híbridos con labels precomputados mezclados con referencias compactas.

### 22.2 Tres mecanismos de hidratación

| Mecanismo | Qué hidrata | Cuándo se carga | Dónde se guarda | Invalidación |
|---|---|---|---|---|
| **Catálogos** | Listas pequeñas, estables, globales (≤ ~500 items): estados, tipos de pozo, unidades, roles, municipios | Al iniciar sesión (`APP_INITIALIZER`) + lazy por feature si son específicos del dominio | NgRx Store (`catalogs` feature state) | ETag + polling ligero cada N minutos (§22.5) |
| **Entidades transversales** | Listas medianas/grandes, dinámicas: operadoras, contratos, campos, usuarios | Bajo demanda: `getById(id)` con cache LRU | NgRx Store o signal cache por servicio (`Map<id, entity>`) | TTL + invalidación explícita tras mutaciones |
| **Selectors composables** | Vista hidratada de una entidad (pozo + estado + operador + contrato) | Tiempo real al renderizar (reactivo) | No se guardan — son `computed()` / memoized selectors | Automática cuando cambia cualquier fuente |

### 22.3 Servicios de catálogo (bootstrap-loaded)

**Dónde viven:**
- Catálogos globales (usados por 2+ dominios) → `shared/services/catalog.service.ts` + Store en `shared/store/catalogs/`.
- Catálogos específicos del dominio (solo usados por un dominio) → `domains/[dominio]/services/{dominio}-catalogs.service.ts` + Store en `domains/[dominio]/store/catalogs/`.

**Patrón del store de catálogos (NgRx):**

```typescript
// shared/store/catalogs/catalogs.state.ts
export interface CatalogsState {
  wellStatuses:  EntityState<WellStatus>;    // normalizado: diccionario por ID
  operators:     EntityState<Operator>;
  measureUnits:  EntityState<MeasureUnit>;
  loaded: {
    wellStatuses: boolean;
    operators:    boolean;
    measureUnits: boolean;
  };
  etags: Record<string, string>;             // ETag por catálogo para polling
}

// shared/store/catalogs/catalogs.selectors.ts
export const selectWellStatusById = (id: WellStatusId) =>
  createSelector(selectWellStatusesEntities, entities => entities[id]);
```

**Bootstrap con `APP_INITIALIZER`:**

```typescript
// app.config.ts
export const appConfig: ApplicationConfig = {
  providers: [
    provideStore(...),
    {
      provide: APP_INITIALIZER,
      multi: true,
      useFactory: (store: Store) => () => {
        // Bloquea el arranque hasta que los catálogos globales mínimos estén listos.
        store.dispatch(CatalogActions.loadCoreCatalogs());
        return firstValueFrom(store.select(selectCoreCatalogsLoaded).pipe(filter(Boolean), take(1)));
      },
      deps: [Store],
    },
  ],
};
```

**División bootstrap mínimo vs. lazy:**
- **Bootstrap mínimo:** solo catálogos realmente globales y pequeños (usuarios roles, estados genéricos, unidades). Objetivo: < 300 ms agregados.
- **Lazy por feature module:** catálogos específicos (ej. catálogo de tipos de forma 100) se cargan al entrar al módulo via `canActivate` guard que despacha la carga.

**Regla:** la carga bootstrap **nunca** debe sumar más de 1 segundo. Si se acerca, dividir en lazy.

### 22.4 Servicios de entidades transversales (cache on-miss)

**Diferencia con catálogos:** entidades transversales **no se precargan** — son demasiado grandes o demasiado dinámicas. Se hidratan bajo demanda y se cachean con TTL.

**Patrón:**

```typescript
// shared/services/operator-cache.service.ts
@Injectable({ providedIn: 'root' })
export class OperatorCacheService {
  private readonly api   = inject(OperatorsApiService);
  private readonly cache = new Map<number, { value: Operator; expiresAt: number }>();
  private readonly inflight = new Map<number, Observable<Operator>>();
  private readonly TTL_MS = 5 * 60 * 1000; // 5 min

  getById(id: number): Observable<Operator> {
    const cached = this.cache.get(id);
    if (cached && cached.expiresAt > Date.now()) return of(cached.value);

    // Dedupe requests concurrentes por el mismo ID
    const inflight = this.inflight.get(id);
    if (inflight) return inflight;

    const req$ = this.api.getById(id).pipe(
      tap(op => this.cache.set(id, { value: op, expiresAt: Date.now() + this.TTL_MS })),
      finalize(() => this.inflight.delete(id)),
      shareReplay(1),
    );
    this.inflight.set(id, req$);
    return req$;
  }

  /** Invalidación explícita tras un update/delete del operador. */
  invalidate(id: number): void { this.cache.delete(id); }
}
```

**Regla de invalidación:** tras cualquier mutación exitosa de la entidad, el **effect** correspondiente invoca `cacheService.invalidate(id)` o directamente `cacheService.set(id, updatedEntity)` si el response trae la entidad completa.

### 22.5 Staleness de catálogos: ETag + polling ligero

Un catálogo cargado en bootstrap puede quedar stale si un admin agrega un tipo nuevo durante la sesión del usuario. Mitigación:

1. El backend expone `GET /api/v1/catalogs/{name}` que devuelve `ETag` derivado del `Version` o `LastModified` del catálogo.
2. Un `CatalogFreshnessService` hace `HEAD /api/v1/catalogs/{name}` cada N minutos (5 min por defecto, configurable por catálogo).
3. Si el `ETag` cambió → despacha `CatalogActions.reload({ name })`.
4. Alternativa para catálogos de alta criticidad: WebSocket/SignalR que emite `catalog.{name}.updated` y dispara el reload inmediato.

**Regla:** el polling de staleness **nunca** corre en paralelo con el usuario haciendo submit — debe pausarse mientras hay mutaciones en vuelo para evitar condiciones de carrera visuales.

### 22.6 Selectors composables (hidratación reactiva)

Esta es la capa donde el `statusId` crudo se convierte en un label visible. Hay dos opciones equivalentes — elegir una por feature según el tamaño del estado:

**Opción A — NgRx selectors memoizados (cuando la entidad vive en Store):**

```typescript
// domains/wells/store/wells.selectors.ts
export const selectWellView = (id: string) => createSelector(
  selectWellById(id),
  selectWellStatusesEntities,      // del store de catálogos global
  selectOperatorsEntities,         // idem
  (well, statuses, operators): WellView | undefined => {
    if (!well) return undefined;
    return {
      ...well,
      statusLabel:    statuses[well.statusId]?.label ?? '—',
      statusColor:    statuses[well.statusId]?.colorHex ?? '#999',
      operatorName:   operators[well.operatorId]?.legalName ?? '—',
    };
  },
);
```

**Opción B — Signals + `computed()` (cuando la entidad vive en el componente/feature):**

```typescript
// well-detail.component.ts
@Component({ ... })
export class WellDetailComponent {
  private readonly store = inject(Store);

  // Fuentes reactivas
  well            = toSignal(this.route.data.pipe(map(d => d['well'] as Well)));
  statusesEntities = this.store.selectSignal(selectWellStatusesEntities);
  operatorsEntities = this.store.selectSignal(selectOperatorsEntities);

  // Vista hidratada — se recomputa solo cuando cambia alguna fuente
  view = computed<WellView | undefined>(() => {
    const w = this.well();
    if (!w) return undefined;
    const status = this.statusesEntities()[w.statusId];
    const op = this.operatorsEntities()[w.operatorId];
    return {
      ...w,
      statusLabel:  status?.label ?? '—',
      statusColor:  status?.colorHex ?? '#999',
      operatorName: op?.legalName ?? '—',
    };
  });
}
```

**Regla:** la hidratación **siempre** es pura y reactiva — nunca side-effects, nunca mutación del modelo. Si un catálogo se actualiza, las vistas se recomputan solas.

### 22.7 Excepción documentada: hidratación server-side

Para **listados grandes con render exigente** (ej. mapa con 5 000 pozos, grilla de producción de 10 000 filas) resolver IDs en el cliente puede lagear aun con `trackBy` + signals. En esos casos el backend puede emitir un **DTO denormalizado específico** con labels ya precomputados.

**Requisitos para la excepción:**
1. Registrar un **ADR** que justifique la denormalización con métricas (render time > 500 ms con el patrón normalizado).
2. El endpoint denormalizado se nombra con sufijo `/view` o `/report` (`GET /api/v1/wells/map-view`, `GET /api/v1/production/daily-report`) — nunca reemplaza el endpoint normalizado.
3. El DTO denormalizado usa un sufijo distintivo en el FE (`WellMapViewDto`, `ProductionReportRowDto`) para que sea evidente que no es el DTO canónico.
4. No se debe usar el DTO denormalizado para mutaciones — solo lectura.

**Antipatrón:** emitir labels precomputados en el endpoint canónico "porque es más fácil en el FE". Eso rompe la evolución barata del schema y duplica la fuente de verdad de los catálogos.

### 22.8 Riesgos del patrón y sus mitigaciones

| Riesgo | Escenario | Mitigación |
|---|---|---|
| **Render lag en listados grandes** | Mapa de 5 000 pozos resolviendo IDs en cliente | `trackBy` + Signals suele bastar; si no, excepción documentada (§22.7) |
| **Entidades transversales grandes** | 500+ contratos imposibles de precargar | No son catálogos — van en cache on-miss con TTL (§22.4) |
| **Primera carga pesada** | Bootstrap con 30 catálogos | División bootstrap mínimo + lazy por feature module (§22.3) |
| **Catálogos stale** | Usuario con app abierta 8h no ve tipos nuevos | ETag + polling ligero; WebSocket para catálogos críticos (§22.5) |
| **Acoplamiento temporal en tests E2E** | Test verifica label antes de que el catálogo cargue | `APP_INITIALIZER` bloquea hasta catálogos listos; tests cargan fixture en setup |

### 22.9 Reglas vinculantes

1. **MUST** — todos los endpoints canónicos (non-`/view`) devuelven datos crudos con IDs.
2. **MUST** — los catálogos globales se registran en `shared/store/catalogs/` y se cargan via `APP_INITIALIZER`.
3. **MUST** — la hidratación ocurre en selectors NgRx o `computed()` de Signals, nunca en el mapper ni en el template.
4. **MUST** — los servicios de entidades transversales implementan dedupe de requests concurrentes e invalidación explícita tras mutaciones.
5. **MUST** — toda denormalización server-side tiene ADR y endpoint con sufijo `/view` o `/report`.
6. **PROHIBIDO** — precargar entidades transversales grandes (operadoras, contratos) en bootstrap.
7. **PROHIBIDO** — componentes que leen `statusId` y lo convierten a label con un `switch` en el template — eso es responsabilidad del selector/computed.
8. **PROHIBIDO** — mutación del modelo hidratado. La vista hidratada (`WellView`) es read-only.

---

## §23 — Autorización en UI: permisos del JWT

> **Principio de defensa en profundidad:** los permisos cumplen **doble función**. En el backend son defensa de seguridad real (CONSTITUTION-ba §7, §12); en el frontend son **render hints** para UX — ocultar botones, menús y rutas que el usuario no puede ejecutar. Nunca se confía solo en el frontend: todo lo que el FE oculta, el BE también valida.

### 23.1 Fuente de los permisos

El JWT de aplicación emitido por `gop.identity` (ver §24) incluye un claim `permissions` (o `scope` string, según convención final) con las capabilities del usuario ya evaluadas para la sesión:

```json
{
  "sub": "u-12345",
  "name": "Daniel Ayala",
  "tenantId": "operator-42",
  "roles": ["OperatorEngineer"],
  "permissions": [
    "Wells.CanView",
    "Wells.CanCreate",
    "Wells.CanEdit",
    "Forms.F101.CanFile",
    "Forms.F101.CanSignEngineer"
  ],
  "exp": 1745270400
}
```

Lista de capabilities vigente: CONSTITUTION-ba §19.1. El FE **no** computa permisos — los lee del token.

### 23.2 `AuthService` como única fuente de verdad

```typescript
// core/auth/auth.service.ts
@Injectable({ providedIn: 'root' })
export class AuthService {
  private readonly _permissions = signal<ReadonlySet<string>>(new Set());

  hydrateFromToken(token: string): void {
    const claims = decodeJwt(token); // sin verificar firma — eso es responsabilidad del BE
    this._permissions.set(new Set(claims.permissions ?? []));
    // ...resto (user, roles, tenantId, exp)
  }

  hasPermission(permission: string | undefined): boolean {
    if (!permission) return true;
    return this._permissions.has(permission);
  }

  hasAnyPermission(permissions: readonly string[]): boolean {
    return permissions.some(p => this._permissions.has(p));
  }

  hasAllPermissions(permissions: readonly string[]): boolean {
    return permissions.every(p => this._permissions.has(p));
  }

  /** Signal reactivo para usar en computed(). */
  readonly permissions = this._permissions.asReadonly();
}
```

**Regla:** ningún componente decodifica el JWT por su cuenta — solo `AuthService` lo hace, al arranque y tras cada refresh/rotación.

### 23.3 Directiva estructural `*appHasPermission`

Para ocultar elementos del DOM cuando el usuario no tiene el permiso:

```typescript
// shared/directives/has-permission.directive.ts
@Directive({ selector: '[appHasPermission]', standalone: true })
export class HasPermissionDirective {
  private readonly view = inject(ViewContainerRef);
  private readonly tpl  = inject(TemplateRef);
  private readonly auth = inject(AuthService);
  private rendered = false;

  @Input({ required: true }) set appHasPermission(permission: string | string[]) {
    const granted = Array.isArray(permission)
      ? this.auth.hasAnyPermission(permission)
      : this.auth.hasPermission(permission);
    if (granted && !this.rendered) {
      this.view.createEmbeddedView(this.tpl);
      this.rendered = true;
    } else if (!granted && this.rendered) {
      this.view.clear();
      this.rendered = false;
    }
  }
}
```

**Uso:**

```html
<!-- Un solo permiso -->
<button *appHasPermission="'Wells.CanCreate'" (click)="create()">Nuevo pozo</button>

<!-- Cualquiera de varios permisos -->
<button *appHasPermission="['Forms.F101.CanApprove', 'Forms.F101.CanReject']">Decidir</button>
```

### 23.4 Pipe `| hasPermission` (para enlaces, `[disabled]`, etc.)

```typescript
@Pipe({ name: 'hasPermission', standalone: true, pure: false })
export class HasPermissionPipe implements PipeTransform {
  private readonly auth = inject(AuthService);
  transform(permission: string | string[]): boolean {
    return Array.isArray(permission)
      ? this.auth.hasAnyPermission(permission)
      : this.auth.hasPermission(permission);
  }
}
```

**Uso:**
```html
<button [disabled]="!('Wells.CanEdit' | hasPermission)">Editar</button>
<a [routerLink]="'/admin'" *ngIf="'Admin.Users.CanManage' | hasPermission">Admin</a>
```

### 23.5 `permissionGuard` para rutas

```typescript
// core/guards/permission.guard.ts
export const permissionGuard = (required: string | string[]): CanActivateFn => () => {
  const auth = inject(AuthService);
  const router = inject(Router);
  const granted = Array.isArray(required) ? auth.hasAnyPermission(required) : auth.hasPermission(required);
  return granted ? true : router.createUrlTree(['/forbidden']);
};

// uso en rutas
{
  path: 'wells/create',
  canActivate: [authGuard, permissionGuard('Wells.CanCreate')],
  loadComponent: () => import('./well-create.component').then(m => m.WellCreateComponent),
}
```

### 23.6 RBAC en el Sidebar (alineación con §17)

El array `NAV_ITEMS` debe pasar a declarar **permisos requeridos**, no solo roles. El filtrado se hace en el smart shell (§17.2):

```typescript
// core/layout/sidebar/nav-items.ts
export interface NavItem {
  key: string;
  icon?: string;
  route?: string;
  /** Permisos requeridos para ver el item. Al menos uno debe coincidir. */
  requiredPermissions: readonly string[];
  children?: NavItem[];
}

export const NAV_ITEMS: NavItem[] = [
  { key: 'wells',      route: '/wells',      requiredPermissions: ['Wells.CanView'] },
  { key: 'operations', route: '/operations', requiredPermissions: ['Forms.F101.CanFile', 'Forms.F101.CanReview'] },
  { key: 'admin',      route: '/admin',      requiredPermissions: ['Admin.Users.CanManage', 'Admin.Roles.CanManage'] },
];
```

> **Transición:** el documento previo usaba `roles: UserRole[]` en `NavItem`. Para features ya implementadas con roles, se mantiene compatibilidad hasta migrar. Las features nuevas **deben** usar `requiredPermissions`. El `blueprint.md` trackea el estado de migración.

### 23.7 Defensa en profundidad — capas que deben estar sincronizadas

| Capa | Responsabilidad | Fuente |
|---|---|---|
| 1. Sidebar / menús / botones | Ocultar si no hay permiso (UX) | `*appHasPermission`, `hasPermission` pipe |
| 2. Route guards | Bloquear navegación por URL directa | `permissionGuard` |
| 3. Servicios del FE | Nada adicional — el BE valida | — |
| 4. Backend (CONSTITUTION-ba §7) | Autorización real, no bypass posible | Policies + decoradores CQRS |

**Regla:** las capas 1 y 2 del FE son **UX**, no seguridad. Cualquier usuario técnico puede evadir capas 1–3. La única defensa real es la capa 4.

### 23.8 Reglas vinculantes

1. **MUST** — `AuthService` es la única fuente de `permissions` en el FE; se hidrata desde el JWT y se reemplaza completamente tras refresh/rotación.
2. **MUST** — toda ruta que expone una acción con capability asociada tiene `permissionGuard` + `authGuard`.
3. **MUST** — todo botón/link que dispara una mutación visible está protegido con `*appHasPermission` o `| hasPermission`.
4. **SHOULD** — las features nuevas declaran `requiredPermissions` en `NavItem`, no `roles`.
5. **PROHIBIDO** — decodificar el JWT fuera de `AuthService`.
6. **PROHIBIDO** — confiar en el FE como única capa de autorización — el BE siempre valida (ver CONSTITUTION-ba §7).

---

## §24 — Flujo de autenticación: federado + local

> **Fuente normativa:** CONSTITUTION-ba §7.1.1 (Token Exchange RFC 8693), §7.1.1.1 (usuarios locales), §19.8 (diagramas de secuencia).

### 24.1 Dos modos de autenticación

`gop.identity` emite **siempre** el mismo JWT de aplicación, independientemente del origen. Lo que cambia es el flujo de obtención:

| Modo | Flujo | Usuarios | UI |
|---|---|---|---|
| **Federado (OIDC)** | Authorization Code + PKCE → Token Exchange | Usuarios ANH + operadores con cuenta corporativa | Botón "Entrar con ANH" → redirect al IdP → callback |
| **Local (Identity)** | Credenciales → JWT directo | Testing pre-federación, operadores/contratistas sin IdP, cuentas técnicas | Formulario email + password |

El flag `environment.auth.allowLocalAuthentication` habilita el formulario local. En PROD puede desactivarse por ADR cuando el IdP de ANH cubra al 100% de los usuarios.

### 24.2 Flujo federado (OIDC + PKCE + Token Exchange)

```
[Usuario] → click "Entrar con ANH"
[FE]     → genera PKCE (code_verifier, code_challenge), guarda en sessionStorage
[FE]     → redirect a /gop.identity/auth/federated/start
[gop.identity] → redirect al IdP externo con code_challenge
[IdP]    → autentica (MFA, etc.)
[IdP]    → redirect a /gop.identity/auth/federated/callback con code
[gop.identity] → intercambia code + code_verifier por id_token con el IdP
[gop.identity] → valida id_token (issuer, audience, firma, exp)
[gop.identity] → Token Exchange RFC 8693 → emite JWT de aplicación
[gop.identity] → redirect al FE: /auth/callback?accessToken=...&refreshToken=...
[FE]     → hydrateFromToken(accessToken) → store refresh token (HttpOnly cookie preferible)
[FE]     → navigate al dashboard o al returnUrl guardado
```

**Rutas del FE involucradas:**

| Ruta | Responsabilidad | Guard |
|---|---|---|
| `/login` | Landing con los dos botones (federado + local) | `noAuthGuard` |
| `/auth/callback` | Recibe tokens del backend, hidrata `AuthService`, navega | — |
| `/auth/error` | Muestra error de autenticación (IdP down, tenant no habilitado) | — |

### 24.3 Flujo local (credenciales)

```
[Usuario] → formulario email + password
[FE]     → POST /gop.identity/auth/local con { email, password }
[gop.identity] → valida con ASP.NET Core Identity (lockout, password history, MFA)
[gop.identity] → emite JWT de aplicación (mismo shape que el federado)
[FE]     → recibe { accessToken, refreshToken, user }
[FE]     → hydrateFromToken, navigate
```

### 24.4 Estructura del JWT de aplicación (alineada con BE §7.2)

```typescript
// core/auth/models/app-jwt.model.ts
export interface AppJwtClaims {
  sub: string;                     // userId
  name: string;                    // displayName
  email: string;
  tenantId: string | null;         // null para usuarios ANH
  authenticationSource: 'External' | 'Local';  // informativo para UI
  roles: string[];
  permissions: string[];           // capabilities ya evaluadas (§23.1)
  iat: number;
  exp: number;
  jti: string;                     // para revocación (CONSTITUTION-ba §7.12)
}
```

**Uso del campo `authenticationSource` en UI:**
- Si es `External` → ocultar "Cambiar contraseña" (la password vive en el IdP).
- Si es `Local` → permitir cambio de contraseña, mostrar política.

### 24.5 Refresh y rotación del token

- El **access token** tiene vida corta (~15 min). El FE lo guarda en memoria (`AuthService`), **nunca** en `localStorage`.
- El **refresh token** vive en cookie `HttpOnly` emitida por `gop.identity` (preferible) o en `sessionStorage` (fallback documentado por ADR).
- Antes de que expire el access token, un `tokenRefreshInterceptor` o un timer en `AuthService` dispara `POST /gop.identity/auth/refresh` y rehidrata.
- Si el refresh falla (reuse detection del BE, §7.1.1) → `sessionExpired` → logout forzado.

### 24.6 Logout

El logout es **siempre** vía NgRx Effect (ver §4.1). Pasos:

1. `POST /gop.identity/auth/logout` (revoca refresh token y access token actual en la revocation list).
2. Si fue federado: redirect al `end_session_endpoint` del IdP (best effort).
3. `authService.clearSession()`.
4. `router.navigate(['/login'])`.

### 24.7 Reglas vinculantes

1. **MUST** — access token en memoria (`AuthService`), nunca `localStorage`.
2. **MUST** — refresh token en cookie `HttpOnly` (preferible) o `sessionStorage` con ADR justificatorio.
3. **MUST** — `AuthService.hydrateFromToken` es el único punto que parsea el JWT.
4. **MUST** — el FE nunca propaga el `id_token` del IdP externo — solo el JWT de aplicación emitido por `gop.identity` (CONSTITUTION-ba §7.1.1).
5. **SHOULD** — ocultar "Cambiar contraseña" cuando `authenticationSource === 'External'`.
6. **PROHIBIDO** — `localStorage.setItem('token', ...)` en cualquier capa.
7. **PROHIBIDO** — mostrar claims del JWT en logs/console.

---

## §25 — Uploads, downloads y exportes

> **Fuente normativa:** CONSTITUTION-ba §9.9.5 (uploads/downloads), §9.9.6 (exportes).

### 25.1 Uploads (archivos adjuntos a formas, evidencias)

**Contrato:**
- `multipart/form-data` — nunca base64 en JSON (salvo casos < 100 KB justificados por ADR).
- Tamaño máximo por archivo: **50 MB** por defecto (configurable por servicio).
- El backend valida MIME real, extensión y corre antivirus — el FE hace validación **pre-flight** para UX.

**Validación pre-flight en el FE:**

```typescript
// shared/services/file-upload.service.ts
export const ALLOWED_MIME = new Set(['application/pdf', 'image/png', 'image/jpeg',
                                     'application/vnd.openxmlformats-officedocument.spreadsheetml.sheet']);
export const MAX_SIZE_BYTES = 50 * 1024 * 1024;

export function validateFile(file: File): ValidationError | null {
  if (file.size > MAX_SIZE_BYTES) return { code: 'FILE_TOO_LARGE', message: `Máximo ${MAX_SIZE_BYTES / 1e6} MB.` };
  if (!ALLOWED_MIME.has(file.type)) return { code: 'INVALID_MIME', message: `Tipo no permitido: ${file.type}.` };
  return null;
}
```

**Envío al backend:**

```typescript
uploadAttachment(formId: string, file: File): Observable<HttpEvent<AttachmentDto>> {
  const fd = new FormData();
  fd.append('file', file, file.name);
  fd.append('formId', formId);
  return this.http.post<AttachmentDto>(API.attachments.base, fd, {
    reportProgress: true,
    observe: 'events',
  });
}
```

**Regla:** el servicio expone eventos (`HttpEvent`) para que el componente pinte la barra de progreso. Nunca `map(() => data)` antes de emitir progreso.

### 25.2 Downloads

**Dos modos según el tamaño:**

| Tamaño | Modo | Implementación |
|---|---|---|
| < 10 MB | Descarga directa vía API | `http.get(url, { responseType: 'blob' })` + `saveAs()` |
| ≥ 10 MB | **Presigned URL** | API retorna `{ url, expiresAt }` — el FE hace `window.location.href = url` o `<a download>` |

**Patrón presigned URL:**

```typescript
// domains/wells/services/wells-api.service.ts
downloadEvidenceLink(wellId: string, evidenceId: string): Observable<PresignedUrl> {
  return this.http.get<PresignedUrl>(API.wells.evidenceDownloadUrl(wellId, evidenceId));
}

// componente
onDownload(evidenceId: string): void {
  this.api.downloadEvidenceLink(this.wellId, evidenceId).subscribe(({ url }) => {
    const a = document.createElement('a');
    a.href = url;
    a.rel = 'noopener';
    a.click();
  });
}
```

**Regla:** nunca embeber el archivo completo en el DOM como `data:` URL — siempre `href` al blob/presigned.

### 25.3 Exportes asíncronos (CSV, Excel, PDF > 10 000 filas)

Endpoints de export con sufijo `/export` pueden devolver:
- `200 + stream`: sincrónico, < 10 000 filas.
- `202 + { operationId, statusUrl }`: asíncrono.

**Patrón async (CONSTITUTION-ba §9.9.6):**

```
[FE]  POST /api/v1/wells/export?format=xlsx  (con filtros)
[BE]  202 { operationId: 'op-abc', statusUrl: '/api/v1/operations/op-abc' }
[FE]  muestra toast persistente "Generando export..."
[FE]  polling GET /api/v1/operations/op-abc cada 3s
[BE]  200 { status: 'InProgress' | 'Completed' | 'Failed', downloadUrl?, error? }
[FE]  cuando status === 'Completed' → disparar download con downloadUrl (presigned)
[FE]  toast "Export listo — Descargar" con acción
```

**Servicio genérico:**

```typescript
// shared/services/async-operation.service.ts
@Injectable({ providedIn: 'root' })
export class AsyncOperationService {
  poll<T>(statusUrl: string, intervalMs = 3000, timeoutMs = 5 * 60 * 1000): Observable<AsyncOperationResult<T>> {
    return timer(0, intervalMs).pipe(
      switchMap(() => this.http.get<AsyncOperationResult<T>>(statusUrl)),
      takeWhile(res => res.status === 'InProgress', true),
      timeout(timeoutMs),
    );
  }
}
```

### 25.4 Reglas vinculantes

1. **MUST** — uploads vía `multipart/form-data`; validación pre-flight de size + MIME.
2. **MUST** — downloads grandes usan presigned URL; no bajar bytes a través del API.
3. **MUST** — exportes > 10 000 filas son asíncronos con `operationId` + polling.
4. **SHOULD** — toast persistente durante operaciones async, no bloqueante.
5. **PROHIBIDO** — base64 de binarios en JSON salvo excepción documentada por ADR (< 100 KB).
6. **PROHIBIDO** — generar PDF/Excel en el cliente para datos de negocio (solo snapshots visuales) — la fuente de verdad del reporte oficial es el backend.

---

## Control de cambios v1.0 → v1.1

**Alineación con CONSTITUTION-ba v1.8 + modelo de datos normalizado + autorización UI:**

- **Header** — añadido versionado (v1.1, fecha, estado, alineación con BE v1.8).
- **§5 Manejo de errores HTTP** — reescrito el `errorInterceptor` para parsear **ProblemDetails RFC 7807** (`title`, `detail`, `errors{}`, `code`, `traceId`). Añadidos casos `409`, `412`, `422`, `429`. Añadido patrón de consumo de errores 422 por campo en formularios.
- **§6.3 Estructura de modelos** — actualizado el ejemplo `WellDTO` a **camelCase** (alineado con BE §9.9.3). Reescrito el rol del mapper: parseo de fechas, tipado de enums, shaping, **sin hidratación** (los IDs se mantienen crudos — la hidratación ocurre en selectors/computed, §22).
- **§21 — nueva sección "Contrato de consumo del backend GOP"** con 8 subsecciones:
  - §21.1 Sin envelope (tabla verb/status/body/header por escenario).
  - §21.2 Fechas ISO 8601 UTC + TZ `America/Bogota` en el FE.
  - §21.3 ProblemDetails RFC 7807 (formato canónico + extensión `code` de GOP).
  - §21.4 ETag + If-Match para concurrencia optimista + manejo de 412.
  - §21.5 `Idempotency-Key` en POSTs críticos (crear/radicar/firmar/aprobar).
  - §21.6 Paginación via header `X-Pagination` + helper `toPagedResult<T>`.
  - §21.7 Query params tipados (sort, filtros, rangos, multivalor).
  - §21.8 Tabla resumen de cumplimiento.
- **§22 — nueva sección "Modelo de datos normalizado e hidratación en cliente"** (patrón *Normalized DTO with Client-Side Hydration*):
  - §22.1 Principio rector: backend emite crudo con IDs; FE hidrata.
  - §22.2 Tres mecanismos: catálogos, entidades transversales, selectors composables.
  - §22.3 Catálogos con `APP_INITIALIZER` + división bootstrap mínimo / lazy por feature.
  - §22.4 Entidades transversales con cache on-miss (TTL + dedupe + invalidación).
  - §22.5 Staleness: ETag + polling ligero + WebSocket para catálogos críticos.
  - §22.6 Selectors composables con NgRx / `computed()` de Signals.
  - §22.7 Excepción documentada: endpoints `/view` denormalizados con ADR.
  - §22.8 Riesgos + mitigaciones (5 riesgos del patrón).
  - §22.9 Reglas vinculantes (MUST / PROHIBIDO).
- **§23 — nueva sección "Autorización en UI: permisos del JWT"**:
  - §23.1 Claim `permissions[]` en el JWT (capabilities ya evaluadas).
  - §23.2 `AuthService` como única fuente de verdad.
  - §23.3 Directiva `*appHasPermission`.
  - §23.4 Pipe `| hasPermission`.
  - §23.5 `permissionGuard` para rutas.
  - §23.6 Sidebar RBAC migra de `roles` a `requiredPermissions` en `NavItem`.
  - §23.7 Defensa en profundidad: UX (FE) ≠ seguridad (BE).
  - §23.8 Reglas vinculantes.
- **§24 — nueva sección "Flujo de autenticación: federado + local"**:
  - §24.1 Dos modos (OIDC + Token Exchange / Identity local).
  - §24.2 Flujo federado OIDC + PKCE + Token Exchange.
  - §24.3 Flujo local (credenciales).
  - §24.4 Estructura del JWT de aplicación.
  - §24.5 Refresh y rotación.
  - §24.6 Logout vía Effect.
  - §24.7 Reglas vinculantes (access token en memoria, refresh en HttpOnly cookie, nunca `localStorage`).
- **§25 — nueva sección "Uploads, downloads y exportes"**:
  - §25.1 Uploads multipart + validación pre-flight.
  - §25.2 Downloads: directo (< 10 MB) o presigned URL (≥ 10 MB).
  - §25.3 Exportes asíncronos con `operationId` + polling.
  - §25.4 Reglas vinculantes.

---

*Fin del CONSTITUTION-fe.md v1.1*
