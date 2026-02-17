# Functional Coherence: Use Cases, Flow, and Behavior

## Introduction

This document demonstrates the **total coherence** between:
- **Use Cases** (what actors want to do)
- **Process Flow** (how the system executes it)
- **States** (lifecycle control)
- **Architecture** (where data is stored)
- **Behavior** (validations and business rules)

### Reference Diagrams

![State Diagram](../../static/img/flows/diagrama-estados-inspeccion.png)
*Figure 1: Inspection State Diagram*

![Lifecycle Flow](../../static/img/flows/flujo-vida-inspeccion.png)
*Figure 2: Inspection Lifecycle Flow - InspectaPro*

As **Functional Analyst**, it is my responsibility to ensure there are no gaps between these components.

---

## Complete Coherence Matrix

| # | Use Case | Actor | Flow Phase | Initial State | Final State | SQL | NoSQL | Critical Validation |
|---|-------------|-------|----------------|----------------|--------------|-----|-------|--------------------|
| 1 | Create new inspection | Administrator | Configuration → 1.2 | - | `DRAFT` | ✅ | ❌ | Valid company FK |
| 2 | Select inspection type | Administrator | Configuration → 1.3 | `DRAFT` | `DRAFT` | ✅ | ⚠️ Template ref. | Type must exist |
| 3 | Assign responsible technician | Administrator | Configuration → 1.4 | `DRAFT` | `ASSIGNED` | ✅ | ❌ | Valid and active technician |
| 4 | Definir fecha y cliente | Administrador | Configuración → 1.5 | `BORRADOR` / `ASIGNADA` | `ASIGNADA` | ✅ | ❌ | Fecha ≥ hoy |
| 5 | Ver inspecciones asignadas | Técnico | Ejecución → 2.1 | `ASIGNADA` | `ASIGNADA` | ✅ | ❌ | Solo sus inspecciones |
| 6 | Abrir inspección (online) | Técnico | Ejecución → 2.2 (Sí) | `ASIGNADA` | `EN_PROGRESO` | ✅ | ✅ Plantilla | Solo técnico asignado |
| 7 | Abrir inspección (offline) | Técnico | Ejecución → 2.2 (No) | `ASIGNADA` | `EN_PROGRESO` | ✅ | ⚠️ Caché | Plantilla en caché |
| 8 | Llenar formulario dinámico | Técnico | Ejecución → 2.3 | `EN_PROGRESO` | `EN_PROGRESO` | - | ⚠️ Local | Auto-guardado cada 30s |
| 9 | Finalizar inspección (online) | Técnico | Cierre → 3.1, 3.2 | `EN_PROGRESO` | `POR_REVISAR` | ✅ | ✅ | Campos obligatorios completos |
| 10 | Finalizar inspección (offline) | Técnico | Cierre → 3.1 | `EN_PROGRESO` | `SINCRONIZANDO` | ✅ | ⚠️ Pendiente | Campos obligatorios completos |
| 11 | Sincronización automática | Sistema | Cierre → 3.2, 3.3, 3.4 | `SINCRONIZANDO` | `POR_REVISAR` | ✅ | ✅ | Conexión restablecida |
| 12 | Ver inspecciones por revisar | Supervisor | Revisión → 4.1 | `POR_REVISAR` | `POR_REVISAR` | ✅ | ✅ | Permisos supervisor |
| 13 | Aprobar inspección | Supervisor | Revisión → 4.2 (Sí) | `POR_REVISAR` | `APROBADA` | ✅ | ✅ Locked | No auto-aprobación |
| 14 | Rechazar inspección | Supervisor | Revisión → 4.2 (No), 4.3 | `POR_REVISAR` | `RECHAZADA` | ✅ | ✅ | Motivo obligatorio |
| 15 | Corregir inspección rechazada | Técnico | Ejecución → 2.3 | `RECHAZADA` | `EN_PROGRESO` | ✅ | ✅ | Solo técnico asignado |
| 16 | Consultar historial | Admin/Auditor | N/A | `APROBADA` | `APROBADA` | ✅ | ✅ | Solo lectura |

---

## Detailed Use Cases

### Use Case 1: Create New Inspection

#### Primary Actor
**Company Administrator**

