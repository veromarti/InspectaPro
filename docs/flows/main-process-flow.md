# Inspection Lifecycle Flow - InspectaPro

![Inspection Lifecycle Flow](../../static/img/flows/flujo-vida-inspeccion.png)
*Figure 1: Complete inspection lifecycle flow showing Configuration, Execution, Closure, and Review phases*

## Flow Overview

The inspection lifecycle in InspectaPro goes through four main phases that strategically integrate SQL and NoSQL databases:

1. **Configuration (SQL)** - Inspection creation and preparation
2. **Execution (Hybrid)** - Field work with offline/online handling
3. **Closure and Synchronization (Hybrid)** - Dynamic data persistence
4. **Review (SQL)** - Validation and cycle closure

This flow ensures that structural data remains in SQL while dynamic execution data is stored in NoSQL.

---

## Phase 1: Configuration (SQL)

### 1.1. Administrator Login

The process begins when an **Administrator** logs into the system.

**Validation:** The system verifies credentials against the SQL database (Users and Roles tables).

**Result:** Authenticated session with administrative permissions.

### 1.2. Create New Inspection

The administrator creates a new inspection in the system.

**SQL Operation:**
```sql
INSERT INTO Inspections (company_id, created_by, created_at, status)
VALUES (?, ?, NOW(), 'DRAFT')
```

**Initial state:** `DRAFT`

The inspection is being configured and is not yet visible to the technician.

### 1.3. Select Inspection Type

The administrator selects the inspection type that defines the base structure.

**Critical data:** The inspection type defines:
- Form structure
- Required fields
- Document sections
- Applicable checklists

**Storage:**
- **SQL:** Reference to inspection type (inspection_type_id)
- **NoSQL:** JSON template of the form associated with the type

### 1.4. Assign Responsible Technician

The administrator assigns a technician who will be responsible for executing the inspection.

**SQL Operation:**
```sql
UPDATE Inspections 
SET assigned_technician_id = ?, status = 'ASSIGNED'
WHERE inspection_id = ?
```

**State change:** `DRAFT` → `ASSIGNED`

**Result:** The technician can now see the inspection in their mobile application.

### 1.5. Define Date and Client

The administrator completes the configuration by defining:
- Scheduled execution date
- Client or site to inspect
- Initial observations

**SQL Storage:**
```sql
UPDATE Inspections 
SET scheduled_date = ?, client_name = ?, client_location = ?
WHERE inspection_id = ?
```

### 1.6. System Generates SQL Record

The system consolidates all structural information in the SQL database.

**Entities involved:**
- Inspections (main record)
- InspectionTypes (reference)
- Users (assigned technician)
- Companies (owning company)

