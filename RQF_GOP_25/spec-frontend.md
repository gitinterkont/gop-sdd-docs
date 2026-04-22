# Especificación Frontend — RQF_GOP_25: Registro de Pozo Antiguo / Perforado

**Versión:** 1.0
**Fecha:** Abril 2026
**Feature:** RQF_GOP_25_RegistroPozoAntiguo
**Estado:** Propuesta — pendiente aprobación del arquitecto humano

---

## 1. Contexto y Objetivo

El sistema GOP 360° requiere una funcionalidad que permita a las operadoras registrar pozos que ya fueron perforados, terminados y —en muchos casos— cuentan con Forma 103 aprobada. A diferencia de la Creación de Pozo Nuevo (RQF_GOP_01, ~18 campos, unilateral), el Registro de Pozo Antiguo es un formulario extenso (~104 campos, 16 secciones, 6 tablas dinámicas) con flujo de aprobación ANH.

**Objetivo funcional:** Proporcionar una interfaz que guíe al operador a través de todas las secciones del registro, aplique validaciones en tiempo real, calcule campos derivados automáticamente, gestione el guardado parcial (borrador), y soporte el ciclo completo de envío → revisión ANH → aprobación/devolución.

---

## 2. Actores y Roles

| Rol | Contexto | Acciones UI |
|-----|----------|-------------|
| **Agente** | Operadora | Crear, editar, consultar, eliminar (en borrador/devuelto), enviar a aprobación, subir adjuntos |
| **Coordinador** | Operadora | Editar (borrador/devuelto), consultar. No puede crear ni enviar. |
| **Ingeniero ANH** | ANH | Consultar, aprobar, devolver con observaciones |
| **Admin GOP** | ANH | Todas las acciones (acceso total cross-operadora) |

---

## 3. Historias de Usuario

| ID | Historia |
|----|----------|
| HU-01 | Como **Agente de operadora**, quiero registrar un pozo antiguo con toda su información técnica, histórica y de localización, para incorporarlo oficialmente al sistema GOP 360°. |
| HU-02 | Como **Agente**, quiero guardar el progreso parcial del formulario como borrador, para poder continuar el diligenciamiento en otro momento. |
| HU-03 | Como **Agente**, quiero que el sistema genere automáticamente el UWI fiscalizado, para garantizar la trazabilidad y unicidad del pozo. |
| HU-04 | Como **Agente**, quiero enviar el registro completo a la ANH para su aprobación, adjuntando mi firma digital como ingeniero de petróleos. |
| HU-05 | Como **Coordinador**, quiero revisar y editar un registro antes del envío, para asegurar la calidad de los datos reportados. |
| HU-06 | Como **Ingeniero ANH**, quiero revisar el registro completo y aprobarlo o devolverlo con observaciones, para validar la información técnica del pozo. |
| HU-07 | Como **Agente**, quiero corregir un registro devuelto por la ANH y reenviarlo, para subsanar las observaciones indicadas. |
| HU-08 | Como **cualquier usuario**, quiero descargar el PDF del registro con la información diligenciada, para contar con un documento oficial imprimible. |
| HU-09 | Como **Agente**, quiero subir documentos soporte de registros eléctricos corridos, para completar el expediente técnico del pozo. |
| HU-10 | Como **Admin GOP**, quiero crear registros de pozo antiguo seleccionando cualquier operadora, para gestionar registros en representación de operadoras. |

---

## 4. Criterios de Aceptación

### CA-01 — Operadora prellenada
**Given** un Agente de operadora autenticado
**When** accede a "Crear Pozo Antiguo"
**Then** el campo Operadora muestra automáticamente su compañía, no es editable, y los contratos filtrados son solo los de esa operadora.

### CA-02 — Área Disponible para ANH
**Given** un usuario ANH selecciona "Área Disponible" como contrato
**When** se carga el formulario
**Then** el campo "Campo" se habilita como texto libre y los campos de Tipo de Contrato y Cuenca no se autocompletan.