#### Description
The administrator creates a new inspection in the system that will later be assigned to a technician for execution.

#### Preconditions
- The administrator is authenticated
- The company has an active subscription
- Inspection types are defined

#### Basic Flow
1. Administrador selecciona "Nueva Inspección"
2. Sistema muestra formulario de creación
3. Administrador ingresa datos básicos
4. Sistema valida datos (FK de empresa)
5. Sistema crea registro en SQL con estado `BORRADOR`
6. Sistema muestra confirmación

#### Corresponding Flow
**Configuration → Phase 1, Step 1.2**

#### Involved States
- **Initial state:** None (inspection does not exist)
- **Resulting state:** `DRAFT`

#### Arquitectura de Datos

**SQL:**
```sql
INSERT INTO inspections (
  id, 
  company_id, 
  created_by, 
  status, 
  created_at
) VALUES (
  'INS-2026-00123',
  ?,
  ?,
  'BORRADOR',
  NOW()
)
```

**NoSQL:**
No interviene en esta fase.

#### Implemented Validations
| Validation | Type | Description |
|------------|------|-------------|
| Company exists | Referential integrity | FK `company_id` must exist |
| Authorized user | Permissions | Only `ADMIN` can create |
| User belongs to company | Multi-tenant | Data isolation |
| Unique ID | Uniqueness | Prevents duplicates |

#### Postconditions
- Inspection created with unique ID
- Initial state `DRAFT`
- Record in `inspection_state_history` table
- Only visible to administrator

#### Verified Coherence ✅
- ✅ Use case ↔ Flow (Configuration)
- ✅ Flow ↔ State (`DRAFT`)
- ✅ State ↔ Architecture (SQL only)
- ✅ Architecture ↔ Validations (FK + permissions)

---

### Caso de Uso 2: Asignar Técnico Responsable

#### Actor Principal
**Administrador de la empresa**

#### Descripción
El administrador asigna un técnico específico que será responsable de ejecutar la inspección en campo.

#### Precondiciones
- Inspección en estado `BORRADOR`
- Técnico existe y está activo
- Técnico pertenece a la misma empresa

#### Flujo Básico
1. Administrador abre inspección en `BORRADOR`
2. Sistema muestra lista de técnicos disponibles
3. Administrador selecciona técnico
4. Administrador define fecha programada
5. Sistema valida técnico (existencia, rol, empresa)
6. Sistema actualiza inspección a estado `ASIGNADA`
7. Sistema envía notificación al técnico

#### Flujo Correspondiente
**Configuración → Fase 1, Paso 1.4**

#### Transición de Estado
```
BORRADOR → ASIGNADA
```

#### Arquitectura de Datos

**SQL:**
```sql
UPDATE inspections 
SET assigned_technician_id = ?,
    status = 'ASIGNADA',
    assigned_at = NOW(),
    scheduled_date = ?
WHERE id = ? AND status = 'BORRADOR'
```

**Registro de transición:**
```sql
INSERT INTO inspection_state_history 
  (inspection_id, old_state, new_state, changed_by, notes)
VALUES 
  ('INS-123', 'BORRADOR', 'ASIGNADA', ?, 'Asignada a técnico TEC-456')
```

#### Validaciones Implementadas
| Validación | SQL | Mensaje de Error |
|------------|-----|------------------|
| Técnico existe | `JOIN users WHERE id = ?` | "Técnico no encontrado" |
| Rol correcto | `JOIN users WHERE role = 'TECHNICIAN'` | "Usuario no es técnico" |
| Misma empresa | `JOIN users WHERE company_id = ?` | "Técnico no pertenece a esta empresa" |
| Estado válido | `WHERE status = 'BORRADOR'` | "Inspección ya fue asignada" |
| Fecha futura | `WHERE scheduled_date >= CURDATE()` | "La fecha debe ser hoy o posterior" |

#### Postcondiciones
- Inspección en estado `ASIGNADA`
- Técnico puede ver la inspección en su app
- Notificación enviada al técnico
- Registro en historial de transiciones

#### Comportamiento Observable
- El técnico recibe push notification: "Nueva inspección asignada"
- La inspección aparece en lista "Pendientes" del técnico
- El administrador ve cambio de estado en el dashboard

