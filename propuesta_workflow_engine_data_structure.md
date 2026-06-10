# Propuesta de Patros de Diseños de arquitectura de datos y funcionalidades
## Workflow Engine Lite basado en definición de tareas y ruteo por acción

## 1. Objetivo

Definir una primera versión de un **workflow engine lite** orientado a procesos donde la progresión del flujo ocurre exclusivamente a partir de **acciones ejecutadas por el usuario** sobre una tarea activa, por ejemplo:

- Aceptar
- Devolver
- Rechazar
- Escalar
- Aprobar

La intención es que exista un **servicio genérico de completado de tareas** que, a partir de la tarea actual y de la acción recibida, consulte la configuración del workflow y determine cuál es la siguiente tarea que debe instanciarse.

Este enfoque busca:

- evitar lógica hardcodeada por flujo dentro del backend
- permitir que los flujos sean configurables por datos
- dejar una base simple pero extensible para soportar condiciones futuras
- mantener una ejecución transaccional y auditable

---

## 2. Alcance de esta primera versión

Esta propuesta cubre un motor de workflow con las siguientes características:

- workflows definidos por tablas
- tareas definidas por configuración
- acciones de usuario como disparadores únicos del flujo
- tabla de ruteo para resolver la siguiente tarea
- soporte opcional a futuro para rutas condicionadas
- historial de acciones ejecutadas
- instanciación de tareas dentro de una instancia de workflow
- finalización del workflow cuando no exista siguiente tarea o cuando se llegue a un nodo final

Esta versión **no** busca todavía resolver:

- timers
- eventos automáticos de sistema
- paralelismo complejo
- subprocesos
- scripting arbitrario
- diseñador visual
- reglas de asignación avanzadas

---

## 3. Principio de funcionamiento

La operación central del motor será un servicio similar a:

`completeTask(taskInstanceId, actionCode, payload, performedBy)`

### Flujo esperado del servicio

1. Consultar la tarea instancia actual.
2. Verificar que esté en estado pendiente.
3. Identificar la definición de la tarea actual.
4. Resolver la acción recibida.
5. Buscar en la tabla de ruteo la transición aplicable para:
   - la definición de tarea actual
   - la acción ejecutada
   - y eventualmente una condición, si existe
6. Registrar la acción ejecutada en el historial.
7. Completar la tarea actual.
8. Instanciar la siguiente tarea si aplica.
9. Si no hay siguiente tarea o la siguiente es terminal, cerrar la instancia del workflow.
10. Confirmar toda la operación en una sola transacción.

---

## 4. Modelo conceptual

### Entidades principales

- **Workflow Definition**: define la estructura lógica del flujo.
- **Task Definition**: define cada tipo de tarea posible dentro del flujo.
- **Action Definition**: catálogo de acciones permitidas.
- **Task Routing**: define hacia qué tarea se debe transicionar según la acción.
- **Workflow Instance**: ejecución concreta de un workflow.
- **Task Instance**: ejecución concreta de una tarea.
- **Task Action Log**: historial de acciones ejecutadas sobre tareas.

---

## 5. Propuesta de tablas

## 5.1. Tabla: `workflow_definition`

### Propósito
Representa la definición general de un workflow. Permite agrupar tareas y versionar la configuración del flujo.

### Columnas propuestas

| Columna | Tipo sugerido | Propósito |
|---|---|---|
| `id` | bigint / uuid | Identificador de la definición |
| `code` | varchar(100) | Código único del workflow |
| `name` | varchar(200) | Nombre funcional |
| `description` | varchar(1000) / text | Descripción de uso |
| `version` | integer | Versión de la definición |
| `status` | varchar(30) | Estado de la definición: DRAFT, ACTIVE, INACTIVE |
| `is_active` | boolean | Indica si la versión está vigente |
| `created_at` | timestamp | Fecha de creación |
| `created_by` | varchar(100) | Usuario creador |
| `updated_at` | timestamp | Fecha de última actualización |
| `updated_by` | varchar(100) | Usuario que actualizó |

### Reglas sugeridas

- `code + version` debe ser único.
- Solo una versión activa por `code`.
- No se debería editar libremente una definición ya activa si existen instancias en ejecución; idealmente se crea una nueva versión.

---

## 5.2. Tabla: `task_definition`

### Propósito
Define cada tarea posible dentro de un workflow. Cada fila representa un nodo del flujo.

### Columnas propuestas