### CA-03 — Operador Inicial "OTRO"
**Given** el operador selecciona "OTRO" en el dropdown de Operador Inicial
**When** se confirma la selección
**Then** se habilita un campo de texto libre (max 200 caracteres) para digitar el nombre del operador.

### CA-04 — Forma 103 = SÍ
**Given** el operador responde SÍ a "¿Cuenta con Forma 103?"
**When** se renderiza la sección del nombre del pozo
**Then** se muestra el mensaje "El nombre del pozo debe ser exactamente igual al de la Forma 103", se habilita campo de texto libre (100 caracteres), y se muestra el campo "Fecha de Aprobación de la Forma 103" como obligatorio.

### CA-05 — Forma 103 = NO (Integración AVM)
**Given** el operador responde NO a "¿Cuenta con Forma 103?"
**When** se renderiza la sección del nombre del pozo
**Then** se despliega un buscador/lista de pozos de AVM con el mensaje informativo de creación en AVM, y se oculta el campo "Fecha de Aprobación de la Forma 103".

### CA-06 — Exclusión mutua de clasificación
**Given** el operador selecciona un valor en Clasificación Final Lahee
**When** el campo cambia
**Then** la Clasificación Final (Desarrollo/Estratigráfico) se deshabilita y se limpia. Y viceversa.

### CA-07 — Estratigráfico → Campo = N/A
**Given** Clasificación Final = Estratigráfico
**When** se confirma la selección
**Then** el campo "Campo" se establece automáticamente en "N/A" y se deshabilita.

### CA-08 — Campos condicionales offshore
**Given** Tipo de Pozo por Ubicación = Costa Afuera
**When** se actualiza el formulario
**Then** se habilitan Espesor de Lámina de Agua y Distancia a la Costa (ambos obligatorios), Elevación del Terreno se establece en 0 y no es editable, y Departamento/Municipio se vuelven opcionales.

### CA-09 — Datum exclusivo
**Given** el operador selecciona un datum Magna Sirgas con un origen
**When** se confirma la selección
**Then** la lista de Datum Bogotá se deshabilita y viceversa.

### CA-10 — Cálculo de espesor en intervalos
**Given** un intervalo de terminación con Desde=1000, Hasta=1025
**When** se completan ambos campos
**Then** Espesor se calcula automáticamente como 25 pies.

### CA-11 — Total Pies Abiertos
**Given** intervalos con espesores 25 (Abierto), 20 (Abierto), 40 (Aislado), 100 (Abierto)
**When** se calcula el Total Pies Abiertos
**Then** el resultado es 145 (excluye Aislado).

### CA-12 — Pruebas condicionales por objetivo F103
**Given** Objetivo Forma 103 = PH
**When** se renderiza el formulario
**Then** se muestra Prueba de Producción, se ocultan Inyectividad y Monitoreo.

### CA-12b — Disposal habilita Inyectividad
**Given** Objetivo Forma 103 = D (Disposal)
**When** se renderiza el formulario
**Then** se muestra la sección de Prueba de Inyectividad (supuesto S-04 confirmado).

### CA-13 — Bloqueo de eliminación
**Given** el registro está en estado "Enviado" o "Aprobado"
**When** la operadora intenta eliminar
**Then** la opción de eliminación no está visible / está deshabilitada.

### CA-14 — PDF condicional
**Given** se descarga el PDF del registro
**When** el objetivo es PH
**Then** solo se visualiza la sección de Producción; las secciones de Inyectividad y Monitoreo no aparecen.

### CA-15 — Pozo padre requerido
**Given** Trayectoria seleccionada = ST, PR o G
**When** se confirma la selección
**Then** se muestra un dropdown con los pozos del contrato para seleccionar el pozo padre/original (RN-05).

### CA-16 — UWI único
**Given** el sistema genera el UWI fiscalizado al guardar
**When** el UWI ya existe
**Then** se muestra error y se impide el guardado.

### CA-17 — Nombre de pozo duplicado
**Given** el operador ingresa un nombre de pozo
**When** el nombre ya existe en el sistema
**Then** se muestra error y se impide la creación.