#### Coherencia Verificada ✅
- ✅ Caso de uso ↔ Flujo (Configuración → Asignación)
- ✅ Flujo ↔ Transición (`BORRADOR` → `ASIGNADA`)
- ✅ Transición ↔ Validaciones (técnico válido)
- ✅ Validaciones ↔ Comportamiento (notificación enviada)

---

### Caso de Uso 3: Ejecutar Inspección en Campo (con Internet)

#### Actor Principal
**Técnico**

#### Descripción
El técnico abre la inspección en el lugar de trabajo, con conexión a internet disponible, y completa el formulario dinámico.

#### Precondiciones
- Inspección en estado `ASIGNADA` o `RECHAZADA`
- Técnico autenticado
- Dispositivo con conexión a internet
- Permisos de ubicación otorgados (opcional)

#### Flujo Básico
1. Técnico abre app móvil
2. Sistema consulta inspecciones asignadas (SQL)
3. Técnico selecciona inspección
4. Sistema actualiza estado a `EN_PROGRESO` (SQL)
5. Sistema descarga plantilla JSON actualizada (NoSQL)
6. Sistema muestra formulario dinámico
7. Técnico completa campos, checkllistas, fotos
8. Sistema guarda progreso cada 30 segundos
9. Técnico presiona "Finalizar"
10. Sistema valida campos obligatorios
11. Sistema guarda documento en NoSQL
12. Sistema actualiza estado a `POR_REVISAR` (SQL)
13. Sistema vincula IDs SQL ↔ NoSQL

#### Flujo Correspondiente
**Ejecución → Fase 2, Pasos 2.1, 2.2 (Sí), 2.3**  
**Cierre → Fase 3, Pasos 3.2, 3.3, 3.4**

#### Transiciones de Estado
```
ASIGNADA → EN_PROGRESO → POR_REVISAR
```

#### Arquitectura de Datos

**SQL (al abrir):**
```sql
UPDATE inspections 
SET status = 'EN_PROGRESO', 
    started_at = NOW()
WHERE id = ? AND assigned_technician_id = ? AND status = 'ASIGNADA'
```

**NoSQL (descarga de plantilla):**
```javascript
db.inspection_templates.findOne({
  inspection_type_id: "tipo_mantenimiento",
  version: "latest"
})
```

**Resultado (ejemplo):**
```json
{
  "inspection_type_id": "tipo_mantenimiento",
  "version": "2.1",
  "sections": [
    {
      "title": "Verificación de Equipos",
      "fields": [
        {
          "id": "motor_funciona",
          "type": "boolean",
          "label": "¿Funciona el motor?",
          "required": true
        },
        {
          "id": "temperatura",
          "type": "number",
          "label": "Temperatura (°C)",
          "required": true,
          "min": -20,
          "max": 150
        },
        {
          "id": "observaciones",
          "type": "text",
          "label": "Observaciones",
          "required": false,
          "maxLength": 500
        }
      ]
    }
  ]
}
```

**NoSQL (al finalizar):**
```javascript
db.inspection_results.insertOne({
  inspection_id: "INS-2026-00123",
  inspection_type: "Mantenimiento Preventivo",
  version: "2.1",
  executed_by: "TEC-456",
  execution_date: ISODate("2026-02-17T10:30:00Z"),
  device_info: {
    device_id: "DEVICE-789",
    os: "Android 13",
    app_version: "1.5.0"
  },
  responses: {
    motor_funciona: true,
    temperatura: 72,
    observaciones: "Equipo en buen estado general"
  },
  photos: [
    {
      "url": "s3://inspectapro-media/INS-123/foto1.jpg",
      "description": "Vista general del equipo",
      "timestamp": ISODate("2026-02-17T10:45:00Z"),
      "geolocation": { lat: 6.2442, lon: -75.5812 }
    }
  ],
  synced_at: ISODate("2026-02-17T14:50:00Z")
})
```

**SQL (al finalizar):**
```sql
UPDATE inspections 
SET status = 'POR_REVISAR',
    completed_at = NOW(),
    synced = TRUE,
    nosql_document_id = '65d9f8a3c4b1e2d3a4567890'
WHERE id = 'INS-2026-00123'
```

