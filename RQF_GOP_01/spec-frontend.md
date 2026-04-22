# Especificación Frontend — RQF_GOP_01: Creación de Pozo Nuevo

**Versión:** 1.0
**Fecha:** Abril 2026
**Feature:** RQF_GOP_01_CreacionPozoNuevo
**Estado:** Propuesta — pendiente aprobación del arquitecto humano

---

## 1. Contexto y Objetivo

Formulario de registro de pozo nuevo (18 campos, 3 secciones + panel UWI) que permite a las operadoras crear la identidad de un pozo como prerrequisito para la Forma 101 (Intención de Perforar). El nombre del pozo se compone desde campos separados (denominación + consecutivo + sufijos), y el UWI fiscalizado se genera algorítmicamente en tiempo real conforme el usuario llena campos.

**Diferenciadores funcionales:**
- Acción unilateral de la operadora (sin aprobación ANH).
- El nombre del pozo es un campo **compuesto** (no texto libre directo).
- El formulario bifurca según trayectoria: {P, ML, O} → denominación+consecutivo; {ST, PR, G} → pozo padre referenciado.
- Clasificación con sub-niveles (Exploratorio → A3/A2a/A2b/A2c/A1).
- Edición y eliminación habilitadas hasta que la F101 sea radicada.

---

## 2. Actores y Roles

| Rol | Contexto | Acciones UI |
|-----|----------|-------------|
| **Agente** | Operadora | Crear, editar, consultar, eliminar, finalizar registro |
| **Coordinador** | Operadora | Editar (Borrador/Creado sin F101), consultar. No puede crear, eliminar ni finalizar |
| **Admin GOP** | ANH | Todas. Creación restringida a clasificación Estratigráfico (RN-15) |

---

## 3. Historias de Usuario

| ID | Historia |
|----|----------|
| HU-01 | Como **Agente**, quiero registrar un pozo nuevo con sus atributos de identificación, clasificación y ubicación, para habilitarlo como prerrequisito de la Forma 101. |
| HU-02 | Como **Agente**, quiero guardar el progreso parcial como borrador, para completar el registro en otro momento. |
| HU-03 | Como **Agente**, quiero que el sistema genere automáticamente el UWI fiscalizado en tiempo real, para verificar su estructura antes de finalizar. |
| HU-04 | Como **Agente**, quiero que al seleccionar trayectoria ST/PR/G el sistema precargue datos del pozo padre y genere el nombre automáticamente, para evitar errores de nomenclatura. |
| HU-05 | Como **Agente**, quiero editar o eliminar un pozo creado mientras no tenga F101 radicada, para corregir errores detectados. |
| HU-06 | Como **Coordinador**, quiero revisar y editar un pozo en borrador, para validar la calidad de los datos antes de la F101. |
| HU-07 | Como **Admin GOP (ANH)**, quiero crear pozos estratigráficos seleccionando contratos de la VAF, para registrar campañas estratigráficas de la ANH. |

---

## 4. Criterios de Aceptación

### CA-01 — Operadora prellenada
**Given** un Agente logueado
**When** accede a "Crear Pozo Nuevo"
**Then** el campo Operadora muestra su compañía, no es editable. Para Admin ANH se muestra un selector de operadoras.

### CA-02 — Autocompletado de contrato
**Given** se selecciona un contrato
**When** la selección se confirma
**Then** Tipo de Contrato y Cuenca se autocomple­tan y no son editables.

### CA-03 — Nombre auto ST
**Given** Trayectoria = ST y se selecciona pozo original "Ballena-1"
**When** el sistema genera el nombre
**Then** concatena `Ballena-1ST[N]` donde N es consecutivo desde 1 validado contra existentes (RN-04).

### CA-04 — Nombre auto G
**Given** Trayectoria = G y se selecciona pozo original "Ballena-1"
**When** el sistema genera el nombre
**Then** concatena `Ballena-1G[N]` (RN-05).

### CA-05 — Nombre auto PR
**Given** Trayectoria = PR y se selecciona pozo original "Ballena-1"
**When** el sistema genera el nombre
**Then** concatena `Ballena-1PR` (RN-06).

