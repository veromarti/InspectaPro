# Inspection States, Decisions, and Transitions

## 1. Lifecycle Overview

Each inspection in InspectaPro follows a structured lifecycle through a **finite state machine**.

The system ensures that inspections move through controlled stages, preventing inconsistencies and ensuring operational integrity.

![Inspection State Diagram](../../static/img/flows/diagrama-estados-inspeccion.png)
*Figure 1: Inspection State Diagram showing system transitions and decisions*

The process is based on **defined states** and **controlled transitions** between them, with **decision points** that determine the process flow.

---

## 2. Inspection States

The InspectaPro system handles **7 main states** during the inspection lifecycle:

### Defined States

| State | Code | Description |
|--------|--------|-------------|
| **DRAFT** | `DRAFT` | The inspection is being configured. Not yet visible to the technician. |
| **ASSIGNED** | `ASSIGNED` | The inspection has been assigned to a technician. Visible in the technician's app, awaiting execution. |
| **IN_PROGRESS** | `IN_PROGRESS` | The technician is actively filling out the form. |
| **SYNCING** | `SYNCING` | Data saved locally, waiting for connection to sync with the server. |
| **UNDER_REVIEW** | `UNDER_REVIEW` | The supervisor is validating the data quality. |
| **APPROVED** | `APPROVED` | Everything is correct. The inspection has been approved and is ready for closure. |
| **REJECTED** | `REJECTED` | Data is missing or there are errors. Returned to the technician for correction. |

---

## 3. Detailed State Description

### 🔹 DRAFT

**Definition:**  
The inspection is being configured. Not yet visible to the technician.

**Who has access?**  
Only the **Administrator** creating the inspection.

**What can the user do?**
- Select inspection type
- Define scheduled date
- Assign client or site
- Modify any field

**What can they NOT do?**
- The technician cannot see this inspection yet
- Cannot be executed

**Stored data:**
- **SQL:** Initial record with `DRAFT` state
- **NoSQL:** No data yet (only type reference exists)

**Next transition:**  
→ `ASSIGNED` (when a technician and date are assigned)

---

### 🔹 ASSIGNED

**Definition:**  
Visible in the technician's app. Awaiting execution.

**Who has access?**
- **Administrator:** Can view and modify
- **Assigned technician:** Can view, cannot edit administrative details

**What can the technician do?**
- View inspection details
- Check client location
- Review inspection type
- Start execution when ready

**What can they NOT do?**
- Cannot change the date
- Cannot reassign to another technician
- Cannot modify the inspection type

**Stored data:**
- **SQL:** State `ASSIGNED`, assigned technician, scheduled date

**Next transition:**  
→ `IN_PROGRESS` (when the technician starts filling out the form)

---

### 🔹 IN_PROGRESS

**Definition:**  
The form is open. Temporary changes are being saved.

**Who has access?**
- **Assigned technician:** Full control over the form
- **Administrator:** View only (cannot edit)

**What can the technician do?**
- Fill out text responses
- Complete checklists
- Capture numeric values
- Attach photos and evidence
- Save partial progress
- Work **offline** (without connection)

**Temporary storage:**
- **Local cache:** Data is saved on the technician's device
- **Deferred synchronization:** If there's no internet, data waits for connection

**When does this state end?**  
When the technician presses the **"Complete inspection"** button.

**Critical validation:**
- The system validates that all **required fields** are complete
- If information is missing → **Does not allow progress**

**Possible transitions:**  
→ `SYNCING` (if completed without internet)  
→ `UNDER_REVIEW` (if completed with internet and syncs correctly)

---

### 🔹 SYNCING

**Definition:**  
Data locally stored, waiting for connection to send to server.

**Why does this state exist?**  
To handle **offline scenarios**. The technician can complete the inspection without internet and the system syncs automatically when connection is restored.

**Who has access?**
- **Technician:** Can see it's pending synchronization
- **Administrator:** Sees state as "Pending synchronization"

**What does the system do automatically?**
- Detects when the device connects to internet
- Sends the complete JSON document to **NoSQL**
- Updates the state in **SQL**
- Links IDs between both databases

**Operations performed during synchronization:**

1. **Save to NoSQL:**
```javascript
db.inspection_results.insertOne({
  inspection_id: "INS-123",
  document: { /* complete responses */ },
  synced_at: ISODate()
})
```

2. **Update in SQL:**
```sql
UPDATE Inspections 
SET status = 'UNDER_REVIEW', synced = TRUE
WHERE inspection_id = 'INS-123'
```

**Next transition:**  
→ `UNDER_REVIEW` (after successful synchronization)

---

### 🔹 UNDER_REVIEW

**Definition:**  
Supervisor validates the data quality.

**Who has access?**
- **Supervisor/Administrator:** Reviews complete results
- **Technician:** Can only view (can no longer edit)

