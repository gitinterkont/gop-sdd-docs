# Especificación Frontend — Forma 202: Informe Mensual de Producción

**Feature:** RQF_GOP_13  
**Versión:** 1.0  
**Fecha:** 2026-04-28  
**Estado:** Propuesta — Pendiente aprobación del arquitecto humano  
**Stack:** Angular 21 (TypeScript)  

---

## 1. Contexto y Objetivo

La Forma 202 es uno de los instrumentos fundamentales para la fiscalización de la producción de hidrocarburos en Colombia. Registra el detalle mensual de producción por pozo y formación productora de cada campo, con volúmenes de petróleo, agua y gas, parámetros de calidad y estado operativo.

**Objetivo:** Implementar la interfaz de usuario para el ciclo completo de la Forma 202: visualización en bandeja, wizard de 3 pasos (encabezado, grilla de producción, firma y envío), cargue masivo de pozos históricos, validación cruzada con Formas 204/205, revisión y aprobación por Supervisor ANH.

---

## 2. Actores y Roles

| Rol | Acciones en la UI |
|---|---|
| **Operador GOP** | Visualiza formas de sus contratos, navega el wizard (3 pasos), edita datos precargados, agrega pozos manuales, carga archivo masivo, firma electrónicamente, envía. Corrige y reenvía si la forma es devuelta. |
| **Supervisor ANH Producción** | Visualiza formas de contratos/campos asignados, revisa la forma (modo solo lectura de datos), consulta log de cambios, aprueba o devuelve. |
| **Admin GOP** | Visualiza todas las formas de todas las operadoras. Acceso total de lectura. Puede crear formas manualmente. |

---

## 3. Historias de Usuario

| ID | Historia |
|---|---|
| HU-F202-01 | Como **Operador GOP**, quiero ver una bandeja con todas mis Formas 202 pendientes y su estado, para priorizar cuáles debo completar antes del vencimiento. |
| HU-F202-02 | Como **Operador GOP**, quiero seleccionar campo, formación, modalidad y tipo de trampa para una Forma 202, para configurar el encabezado antes de cargar la producción. |
| HU-F202-03 | Como **Operador GOP**, quiero que al hacer clic en "Consultar" se precarguen los pozos activos desde AVM con sus datos de producción, para no digitarlos manualmente. |
| HU-F202-04 | Como **Operador GOP**, quiero editar cualquier dato precargado desde AVM y que el sistema registre automáticamente mis cambios, para poder corregir datos incorrectos con trazabilidad. |
| HU-F202-05 | Como **Operador GOP**, quiero agregar pozos históricos de forma individual o mediante carga masiva desde un archivo Excel, para reportar campos con pozos no disponibles en AVM. |
| HU-F202-06 | Como **Operador GOP**, quiero ver la fila TOTAL con sumatorias y promedios calculados automáticamente, para verificar la consistencia de los datos antes de enviar. |
| HU-F202-07 | Como **Operador GOP**, quiero ejecutar la validación cruzada contra Formas 204 y 205 y ver el resultado como advertencia, para anticipar posibles inconsistencias. |
| HU-F202-08 | Como **Operador GOP**, quiero firmar electrónicamente con mi nombre e ingresando matrícula profesional y enviar la forma, para cumplir con el plazo normativo. |
| HU-F202-09 | Como **Operador GOP**, quiero recibir una alerta en la bandeja cuando falte 1 día hábil para el vencimiento, para no incurrir en incumplimiento. |
| HU-F202-10 | Como **Operador GOP**, quiero corregir una forma devuelta por el Supervisor y reenviarla dentro de los 3 días hábiles, para completar el ciclo. |
| HU-F202-11 | Como **Supervisor ANH**, quiero revisar la Forma 202 enviada por el operador en modo lectura, consultar el log de modificaciones sobre datos AVM, y aprobar o devolver con observaciones. |
| HU-F202-12 | Como **Supervisor ANH**, quiero firmar electrónicamente la aprobación para generar el PDF oficial con doble firma. |
| HU-F202-13 | Como cualquier rol autorizado, quiero descargar el PDF generado de la Forma 202, para consulta o archivo. |

---

## 4. Criterios de Aceptación