### CA-18 — Formaciones: base = tope siguiente
**Given** se ingresan múltiples formaciones en la tabla dinámica
**When** se agrega la segunda formación
**Then** la Base de la primera formación se calcula automáticamente como el Tope MD de la segunda. La última Base corresponde a la Profundidad Total MD.

### CA-19 — Redondeo de coordenadas de formaciones
**Given** el operador ingresa Tope MD = 6200.67
**When** se aplica RN-23
**Then** el valor se redondea a 6201 (≥ 0.50 → siguiente unidad).

### CA-20 — No. Disparos por Pie = N/A si no es Cañoneo
**Given** tipo de terminación ≠ Cañoneo
**When** se muestra la tabla de intervalos
**Then** la columna "No. de Disparos por Pie" muestra "N/A" y no es editable.

### CA-21 — Mensaje normativo OH
**Given** Tipo de Pozo por Terminación = OH (Hueco Abierto)
**When** se selecciona
**Then** se muestra el mensaje normativo de la Resolución 40537/2024 Art. 21 Parágrafo 3.

### CA-22 — Notificación de cambio de estado
**Given** un registro cambia de estado (enviado, aprobado, devuelto)
**When** la transición se completa
**Then** la operadora recibe notificación (in-app y/o email, RN-29).

### CA-23 — Yacimiento alimentado por formaciones
**Given** se capturaron formaciones en la Sección 11
**When** el operador llega a la Sección 12 (Yacimiento)
**Then** el dropdown de Yacimiento muestra las formaciones capturadas como opciones.

### CA-24 — Guardado parcial
**Given** el operador ha diligenciado al menos 1 campo
**When** hace clic en "Guardar Borrador"
**Then** el registro se guarda en estado Borrador sin validar campos obligatorios.

### CA-25 — Envío con validación completa
**Given** el operador hace clic en "Enviar a Aprobación"
**When** existen campos obligatorios vacíos o inválidos
**Then** el sistema impide el envío, resalta los campos con error y muestra mensajes específicos.

---

## 5. Flujos de UI

### 5.1 Flujo principal — Creación y envío

```
[Lista de registros] → [Botón "Crear Pozo Antiguo"]
  → [Wizard / Formulario multi-sección]
    → Sección 1: Información Contractual
    → Sección 2: Detalles Técnicos
    → Sección 3: Ubicación Geográfica
    → Sección 4: Identificadores (UWI auto)
    → Sección 5: Localización - Origen Inicial
    → Sección 6: Localización - Magna Sirgas Nacional
    → Sección 7: Coordenadas Geográficas
    → Sección 8: Información Física
    → Sección 9: Tipo de Terminación (tabla dinámica)
    → Sección 10: Pruebas (condicional)
    → Sección 11: Formaciones Encontradas (tabla dinámica)
    → Sección 12: Yacimiento
    → Sección 13: Tuberías de Revestimiento (tabla dinámica)
    → Sección 14: Registros Corridos (tabla dinámica + adjuntos)
    → Sección 15: Arenas de Agua Dulce (tabla dinámica)
    → Sección 16: Firmas y Acciones
  → [Guardar Borrador] | [Enviar a Aprobación]
```

### 5.2 Flujo de aprobación ANH

```
[Bandeja de registros pendientes (ANH)]
  → [Seleccionar registro]
  → [Vista de detalle completo (solo lectura)]
  → [Botón "Aprobar"] → Firma ANH → [Confirmación]
  → [Botón "Devolver"] → Observaciones → [Confirmación]
```

### 5.3 Flujo de corrección post-devolución

```
[Lista de registros (filtro: Devuelto)]
  → [Seleccionar registro devuelto]
  → [Vista con observaciones de ANH destacadas]
  → [Editar secciones necesarias]
  → [Guardar Borrador] | [Reenviar a Aprobación]
```

### 5.4 Transiciones de pantalla por estado