| Columna | Tipo sugerido | Propósito |
|---|---|---|
| `id` | bigint / uuid | Identificador de la tarea definida |
| `workflow_definition_id` | fk | Referencia al workflow al que pertenece |
| `code` | varchar(100) | Código único de la tarea dentro del workflow |
| `name` | varchar(200) | Nombre funcional de la tarea |
| `description` | text | Descripción opcional |
| `task_type` | varchar(50) | Tipo de tarea, por ejemplo HUMAN |
| `is_start` | boolean | Indica si puede ser tarea inicial |
| `is_end` | boolean | Indica si representa un nodo terminal |
| `allows_comment` | boolean | Si la tarea permite comentarios |
| `requires_comment_on_reject` | boolean | Si ciertas acciones exigen comentario |
| `form_key` | varchar(150) | Referencia opcional a formulario o UI |
| `sla_hours` | integer | Tiempo esperado de atención |
| `is_active` | boolean | Habilitada o no |
| `created_at` | timestamp | Fecha de creación |
| `created_by` | varchar(100) | Usuario creador |
| `updated_at` | timestamp | Fecha de actualización |
| `updated_by` | varchar(100) | Usuario actualizador |

### Reglas sugeridas

- `workflow_definition_id + code` debe ser único.
- Puede existir una sola tarea inicial por workflow en esta primera versión.
- `is_end = true` identifica tareas terminales si se decide usar nodo final explícito.

---

## 5.3. Tabla: `action_definition`

### Propósito
Catálogo de acciones que el usuario puede ejecutar sobre una tarea.

### Columnas propuestas

| Columna | Tipo sugerido | Propósito |
|---|---|---|
| `id` | bigint / uuid | Identificador de la acción |
| `code` | varchar(100) | Código técnico, por ejemplo APPROVE, REJECT, RETURN |
| `name` | varchar(200) | Nombre visible |
| `description` | text | Descripción opcional |
| `is_active` | boolean | Habilitada o no |
| `created_at` | timestamp | Fecha de creación |
| `created_by` | varchar(100) | Usuario creador |
| `updated_at` | timestamp | Fecha de actualización |
| `updated_by` | varchar(100) | Usuario actualizador |

### Reglas sugeridas

- `code` debe ser único.
- Conviene reutilizar acciones comunes entre workflows.

---

## 5.4. Tabla: `task_definition_action`

### Propósito
Define qué acciones están permitidas para una tarea específica. Sirve para validar desde backend y exponer opciones correctas en UI.

### Columnas propuestas

| Columna | Tipo sugerido | Propósito |
|---|---|---|
| `id` | bigint / uuid | Identificador |
| `task_definition_id` | fk | Tarea a la que aplica |
| `action_definition_id` | fk | Acción permitida |
| `display_order` | integer | Orden sugerido para UI |
| `button_label` | varchar(100) | Texto de botón opcional |
| `button_style` | varchar(50) | Estilo visual opcional: primary, danger, secondary |
| `requires_comment` | boolean | Si la acción exige comentario |
| `is_active` | boolean | Habilitada o no |
| `created_at` | timestamp | Fecha de creación |
| `created_by` | varchar(100) | Usuario creador |

### Reglas sugeridas

- `task_definition_id + action_definition_id` debe ser único.
- Solo se deberían poder ejecutar acciones presentes en esta tabla.

---

## 5.5. Tabla: `task_routing`

### Propósito
Es la tabla central del motor. Define la siguiente tarea a instanciar según la tarea actual y la acción recibida. También deja preparada la posibilidad de agregar condiciones futuras.

### Columnas propuestas

| Columna | Tipo sugerido | Propósito |
|---|---|---|
| `id` | bigint / uuid | Identificador de la regla de ruteo |
| `workflow_definition_id` | fk | Redundante útil para integridad y consultas |
| `task_definition_id` | fk | Tarea origen |
| `action_definition_id` | fk | Acción disparadora |
| `next_task_definition_id` | fk nullable | Tarea destino; null si la acción finaliza el workflow |
| `condition_type` | varchar(50) nullable | Tipo de condición futura, por ejemplo EXPRESSION o JSON_RULE |
| `condition_value` | text nullable | Expresión o estructura serializada |
| `priority` | integer | Orden de evaluación, menor valor = mayor prioridad |
| `is_default` | boolean | Indica si es la ruta por defecto cuando ninguna condición cumple |
| `is_active` | boolean | Habilitada o no |
| `created_at` | timestamp | Fecha de creación |
| `created_by` | varchar(100) | Usuario creador |
| `updated_at` | timestamp | Fecha de actualización |
| `updated_by` | varchar(100) | Usuario actualizador |

### Uso esperado en esta primera versión