### CA-F202-01 — Bandeja de Formas 202
**Given** un Operador GOP autenticado con contratos asignados  
**When** accede a la ruta de Formas 202  
**Then** visualiza una tabla paginada con las formas de sus contratos, mostrando: consecutivo, campo, formación, modalidad, mes/año, estado, versión, fecha de creación, indicador de vencimiento próximo  
**And** puede filtrar por año, mes, campo, formación, estado  
**And** puede ordenar por cualquier columna  

### CA-F202-02 — Encabezado (Paso 1)
**Given** un Operador GOP que accede a una Forma 202 en estado `Registration`  
**When** carga el Paso 1 del wizard  
**Then** ve los campos autocompletados (operador, contrato, campo, formación, modalidad, mes/año) en modo solo lectura  
**And** puede seleccionar tipo de trampa (Estructural, Estratigráfica, Mixta) desde un dropdown  
**And** el botón "Consultar" / "Cargar Producción" está habilitado solo si todos los campos están completos (RN-07)  

### CA-F202-03 — Precarga desde AVM
**Given** un Operador GOP en el Paso 1 con todos los campos completos  
**When** hace clic en "Consultar"  
**Then** el sistema invoca la API de precarga (`POST /{id}/preload`)  
**And** muestra un indicador de carga ("Consultando AVM...")  
**And** al completar, navega automáticamente al Paso 2 con la grilla precargada  
**And** si AVM falla (502), muestra mensaje de error con opción de registro manual (RN-25)  

### CA-F202-04 — Grilla de Producción (Paso 2)
**Given** la precarga completada  
**When** el Paso 2 se muestra  
**Then** se visualiza una tabla con las 22 columnas organizadas en grupos: Identificación, Días, Petróleo, Agua, Gas, Calidad, Estado  
**And** cada fila muestra los datos del pozo con el origen (AVM / Manual / Masivo) como indicador visual  
**And** la fila TOTAL se muestra al pie con valores calculados  
**And** la grilla soporta paginación (50 filas por defecto, máximo 500 para campos grandes)  
**And** soporta búsqueda por nombre de pozo y filtro por estado/origen de datos  

### CA-F202-05 — Edición en línea con trazabilidad
**Given** un Operador GOP viendo la grilla en estado `Registration`  
**When** modifica un valor de una celda precargada desde AVM  
**Then** la celda se marca visualmente como "modificada" (indicador de color/ícono)  
**And** el campo `isModified` del registro se actualiza  
**And** el contador de modificaciones se actualiza en el resumen  
**And** los cambios se persisten al salir de la celda (auto-guardado al hacer blur)  

### CA-F202-06 — Validación de acumulados
**Given** un pozo con acumulado del mes anterior  
**When** el operador ingresa un acumulado menor al del mes anterior  
**Then** la celda muestra error de validación inline: "El acumulado no puede disminuir respecto al mes anterior"  
**And** el error impide guardar esa fila hasta que se corrija  

### CA-F202-07 — Cargue individual de pozo
**Given** un Operador GOP en la grilla (Paso 2)  
**When** hace clic en "Agregar pozo"  
**Then** se muestra un formulario/fila editable con todos los 22 campos vacíos  
**And** aplican las mismas validaciones que para edición  
**And** al guardar, la fila aparece en la grilla marcada como origen "Manual"  

### CA-F202-08 — Cargue masivo desde Excel
**Given** un Operador GOP en la grilla  
**When** hace clic en "Carga masiva" y selecciona un archivo .xlsx  
**Then** el sistema sube el archivo (`POST /{id}/well-productions/bulk-upload`)  
**And** muestra un indicador de progreso  
**And** al completar, muestra resumen: total procesados, exitosos, fallidos  
**And** si hay errores, muestra detalle por fila (número de fila, pozo, campo, mensaje)  
**And** las filas exitosas aparecen en la grilla con origen "BulkLoad"  
**And** puede descargar la plantilla Excel desde un enlace visible  

### CA-F202-09 — Fila TOTAL
**Given** datos en la grilla  
**When** se actualiza cualquier valor  
**Then** la fila TOTAL recalcula automáticamente  
**And** sumatorias para: días, producción diaria/mensual/acumulada (petróleo, agua, gas)  
**And** promedios aritméticos para: factores de campo  
**And** BSW Total calculado por fórmula  
**And** API Gravity Total por promedio ponderado vía gravedad específica  
**And** RGP Total calculado por fórmula  
**And** si algún denominador es cero, muestra "N/A" en la celda correspondiente  