### CA-06 — Nombre con sufijo P/ML/O
**Given** Trayectoria = O y denominación "Caño Sur Este" + consecutivo "325"
**When** el sistema compone el nombre
**Then** el resultado es "Caño Sur Este-325O" con el sufijo "O" al final (RN-07).

### CA-07 — Campos bloqueados para ST/PR/G
**Given** Trayectoria ∈ {ST, PR, G} y se selecciona pozo padre
**When** se cargan los datos
**Then** se bloquean: Nombre (auto), Cuenca, Ubicación, Departamento, Municipio, Código DANE, Cluster. Los demás quedan habilitados (RN-08).

### CA-08 — A3 → Campo Exploratorio
**Given** Clasificación = Exploratorio y Sub-clasificación = A3
**When** se selecciona
**Then** Campo muestra "CAMPO EXPLORATORIO", no editable (RN-11).

### CA-09 — Estratigráfico → Campo N/A
**Given** Clasificación = Estratigráfico
**When** se selecciona
**Then** Campo se fija en "N/A", deshabilitado (RN-14).

### CA-10 — ANH solo Estratigráfico
**Given** usuario ANH (Admin GOP)
**When** intenta crear pozo
**Then** solo puede seleccionar Clasificación = Estratigráfico. Las demás opciones están deshabilitadas (RN-15).

### CA-11 — Pregunta nombre de campo (Desarrollo)
**Given** Clasificación = Desarrollo
**When** se accede al campo de denominación
**Then** se muestra la pregunta "¿El pozo va a llevar el nombre del campo?" (SÍ/NO). Si SÍ, se precarga el nombre del campo como prefijo y el usuario solo ingresa consecutivo. Si NO, se habilita denominación libre (RN-20).

### CA-12 — Validación denominación sin siglas
**Given** el operador ingresa denominación "ST Pozo"
**When** se valida el campo
**Then** se muestra error: "No se permiten siglas reservadas (ST, G, PR, P, ML, Piloto)" (RN-17).

### CA-13 — Validación nombre único
**Given** el nombre compuesto "RUBIALES 157O" ya existe
**When** se intenta finalizar
**Then** el sistema impide la creación con mensaje "Ya existe un pozo con este nombre" (RN-19).

### CA-14 — UWI único
**Given** el UWI generado ya existe
**When** se intenta finalizar
**Then** el sistema impide la creación con mensaje "UWI duplicado" (RN-37).

### CA-15 — UWI estructura correcta
**Given** pozo RUBIALES 323, cluster RUBIALES 323, Meta (50), Puerto Gaitán (568)
**When** el sistema genera la sigla UWI
**Then** produce código con excepción clúster=pozo: sufijo "C" en cluster (RN-32/CA-11 HU).

### CA-16 — Bloqueo post-F101
**Given** un pozo en estado Creado con F101 radicada
**When** se intenta editar o eliminar
**Then** las opciones están deshabilitadas (RN-40).

### CA-17 — Warning OH
**Given** Tipo de Terminación = OH (Hueco abierto)
**When** se selecciona
**Then** se muestra mensaje normativo de Resolución 40537 Art. 21 (RN-23).

### CA-18 — Departamento/Municipio filtrado
**Given** se selecciona un departamento
**When** se carga la lista de municipios
**Then** solo aparecen municipios del departamento seleccionado y el código DANE se autocompleta (RN-26).

### CA-19 — Offshore: Dpto/Mpio opcional
**Given** Tipo Ubicación = Costa Afuera
**When** se renderiza la sección de ubicación
**Then** Departamento y Municipio se marcan como opcionales (PA-02 adoptado).

### CA-20 — Warning taponamiento ST
**Given** Trayectoria = ST y el pozo padre NO tiene F108/109 aprobada
**When** se cargan los datos del padre
**Then** se muestra warning (no bloqueante): "El pozo original no tiene programa de taponamiento aprobado" (RN-09, V1 warning).

### CA-21 — Preview UWI reactivo
**Given** el operador va llenando campos (departamento, municipio, denominación, consecutivo, cluster, ángulo, trayectoria, objetivo, terminación)
**When** se modifica cualquiera de estos campos
**Then** el panel de UWI se actualiza en tiempo real mostrando el UWI calculado y sus componentes desagregados.