En la primera versión, la mayoría de filas tendrán:

- `condition_type = null`
- `condition_value = null`
- `priority = 1`
- `is_default = false`

Es decir, una relación simple:

**tarea actual + acción = siguiente tarea**

### Uso esperado a futuro

Si en una siguiente fase se requieren rutas condicionales, el motor podrá:

1. buscar todas las filas para la combinación `task_definition_id + action_definition_id`
2. ordenarlas por `priority`
3. evaluar las condiciones
4. seleccionar la primera que cumpla
5. si ninguna cumple, usar la fila `is_default = true`

### Reglas sugeridas

- Si no se usan condiciones, debería existir una única ruta por `task_definition_id + action_definition_id`.
- Si se usan condiciones, debe existir como máximo una fila `is_default = true` por `task_definition_id + action_definition_id`.
- `next_task_definition_id` debe pertenecer al mismo workflow.
- No se recomienda que `next_task_definition_id` sea una expresión; la condición debe decidir si aplica la transición, no construir dinámicamente el destino.

---

## 5.6. Tabla: `workflow_instance`

### Propósito
Representa una ejecución concreta de una definición de workflow.

### Columnas propuestas

| Columna | Tipo sugerido | Propósito |
|---|---|---|
| `id` | bigint / uuid | Identificador de la instancia |
| `workflow_definition_id` | fk | Definición usada |
| `business_key` | varchar(150) nullable | Identificador del caso de negocio |
| `status` | varchar(30) | Estado: CREATED, ACTIVE, COMPLETED, CANCELLED |
| `started_at` | timestamp | Fecha de inicio |
| `started_by` | varchar(100) | Usuario o sistema que inició |
| `completed_at` | timestamp nullable | Fecha de finalización |
| `completed_by` | varchar(100) nullable | Usuario o sistema que finalizó |
| `current_task_instance_id` | fk nullable | Referencia opcional a la tarea actual |
| `context_data` | json / text nullable | Datos del caso necesarios para evaluación o trazabilidad |
| `created_at` | timestamp | Fecha de creación |
| `updated_at` | timestamp | Fecha de actualización |

### Notas

- `business_key` permite enlazar la instancia con una entidad de negocio externa.
- `context_data` puede ayudar más adelante para condiciones o auditoría.
- `current_task_instance_id` es opcional; puede derivarse buscando la tarea pendiente actual, pero almacenarlo puede simplificar consultas.

---

## 5.7. Tabla: `task_instance`

### Propósito
Representa una tarea concreta creada dentro de una instancia de workflow.

### Columnas propuestas

| Columna | Tipo sugerido | Propósito |
|---|---|---|
| `id` | bigint / uuid | Identificador |
| `workflow_instance_id` | fk | Instancia de workflow a la que pertenece |
| `task_definition_id` | fk | Definición de tarea utilizada |
| `status` | varchar(30) | Estado: PENDING, COMPLETED, CANCELLED |
| `assigned_user_id` | varchar(100) nullable | Usuario asignado |
| `assigned_role_code` | varchar(100) nullable | Rol asignado |
| `sequence_no` | integer | Orden de creación dentro del workflow |
| `created_at` | timestamp | Fecha de creación |
| `created_by` | varchar(100) | Usuario o sistema que la generó |
| `started_at` | timestamp nullable | Fecha en que fue tomada o abierta |
| `completed_at` | timestamp nullable | Fecha de completado |
| `completed_by` | varchar(100) nullable | Usuario que la completó |
| `completion_action_id` | fk nullable | Acción con que se completó |
| `comment` | text nullable | Comentario capturado al completar |

### Reglas sugeridas

- Debe existir una sola tarea pendiente activa por workflow en esta primera versión si el flujo es estrictamente secuencial.
- `sequence_no` ayuda a ordenar historial sin depender solo de timestamps.

---

## 5.8. Tabla: `task_action_log`

### Propósito
Auditar cada acción ejecutada por el usuario sobre una tarea.

### Columnas propuestas

| Columna | Tipo sugerido | Propósito |
|---|---|---|
| `id` | bigint / uuid | Identificador |
| `workflow_instance_id` | fk | Instancia relacionada |
| `task_instance_id` | fk | Tarea sobre la que se actuó |
| `task_definition_id` | fk | Tarea definida correspondiente |
| `action_definition_id` | fk | Acción ejecutada |
| `performed_by` | varchar(100) | Usuario que ejecutó |
| `performed_at` | timestamp | Fecha y hora |
| `comment` | text nullable | Comentario ingresado |
| `routing_id` | fk nullable | Regla de ruteo seleccionada |
| `result_next_task_definition_id` | fk nullable | Tarea destino resuelta |
| `result_next_task_instance_id` | fk nullable | Tarea instancia creada |
| `request_payload` | json / text nullable | Payload recibido por el servicio |
| `evaluation_trace` | json / text nullable | Resultado de evaluación de condiciones si aplica |