| Estado actual | Acciones visibles (Agente) | Acciones visibles (Ing. ANH) |
|---------------|---------------------------|------------------------------|
| `DRAFT` | Editar, Guardar Borrador, Enviar, Eliminar, Descargar PDF (preview) | — |
| `SUBMITTED` | Solo consulta, Descargar PDF (preview) | Aprobar, Devolver, Descargar PDF |
| `RETURNED` | Editar, Guardar Borrador, Reenviar, Eliminar, Ver observaciones | Solo consulta |
| `APPROVED` | Solo consulta, Descargar PDF (oficial) | Solo consulta, Descargar PDF |

---

## 6. Componentes Funcionales Requeridos

### 6.1 Página de lista de registros
- Tabla paginada con filtros por estado, contrato, nombre de pozo, UWI, rango de fechas.
- Botón "Crear Pozo Antiguo" (visible solo para Agente y Admin).
- Indicador visual del estado de cada registro (badge con color).
- Acciones contextuales por fila (editar, eliminar, ver detalle) según estado y rol.

### 6.2 Formulario multi-sección (wizard o scroll)
- Navegación entre 16 secciones con indicador de progreso.
- Cada sección debe poder guardarse parcialmente.
- Campos condicionales que se muestran/ocultan dinámicamente según selecciones previas.
- Campos automáticos/calculados actualizados en tiempo real.
- Barra de acciones fija: "Guardar Borrador" (siempre visible), "Enviar a Aprobación" (sección final).
- Preview del UWI fiscalizado actualizado en tiempo real conforme se llenan campos relevantes.

### 6.3 Tablas dinámicas (6 instancias)
- Agregar/eliminar filas con botón.
- Campos calculados por fila (espesor, base de formación).
- Totalizadores (Total Pies Abiertos).
- Validación por fila.
- Máximo de filas configurable: Intervalos de terminación (50), Formaciones (50), Tuberías (20), Registros (30), Arenas de agua dulce (20).

### 6.4 Selector de datum con exclusión mutua
- Dos dropdowns (Magna Sirgas, Bogotá) con comportamiento de radio visual.
- Al seleccionar uno, el otro se deshabilita y limpia.
- Encabezado informativo: "Hasta el año 2005 referirse al Datum Bogotá y a partir del 2005 referirse al Datum Magna-Sirgas" (RN-13).

### 6.5 Panel de observaciones ANH
- Visible cuando estado = `RETURNED`.
- Muestra historial de observaciones con fecha, autor y texto.
- Destacado visual (banner/alert) en la parte superior del formulario.

### 6.6 Componente de firma
- Captura nombre del ingeniero y número de matrícula profesional.
- Verificación de que el firmante tiene `ProfessionalLicense` válida.
- Sección de firma ANH visible solo en modo aprobación.

### 6.7 Upload de adjuntos para registros eléctricos
- Asociado a cada fila de la tabla "Registros Corridos".
- Drag & drop o botón de selección.
- Validación de tipo de archivo y tamaño en cliente.
- Indicador de progreso de upload.
- Preview del nombre del archivo adjuntado.

### 6.8 Vista de detalle (solo lectura)
- Muestra toda la información del registro en formato presentación.
- Usada por Ingeniero ANH para revisión.
- Botones de acción (Aprobar/Devolver) visibles según rol y estado.
- Descargar PDF.

---

## 7. Reglas de Validación de Formularios (Client-side)

> **Principio:** Las validaciones en cliente **nunca** son la única defensa. Son complemento UX; el backend revalida todo.

### 7.1 Validaciones por sección

**Sección 1 — Contrato Actual:**
| Campo | Regla | Mensaje |
|-------|-------|---------|
| Contrato | Requerido | "Seleccione un contrato" |

**Sección 1 — Contrato Histórico:**
| Campo | Regla | Mensaje |
|-------|-------|---------|
| Operador Inicial | Requerido (dropdown o texto si "OTRO") | "Seleccione o ingrese el operador inicial" |
| Contrato Inicial | Requerido, max 100 chars | "Ingrese el contrato inicial" |
| Tipo Contrato Inicial | Requerido | "Seleccione el tipo de contrato" |