#### Validaciones Implementadas

**Al abrir:**
| Validación | Ubicación | Descripción |
|------------|-----------|-------------|
| Solo técnico asignado | SQL WHERE | Previene acceso no autorizado |
| Estado válido | SQL WHERE | Solo `ASIGNADA` o `RECHAZADA` |
| Suscripción activa | SQL JOIN | Empresa debe tener suscripción |

**Durante ejecución:**
| Validación | Ubicación | Descripción |
|------------|-----------|-------------|
| Tipo de dato correcto | App móvil | Número solo acepta numéricos |
| Rango válido | App móvil | Temperatura entre -20 y 150 |
| Longitud máxima | App móvil | Observaciones ≤ 500 caracteres |
| Formato de foto | App móvil | Solo JPG/PNG, máx 5MB |

**Al finalizar:**
| Validación | Ubicación | Descripción |
|------------|-----------|-------------|
| Campos obligatorios completos | App móvil | Todos los `required: true` |
| Al menos una foto (si aplicable) | App móvil | Según tipo de inspección |
| Firmas capturadas | App móvil | Si el tipo lo requiere |

#### Postcondiciones
- Inspección en estado `POR_REVISAR`
- Documento JSON completo en NoSQL
- IDs vinculados entre SQL y NoSQL
- Supervisor puede revisar la inspección

#### Comportamiento Observable
- Formulario se carga dinámicamente
- Progreso guardado automáticamente (spinner)
- Botón "Finalizar" deshabilitado hasta completar campos obligatorios
- Confirmación visual: "Inspección enviada correctamente"
- La inspección desaparece de "Pendientes" del técnico

#### Coherencia Verificada ✅
- ✅ Caso de uso ↔ Flujo (Ejecución + Cierre)
- ✅ Flujo ↔ Estados (`ASIGNADA` → `EN_PROGRESO` → `POR_REVISAR`)
- ✅ Estados ↔ Arquitectura (SQL para estado, NoSQL para contenido)
- ✅ Arquitectura ↔ Vinculación (IDs compartidos)
- ✅ Validaciones ↔ Comportamiento (bloqueo hasta completar campos)

---

### Caso de Uso 4: Ejecutar Inspección Offline

#### Actor Principal
**Técnico**

#### Descripción
El técnico trabaja en un lugar sin cobertura de internet. El sistema permite trabajo completo offline con sincronización diferida.

#### Precondiciones
- Inspección en estado `ASIGNADA`
- Plantilla de formulario previamente descargada en caché
- App móvil instalada con permisos de almacenamiento

#### Flujo Básico
1. Técnico abre app (sin internet)
2. Sistema detecta ausencia de conectividad
3. Sistema muestra ícono de "Modo Offline"
4. Sistema carga plantilla desde caché local
5. Técnico completa formulario
6. Sistema guarda respuestas en base de datos local
7. Técnico presiona "Finalizar"
8. Sistema valida campos obligatorios
9. Sistema actualiza estado local a `SINCRONIZANDO`
10. Sistema muestra: "Se sincronizará cuando haya conexión"
11. **[Tiempo después]** Dispositivo se conecta a internet
12. Sistema detecta conexión automáticamente
13. Sistema sincroniza datos al servidor
14. Sistema limpia caché local

#### Flujo Correspondiente
**Ejecución → Fase 2, Pasos 2.2 (No), 2.3**  
**Cierre → Fase 3, Paso 3.1 (offline) → Pasos 3.2, 3.3, 3.4 (cuando hay conexión)**

#### Transiciones de Estado
```
ASIGNADA → EN_PROGRESO (local) → SINCRONIZANDO (local) 
       → [conexión restablecida] → POR_REVISAR (servidor)
```

#### Arquitectura de Datos

**Caché Local (IndexedDB / SQLite):**
```javascript
localDB.inspection_cache.put({
  inspection_id: "INS-2026-00123",
  status: "SINCRONIZANDO",
  offline_started_at: "2026-02-17T10:30:00",
  offline_completed_at: "2026-02-17T14:45:00",
  responses: {
    motor_funciona: true,
    temperatura: 72,
    observaciones: "Trabajando sin conexión"
  },
  photos_base64: [
    "data:image/jpeg;base64,/9j/4AAQSkZJRgABAQAAAQ..."
  ],
  pending_sync: true,
  sync_attempts: 0
})
```

