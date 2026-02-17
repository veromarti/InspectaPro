# Flujo de Vida de una Inspección - InspectaPro

![Flujo de Vida de una Inspección](../../static/img/flows/flujo-vida-inspeccion.png)
*Figura 1: Flujo completo de vida de una inspección mostrando las fases de Configuración, Ejecución, Cierre y Revisión*

## Visión General del Flujo

El ciclo de vida de una inspección en InspectaPro atraviesa cuatro fases principales que integran de manera estratégica las bases de datos SQL y NoSQL:

1. **Configuración (SQL)** - Creación y preparación de la inspección
2. **Ejecución (Híbrido)** - Trabajo en campo con manejo offline/online
3. **Cierre y Sincronización (Híbrido)** - Persistencia de datos dinámicos
4. **Revisión (SQL)** - Validación y cierre del ciclo

Este flujo garantiza que los datos estructurales permanezcan en SQL mientras que los datos dinámicos de ejecución se almacenen en NoSQL.

---

## Fase 1: Configuración (SQL)

### 1.1. Inicio de Sesión del Administrador

El proceso comienza cuando un **Administrador** inicia sesión en el sistema.

**Validación:** El sistema verifica credenciales contra la base de datos SQL (tabla Users y Roles).

**Resultado:** Sesión autenticada con permisos administrativos.

### 1.2. Crear Nueva Inspección

El administrador crea una nueva inspección en el sistema.

**Operación SQL:**
```sql
INSERT INTO Inspections (company_id, created_by, created_at, status)
VALUES (?, ?, NOW(), 'BORRADOR')
```

**Estado inicial:** `BORRADOR`

La inspección se está configurando y aún no es visible para el técnico.

### 1.3. Seleccionar Tipo de Inspección

El administrador selecciona el tipo de inspección que define la estructura base.

**Dato crítico:** El tipo de inspección define:
- Estructura de formulario
- Campos obligatorios
- Secciones del documento
- Checklists aplicables

**Almacenamiento:**
- **SQL:** Referencia al tipo de inspección (inspection_type_id)
- **NoSQL:** Plantilla JSON del formulario asociado al tipo

### 1.4. Asignar Técnico Responsable

El administrador asigna un técnico que será responsable de ejecutar la inspección.

**Operación SQL:**
```sql
UPDATE Inspections 
SET assigned_technician_id = ?, status = 'ASIGNADA'
WHERE inspection_id = ?
```

**Cambio de estado:** `BORRADOR` → `ASIGNADA`

**Resultado:** El técnico ahora puede ver la inspección en su aplicación móvil.

### 1.5. Definir Fecha y Cliente

El administrador completa la configuración definiendo:
- Fecha programada de ejecución
- Cliente o sitio a inspeccionar
- Observaciones iniciales

**Almacenamiento SQL:**
```sql
UPDATE Inspections 
SET scheduled_date = ?, client_name = ?, client_location = ?
WHERE inspection_id = ?
```

### 1.6. Sistema Genera Registro en SQL

El sistema consolida toda la información estructural en la base de datos SQL.

**Entidades involucradas:**
- Inspections (registro principal)
- InspectionTypes (referencia)
- Users (técnico asignado)
- Companies (empresa propietaria)

**Estado final de la fase:** `ASIGNADA` (Visible en la App del técnico, esperando ejecución)

---

## Fase 2: Ejecución (Híbrido)

### 2.1. Técnico Recibe Notificación

El técnico recibe una notificación en su dispositivo móvil.

**Mecanismo:** Push notification o sincronización de la aplicación.

**Datos comunicados:**
- Inspección asignada
- Cliente
- Fecha programada
- Tipo de inspección

### 2.2. Técnico Abre la App en Campo

El técnico abre la aplicación móvil en el lugar de la inspección.

**Punto de decisión crítico:** ¿Existe conexión a internet?

#### Decisión: ¿Existe conexión?

##### Sí → Descargar Plantilla JSON

Si hay conexión, el sistema descarga la plantilla más reciente desde NoSQL.

**Operación NoSQL:**
```javascript
db.inspection_templates.findOne({ 
  inspection_type_id: "tipo_x", 
  version: "latest" 
})
```

**Datos descargados:**
- Estructura del formulario
- Preguntas
- Campos obligatorios
- Opciones de respuesta

**Ventaja:** El técnico trabaja con la versión más actualizada del formulario.

##### No → Usar Caché Local (Offline)

Si no hay conexión, el sistema utiliza la última versión almacenada localmente.