### CA-F202-10 — Validación cruzada 204/205
**Given** un Operador o Supervisor con la forma abierta  
**When** hace clic en "Validar contra 204/205"  
**Then** el sistema ejecuta la validación cruzada  
**And** muestra resultado por tipo (Petróleo vs 204, Gas vs 205)  
**And** si hay diferencia, muestra badge "Warning" con el detalle del valor esperado vs encontrado  
**And** la alerta es informativa, no bloquea el envío  
**And** si la Forma 204 o 205 no existe, muestra "No disponible"  

### CA-F202-11 — Firma y Envío (Paso 3)
**Given** un Operador GOP en el Paso 3 con datos válidos en la grilla  
**When** ingresa nombre del ingeniero, matrícula profesional, y confirma la firma  
**Then** el botón "Firmar y Enviar" se habilita  
**And** al hacer clic, el sistema llama a `POST /{id}/actions/sign-and-submit`  
**And** muestra confirmación: "La Forma 202 ha sido firmada y enviada. El PDF se generará en breve."  
**And** el estado cambia a `Submitted` en la bandeja  
**And** el wizard pasa a modo solo lectura  

### CA-F202-12 — Devolución y corrección
**Given** una Forma 202 en estado `Registration` por devolución  
**When** el Operador accede a la forma  
**Then** ve un banner con el motivo de devolución y los pozos señalados (si aplica)  
**And** puede editar los datos y volver a firmar y enviar  
**And** ve un indicador de plazo restante (3 días hábiles)  

### CA-F202-13 — Revisión por Supervisor ANH
**Given** un Supervisor ANH con una forma en estado `Submitted` en su bandeja  
**When** abre la forma  
**Then** ve toda la información en modo solo lectura  
**And** puede acceder al log de modificaciones (pestaña o panel lateral)  
**And** puede filtrar el log por pozo y campo modificado  
**And** tiene botones "Aprobar" y "Devolver"  
**And** el botón "Devolver" solo está habilitado antes del día 19 del mes  

### CA-F202-14 — Aprobación por Supervisor
**Given** un Supervisor ANH revisando una forma `Submitted`  
**When** hace clic en "Aprobar", ingresa su nombre/matrícula y confirma  
**Then** la forma cambia a estado `Approved`  
**And** queda completamente bloqueada  
**And** se muestra confirmación: "La Forma 202 ha sido aprobada. El PDF final se generará en breve."  

### CA-F202-15 — Descarga de PDF
**Given** una forma con PDF generado (post-envío o post-aprobación)  
**When** el usuario hace clic en "Descargar PDF"  
**Then** se descarga el archivo PDF correspondiente al estado actual  

---

## 5. Flujos de UI

### 5.1 Flujo Principal — Bandeja → Wizard → Envío

```
[Bandeja de Formas 202]
    ↓ clic en fila
[Wizard Paso 1 — Encabezado]
    ↓ clic "Consultar"
    ↓ (indicador de carga: precarga AVM)
[Wizard Paso 2 — Grilla de Producción]
    ↓ editar / agregar / carga masiva
    ↓ clic "Siguiente"
[Wizard Paso 3 — Firma y Envío]
    ↓ ingresar datos firmante + confirmar
    ↓ clic "Firmar y Enviar"
[Confirmación de envío]
    ↓ redirige a bandeja
[Bandeja — estado actualizado: Submitted]
```

### 5.2 Flujo Alterno — Devolución

```
[Bandeja — Supervisor hace clic en forma Submitted]
[Vista de Revisión — solo lectura + log de cambios]
    ↓ clic "Devolver"
    ↓ ingresa motivo (obligatorio)
[Confirmación de devolución]

[Bandeja Operador — forma vuelve a Registration con banner de devolución]
[Wizard Paso 2 — edición habilitada, pozos señalados resaltados]
    ↓ corrige datos
[Wizard Paso 3 — firma y reenvío]
```

### 5.3 Transiciones de Pantalla por Estado

| Estado Forma | Operador ve... | Supervisor ve... |
|---|---|---|
| `Initial` | No visible (creada por sistema) | No visible |
| `Registration` | Wizard editable (3 pasos) | No visible (aún no enviada) |
| `Submitted` | Wizard en solo lectura | Vista de revisión + botones Aprobar/Devolver |
| `Approved` | Wizard en solo lectura + PDF | Solo lectura + PDF |

---

## 6. Componentes Funcionales Requeridos