**What does the supervisor analyze?**
- Completeness of required fields
- Response coherence
- Quality of photographic evidence
- Compliance with company standards

**Queried data:**
- **SQL:** Metadata (dates, technician, client)
- **NoSQL:** Complete response document

**Critical decision point:**  
The supervisor must make an **approval decision**.

**Possible transitions:**  
→ `APPROVED` (if everything is correct)  
→ `REJECTED` (if data is missing or there are errors)

---

### 🔹 APPROVED

**Definition:**  
Everything is correct. The inspection meets requirements.

**Who has access?**
- **All roles:** Read-only
- **No one can modify:** The document is immutable

**What does the system record?**
- Approval date
- User who approved
- Approval reason (optional)

**SQL Operation:**
```sql
UPDATE Inspections 
SET status = 'APPROVED', 
    reviewed_by = ?,
    reviewed_at = NOW(),
    approval_notes = ?
WHERE inspection_id = ?
```

**Next transition:**  
→ End of cycle (Historical)

**Subsequent use:**
- Historical queries
- Audit reports
- Compliance analysis

---

### 🔹 REJECTED

**Definition:**  
Data is missing or there are errors. Returned to the technician for correction.

**Who has access?**
- **Supervisor:** Has identified problems
- **Technician:** Receives rejection notification

**What information is communicated to the technician?**
- Rejection reason
- Specific fields requiring correction
- Supervisor observations
- Resubmission deadline (optional)

**SQL Operation:**
```sql
UPDATE Inspections 
SET status = 'REJECTED', 
    reviewed_by = ?,
    rejection_reason = ?,
    rejected_at = NOW()
WHERE inspection_id = ?
```

**What can the technician do?**
- Review observations
- Reopen the form
- Correct the indicated data
- Resubmit the inspection

**Next transition:**  
→ `IN_PROGRESS` (when the technician reopens the inspection to correct)

**Return flow:**  
The cycle returns through:  
`IN_PROGRESS` → `SYNCING` → `UNDER_REVIEW` → `APPROVED`

---

## 4. State Transition Diagram

```
                    Administrator creates the request
                              |
                              ↓
                          [DRAFT]
         The inspection is being configured.
        Not yet visible to the technician.
                              |
                   Technician and date assigned
                              ↓
                         [ASSIGNED]
              Visible in the technician's app.
                    Awaiting execution.
                              |
                   Technician starts filling
                              ↓
                      [IN_PROGRESS]
              The form is open.
           Temporary changes are being saved.
                              |
          ┌───────────────────┴───────────────────┐
          |                                       |
  Technician completes                Technician completes
  (without internet)                     (with internet)
          |                                       |
          ↓                                       |
     [SYNCING]                                    |
  Data locally stored,                            |
  waiting for connection.                         |
          |                                       |
   Connection restored                            |
          |                                       |
          └───────────────────┬───────────────────┘
                              ↓
                      [UNDER_REVIEW]
             Supervisor validates quality.
                              |
                   ┌──────────┴──────────┐
                   |                     |
            Everything OK           Data missing
                   |                     |
                   ↓                     ↓
             [APPROVED]             [REJECTED]
                   |                     |
                   |          Returned to technician
                   |                     |
                   |                     └────────┐
                   |                              |
            End of cycle                          ↓
             (Historical)                   [IN_PROGRESS]
                                           (cycle repeats)
```

---

## 5. System Decision Points

### 🎯 Decision 1: Is there internet connection?

**Context:** The technician opens the app in the field.

**Options:**
- **Yes** → Downloads updated JSON template from NoSQL
- **No** → Uses local cache (offline mode)

**Impact:** Determines if the technician works with the most recent form version or a previously stored version.

---

### 🎯 Decision 2: Are required fields complete?

**Context:** The technician attempts to complete the inspection.

**Validation:**
```javascript
if (requiredFields.some(field => !field.value)) {
  showError("You must complete all required fields");
  blockCompletion();
}
```

**Options:**
- **Yes** → Allows completing inspection
- **No** → Blocks completion, shows missing fields

**Impact:** Ensures minimum data quality before sending to review.

---

### 🎯 Decision 3: Is there internet when completing?

**Context:** The technician has completed all fields and presses "Complete".

**Options:**
- **Yes** → Syncs immediately → State `UNDER_REVIEW`
- **No** → State `SYNCING` (deferred synchronization)

**Impact:** Allows working completely offline without data loss.

---

### 🎯 Decision 4: Approved by supervisor?

**Context:** The supervisor reviews the inspection.

**Evaluation criteria:**
- Information completeness
- Evidence quality
- Response coherence
- Standard compliance

**Options:**
- **Yes** → State `APPROVED` → End of cycle
- **No** → State `REJECTED` → Notifies technician

**Impact:** Determines if the inspection is closed or returns to execution.

---

## 6. Transition Rules

State transitions are controlled by the system to ensure process consistency.