### CA-22 — Primer pozo Estratigráfico
**Given** no existe pozo Estratigráfico para el contrato
**When** se crea el primero
**Then** se permite digitar nombre alfabético y el sistema agrega "-1" como consecutivo (RN-21).

### CA-23 — Segundo pozo Estratigráfico en adelante
**Given** ya existe un pozo Estratigráfico en el contrato
**When** se crea el siguiente
**Then** se ofrece opción "Consecutivo" (auto-incrementa) o "Nuevo nombre" (aplica regla de primer pozo, RN-22).

---

## 5. Flujos de UI

### 5.1 Layout general (basado en prototipo confirmado)

```
┌──────────────────────────────────────────────────────────────┐
│                    HEADER (título + botones)                  │
│         [Guardar Borrador]          [Finalizar Registro]     │
├──────────┬──────────────────────────────┬───────────────────┤
│ STEPPER  │        FORMULARIO            │    RESUMEN        │
│ (left)   │     (center, 3 pasos)        │    (right)        │
│          │                              │                   │
│ ● Paso 1 │ ┌──────────────────────────┐ │ Nombre: ---       │
│   Info    │ │ Contenido del paso       │ │ UWI: ---          │
│   Contrato│ │ activo con transiciones  │ │ Estado: Borrador  │
│          │ │ animadas                  │ │                   │
│ ○ Paso 2 │ └──────────────────────────┘ │                   │
│   Datos   │                              │ [Ayuda contextual]│
│   Pozo   │ ┌──────────────────────────┐ │                   │
│          │ │ Nav: [Anterior] [Sgte]   │ │                   │
│ ○ Paso 3 │ └──────────────────────────┘ │                   │
│   Ubic.  │                              │                   │
│   Geog.  │                              │                   │
│          │                              │                   │
│ Progreso │                              │                   │
│ [====  ] │                              │                   │
└──────────┴──────────────────────────────┴───────────────────┘
```

**Grid:** 12 columnas. Stepper = 3 cols. Formulario = 6 cols. Resumen = 3 cols. Stepper y Resumen `sticky top-4`.

### 5.2 Paso 1 — Información del Contrato

| Campo | Tipo | Editable | Comportamiento |
|-------|------|----------|----------------|
| Operadora | Texto auto | No | Prellenado desde sesión. Admin ANH: selector |
| Contrato | Dropdown | Sí | Filtra por operadora. ANH: incluye contratos VAF |
| Tipo de Contrato | Texto auto | No | Autocompleta al seleccionar contrato |
| Cuenca | Texto auto | No | Autocompleta al seleccionar contrato |

### 5.3 Paso 2 — Datos del Pozo

**Bifurcación por trayectoria:**

| Si trayectoria ∈ {P, ML, O} | Si trayectoria ∈ {ST, PR, G} |
|------------------------------|-------------------------------|
| Mostrar: Denominación + Consecutivo | Mostrar: Pozo de Referencia (dropdown) |
| Nombre compuesto visible (disabled) con sufijo | Nombre auto-generado (disabled) |
| Clasificación habilitada | Campos RN-08 bloqueados |

**Sección condicional — Sub-clasificación:**
- Visible solo si Clasificación = Exploratorio.
- Opciones: A3, A2a, A2b, A2c, A1.

**Campo "Campo":**
- Exploratorio A3 → "CAMPO EXPLORATORIO" disabled.
- Exploratorio A2/A1 → Dropdown de campos + "Campo Exploratorio". Si A3 del contrato tiene F103 Lahee B3 → campo auto (RN-12).
- Desarrollo → Dropdown de campos aprobados (sin "Campo Exploratorio"). Pregunta RN-20 antes de nombre.
- Estratigráfico → "N/A" disabled.

**Warning OH:** Banner naranja con texto normativo Resolución 40537 al seleccionar terminación OH.

### 5.4 Paso 3 — Ubicación Geográfica