### 6.1 Bandeja de Formas 202
- Tabla paginada y ordenable
- Filtros: año, mes, campo, formación, estado
- Badge de estado con colores diferenciados
- Indicador de vencimiento próximo (alerta visual en rojo si faltan ≤ 1 día hábil)
- Conteo total de registros
- Botón de acceso al wizard por cada fila

### 6.2 Wizard de 3 Pasos
- Navegación secuencial con indicador de progreso (stepper)
- Navegación hacia atrás permitida en estado `Registration`
- Estado del wizard persistido (no se pierde al navegar entre pasos)
- En modo solo lectura (estados `Submitted`/`Approved`): todos los pasos visibles pero sin edición

### 6.3 Encabezado del Formulario (Paso 1)
- Campos autocompletados en solo lectura: operador, contrato, campo, formación, modalidad, mes, año, consecutivo, estado, fecha de creación, radicado
- Dropdown editable: tipo de trampa
- Textarea editable: observaciones (máx. 2000 chars, con contador)
- Botón "Consultar" / "Cargar Producción"

### 6.4 Grilla de Producción (Paso 2)
- Tabla con 22 columnas agrupadas en secciones colapsables: Identificación (3), Días (2), Petróleo (4), Agua (4), Gas (4), Calidad (3), Estado (2)
- Edición inline por celda (clic para editar, blur para guardar)
- Indicador visual de celdas modificadas vs originales AVM
- Fila TOTAL fija al pie (sticky footer)
- Paginación configurable (50/100/200/500 filas)
- Búsqueda por nombre de pozo
- Filtros por estado de pozo, tipo de pozo, origen de datos
- Botón "Agregar pozo" → formulario de fila nueva
- Botón "Carga masiva" → diálogo de upload + enlace a plantilla descargable
- Botón "Validar contra 204/205" → muestra resultado inline
- Indicador de origen de datos por fila (ícono: AVM / Manual / Excel)

### 6.5 Panel de Firma y Envío (Paso 3)
- Sección "Firma del Operador":
  - Campo texto: Nombre del ingeniero (máx. 50 chars, obligatorio)
  - Campo numérico: N.° Matrícula (7 dígitos, obligatorio)
  - Checkbox de confirmación: "Confirmo que la información es veraz"
  - Fecha de presentación (automática, solo lectura)
- Sección "Aprobación ANH" (solo lectura hasta que el supervisor apruebe):
  - Nombre ingeniero ANH, matrícula, fecha de aprobación
- Sección "Observaciones":
  - Textarea (readonly si no hay observaciones; inline con paso 1 si tiene)
- Botón "Firmar y Enviar" (habilitado solo si todos los campos están completos y checkbox confirmado)

### 6.6 Vista de Revisión (Supervisor)
- Misma información que el wizard pero en modo completamente readonly
- Panel o pestaña adicional: "Log de Modificaciones"
  - Tabla filtrable: pozo, campo modificado, valor original, valor nuevo, usuario, fecha
  - Paginada
- Botones de acción: "Aprobar" y "Devolver"
- Al devolver: modal con campo de texto obligatorio (motivo) y selección opcional de pozos afectados

### 6.7 Banner de Devolución
- Visible cuando la forma está en `Registration` y tiene historial de devolución
- Muestra: motivo de la devolución, fecha, plazo restante
- Resalta los pozos señalados (si se indicaron en la devolución)

### 6.8 Diálogo de Carga Masiva
- Zona de drag-and-drop o selector de archivo
- Solo acepta .xlsx, máximo 10 MB
- Enlace "Descargar plantilla" visible
- Indicador de progreso durante la carga
- Resultado post-carga: tabla de errores/warnings por fila

---

## 7. Reglas de Validación de Formularios (Client-side)

Todas las validaciones client-side son réplica de las validaciones de backend. **La validación en cliente nunca es la única defensa** — el backend siempre revalida.