**Después de sincronización:**
1. Datos se envían a NoSQL (servidor)
2. Estado se actualiza en SQL (servidor)
3. Caché local se limpia

#### Validaciones Implementadas

**Antes de permitir offline:**
| Validación | Ubicación | Descripción |
|------------|-----------|-------------|
| Plantilla en caché | App móvil | Debe existir versión local |
| Espacio disponible | App móvil | Al menos 50MB libres |
| Permisos de almacenamiento | App móvil | Usuario debe otorgar permisos |

**Durante sincronización:**
| Validación | Ubicación | Descripción |
|------------|-----------|-------------|
| Inspección aún asignada | Servidor SQL | No fue reasignada mientras offline |
| Técnico aún válido | Servidor SQL | Cuenta no fue desactivada |
| Suscripción activa | Servidor SQL | Empresa no suspendió servicio |

#### Postcondiciones
- Datos sincronizados al servidor
- Inspección en estado `POR_REVISAR`
- Caché local limpiado
- Notificación de sincronización exitosa

#### Comportamiento Observable: Comportamiento Observable
- **Durante offline:**
  - Ícono de "sin conexión" visible
  - Banner: "Trabajando sin internet - Los datos se guardan localmente"
  - Progreso guardado localmente cada 10 segundos
  - Al finalizar: "Inspección guardada. Se enviará cuando haya conexión"
  
- **Durante sincronización:**
  - Barra de progreso: "Sincronizando inspección INS-123... 45%"
  - Al completar: "Sincronización exitosa ✓"

#### Manejo de Errores
| Error | Acción del Sistema |
|-------|---------------------|
| Sin espacio en dispositivo | Muestra alerta: "Libera espacio para continuar" |
| Sincronización fallida | Reintenta automáticamente en 5 minutos |
| Conflicto de versión | Notifica que la inspección fue modificada |

#### Coherencia Verificada ✅
- ✅ Caso de uso ↔ Flujo (Ejecución offline + Sincronización diferida)
- ✅ Flujo ↔ Estado (`SINCRONIZANDO` temporal)
- ✅ Estado ↔ Arquitectura (Caché local → Servidor)
- ✅ Arquitectura ↔ Validaciones (Plantilla en caché requerida)
- ✅ Validaciones ↔ Comportamiento (Indicadores visuales de offline)

---

### Caso de Uso 5: Aprobar Inspección

#### Actor Principal
**Supervisor**

#### Descripción
El supervisor revisa los resultados de la inspección y la aprueba porque cumple con todos los requisitos de calidad.

#### Precondiciones
- Inspección en estado `POR_REVISAR`
- Supervisor autenticado
- Supervisor pertenece a la misma empresa

#### Flujo Básico
1. Supervisor accede a "Inspecciones Pendientes"
2. Sistema muestra lista filtrada por estado `POR_REVISAR` (SQL)
3. Supervisor selecciona inspección
4. Sistema carga metadatos (SQL) y contenido completo (NoSQL)
5. Supervisor revisa respuestas, fotos y evidencias
6. Supervisor presiona "Aprobar"
7. Sistema solicita confirmación (opcional: agregar nota)
8. Supervisor confirma
9. Sistema valida permisos
10. Sistema actualiza estado a `APROBADA` (SQL)
11. Sistema bloquea documento (NoSQL)
12. Sistema registra en historial
13. Sistema muestra confirmación

#### Flujo Correspondiente
**Revisión → Fase 4, Pasos 4.1, 4.2 (Sí)**

#### Transición de Estado
```
POR_REVISAR → APROBADA (estado final)
```

#### Arquitectura de Datos

**SQL (consulta inicial):**
```sql
SELECT i.id, i.scheduled_date, i.completed_at, 
       u.name AS technician_name, 
       c.name AS client_name,
       i.nosql_document_id
FROM inspections i
JOIN users u ON i.assigned_technician_id = u.id
JOIN companies c ON i.company_id = c.id
WHERE i.status = 'POR_REVISAR' 
  AND i.company_id = ? -- multi-tenant isolation
ORDER BY i.completed_at ASC
```