| Campo | Tipo | Editable | Comportamiento |
|-------|------|----------|----------------|
| Tipo Ubicación | Dropdown (Continental/Offshore) | Sí (auto si ST/PR/G) | Condiciona obligatoriedad Dpto/Mpio |
| Departamento | Dropdown | Sí (blocked si ST/PR/G) | Validación Mapa de Tierras (RN-25) |
| Código DANE Dpto | Auto | No | Autocompleta |
| Municipio | Dropdown filtrado | Sí (blocked si ST/PR/G) | Solo municipios del depto (RN-26) |
| Código DANE Mpio | Auto | No | Autocompleta |
| Cluster-Locación | Dropdown | Sí (blocked si ST/PR/G) | Incluye opción "Crear nuevo" |

**Panel UWI Fiscalizado:**
- Siempre visible al final de este paso.
- Fondo oscuro destacado, tipografía grande.
- Muestra UWI calculado en tiempo real o "DATOS INCOMPLETOS" si faltan campos.
- Badge "Generado Automáticamente" + "Metodología PPDM".

### 5.5 Panel de Resumen (sidebar derecho)

Sticky. Muestra en todo momento:
- **Nombre:** nombre compuesto actual o "---"
- **UWI:** UWI calculado o "---"
- **Estado:** "Borrador" / "Listo para registro"
- **Ayuda contextual:** texto breve del paso actual + enlace a manual.

### 5.6 Acciones del formulario

| Acción | Ubicación | Comportamiento |
|--------|-----------|----------------|
| Guardar Borrador | Header | Guarda sin validar obligatoriedad. API: POST (si nuevo) o PUT |
| Finalizar Registro | Header | Valida todos los campos. API: POST/PUT + POST finalize |
| Anterior | Nav inferior | Retrocede un paso |
| Siguiente | Nav inferior | Avanza (en último paso: "Finalizar Registro") |

### 5.7 Modal de éxito

**Borrador guardado:** modal con icono de guardado, mensaje "La información ha sido guardada en su bandeja de borradores".

**Registro finalizado:** modal con icono de check + confetti, mensaje "El pozo ha sido radicado correctamente". Redirige a `/wells/manage` tras cerrar.

---

## 6. Componentes Funcionales Requeridos

### 6.1 Página de lista de pozos
- Tabla paginada con filtros (estado, contrato, nombre, UWI, clasificación).
- Botón "Crear Pozo Nuevo" (visible para Agente y Admin).
- Acciones por fila: Editar, Eliminar (solo si `hasFiledForm101 = false`), Ver detalle.
- Indicador visual de F101 radicada (badge lock).

### 6.2 Formulario stepper de 3 pasos
- Navegación lateral con indicador de paso activo.
- Transiciones animadas entre pasos.
- Barra de progreso (% completado).
- Campos condicionales según trayectoria y clasificación.

### 6.3 Componente de nombre compuesto
- Combina: prefijo (campo o denominación) + denominación + "-" + consecutivo + sufijo de trayectoria.
- Campo resultante deshabilitado con badge de sufijo.
- Para ST/PR/G: muestra nombre auto-generado desde pozo padre.

### 6.4 Panel de UWI reactivo
- Se recalcula cada vez que cambia un campo relevante.
- Invoca endpoint `preview-uwi` (debounced, ~300ms) o cálculo client-side duplicado.
- Muestra componentes desagregados y estado de unicidad.

### 6.5 Selector de pozo padre
- Dropdown filtrado por contrato actual.
- Muestra nombre + UWI del pozo.
- Al seleccionar, precarga campos bloqueados y verifica F108/109.

### 6.6 Pregunta condicional "¿Nombre del campo?" (Desarrollo)
- Toggle SÍ/NO visible solo cuando clasificación = Desarrollo.
- Si SÍ: precarga nombre del campo como prefijo, usuario solo ingresa consecutivo.
- Si NO: habilita denominación libre.

### 6.7 Selector de nombre Estratigráfico (2do+ pozo)
- Opciones: "Consecutivo" (auto) o "Nuevo nombre" (libre).
- Visible solo si ya existe ≥1 pozo estratigráfico en el contrato.

---

## 7. Reglas de Validación de Formularios (Client-side)

### 7.1 Validación de denominación (RN-16/RN-17)