**Mecanismo:** Base de datos local (SQLite o IndexedDB) en el dispositivo móvil.

**Estrategia offline-first:**
- La app permite trabajar completamente sin conexión
- Los cambios se guardan localmente
- La sincronización ocurre cuando se restablece la conexión

**Estado durante ejecución offline:** `EN_PROGRESO`

### 2.3. Técnico Llena el Formulario Dinámico

El técnico completa el formulario de inspección.

**Estructura del documento JSON generado:**

```json
{
  "inspection_id": "INS-2026-00123",
  "inspection_type": "Mantenimiento Preventivo",
  "version": "2.1",
  "executed_by": "TEC-456",
  "execution_date": "2026-02-17T10:30:00Z",
  "responses": {
    "text_fields": [
      { "question": "Observaciones generales", "answer": "Equipo en buen estado" }
    ],
    "checklists": [
      { "item": "¿Funciona el motor?", "checked": true },
      { "item": "¿Hay fugas?", "checked": false }
    ],
    "numeric_fields": [
      { "metric": "Temperatura", "value": 72, "unit": "°C" }
    ]
  },
  "photos": [
    { "url": "s3://bucket/foto1.jpg", "description": "Vista general" }
  ],
  "evidences": [
    { "type": "firma", "data": "base64..." }
  ]
}
```

**Características del documento:**
- **Flexible:** Soporta múltiples tipos de respuesta
- **Versionado:** Incluye número de versión del formulario
- **Completo:** Contiene toda la información de la ejecución

### 2.4. Validar Campos Obligatorios

Antes de finalizar, el sistema valida que:
- Todos los campos obligatorios estén completos
- Los formatos de datos sean correctos
- Las evidencias requeridas estén adjuntas

**Operación:** Validación en cliente (aplicación móvil)

**Si falta información:** El sistema no permite avanzar hasta completar los campos requeridos.

---

## Fase 3: Cierre y Sincronización (Híbrido)

### 3.1. Técnico Finaliza Inspección

El técnico marca la inspección como finalizada en la aplicación.

**Acción del usuario:** Presiona botón "Finalizar inspección"

**Si hay conexión internet:**
→ Procede a sincronización inmediata

**Si NO hay conexión internet:**
→ El estado cambia a `SINCRONIZANDO` (pendiente)
→ Los datos permanecen en caché local
→ La sincronización ocurrirá automáticamente cuando se restablezca la conexión

### 3.2. Sistema Guarda el Documento JSON (NoSQL)

Una vez conectado, el sistema guarda el documento completo de inspección en la base de datos NoSQL.

**Operación NoSQL:**
```javascript
db.inspection_results.insertOne({
  inspection_id: "INS-2026-00123",
  document: { /* contenido completo */ },
  synced_at: ISODate("2026-02-17T14:45:00Z"),
  device_id: "DEVICE-789"
})
```

**Resultado:** Los datos dinámicos están persistidos en NoSQL.

### 3.3. Sistema Actualiza Estado a COMPLETADA (SQL)

Paralelamente, el sistema actualiza el registro estructural en SQL.

**Operación SQL:**
```sql
UPDATE Inspections 
SET status = 'COMPLETADA', 
    completed_at = NOW(),
    synced = TRUE
WHERE inspection_id = ?
```

**Cambio de estado:** `EN_PROGRESO` → `COMPLETADA`

### 3.4. Vincular ID SQL ↔ ID NoSQL

El sistema establece la conexión entre ambas bases de datos.

**Estrategia de vinculación:**
- El documento NoSQL incluye el `inspection_id` de SQL
- El registro SQL puede opcionalmente referenciar el `document_id` de NoSQL

**Ejemplo de relación:**

SQL:
```
inspection_id: INS-2026-00123
status: COMPLETADA
nosql_document_id: "65d9f8a3c4b1e2d3a4567890"
```

NoSQL:
```json
{
  "_id": "65d9f8a3c4b1e2d3a4567890",
  "inspection_id": "INS-2026-00123",
  "document": { ... }
}
```

**Resultado:** Los datos estructurales (SQL) y dinámicos (NoSQL) están conectados mediante identificadores compartidos.

---

## Fase 4: Revisión (SQL)

### 4.1. Supervisor Revisa Resultados

Un **supervisor** (usuario con permisos de revisión) accede al sistema para validar la inspección.

**Datos consultados:**
- **SQL:** Estado, fechas, técnico responsable, cliente
- **NoSQL:** Documento completo de respuestas y evidencias