### Beneficio

Permite entender después:

- qué acción ejecutó el usuario
- qué ruta resolvió el sistema
- qué tarea siguiente se creó
- con qué comentarios o payload ocurrió

---

## 6. Funcionalidades del motor

## 6.1. Crear definición de workflow

Permite registrar un workflow con su código, nombre y versión.

### Debe permitir

- crear definición en borrador
- activar o desactivar versiones
- consultar definición vigente

---

## 6.2. Registrar tareas definidas

Permite crear los nodos del flujo.

### Debe permitir

- crear tarea
- marcar tarea inicial
- marcar tarea terminal
- asociar metadatos funcionales

---

## 6.3. Registrar acciones

Permite mantener el catálogo de acciones.

### Debe permitir

- crear acción
- activar o desactivar acción
- consultar acciones por código

---

## 6.4. Asociar acciones permitidas a cada tarea

Permite determinar qué botones o acciones pueden ejecutarse sobre una tarea específica.

### Debe permitir

- asociar acción a tarea
- definir etiqueta visible del botón
- definir si requiere comentario
- consultar acciones disponibles para una tarea

---

## 6.5. Configurar ruteo

Permite definir a qué tarea se transiciona según acción.

### Debe permitir

- registrar transición simple
- indicar siguiente tarea
- configurar prioridad
- marcar ruta default
- dejar condición preparada para futuro

### Validaciones importantes

- no permitir rutas duplicadas inconsistentes
- no permitir más de un default por combinación tarea + acción
- no permitir que la tarea destino pertenezca a otro workflow

---

## 6.6. Iniciar instancia de workflow

Crea una ejecución real del flujo.

### Flujo esperado

1. recibir código o id del workflow
2. resolver versión activa
3. identificar tarea inicial
4. crear `workflow_instance`
5. crear primera `task_instance`
6. dejar el workflow en estado ACTIVE

---

## 6.7. Consultar tarea actual de una instancia

Permite a backend o UI conocer cuál es la tarea pendiente actual.

### Debe retornar

- datos de la tarea actual
- acciones disponibles
- asignación
- metadatos básicos

---

## 6.8. Completar tarea

Es la funcionalidad central del motor.

### Firma conceptual

`completeTask(taskInstanceId, actionCode, payload, performedBy, comment)`

### Validaciones mínimas

- la tarea debe existir
- la tarea debe estar en estado PENDING
- la acción debe estar permitida para la definición de tarea
- debe existir una ruta válida o definirse explícitamente que la acción finaliza el flujo

### Lógica interna sugerida

1. bloquear la tarea instancia para evitar concurrencia
2. resolver definición de tarea actual
3. resolver acción por código
4. validar que la acción exista en `task_definition_action`
5. obtener reglas candidatas de `task_routing`
6. si no hay condiciones, elegir la única ruta esperada
7. si hay condiciones, evaluarlas por prioridad
8. si ninguna cumple, usar default si existe
9. registrar en `task_action_log`
10. marcar tarea actual como COMPLETED
11. crear siguiente `task_instance` si aplica
12. actualizar `workflow_instance.current_task_instance_id`
13. si no aplica siguiente tarea, marcar workflow como COMPLETED
14. confirmar transacción

### Resultado esperado

- tarea actual completada
- siguiente tarea creada o workflow finalizado
- historial auditado

---

## 6.9. Consultar historial del workflow

Permite reconstruir el avance del caso.

### Debe mostrar

- tareas creadas
- acciones ejecutadas
- usuarios participantes
- timestamps
- comentarios
- decisiones de ruteo

---

## 7. Comportamiento de `priority` e `is_default`

## `priority`

Sirve para definir el orden de evaluación cuando existan múltiples rutas para la misma tarea y acción.

### Regla

- menor valor = mayor prioridad
- el motor debe evaluar en orden ascendente y quedarse con la primera condición válida

## `is_default`

Sirve como ruta de fallback cuando ninguna condición aplica.

### Regla

- puede existir máximo una ruta default por combinación tarea + acción
- solo se usa si ninguna otra transición condicionada resulta válida

### En la primera versión