**NoSQL (consulta de contenido):**
```javascript
db.inspection_results.findOne({
  inspection_id: "INS-2026-00123"
})
```

**SQL (aprobación):**
```sql
UPDATE inspections 
SET status = 'APROBADA',
    reviewed_by = ?,
    reviewed_at = NOW(),
    approval_notes = ?
WHERE id = ? AND status = 'POR_REVISAR'
```

**Registro de auditoría:**
```sql
INSERT INTO inspection_state_history 
  (inspection_id, old_state, new_state, changed_by, notes)
VALUES 
  ('INS-123', 'POR_REVISAR', 'APROBADA', 'SUP-02', 'Aprobada sin observaciones')
```

**NoSQL (bloqueo de documento):**
```javascript
db.inspection_results.updateOne(
  { inspection_id: "INS-2026-00123" },
  { 
    $set: { 
      locked: true,
      approved_at: ISODate(),
      approved_by: "SUP-02"
    }
  }
)
```

#### Validaciones Implementadas
| Validación | Tipo | Mensaje de Error |
|------------|------|------------------|
| Usuario tiene rol SUPERVISOR/ADMIN | Permisos | "No tienes permisos para aprobar inspecciones" |
| Supervisor pertenece a la empresa | Multi-tenant | "No puedes aprobar inspecciones de otra empresa" |
| Inspección está en `POR_REVISAR` | Estado | "Esta inspección no está pendiente de revisión" |
| Supervisor ≠ Técnico | Conflicto de interés | "No puedes aprobar tu propia inspección" |

#### Postcondiciones
- Inspección en estado final `APROBADA`
- Documento NoSQL bloqueado (inmutable)
- Registro completo en historial de auditoría
- Inspección disponible solo para consulta

#### Comportamiento Observable
- La inspección desaparece de "Pendientes de Revisión"
- Aparece en "Historial - Aprobadas" con ícono verde ✓
- Técnico recibe notificación: "Inspección INS-123 aprobada"
- Documento es read-only (botones de edición ocultos)

#### Coherencia Verificada ✅
- ✅ Caso de uso ↔ Flujo (Revisión → Decisión Sí)
- ✅ Flujo ↔ Estado (`POR_REVISAR` → `APROBADA`)
- ✅ Estado ↔ Arquitectura (SQL + NoSQL bloqueado)
- ✅ Arquitectura ↔ Auditoría (Registro en historial)
- ✅ Validaciones ↔ Comportamiento (No auto-aprobación)

---

### Caso de Uso 6: Rechazar Inspección

#### Actor Principal
**Supervisor**

#### Descripción
El supervisor identifica errores, información faltante o incumplimiento de estándares, y devuelve la inspección al técnico para corrección.

#### Precondiciones
- Inspección en estado `POR_REVISAR`
- Supervisor autenticado
- Motivo de rechazo definido

#### Flujo Básico
1. Supervisor revisa inspección (igual que caso de uso 5)
2. Supervisor identifica problemas
3. Supervisor presiona "Rechazar"
4. Sistema muestra formulario de rechazo:
   - **Motivo** (obligatorio, texto libre)
   - **Campos a corregir** (checklist)
   - **Severidad** (Menor, Moderada, Crítica)
5. Supervisor completa formulario
6. Sistema valida que el motivo esté presente
7. Sistema actualiza estado a `RECHAZADA` (SQL)
8. Sistema registra motivo de rechazo
9. Sistema envía notificación al técnico
10. Sistema muestra confirmación

#### Flujo Correspondiente
**Revisión → Fase 4, Pasos 4.2 (No), 4.3**

#### Transición de Estado
```
POR_REVISAR → RECHAZADA
```

**Flujo de retorno permitido:**
```
RECHAZADA → EN_PROGRESO (técnico reabre para corregir)
```

#### Arquitectura de Datos

**SQL:**
```sql
UPDATE inspections 
SET status = 'RECHAZADA',
    reviewed_by = ?,
    rejected_at = NOW(),
    rejection_reason = ?,
    rejection_severity = ?
WHERE id = ? AND status = 'POR_REVISAR'
```