| Campo | Regla | Mensaje de error |
|---|---|---|
| `trapType` | Requerido. Uno de: Structural, Stratigraphic, Mixed | "Seleccione un tipo de trampa" |
| `observations` | Opcional. Máx. 2000 caracteres | "Máximo 2000 caracteres" |
| `wellName` | Requerido. Máx. 100 caracteres | "Nombre del pozo es obligatorio" |
| `municipalityDaneCode` | Requerido. Exactamente 5 dígitos numéricos | "Código DANE debe ser de 5 dígitos" |
| `productionMethod` | Requerido. Lista cerrada | "Seleccione método de producción" |
| Todos los campos numéricos de producción | Requeridos. ≥ 0. Máximo 2 decimales (5 para factores) | "El valor no puede ser negativo" |
| `bswPercent` | 0 ≤ valor ≤ 100. 3 decimales | "BSW debe estar entre 0% y 100%" |
| `apiGravity` | ≥ 0. 1 decimal | "Gravedad API no puede ser negativa" |
| Acumulados (petróleo, agua, gas, días) | ≥ acumulado del mes anterior (cuando existe) | "El acumulado no puede ser menor al del mes anterior (X)" |
| `engineerFullName` | Requerido. Máx. 50 caracteres | "Nombre del ingeniero es obligatorio" |
| `professionalLicenseNumber` | Requerido. Exactamente 7 dígitos | "Matrícula debe ser de 7 dígitos" |
| `confirmSignature` | Debe ser `true` para habilitar envío | "Debe confirmar la firma" |
| `returnReason` | Requerido (al devolver). Máx. 2000 caracteres | "Indique el motivo de la devolución" |
| Archivo Excel | Solo .xlsx. Máx. 10 MB | "Solo se aceptan archivos Excel (.xlsx) de hasta 10 MB" |

---

## 8. Estados de la Interfaz

### 8.1 Máquina de Estados de la UI

```
[Cargando]
   ↓ datos cargados
[Datos OK] ← estado estable: wizard habilitado
   ↓ error de red
[Error de Red] → botón "Reintentar"
   ↓ error de negocio (409, 422)
[Error de Negocio] → mensaje específico + acción sugerida
   ↓ sesión expirada (401)
[Sesión Expirada] → modal "Su sesión ha expirado" → redirige a login
   ↓ sin permisos (403)
[Sin Permisos] → pantalla de acceso denegado
   ↓ datos vacíos
[Estado Vacío] → mensaje "No hay formas para el período seleccionado"
   ↓ precarga AVM en progreso
[Precargando AVM] → spinner + texto "Consultando datos de producción..."
   ↓ precarga AVM fallida (502)
[Error AVM] → "No se pudo conectar con AVM. Puede registrar datos manualmente." + botón "Continuar sin precarga"
   ↓ carga masiva en progreso
[Cargando Archivo] → barra de progreso
   ↓ guardado automático
[Guardando] → indicador sutil de auto-guardado (spinner en celda)
```

### 8.2 Estados por Componente

| Componente | Estados posibles |
|---|---|
| Bandeja | Cargando, Datos OK, Vacío, Error de Red |
| Wizard Paso 1 | Cargando, Datos OK, Error de Red |
| Wizard Paso 2 (grilla) | Cargando, Precargando AVM, Error AVM, Datos OK, Vacío (sin pozos), Guardando, Error de Negocio |
| Wizard Paso 3 (firma) | Datos OK, Enviando, Enviado, Error de Envío |
| Vista de Revisión (Supervisor) | Cargando, Datos OK, Error de Red |
| Diálogo de Carga Masiva | Selección archivo, Cargando, Resultado OK, Resultado con errores |
| Log de Modificaciones | Cargando, Datos OK, Vacío |

---

## 9. NFRs Frontend

### 9.1 Performance
| Métrica | Objetivo |
|---|---|
| LCP (Largest Contentful Paint) | < 2.5s en bandeja; < 3.0s en wizard con grilla |
| INP (Interaction to Next Paint) | < 200ms |
| TTI (Time to Interactive) | < 3.0s |
| Renderizado de grilla (500 filas) | < 1.5s incluyendo scroll virtual |
| Guardado inline (blur de celda) | < 500ms percibido por el usuario |

### 9.2 Accesibilidad (WCAG 2.1 AA)
- Navegación completa por teclado en la grilla (Tab entre celdas, Enter para editar, Escape para cancelar)
- Contraste mínimo 4.5:1 para texto, 3:1 para elementos de UI grandes
- Etiquetas ARIA para la grilla (`role="grid"`, `aria-label` en celdas editables)
- Lectores de pantalla: anuncio de cambios de estado, errores de validación, confirmaciones
- Focus visible en todos los elementos interactivos
- Skip links para navegación rápida dentro del wizard

### 9.3 Responsividad
| Breakpoint | Comportamiento |
|---|---|
| Desktop (≥ 1280px) | Grilla completa con scroll horizontal para las 22 columnas |
| Tablet (768px — 1279px) | Grilla con columnas agrupadas colapsables; grupos prioritarios visibles |
| Mobile (< 768px) | Bandeja adaptada a tarjetas; grilla no editable (solo consulta). Edición redirige a desktop. |