**Regla:** La denominación debe contener palabras completas, no siglas ni abreviaturas.
- Palabras < 4 caracteres se rechazan **salvo** whitelist: `DE, Y, EL, LA, EN, AL, DEL, LOS, LAS, SAN, SUR, MAR, SOL, RIO, LUZ, PAZ, REY, PAN, CAL, SAL, GAS, OIL`.
- Prohibido contener: `ST`, `G`, `PR`, `P`, `ML`, `PILOTO` como palabra completa o sufijo.
- No se permiten caracteres después de la numeración (RN-18).

### 7.2 Tabla de validaciones al finalizar

| Campo | Regla | Mensaje de error |
|-------|-------|------------------|
| Contrato | Requerido | "Seleccione un contrato" |
| Trayectoria | Requerido | "Seleccione tipo de trayectoria" |
| Pozo Padre | Requerido si tray ∈ {ST, PR, G} | "Seleccione pozo de referencia" |
| Clasificación | Requerido | "Seleccione clasificación" |
| Sub-clasificación | Requerida si Exploratorio | "Seleccione sub-clasificación" |
| Denominación | Requerida si tray ∈ {P, ML, O}; sin siglas | "La denominación no puede contener siglas" |
| Consecutivo | Requerido si tray ∈ {P, ML, O} | "Ingrese el consecutivo" |
| Campo | Requerido salvo Estratigráfico | "Seleccione el campo" |
| Ángulo | Requerido | "Seleccione tipo de ángulo" |
| Objetivo | Requerido | "Seleccione el objetivo" |
| Terminación | Requerido | "Seleccione tipo de terminación" |
| Departamento | Requerido si Continental | "Seleccione departamento" |
| Municipio | Requerido si Continental | "Seleccione municipio" |
| Cluster | Requerido | "Seleccione cluster/locación" |

### 7.3 Validación al guardar borrador
- Sin validación de obligatoriedad.
- Solo formato/tipo si el campo tiene valor.

---

## 8. Estados de la Interfaz

### 8.1 Máquina de estados del formulario

```
[INITIAL] → (carga catálogos) → [LOADING]
  → (éxito) → [READY] → (edita) → [EDITING]
    → (guardar) → [SAVING] → [SAVED]
    → (finalizar) → [VALIDATING]
      → (ok) → [FINALIZING] → [FINALIZED] → redirect
      → (errores) → [VALIDATION_ERRORS]
  → (error catálogos) → [CATALOG_ERROR] → (retry)
```

### 8.2 Estados de la lista

| Estado | UI |
|--------|-----|
| Cargando | Skeleton |
| Datos | Tabla + paginación |
| Vacío | "No hay pozos. Cree el primero." |
| Error | Banner + retry |

---

## 9. NFRs Frontend

### 9.1 Performance

| Métrica | Objetivo |
|---------|----------|
| LCP | < 2.0s (formulario simple) |
| INP | < 150ms |
| TTI | < 2.0s |
| Recálculo UWI | < 300ms percibido |

### 9.2 Accesibilidad (WCAG 2.1 AA)
- Tab navigation entre campos y pasos.
- Contraste 4.5:1. ARIA labels en dropdowns.
- Anuncios en lectores de pantalla para errores y cambios de paso.
- Focus trapping en modal de éxito.

### 9.3 Responsividad
- Desktop (≥1200px): 3 columnas (stepper + form + resumen).
- Tablet (768-1199px): stepper colapsable, resumen debajo.
- Mobile (<768px): stepper horizontal simplificado, resumen colapsable.

### 9.4 Internacionalización
- Idioma: Español (es-CO).
- Fechas: `dd/MM/yyyy` en UI, ISO 8601 en API.

### 9.5 Navegadores
- Chrome, Edge, Firefox, Safari (últimas 2 versiones).

---

## 10. Seguridad Frontend

- Tokens gestionados según CONSTITUTION-fe.md.
- Contenido dinámico renderizado con binding seguro (no `innerHTML`).
- Sesión expirada: auto-guardado de borrador → redirect a login.
- RN-15 (ANH solo Estratigráfico): el dropdown de clasificación muestra solo "Estratigráfico" para rol ANH. El backend revalida.
- No loguear datos sensibles en consola.
- Control de acceso visual: botones de acción según rol y estado.

---

## 11. Edge Cases