Aunque probablemente no se usen todavía condiciones reales, conviene incluir estas columnas desde ya para dejar el modelo listo para evolucionar sin romper la estructura.

---

## 8. Recomendaciones de diseño

## 8.1. Mantener `next_task_definition_id` explícito

No se recomienda que la tarea destino sea una expresión dinámica. La lógica debe ser:

- la condición decide si aplica la transición
- la transición apunta explícitamente a la tarea destino

Esto vuelve el flujo:

- más legible
- más auditable
- más fácil de validar
- más fácil de consultar

---

## 8.2. Ejecutar todo en una sola transacción

Completar tarea, registrar historial y crear siguiente tarea debe ser atómico.

Si algo falla, no puede quedar:

- la tarea actual completada sin siguiente tarea
- o la siguiente tarea creada sin historial

---

## 8.3. Prevenir doble ejecución

Se debe contemplar protección frente a doble click o reintentos.

### Opciones

- lock transaccional sobre `task_instance`
- validar estado antes de completar
- idempotency key si el canal lo requiere

---

## 8.4. Versionar definiciones

No conviene cambiar una definición activa usada por instancias ya iniciadas.

Recomendación:

- cada cambio estructural importante genera nueva versión
- las instancias existentes siguen asociadas a la versión con la que nacieron

---

## 8.5. Separar definición e instancia

Es clave distinguir:

- lo que define cómo debe comportarse el flujo
- de lo que ocurre en una ejecución real

Por eso el modelo separa:

- `workflow_definition` y `task_definition`
- de `workflow_instance` y `task_instance`

---

## 9. Restricciones e índices sugeridos

## Restricciones sugeridas

- `workflow_definition(code, version)` único
- `task_definition(workflow_definition_id, code)` único
- `action_definition(code)` único
- `task_definition_action(task_definition_id, action_definition_id)` único
- en modo simple: `task_routing(task_definition_id, action_definition_id)` único cuando no haya condiciones
- en modo condicionado: regla de unicidad para controlar solo un default por combinación

## Índices sugeridos

- índice por `workflow_instance(status)`
- índice por `task_instance(workflow_instance_id, status)`
- índice por `task_instance(assigned_user_id, status)`
- índice por `task_routing(task_definition_id, action_definition_id, is_active)`
- índice por `task_action_log(workflow_instance_id, performed_at)`

---

## 10. Flujo de ejemplo

Supóngase el siguiente workflow:

- Tarea 1: Revisión
- Tarea 2: Corrección
- Tarea 3: Aprobado

Y las siguientes rutas:

- Revisión + Aprobar -> Aprobado
- Revisión + Devolver -> Corrección
- Revisión + Rechazar -> fin del workflow
- Corrección + Enviar -> Revisión

### Ejecución

1. Se inicia una instancia del workflow.
2. El motor crea una `task_instance` de Revisión.
3. El usuario presiona Devolver.
4. El servicio consulta `task_routing`.
5. Encuentra que Revisión + Devolver lleva a Corrección.
6. Completa la tarea actual.
7. Registra el evento en `task_action_log`.
8. Crea una nueva `task_instance` de Corrección.
9. Actualiza la instancia de workflow.

---

## 11. Posible evolución futura

Esta arquitectura permite crecer más adelante hacia:

- condiciones evaluables
- asignación automática por rol o área
- notificaciones
- SLA y vencimientos
- colas de trabajo por usuario o rol
- paralelismo limitado
- cancelación o reversiones
- visualización del grafo del workflow

La recomendación es mantener la primera implementación lo más simple posible, pero con columnas y separación conceptual suficientes para no rediseñar todo cuando aparezcan casos de uso nuevos.

---

## 12. Conclusión

La propuesta consiste en construir un workflow engine liviano, declarativo y orientado a acciones de usuario, donde la regla principal de avance sea:

**tarea actual + acción ejecutada => siguiente tarea definida**

El motor debe operar sobre una arquitectura de datos separando claramente:

- definiciones del flujo
- instancias del flujo
- reglas de ruteo
- historial de ejecución

Esto permite que el comportamiento del proceso esté gobernado por configuración y no por lógica hardcodeada, dejando además preparada la estructura para evolucionar a rutas condicionadas en una siguiente fase sin romper el modelo central.

---

## 13. Siguiente paso recomendado

Como siguiente paso, conviene convertir esta propuesta en:

1. un modelo relacional definitivo con tipos concretos según la base de datos elegida
2. un conjunto de constraints y reglas de integridad
3. pseudocódigo o contrato técnico del servicio `completeTask`
4. ejemplos de inserts de configuración para un workflow real