### Mandatory Rules

1. **An inspection CANNOT move to `ASSIGNED` without first being in `DRAFT`**
   - Validation: Must have record with `DRAFT` state

2. **An inspection CANNOT move to `IN_PROGRESS` without being `ASSIGNED`**
   - Validation: Must have assigned technician and defined date

3. **An inspection CANNOT move to `UNDER_REVIEW` without completing required fields**
   - Validation: Required fields checklist at 100%

4. **An inspection CANNOT be `APPROVED` without going through `UNDER_REVIEW`**
   - Validation: Must have review record

5. **A `REJECTED` inspection must return to `IN_PROGRESS` before being approved**
   - Validation: Cannot approve directly from `REJECTED`

6. **An `APPROVED` inspection is immutable**
   - Validation: No state change is allowed after approval

### Allowed Transitions Table

| Current State | Allowed Transitions | Responsible Actor |
|---------------|-------------------------|-------------------|
| `DRAFT` | → `ASSIGNED` | Administrator |
| `ASSIGNED` | → `IN_PROGRESS` | Technician |
| `IN_PROGRESS` | → `SYNCING`, `UNDER_REVIEW` | Technician (upon completion) |
| `SYNCING` | → `UNDER_REVIEW` | System (automatic) |
| `UNDER_REVIEW` | → `APPROVED`, `REJECTED` | Supervisor |
| `REJECTED` | → `IN_PROGRESS` | Technician (upon reopening) |
| `APPROVED` | ❌ No transitions | N/A (final state) |

---

## 7. State Traceability

Each state transition is recorded in a history table.

### SQL Table: `inspection_state_history`

```sql
CREATE TABLE inspection_state_history (
  id INT PRIMARY KEY AUTO_INCREMENT,
  inspection_id VARCHAR(50) NOT NULL,
  old_state VARCHAR(20),
  new_state VARCHAR(20) NOT NULL,
  changed_by INT NOT NULL,
  changed_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  notes TEXT,
  FOREIGN KEY (inspection_id) REFERENCES inspections(id),
  FOREIGN KEY (changed_by) REFERENCES users(id)
);
```

### Historical Record Example

| inspection_id | old_state | new_state | changed_by | changed_at | notes |
|---------------|-----------|-----------|------------|------------|-------|
| INS-123 | NULL | DRAFT | ADM-01 | 2026-02-14 09:00:00 | Inspection created |
| INS-123 | DRAFT | ASSIGNED | ADM-01 | 2026-02-14 09:15:00 | Assigned to TEC-05 |
| INS-123 | ASSIGNED | IN_PROGRESS | TEC-05 | 2026-02-15 10:30:00 | Technician started execution |
| INS-123 | IN_PROGRESS | SYNCING | TEC-05 | 2026-02-15 14:45:00 | Completed without connection |
| INS-123 | SYNCING | UNDER_REVIEW | SYSTEM | 2026-02-15 15:10:00 | Successful synchronization |
| INS-123 | UNDER_REVIEW | APPROVED | SUP-02 | 2026-02-16 08:20:00 | Inspection approved |

**Traceability advantages:**
- Complete change audit
- Identification of responsible parties per transition
- Execution time analysis per phase
- Regulatory compliance
- Conflict resolution

---

## 8. Process Integrity

The controlled lifecycle model ensures:

### ✅ Clear Operational Responsibility
Each state has a defined responsible party:
- `DRAFT`, `ASSIGNED` → Administrator
- `IN_PROGRESS`, `SYNCING` → Technician
- `UNDER_REVIEW`, `APPROVED`, `REJECTED` → Supervisor

### ✅ Structured Validation
Transitions only occur when specific conditions are met.

### ✅ Prevention of Unauthorized Changes
A technician cannot approve their own inspection.
An administrator cannot modify an inspection in progress.

### ✅ Decision Traceability
Each state change is historically recorded with who, when, and why.

---

## 9. Consistency with Hybrid Architecture

State management is directly connected to the SQL + NoSQL architecture:

| State | SQL | NoSQL |
|--------|-----|-------|
| `DRAFT` | ✅ Record created | ❌ No data |
| `ASSIGNED` | ✅ Technician assigned | ❌ Only type reference |
| `IN_PROGRESS` | ✅ State updated | ⚠️ Local cache on device |
| `SYNCING` | ✅ Temporary state | ⚠️ Pending persistence |
| `UNDER_REVIEW` | ✅ Review state | ✅ Complete JSON document |
| `APPROVED` | ✅ Final state | ✅ Immutable document |
| `REJECTED` | ✅ Temporary state | ✅ Queryable document |

**Conclusion:**  
The state model ensures that the system maintains:
- **Structural consistency in SQL**
- **Operational flexibility in NoSQL**
- **Complete lifecycle traceability**

This design is fundamental to ensuring inspection reliability and business accountability.