| # | Escenario | Comportamiento |
|---|-----------|----------------|
| EC-01 | Usuario no autenticado | Redirect a login |
| EC-02 | Sin rol requerido | Página 403 |
| EC-03 | Sesión expira durante edición | Auto-guardado borrador → redirect login |
| EC-04 | Pérdida de conexión | Mensaje offline, retry al reconectar |
| EC-05 | Backend > 5s | Spinner + cancelación tras 10s |
| EC-06 | Error 5xx | Banner error + retry |
| EC-07 | Nombre duplicado | Error inline en campo nombre |
| EC-08 | UWI duplicado | Error en panel UWI |
| EC-09 | Doble clic en "Finalizar" | Botón disabled al primer clic |
| EC-10 | Dos usuarios editando mismo pozo | ETag → 409 → "Modificado por otro usuario" |
| EC-11 | Pozo padre sin F108/109 (ST) | Warning no bloqueante (V1) |
| EC-12 | Contrato sin cuenca | "Cuenca no disponible" |
| EC-13 | Catálogos no cargan | Estado error, retry, sin formulario |
| EC-14 | F101 radicada → intenta editar | Botones deshabilitados + tooltip explicativo |
| EC-15 | ANH intenta clasificación ≠ Estratigráfico | Dropdown restringido + mensaje |

---

## 12. Criterios de Testing

### 12.1 Cobertura mínima
- **80%** en lógica de negocio (composición de nombre, cálculo UWI, validaciones, condicionales de clasificación/trayectoria).
- **70%** en servicios de API.

### 12.2 E2E obligatorios
1. Crear pozo Original de Desarrollo → verificar nombre + UWI.
2. Crear pozo ST → verificar precarga de padre + nombre auto.
3. Crear pozo Exploratorio A3 → campo = "CAMPO EXPLORATORIO".
4. Crear pozo Estratigráfico → campo = "N/A".
5. ANH intenta crear Desarrollo → bloqueado (RN-15).
6. Duplicado de nombre → error.
7. Duplicado de UWI → error.
8. Guardar borrador parcial → recargar → datos preservados.
9. Finalizar con campos faltantes → errores mostrados.
10. Editar pozo con F101 radicada → bloqueado.
11. Warning OH al seleccionar Open Hole.
12. Offshore → Dpto/Mpio opcionales.

---

## 13. Supuestos Confirmados

| ID | Supuesto | Origen |
|----|----------|--------|
| S-01 | No crea Procedure ni Workflow; el Well se crea directamente | A-07 confirmado |
| S-02 | Nombre es campo compuesto: denominación + consecutivo + sufijo | P-01/pregunta 2 confirmado |
| S-03 | Layout: stepper de 3 pasos (adoptado del prototipo) | D-02/pregunta 3 confirmado |
| S-04 | Ejemplo sigla "ANH MORICHE" → ANHMO (no ANHJUDEAC) | I-01/pregunta 4 confirmado |
| S-05 | RN-09: warning no bloqueante en V1 si no hay F108 | A-01/pregunta 5 confirmado |
| S-06 | Datos de pozo padre = atributos del Well (no payload F101) | A-02 confirmado |
| S-07 | "A3 asociado" = por contrato; múltiples → se listan todos los campos | A-03 confirmado |
| S-08 | Desarrollo: nombre del campo como pregunta interactiva (RN-20) | A-04 confirmado |
| S-09 | "Primer pozo estratigráfico" = por contrato | A-05 confirmado |
| S-10 | Offshore: Dpto/Mpio opcionales | A-06 (PA-02) confirmado |
| S-11 | Datos maestros en catálogos internos GOP | A-08 confirmado |

---

## 14. Puntos de Decisión Pendientes

| # | Punto | Impacto |
|---|-------|---------|
| PD-FE-01 | Catálogo de contratos VAF para ANH — API compartida o filtro especial | Paso 1 |
| PD-FE-02 | Validación Mapa de Tierras (RN-25/26) — síncrona o asíncrona | Paso 3 |

---

## 15. Anexos

- **API Contract:** `api-contract.md`
- **OpenAPI Spec:** `openapi.yaml`
- **Spec Backend:** `spec-backend.md`