**Sección 2 — Detalles Técnicos:**
| Campo | Regla | Mensaje |
|-------|-------|---------|
| Trayectoria | Requerido | "Seleccione el tipo de trayectoria" |
| Pozo Padre | Requerido si trayectoria ∈ {ST, PR, G} | "Seleccione el pozo padre/original" |
| ¿Cuenta con F103? | Requerido | "Indique si cuenta con Forma 103" |
| Nombre del Pozo | Requerido, max 100 chars | "Ingrese el nombre del pozo" |
| Clasificación | Al menos uno (Lahee XOR Desarrollo/Estratigráfico) | "Seleccione una clasificación final" |
| Campo | Requerido (salvo Estratigráfico o Área Disponible) | "Seleccione el campo" |
| Ángulo | Requerido | "Seleccione el tipo de ángulo" |
| Objetivo F103 | Requerido | "Seleccione el objetivo según Forma 103" |
| Estado Actual | Requerido | "Seleccione el estado actual del pozo" |
| Objetivo Actual | Requerido | "Seleccione el objetivo actual" |
| Terminación | Requerido | "Seleccione el tipo de terminación" |

**Sección 3 — Ubicación:**
| Campo | Regla | Mensaje |
|-------|-------|---------|
| Departamento | Requerido si Continental | "Seleccione el departamento" |
| Municipio | Requerido si Continental; filtrado por departamento (RN-11) | "Seleccione el municipio" |
| Cluster | Requerido | "Seleccione el cluster/locación" |

**Secciones 5-7 — Coordenadas:**
| Campo | Regla | Mensaje |
|-------|-------|---------|
| Datum (Sección 5) | Requerido, exclusión mutua | "Seleccione un datum" |
| Coordenadas Norte/Este | Requerido, numérico, max 2 decimales | "Ingrese coordenada válida" |
| Latitud | Requerido, rango [-90, 90], formato ±gg.gggggg | "Latitud fuera de rango" |
| Longitud | Requerido, rango [-180, 0], formato -gg.gggggg | "Longitud debe ser negativa (oeste)" |

**Sección 8 — Información Física:**
| Campo | Regla | Mensaje |
|-------|-------|---------|
| Profundidades (MD, TVD, TVDss) | Requerido, > 0 | "Ingrese profundidad válida" |
| Fechas | Requerido, formato fecha válido | "Ingrese fecha válida" |
| Fecha Inicio < Fecha Fin | Cross-field | "La fecha de inicio debe ser anterior a la de fin" |
| Fecha Fin ≤ Fecha Terminado | Cross-field | "La fecha fin de perforación debe ser anterior o igual a fecha terminado" |
| Observaciones | Requerido, max 300 chars | "Ingrese observaciones" |
| F103 Approval Date | Requerido si hasForm103 = SÍ | "Ingrese la fecha de aprobación de la Forma 103" |

**Sección 9 — Intervalos:**
| Campo | Regla | Mensaje |
|-------|-------|---------|
| Al menos 1 intervalo | Requerido | "Agregue al menos un intervalo" |
| Desde < Hasta | Por fila | "Desde debe ser menor que Hasta" |
| Disparos por Pie | Requerido si tipo = Cañoneo, ≥ 0 | "Ingrese disparos por pie" |

**Secciones 10.1-10.3 — Pruebas (condicionales):**
- Todos los campos de la sección visible son requeridos.
- Valores numéricos > 0 donde aplique.
- BSW % entre 0 y 100.

**Sección 11 — Formaciones:**
| Campo | Regla | Mensaje |
|-------|-------|---------|
| Al menos 1 formación | Requerido | "Agregue al menos una formación" |
| Nombre formación | Requerido | "Ingrese nombre de la formación" |
| Topes en orden ascendente | Cross-row: Tope MD(n+1) > Tope MD(n) | "Las formaciones deben estar en orden de profundidad" |

**Sección 13 — Tuberías:**
| Campo | Regla | Mensaje |
|-------|-------|---------|
| Al menos 1 tubería | Requerido | "Agregue al menos una tubería" |
| Todos los campos por fila | Requerido | "Complete todos los campos de la tubería" |