**Visualización:** El sistema combina información de ambas fuentes para presentar una vista unificada.

### 4.2. Decisión: ¿Aprobada?

El supervisor analiza los resultados y toma una decisión crítica.

#### Sí → Estado Final: CERRADA/AUDITADA

Si la inspección cumple con todos los requisitos:

**Operación SQL:**
```sql
UPDATE Inspections 
SET status = 'CERRADA', 
    reviewed_by = ?,
    reviewed_at = NOW(),
    approval_status = 'APROBADA'
WHERE inspection_id = ?
```

**Estado final:** `CERRADA/AUDITADA`

**Consecuencia:** 
- La inspección es inmutable (read-only)
- Se registra en el historial
- Puede ser consultada con fines de auditoría

#### No → Estado: RECHAZADA/OBSERVADA

Si la inspección requiere correcciones:

**Operación SQL:**
```sql
UPDATE Inspections 
SET status = 'RECHAZADA', 
    reviewed_by = ?,
    reviewed_at = NOW(),
    rejection_reason = ?
WHERE inspection_id = ?
```

**Estado:** `RECHAZADA/OBSERVADA`

### 4.3. Notificar al Técnico para Corrección

Si la inspección fue rechazada, el sistema notifica al técnico.

**Contenido de la notificación:**
- Motivo del rechazo
- Campos o secciones que requieren corrección
- Plazo para re-ejecución

**Flujo de retorno:**

El técnico vuelve a la **Fase 2 (Ejecución)** para:
- Revisar observaciones del supervisor
- Corregir los datos incorrectos
- Volver a enviar la inspección

**Cambio de estado:** `RECHAZADA` → `EN_PROGRESO` (al reabrir)

---

## Fin del Ciclo

Una vez que la inspección alcanza el estado `CERRADA/AUDITADA`, el ciclo de vida finaliza.

**Resultado:**
- Datos estructurales conservados en SQL
- Datos dinámicos preservados en NoSQL
- Trazabilidad completa del proceso
- Historial inmutable para auditorías

---

## Resumen de Interacción SQL ↔ NoSQL

| Fase | SQL | NoSQL |
|------|-----|-------|
| Configuración | ✅ Crea registro, asigna técnico, define estado | ❌ No interviene |
| Ejecución | ✅ Consulta metadatos, valida permisos | ✅ Proporciona plantilla JSON, almacena caché offline |
| Sincronización | ✅ Actualiza estado a COMPLETADA | ✅ Guarda documento completo de respuestas |
| Revisión | ✅ Actualiza estado final, registra aprobación/rechazo | ✅ Consultado para visualización de respuestas |

---

## Decisiones Clave en el Flujo

### Decisión 1: ¿Existe conexión?
- **Sí** → Descarga plantilla actualizada desde NoSQL
- **No** → Usa caché local (modo offline)

### Decisión 2: ¿Campos válidos?
- **Sí** → Permite finalizar inspección
- **No** → Bloquea finalización hasta completar campos obligatorios

### Decisión 3: ¿Internet disponible al finalizar?
- **Sí** → Sincroniza inmediatamente
- **No** → Estado SINCRONIZANDO (sincronización diferida)

### Decisión 4: ¿Aprobada por supervisor?
- **Sí** → Estado final CERRADA/AUDITADA
- **No** → Estado RECHAZADA, notifica al técnico

---

## Trazabilidad del Estado

Durante todo el flujo, el sistema mantiene registro histórico de transiciones de estado:

**Tabla SQL: inspection_state_history**
```sql
| inspection_id | old_state    | new_state    | changed_by | changed_at          |
|---------------|--------------|--------------|------------|---------------------|
| INS-123       | NULL         | BORRADOR     | ADM-01     | 2026-02-14 09:00:00 |
| INS-123       | BORRADOR     | ASIGNADA     | ADM-01     | 2026-02-14 09:15:00 |
| INS-123       | ASIGNADA     | EN_PROGRESO  | TEC-05     | 2026-02-15 10:30:00 |
| INS-123       | EN_PROGRESO  | COMPLETADA   | TEC-05     | 2026-02-15 14:45:00 |
| INS-123       | COMPLETADA   | CERRADA      | SUP-02     | 2026-02-16 08:20:00 |
```

**Ventajas de la trazabilidad:**
- Auditoría completa de cambios
- Identificación de responsables
- Análisis de tiempos de ejecución
- Cumplimiento normativo