**Tabla de observaciones (opcional):**
```sql
INSERT INTO inspection_rejections 
  (inspection_id, rejected_by, reason, fields_to_correct)
VALUES 
  ('INS-123', 'SUP-02', 'Temperatura fuera de rango esperado', '["temperatura", "observaciones"]')
```

**NoSQL:**
El documento permanece sin cambios (no se bloquea).

#### Validaciones Implementadas
| Validación | Obligatoriedad | Descripción |
|------------|----------------|-------------|
| Motivo de rechazo | ✅ Obligatorio | Mínimo 20 caracteres |
| Campos a corregir | ⚠️ Recomendado | Ayuda al técnico a identificar problemas |
| Severidad | ⚠️ Recomendado | Prioriza el trabajo del técnico |

#### Postcondiciones
- Inspección en estado `RECHAZADA`
- Técnico notificado con motivo específico
- Registro en historial de transiciones
- Inspección vuelve a ser editable para el técnico

#### Comportamiento Observable
- Técnico recibe notificación push: "Inspección INS-123 requiere corrección"
- Al abrir la inspección, el técnico ve banner rojo:
  ```
  ⚠️ Inspección rechazada por Supervisor Juan Pérez
  Motivo: Temperatura fuera de rango esperado
  Campos a corregir: Temperatura, Observaciones
  ```
- Los campos señalados se resaltan en amarillo
- Botón "Reenviar correcciones" habilitado

#### Coherencia Verificada ✅
- ✅ Caso de uso ↔ Flujo (Revisión → Decisión No)
- ✅ Flujo ↔ Estado (`POR_REVISAR` → `RECHAZADA`)
- ✅ Estado ↔ Ciclo de retorno (`RECHAZADA` → `EN_PROGRESO`)
- ✅ Arquitectura ↔ Validaciones (Motivo obligatorio)
- ✅ Validaciones ↔ Notificación (Técnico informado)

---

## Coherencia con Diagrama de Estados

Todos los casos de uso respetan las transiciones definidas en el diagrama de estados finitos:

```
                    Administrativo crea la solicitud
                              |
                              ↓
                        [BORRADOR]
         La inspección se está configurando
                              |
                   Se asigna técnico y fecha
                              ↓
                        [ASIGNADA]
              Visible en la App del técnico
                              |
                   Técnico inicia llenado
                              ↓
                      [EN_PROGRESO]
              El formulario está abierto
                              |
          ┌───────────────────┴───────────────────┐
          |                                       |
  Técnico finaliza                    Técnico finaliza
  (sin internet)                      (con internet)
          |                                       |
          ↓                                       ↓
  [SINCRONIZANDO]                        [POR_REVISAR]
          |                                       |
   Conexión restablecida                         |
          |                                       |
          └───────────────────┬───────────────────┘
                              ↓
                      [POR_REVISAR]
             Supervisor valida la calidad
                              |
                   ┌──────────┴──────────┐
                   |                     |
            Todo correcto          Faltan datos
                   |                     |
                   ↓                     ↓
             [APROBADA]             [RECHAZADA]
            (Estado final)                |
                                          |
                               Se devuelve al técnico
                                          |
                                          ↓
                                   [EN_PROGRESO]
                                   (ciclo se repite)
```

**Verificación:**
- ✅ No existen transiciones "mágicas" que salten estados
- ✅ Cada transición tiene un caso de uso asociado
- ✅ Cada caso de uso resulta en una transición válida
- ✅ El ciclo de retorno (`RECHAZADA` → `EN_PROGRESO`) está documentado

---

## Coherencia con Arquitectura Híbrida

### Separación Clara de Responsabilidades

| Tipo de Dato | Almacenamiento | Casos de Uso Involucrados |
|--------------|----------------|---------------------------|
| Empresa, Usuarios, Roles | SQL | Todos (validación de permisos) |
| Estado de inspección | SQL | 1, 2, 3, 4, 9, 10, 11, 13, 14, 15 |
| Técnico asignado | SQL | 2, 5, 6, 7 |
| Fechas (creación, asignación, ejecución, revisión) | SQL | 1, 2, 4, 9, 13, 14 |
| Plantilla de formulario (versionada) | NoSQL | 6, 7 |
| Respuestas del técnico | NoSQL | 8, 9, 10 |
| Fotos y evidencias | NoSQL (referencia) + S3 (archivo) | 8, 12, 13 |
| Historial de transiciones | SQL | Todos (auditoría) |