**Sección 15 — Arenas de agua dulce:**
| Campo | Regla | Mensaje |
|-------|-------|---------|
| Al menos 1 arena | Requerido | "Agregue al menos una arena de agua dulce" |
| Tope < Base | Por fila | "Tope debe ser menor que Base" |

### 7.2 Validación al guardar borrador
- **Sin validación de obligatoriedad.** Cualquier estado parcial se acepta.
- Solo validación de formato/tipo (si un campo tiene valor, que sea del tipo correcto).

### 7.3 Validación al enviar a aprobación
- **Validación completa** de todos los campos obligatorios y reglas cross-field.
- Si hay errores, scroll al primer campo con error y panel de resumen de errores visible.

---

## 8. Estados de la Interfaz

### 8.1 Máquina de estados del formulario

```
[INITIAL] → (carga de catálogos)
  → [LOADING_CATALOGS] → (éxito) → [READY]
                        → (error) → [CATALOG_ERROR] → (retry) → [LOADING_CATALOGS]

[READY] → (usuario edita) → [EDITING]
  → (guardar borrador) → [SAVING] → (éxito) → [SAVED]
                                   → (error) → [SAVE_ERROR]
  → (enviar) → [VALIDATING] → (ok) → [SUBMITTING] → (éxito) → [SUBMITTED]
                                                    → (error) → [SUBMIT_ERROR]
                             → (errores) → [VALIDATION_ERRORS]

[SAVED] → (usuario edita) → [EDITING]
```

### 8.2 Estados de la página de lista

| Estado | Condición | UI |
|--------|-----------|-----|
| Cargando | Fetch inicial o cambio de filtro | Skeleton/spinner en tabla |
| Datos | Respuesta con items.length > 0 | Tabla con datos y paginación |
| Vacío | items.length = 0 | Mensaje "No se encontraron registros" + botón crear |
| Error de red | Fallo en fetch | Banner de error con botón reintentar |
| Sin permisos | 403 | Mensaje "No tiene permisos para ver esta sección" |

### 8.3 Estados de upload de adjuntos

| Estado | UI |
|--------|-----|
| Idle | Botón "Adjuntar" o zona drop |
| Uploading | Progress bar con porcentaje |
| Uploaded | Nombre del archivo + botón eliminar |
| Error | Mensaje de error + botón reintentar |

---

## 9. NFRs Frontend

### 9.1 Performance
| Métrica | Objetivo |
|---------|----------|
| LCP (Largest Contentful Paint) | < 2.5s |
| INP (Interaction to Next Paint) | < 200ms |
| TTI (Time to Interactive) | < 3.0s para el formulario completo |
| Bundle size (lazy-loaded del dominio wells) | < 250 KB gzipped |
| Tiempo de guardado de borrador | < 1.5s (p95) percibido por el usuario |

### 9.2 Accesibilidad (WCAG 2.1 AA)
- Navegación completa por teclado (Tab/Shift+Tab entre campos, Enter para acciones).
- Contraste mínimo 4.5:1 en texto, 3:1 en elementos interactivos.
- Etiquetas ARIA en tablas dinámicas (`aria-label`, `aria-describedby` para errores).
- Anuncios de lectores de pantalla en cambios de estado (guardado, error, envío).
- Focus trapping en modales de confirmación.
- Skip links para navegación entre secciones del wizard.

### 9.3 Responsividad
| Breakpoint | Comportamiento |
|------------|---------------|
| Desktop (≥1200px) | Layout completo, tablas dinámicas con scroll horizontal mínimo |
| Tablet (768-1199px) | Formulario apilado, tablas dinámicas con scroll horizontal |
| Mobile (< 768px) | Secciones colapsables, tablas dinámicas como cards por fila |

### 9.4 Internacionalización
- Idioma principal: **Español (es-CO)**.
- Formato de fechas: `dd/MM/yyyy` en UI, ISO 8601 en API.
- Formato numérico: separador de miles `.`, separador decimal `,` en UI. En API: notación estándar (`.` decimal).
- Unidades: pies, metros, barriles, kscf — según campo (no se convierten, se muestran tal cual).