### 9.4 Internacionalización
- Idioma primario: Español (CO)
- Formato de fechas: DD/MM/AAAA en UI, ISO 8601 en API
- Separador de miles: punto (1.250.000)
- Separador decimal: coma (0,985)
- Moneda/unidades: BBL (barriles), KPC (miles de pies cúbicos), BOPD, BWPD
- Textos externalizados en archivos de localización

### 9.5 Compatibilidad de Navegadores
- Chrome ≥ 120
- Edge ≥ 120
- Firefox ≥ 115
- Safari ≥ 17

---

## 10. Seguridad Frontend (OWASP Top 10 aplicable)

### 10.1 Almacenamiento de Tokens
- El token JWT se almacena en memoria (variable de servicio) o en `httpOnly` cookie según la estrategia definida por CONSTITUTION-fe.md.
- **Nunca** en `localStorage` ni `sessionStorage` sin justificación documentada.
- Refresh token manejado vía endpoint dedicado; nunca expuesto al JavaScript del cliente.

### 10.2 Prevención XSS
- Todo contenido dinámico renderizado mediante binding de Angular (`{{ }}` o `[property]`), que escapa automáticamente.
- **Prohibido** usar `innerHTML` con datos del usuario o del backend sin sanitización explícita.
- Los campos de observaciones y motivos de devolución se renderizan como texto plano, nunca como HTML.

### 10.3 CSRF
- Si se usan cookies para autenticación, el backend provee un token anti-CSRF que el frontend envía en cada mutación (POST/PUT/PATCH/DELETE).
- Si se usa Bearer JWT en header, CSRF no aplica (stateless).

### 10.4 Validación Client-side
- Todas las validaciones del §7 se implementan en el cliente para UX inmediata.
- **Nunca** son la única defensa. El backend siempre revalida.
- Los mensajes de error del backend se muestran tal cual (ya sanitizados).

### 10.5 Manejo de Sesión
- Sesión expirada (401): interceptor HTTP global detecta el 401, muestra modal "Sesión expirada" y redirige a login.
- Renovación de token: el interceptor intenta refresh antes de mostrar el modal. Si el refresh falla, redirige.
- Inactividad: timeout configurable (alineado con CONSTITUTION-ba.md §12.6).

### 10.6 Datos Sensibles
- No se loguean tokens, matrículas ni datos de ingenieros en la consola del navegador.
- En producción, `console.log` de datos sensibles está prohibido.
- Las herramientas de desarrollo no deben exponer payloads con datos sensibles en texto plano.

### 10.7 Control de Acceso Visual por Rol
- Componentes y botones se muestran/ocultan según el rol del usuario autenticado.
- El ocultamiento visual **nunca** reemplaza la autorización del backend. Si un usuario manipula el DOM y envía una petición no autorizada, el backend retorna 403.
- Rutas protegidas por guards que verifican rol antes de cargar el componente.

---

## 11. Edge Cases