### Verificación de Consistencia

**Principio:** Cada caso de uso accede solo a los datos necesarios en la capa correcta.

**Ejemplos:**
- ✅ Caso de uso "Crear inspección" → Solo SQL (correcto)
- ✅ Caso de uso "Llenar formulario" → Híbrido (correcto: plantilla en NoSQL, estado en SQL)
- ✅ Caso de uso "Aprobar inspección" → Híbrido (correcto: metadatos SQL, contenido NoSQL)
- ❌ Hipotético "Crear empresa" → NoSQL únicamente (incorrecto, debe ser SQL)

---

## Coherencia con Requerimientos de Negocio

| Requerimiento de Negocio | Implementación en Casos de Uso |
|--------------------------|--------------------------------|
| "Los técnicos deben poder trabajar sin internet" | Caso de uso 4: Ejecución offline con sincronización diferida |
| "Los formularios pueden cambiar con el tiempo sin afectar el sistema" | Caso de uso 6, 7: Plantillas versionadas en NoSQL |
| "Las inspecciones aprobadas no se pueden modificar" | Caso de uso 13: Documento bloqueado en NoSQL después de aprobación |
| "Debe haber trazabilidad de quién hizo qué y cuándo" | Todos los casos de uso: Registro en `inspection_state_history` |
| "Cada empresa solo ve sus propias inspecciones" | Todos los casos de uso: Validación `WHERE company_id = ?` en SQL |
| "Los supervisores pueden devolver inspecciones incorrectas" | Caso de uso 14, 15: Estado `RECHAZADA` con ciclo de retorno |
| "Las inspecciones deben poder incluir fotos y evidencias" | Caso de uso 8: Captura de multimedia en formulario dinámico |
| "El sistema debe funcionar en dispositivos móviles" | Caso de uso 4: Caché local en IndexedDB/SQLite |

**Resultado:** Todos los requerimientos de negocio están cubiertos por casos de uso específicos.

---

## Verificación Final de Coherencia

### ✅ Casos de Uso → Flujo de Proceso
Cada caso de uso corresponde a uno o más pasos específicos del flujo de vida de la inspección.

### ✅ Flujo de Proceso → Estados
Cada paso del flujo resulta en un estado específico o transición de estado controlada.

### ✅ Estados → Transiciones
Cada estado tiene transiciones válidas definidas y bloqueadas.

### ✅ Transiciones → Arquitectura
Cada transición se refleja en cambios en SQL y/o NoSQL según corresponda.

### ✅ Arquitectura → Validaciones
Cada operación en SQL o NoSQL tiene validaciones de negocio implementadas.

### ✅ Validaciones → Comportamiento Observable
Cada validación resulta en un comportamiento visible y comprensible para el usuario.

### ✅ Comportamiento → Casos de Uso
El comportamiento observable cumple con las expectativas definidas en los casos de uso.

**El ciclo de coherencia está completo.**

---

## Conclusión: Sistema Funcionalmente Coherente

Como **Analista Funcional** (rol MONTERROSA), confirmo que el sistema InspectaPro presenta **coherencia total** entre todos sus componentes:

1. **No existen brechas** entre casos de uso y flujo de proceso
2. **No existen estados huérfanos** sin casos de uso asociados
3. **No existen datos desconectados** entre SQL y NoSQL
4. **No existen transiciones inválidas** no controladas
5. **No existen validaciones inconsistentes** entre capas
6. **No existen comportamientos inesperados** no documentados

**Todos los componentes están alineados:**
- Negocio ← correctamente interpretado
- Casos de uso ← mapean al flujo de proceso
- Flujo de proceso ← controla transiciones de estado
- Estados ← persisten en arquitectura híbrida
- Arquitectura ← soporta requerimientos de negocio
- Validaciones ← garantizan integridad operativa
- Comportamiento ← cumple expectativas del usuario

Este análisis cumple con la responsabilidad de **MONTERROSA** de asegurar coherencia entre uso y comportamiento del sistema.