### 9.5 Compatibilidad de navegadores
- Chrome (últimas 2 versiones)
- Edge (últimas 2 versiones)
- Firefox (últimas 2 versiones)
- Safari (últimas 2 versiones en macOS)

---

## 10. Seguridad Frontend (OWASP Top 10 aplicable)

### 10.1 Almacenamiento de tokens
- Los tokens JWT se almacenan según la estrategia definida en CONSTITUTION-fe.md (preferencia por `httpOnly cookies` o mecanismo seguro equivalente, no `localStorage` para tokens de sesión).

### 10.2 Prevención XSS
- Todo contenido dinámico renderizado mediante binding seguro del framework (no `innerHTML` sin sanitización).
- Campos de texto libre (observaciones, nombre de pozo, etc.) sanitizados antes de renderizar.
- CSP headers configurados a nivel de servidor.

### 10.3 CSRF
- Protección CSRF en formularios que mutan estado (según mecanismo del framework).

### 10.4 Validación en cliente como UX, no como defensa
- Todas las validaciones de sección 7 son **UX aid**. El backend revalida íntegramente.
- Nunca se confía en datos manipulados del lado del cliente.

### 10.5 Manejo de sesión expirada
- Si un request devuelve `401`, se intenta refresh del token.
- Si el refresh falla, se redirige a login preservando la URL actual como return URL.
- Se muestra aviso: "Su sesión ha expirado. Inicie sesión nuevamente."
- **Guardado automático de borrador** antes de redireccionar (si hay datos no guardados).

### 10.6 Ocultamiento de datos sensibles
- No loguear tokens ni datos de matrícula profesional en console o herramientas de desarrollo en producción.
- Matrículas profesionales mostradas parcialmente enmascaradas en vistas de lista (ej: `PET-***45`).

### 10.7 Control de acceso visual por rol
- Botones de acción visibles según rol y estado (sección 5.4).
- Rutas protegidas por guards de rol.
- Datos fuera del alcance del tenant no se solicitan ni muestran.

---

## 11. Edge Cases

| # | Escenario | Comportamiento esperado |
|---|-----------|------------------------|
| EC-01 | Usuario no autenticado accede a la ruta | Redirect a login con return URL |
| EC-02 | Usuario autenticado sin rol requerido accede a la ruta | Página 403 con mensaje "No tiene permisos" |
| EC-03 | Sesión expira durante el diligenciamiento | Auto-guardado de borrador → redirect a login → al regresar, se carga el borrador guardado |
| EC-04 | Pérdida de conexión durante guardado | Mensaje "Sin conexión. Los cambios se guardarán cuando se restablezca." Retry automático al reconectar |
| EC-05 | Respuesta del backend > 5s | Spinner con texto "Guardando..." y posibilidad de cancelar tras 10s |
| EC-06 | Error 5xx del backend | Banner de error "Error del servidor. Intente nuevamente." con botón retry |
| EC-07 | UWI duplicado al crear | Mensaje de error en campo UWI con texto descriptivo |
| EC-08 | Nombre de pozo duplicado | Mensaje de error inline en campo de nombre |
| EC-09 | Doble clic en "Enviar a Aprobación" | Botón se deshabilita al primer clic. Spinner de carga. Se previene doble envío |
| EC-10 | Dos usuarios editan el mismo borrador | Optimistic concurrency via ETag. El segundo recibe error 409. Mensaje: "El registro fue modificado por otro usuario. Recargue para ver los cambios." |
| EC-11 | Archivo adjunto excede tamaño máximo | Validación en cliente antes de upload. Mensaje "El archivo excede el tamaño máximo permitido (10 MB)" |
| EC-12 | Tipo de archivo no permitido | Validación en cliente. Mensaje "Tipo de archivo no soportado" |
| EC-13 | Lista de AVM vacía o servicio no disponible | Mensaje informativo y campo de texto libre como fallback |
| EC-14 | Catálogos no cargan al iniciar | Estado CATALOG_ERROR con botón "Reintentar". No se permite editar hasta que carguen |
| EC-15 | Contrato sin cuenca asociada | Campo "Cuenca" muestra "No disponible" y no bloquea el guardado de borrador |
| EC-16 | Formaciones en orden incorrecto de profundidad | Validación cross-row al completar la tabla. Alerta visual en filas fuera de orden |