**Phase final state:** `ASSIGNED` (Visible in technician's app, awaiting execution)

---

## Phase 2: Execution (Hybrid)

### 2.1. Technician Receives Notification

The technician receives a notification on their mobile device.

**Mechanism:** Push notification or application synchronization.

**Communicated data:**
- Assigned inspection
- Client
- Scheduled date
- Inspection type

### 2.2. Technician Opens App in Field

The technician opens the mobile application at the inspection location.

**Critical decision point:** Is there internet connection?

#### Decision: Connection Available?

##### Yes → Download JSON Template

If there is connection, the system downloads the most recent template from NoSQL.

**NoSQL Operation:**
```javascript
db.inspection_templates.findOne({ 
  inspection_type_id: "type_x", 
  version: "latest" 
})
```

**Downloaded data:**
- Form structure
- Questions
- Required fields
- Response options

**Advantage:** The technician works with the most updated version of the form.

##### No → Use Local Cache (Offline)

If there is no connection, the system uses the last version stored locally.

**Mechanism:** Local database (SQLite or IndexedDB) on the mobile device.

**Offline-first strategy:**
- The app allows working completely without connection
- Changes are saved locally
- Synchronization occurs when connection is restored

**State during offline execution:** `IN_PROGRESS`

### 2.3. Technician Fills Dynamic Form

The technician completes the inspection form.

**Generated JSON document structure:**

```json
{
  "inspection_id": "INS-2026-00123",
  "inspection_type": "Preventive Maintenance",
  "version": "2.1",
  "executed_by": "TEC-456",
  "execution_date": "2026-02-17T10:30:00Z",
  "responses": {
    "text_fields": [
      { "question": "General observations", "answer": "Equipment in good condition" }
    ],
    "checklists": [
      { "item": "Does the engine work?", "checked": true },
      { "item": "Are there leaks?", "checked": false }
    ],
    "numeric_fields": [
      { "metric": "Temperature", "value": 72, "unit": "°C" }
    ]
  },
  "photos": [
    { "url": "s3://bucket/photo1.jpg", "description": "General view" }
  ],
  "evidences": [
    { "type": "signature", "data": "base64..." }
  ]
}
```

**Document characteristics:**
- **Flexible:** Supports multiple response types
- **Versioned:** Includes form version number
- **Complete:** Contains all execution information

### 2.4. Validate Required Fields

Before finalizing, the system validates that:
- All required fields are complete
- Data formats are correct
- Required evidence is attached

**Operation:** Client-side validation (mobile application)

**If information is missing:** The system does not allow progress until required fields are completed.

---

## Phase 3: Closure and Synchronization (Hybrid)

### 3.1. Technician Completes Inspection

The technician marks the inspection as completed in the application.

**User action:** Presses "Complete inspection" button

**If internet connection available:**
→ Proceeds to immediate synchronization

**If NO internet connection:**
→ State changes to `SYNCING` (pending)
→ Data remains in local cache
→ Synchronization will occur automatically when connection is restored

### 3.2. System Saves JSON Document (NoSQL)

Once connected, the system saves the complete inspection document in the NoSQL database.

**NoSQL Operation:**
```javascript
db.inspection_results.insertOne({
  inspection_id: "INS-2026-00123",
  document: { /* complete content */ },
  synced_at: ISODate("2026-02-17T14:45:00Z"),
  device_id: "DEVICE-789"
})
```

**Result:** Dynamic data is persisted in NoSQL.

### 3.3. System Updates State to COMPLETED (SQL)

In parallel, the system updates the structural record in SQL.

**SQL Operation:**
```sql
UPDATE Inspections 
SET status = 'COMPLETED', 
    completed_at = NOW(),
    synced = TRUE
WHERE inspection_id = ?
```

**State change:** `IN_PROGRESS` → `COMPLETED`

### 3.4. Link SQL ID ↔ NoSQL ID

The system establishes the connection between both databases.

**Linking strategy:**
- The NoSQL document includes the SQL `inspection_id`
- The SQL record can optionally reference the NoSQL `document_id`

**Relationship example:**

SQL:
```
inspection_id: INS-2026-00123
status: COMPLETED
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

**Result:** Structural (SQL) and dynamic (NoSQL) data are connected through shared identifiers.

---

## Phase 4: Review (SQL)

### 4.1. Supervisor Reviews Results

A **supervisor** (user with review permissions) accesses the system to validate the inspection.

**Queried data:**
- **SQL:** State, dates, responsible technician, client
- **NoSQL:** Complete document of responses and evidence

**Visualization:** The system combines information from both sources to present a unified view.

### 4.2. Decision: Approved?

The supervisor analyzes the results and makes a critical decision.

#### Yes → Final State: CLOSED/AUDITED

If the inspection meets all requirements:

**SQL Operation:**
```sql
UPDATE Inspections 
SET status = 'CLOSED', 
    reviewed_by = ?,
    reviewed_at = NOW(),
    approval_status = 'APPROVED'
WHERE inspection_id = ?
```

**Final state:** `CLOSED/AUDITED`

**Consequence:** 
- The inspection is immutable (read-only)
- Recorded in history
- Can be consulted for audit purposes

#### No → State: REJECTED/OBSERVED

If the inspection requires corrections:

**SQL Operation:**
```sql
UPDATE Inspections 
SET status = 'REJECTED', 
    reviewed_by = ?,
    reviewed_at = NOW(),
    rejection_reason = ?
WHERE inspection_id = ?
```

**State:** `REJECTED/OBSERVED`

### 4.3. Notify Technician for Correction

If the inspection was rejected, the system notifies the technician.

**Notification content:**
- Rejection reason
- Fields or sections requiring correction
- Re-execution deadline

**Return flow:**

The technician returns to **Phase 2 (Execution)** to:
- Review supervisor observations
- Correct incorrect data
- Resubmit the inspection

**State change:** `REJECTED` → `IN_PROGRESS` (upon reopening)

---

## Cycle End

Once the inspection reaches the `CLOSED/AUDITED` state, the lifecycle ends.

**Result:**
- Structural data preserved in SQL
- Dynamic data preserved in NoSQL
- Complete process traceability
- Immutable history for audits

---

## SQL ↔ NoSQL Interaction Summary

| Phase | SQL | NoSQL |
|------|-----|-------|
| Configuration | ✅ Creates record, assigns technician, defines state | ❌ Does not intervene |
| Execution | ✅ Queries metadata, validates permissions | ✅ Provides JSON template, stores offline cache |
| Synchronization | ✅ Updates state to COMPLETED | ✅ Saves complete response document |
| Review | ✅ Updates final state, records approval/rejection | ✅ Queried for response visualization |

---

## Key Decisions in the Flow

### Decision 1: Connection Available?
- **Yes** → Downloads updated template from NoSQL
- **No** → Uses local cache (offline mode)

### Decision 2: Fields Valid?
- **Yes** → Allows completing inspection
- **No** → Blocks completion until required fields are filled

### Decision 3: Internet Available Upon Completion?
- **Yes** → Synchronizes immediately
- **No** → SYNCING state (deferred synchronization)

### Decision 4: Approved by Supervisor?
- **Yes** → Final state CLOSED/AUDITED
- **No** → REJECTED state, notifies technician

---

## State Traceability

Throughout the flow, the system maintains historical record of state transitions:

**SQL Table: inspection_state_history**
```sql
| inspection_id | old_state    | new_state    | changed_by | changed_at          |
|---------------|--------------|--------------|------------|---------------------|
| INS-123       | NULL         | DRAFT        | ADM-01     | 2026-02-14 09:00:00 |
| INS-123       | DRAFT        | ASSIGNED     | ADM-01     | 2026-02-14 09:15:00 |
| INS-123       | ASSIGNED     | IN_PROGRESS  | TEC-05     | 2026-02-15 10:30:00 |
| INS-123       | IN_PROGRESS  | COMPLETED    | TEC-05     | 2026-02-15 14:45:00 |
| INS-123       | COMPLETED    | CLOSED       | SUP-02     | 2026-02-16 08:20:00 |
```

**Traceability advantages:**
- Complete change audit
- Identification of responsible parties
- Execution time analysis
- Regulatory compliance