| # | Escenario | Comportamiento Esperado |
|---|---|---|
| EC-01 | Usuario no autenticado accede a `/production/form-202` | Redirige a login con URL de retorno |
| EC-02 | Usuario autenticado sin rol para Forma 202 | Muestra pantalla "Acceso denegado" con mensaje claro |
| EC-03 | Sesión expira mientras el operador edita la grilla | Modal de sesión expirada. Los cambios no guardados se pierden (el auto-guardado por celda minimiza pérdida). El modal ofrece re-login y retorno a la forma. |
| EC-04 | Pérdida de conexión durante auto-guardado de celda | La celda muestra indicador de error. Al restaurar conexión, ofrece reintentar. No se avanza al siguiente paso hasta resolver. |
| EC-05 | Backend responde con timeout (>10s) en precarga AVM | Muestra mensaje "La consulta a AVM está tardando más de lo esperado" con opciones: "Esperar" o "Continuar sin precarga (registro manual)". |
| EC-06 | Backend retorna 5xx en cualquier operación | Muestra toast de error genérico: "Ocurrió un error inesperado. Intente de nuevo. Si persiste, contacte soporte." con `traceId` visible para reporte. |
| EC-07 | Grilla vacía (sin pozos precargados ni manuales) | Muestra estado vacío con mensaje: "No se encontraron pozos para esta selección. Puede agregar pozos manualmente o mediante carga masiva." |
| EC-08 | Archivo Excel de carga masiva con formato incorrecto | Muestra error: "El archivo no tiene el formato esperado. Descargue la plantilla oficial." |
| EC-09 | Archivo Excel excede 10 MB | Validación client-side antes de enviar: "El archivo excede el tamaño máximo de 10 MB." |
| EC-10 | Doble clic en "Firmar y Enviar" | El botón se deshabilita inmediatamente tras el primer clic (debounce). Muestra spinner. El backend es idempotente. |
| EC-11 | Operador intenta editar una forma en estado `Submitted` o `Approved` | Todos los campos en solo lectura. No hay botones de edición visibles. |
| EC-12 | Supervisor intenta devolver después del día 19 | Botón "Devolver" deshabilitado con tooltip: "No se puede devolver después del día 19 del mes." |
| EC-13 | Campo con +2000 pozos (ej. La Cira) | La grilla usa virtualización de filas (solo renderiza las visibles). Paginación forzada. La carga masiva es el mecanismo principal. |
| EC-14 | Datos parciales del backend (campos null) | Celdas null se muestran como "—" (guion). No se calculan en la fila TOTAL. |
| EC-15 | Operador accede a forma de otro operador/contrato | Backend retorna 403. UI muestra "No tiene permisos para acceder a esta forma." |

---

## 12. Criterios de Testing y Cobertura

### 12.1 Niveles Requeridos

| Nivel | Alcance | Herramienta sugerida |
|---|---|---|
| Unitarias | Componentes individuales (bandeja, wizard steps, grilla, cálculos de totales), pipes, validadores | Framework de testing de Angular |
| Integración | Flujo completo de wizard (paso 1 → 2 → 3), interacción con servicios HTTP mockeados | Framework de testing de Angular |
| E2E | Flujos críticos end-to-end contra backend real o mock server | Herramienta E2E compatible con Angular |

### 12.2 Cobertura Mínima
- **80%** en lógica de negocio de componentes (cálculos de totales, validaciones, transformaciones)
- **70%** en componentes de presentación
- **90%** en pipes y utilidades de cálculo

### 12.3 Escenarios Críticos que DEBEN tener E2E

| # | Escenario E2E |
|---|---|
| 1 | Flujo completo: login → bandeja → abrir forma → precarga → editar celda → firmar → enviar → verificar estado en bandeja |
| 2 | Cargue masivo: subir archivo con errores parciales → verificar filas cargadas y errores reportados |
| 3 | Devolución: supervisor devuelve → operador corrige → reenvía → supervisor aprueba |
| 4 | Acceso denegado: usuario sin rol intenta acceder a ruta protegida → redirige |
| 5 | Sesión expirada durante edición → modal → re-login → retorno |
| 6 | Validación de acumulados: ingresar acumulado menor → error inline visible |
| 7 | Campo con grilla grande (>100 pozos): paginación y búsqueda funcionan correctamente |

---

## 13. Anexos

- **API Contract:** `api-contract.md` (en este directorio)
- **OpenAPI Spec:** `openapi.yaml` (en este directorio)
- **Spec Backend:** `spec-backend.md` (en este directorio)
- **Modelo de Datos:** `data-model.md` (raíz del repositorio)
- **Fuente funcional:** RQF_GOP_13_Forma202 V1 + BP_GOP_13_Forma202 V1

---

## 14. Puntos de Decisión Pendientes

| # | Punto | Impacto |
|---|---|---|
| PD-FE-01 | La virtualización de la grilla para campos con +2000 pozos puede requerir una librería específica de tabla virtual. La decisión de librería la toma el arquitecto frontend. | Performance de grilla |
| PD-FE-02 | El mecanismo de auto-guardado (por celda al blur vs. botón "Guardar" explícito) debe validarse con UX. Este spec asume auto-guardado por blur para minimizar pérdida de datos. | UX / flujo de edición |
| PD-FE-03 | El formato de visualización de números (separador de miles punto, decimal coma) debe confirmarse con el equipo funcional ANH, dado que los reportes regulatorios usan ambas convenciones. | Internacionalización |
| PD-FE-04 | La plantilla Excel para carga masiva debe incluir validaciones nativas de Excel (data validation en celdas) para reducir errores antes del upload. El alcance de estas validaciones lo define el equipo funcional. | UX de carga masiva |