---

## 12. Criterios de Testing y Cobertura

### 12.1 Niveles requeridos

| Nivel | Alcance | Herramienta sugerida |
|-------|---------|---------------------|
| Unitarias (componentes) | Validaciones de formulario, cálculos (espesor, total pies abiertos, UWI, redondeo), lógica condicional (exclusión mutua, campos offshore) | Framework de testing del stack |
| Unitarias (servicios) | Mapeo DTOs, transformaciones de datos, lógica de estado | Framework de testing del stack |
| Integración | Interacción entre secciones del wizard, flujos de guardado/envío con mock de API | Framework de testing del stack |
| E2E | Flujos críticos completos contra backend real o mock server | Framework E2E del stack |

### 12.2 Cobertura mínima sugerida
- **80%** en lógica de negocio de componentes (cálculos, validaciones, condicionales).
- **70%** en servicios de API y mapeo de datos.

### 12.3 Escenarios E2E obligatorios
1. Flujo completo: Crear borrador → guardar → completar → enviar a aprobación → verificar estado SUBMITTED.
2. Flujo de corrección: Registro devuelto → editar → reenviar.
3. Flujo de aprobación ANH: Registro enviado → aprobar → verificar estado APPROVED y creación de Well.
4. Validación de exclusión mutua de clasificación.
5. Cálculos de tablas dinámicas (espesor, total pies abiertos, base de formaciones).
6. Campos condicionales offshore vs continental.
7. Intento de eliminación en estado no permitido.
8. Concurrencia: dos sesiones editando el mismo registro.
9. Upload y eliminación de adjuntos de registros eléctricos.

---

## 13. Supuestos Confirmados Documentados

| ID | Supuesto | Origen |
|----|----------|--------|
| S-A01 | El formulario se especifica como wizard multi-paso con guardado parcial por sección | A-01, confirmado |
| S-A02 | El operador ingresa coordenadas en 3 sistemas; el backend transforma las geográficas al CRS canónico | A-02, confirmado |
| S-A05 | La clasificación final (Lahee vs Desarrollo/Estratigráfico) es de libre elección por el operador | A-05, confirmado |
| S-A07 | UWI SGC se captura como texto libre en V1 (integración SGC pendiente) | A-07, confirmado |
| S-A08 | Coordinador puede editar en Borrador/Devuelto pero no puede crear ni enviar | A-08, confirmado |
| S-I01 | Ítem 14 (Objetivo F103) e Ítem 17 (Objetivo Actual) son campos distintos | I-01, confirmado |
| S-I03 | "Fecha Aprobación F103" es condicional: obligatoria solo si hasForm103 = SÍ | I-03, confirmado |
| S-S04 | Disposal (D) habilita la sección de Prueba de Inyectividad | S-04, confirmado |

---

## 14. Puntos de Decisión Pendientes

| # | Punto | Impacto |
|---|-------|---------|
| PD-FE-01 | Wizard multi-paso vs scroll continuo: decisión de UX final tras prototipo. El spec asume wizard. | Estructura del componente principal |
| PD-FE-02 | Formatos y tamaño máximo de adjuntos (PA-05). Propuesta temporal: PDF/TIFF/LAS, max 10 MB | Validación de upload |
| PD-FE-03 | Lista exacta de orígenes MAGNA-SIRGAS y zonas Datum Bogotá con códigos EPSG | Catálogos de la Sección 5 |
| PD-FE-04 | Catálogos de tuberías (diámetro, grado, peso) — pendientes de poblar (S-01) | Dropdowns de Sección 13 |
| PD-FE-05 | Catálogo de tipos de registro eléctrico — pendiente de poblar | Dropdown de Sección 14 |

---

## 15. Anexos

- **API Contract:** `api-contract.md` — Contrato de interfaz HTTP.
- **OpenAPI Spec:** `openapi.yaml` — Especificación validable.
- **Spec Backend:** `spec-backend.md` — Especificación funcional del backend.
