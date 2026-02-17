# Functional Impact of Hybrid Architecture on InspectaPro

## Introduction

The hybrid architecture of InspectaPro is not an arbitrary technological decision, but a **direct response to the functional needs of the business**.

The system combines:
- **SQL (Relational database)** → Structural control, governance, and traceability
- **NoSQL (Document database)** → Operational flexibility, form evolution, and scalability

This document explains how the hybrid architecture functionally impacts each phase of the inspection lifecycle.

---

## 1. Separation of Functional Responsibilities

The hybrid architecture directly influences how system responsibilities are distributed.

### SQL Governs Structure and Control

**Responsibilities:**
- Management of companies, users, and roles
- Control of inspection lifecycle
- State transitions
- Technician assignment
- Change traceability
- Audit and regulatory compliance

**Why SQL:**
These entities require:
- Stable structure
- Referential integrity
- ACID transactions
- Complex relational queries
- Normalization to avoid redundancies

### NoSQL Manages Dynamic Execution

**Responsibilities:**
- Storage of inspection responses
- Dynamic and versioned forms
- Evidence (references to photos, signatures)
- Variable data structures per inspection type
- Offline cache on mobile devices

**Why NoSQL:**
Execution data is:
- Heterogeneous (each inspection type is different)
- Evolutionary (forms change over time)
- Nested (complex JSON structures)
- Flexible (no fixed schema)

---

## 2. Impact on Each Lifecycle Phase

### Phase 1: Configuration (SQL)

**Functional operation:**  
The administrator creates and configures an inspection.

**Managed data:**
- Owning company
- Assigned technician
- Inspection type (reference)
- Scheduled date
- Initial state: `DRAFT`

**Involved architecture:**  
**SQL only.**

**Why:**  
In this phase, only **structural and relational** data is managed:
- The inspection must belong to a company (FK: `company_id`)
- Must have an assigned technician (FK: `technician_id`)
- Must reference a valid type (FK: `inspection_type_id`)

**Typical SQL query:**
```sql
INSERT INTO inspections (company_id, inspection_type_id, assigned_technician_id, status, scheduled_date)
VALUES (?, ?, ?, 'DRAFT', ?);
```

**Functional advantage:**  
- Guaranteed referential integrity
- Impossible to assign an inspection to a non-existent technician
- Automatic validation of permissions by company

---

### Phase 2: Execution (Hybrid)

**Functional operation:**  
The technician fills out the inspection form in the field.

**Critical point:** This phase combines **SQL and NoSQL**.

#### 2.1. SQL Controls the State

When the technician opens the app:

```sql
SELECT i.id, i.inspection_type_id, i.scheduled_date, i.status, it.name
FROM inspections i
JOIN inspection_types it ON i.inspection_type_id = it.id
WHERE i.assigned_technician_id = ? AND i.status = 'ASSIGNED'
```

**Result:** List of pending inspections.

#### 2.2. NoSQL Provides the Dynamic Form

Once the technician selects an inspection:

```javascript
db.inspection_templates.findOne({
  inspection_type_id: "maintenance_type",
  version: "2.1"
})
```

**Result:** JSON document with form structure:

```json
{
  "inspection_type_id": "maintenance_type",
  "version": "2.1",
  "sections": [
    {
      "title": "Equipment Verification",
      "fields": [
        { "id": "engine_works", "type": "boolean", "label": "Does the engine work?", "required": true },
        { "id": "temperature", "type": "number", "label": "Temperature (°C)", "required": true },
        { "id": "observations", "type": "text", "label": "Observations", "required": false }
      ]
    }
  ]
}
```

#### 2.3. Local Cache for Offline Work

If there is no connection, the system uses a local copy of the form.

**Storage:**  
IndexedDB or SQLite on the mobile device.

**Deferred synchronization:**  
Data is saved locally and sent to the server when connection is restored.

**State during offline:** `SYNCING`

**Why hybrid:**
- **SQL:** Maintains the `IN_PROGRESS` state (lifecycle control)
- **NoSQL:** Temporarily stores responses (structure flexibility)

**Functional advantage:**
- The technician can work without internet
- Data is not lost
- Synchronization is automatic

---

### Phase 3: Closure and Synchronization (Critical Hybrid)

**Functional operation:**  
The technician completes the inspection and the system synchronizes the data.

**Data flow:**

#### Step 1: Local Validation

Before sending, the system validates:
```javascript
const requiredFields = form.fields.filter(f => f.required);
if (requiredFields.some(field => !field.value)) {
  throw new Error("Required fields are missing");
}
```

#### Step 2: Save to NoSQL

```javascript
db.inspection_results.insertOne({
  inspection_id: "INS-2026-00123",
  inspection_type: "Preventive Maintenance",
  version: "2.1",
  executed_by: "TEC-456",
  execution_date: ISODate("2026-02-17T10:30:00Z"),
  responses: {
    engine_works: true,
    temperature: 72,
    observations: "Equipment in good condition"
  },
  photos: [
    { url: "s3://bucket/photo1.jpg", description: "General view" }
  ]
})
```

#### Step 3: Update in SQL

```sql
UPDATE inspections 
SET status = 'UNDER_REVIEW', 
    completed_at = NOW(),
    synced = TRUE,
    nosql_document_id = '65d9f8a3c4b1e2d3a4567890'
WHERE inspection_id = 'INS-2026-00123'
```

#### Step 4: ID Linking

**Connection strategy:**

| Database | Field | Value |
|---------------|-------|-------|
| SQL | `inspection_id` | `INS-2026-00123` |
| SQL | `nosql_document_id` | `65d9f8a3c4b1e2d3a4567890` |
| NoSQL | `inspection_id` | `INS-2026-00123` (reference to SQL) |
| NoSQL | `_id` | `65d9f8a3c4b1e2d3a4567890` |

**Why hybrid:**
- **SQL:** Maintains reference and state (traceability)
- **NoSQL:** Stores complete content (flexibility)

**Functional advantage:**
- Fast queries by state → SQL
- Full content queries → NoSQL
- Both databases are synchronized via shared IDs

---

### Phase 4: Review (Hybrid Query)

**Functional operation:**  
The supervisor reviews inspection results.

#### Query Scenario

**SQL Query (Metadata):**
```sql
SELECT i.id, i.status, i.completed_at, u.name AS technician, c.name AS company
FROM inspections i
JOIN users u ON i.assigned_technician_id = u.id
JOIN companies c ON i.company_id = c.id
WHERE i.status = 'UNDER_REVIEW'
ORDER BY i.completed_at DESC
```

**Result:**
- List of inspections pending review
- Structural information (who, when, where)

**NoSQL Query (Complete Content):**
```javascript
db.inspection_results.findOne({
  inspection_id: "INS-2026-00123"
})
```

**Result:**
- Complete JSON document with all responses
- Photographic evidence
- Technician observations

**User presentation:**  
The system combines both results to show a unified view:

```
Inspection: INS-2026-00123
Status: UNDER_REVIEW
Technician: Juan Pérez
Date: 2026-02-17 14:45
Client: Company XYZ

--- Responses ---
Does the engine work?: Yes
Temperature: 72°C
Observations: Equipment in good condition

--- Evidence ---
[Photo 1: General view]
```

**Why hybrid:**
- **SQL:** Provides organizational context
- **NoSQL:** Provides dynamic content

**Functional advantage:**
- Fast queries to filter by company, technician, or date → SQL
- Full access to detailed responses → NoSQL

---

## 3. Form Evolution Without System Refactoring

**Critical requirement:**  
Inspection forms must be able to evolve without affecting the system.

### Real Scenario

**Month 1:**  
Inspection type "Electrical Maintenance" version 1.0 has 10 questions.

**Month 6:**  
The client company requests adding 5 new questions.

**Month 12:**  
An obsolete question is removed and another is modified.

### If the system were entirely SQL

**Problem:**
Each change would require:

```sql
-- Add column
ALTER TABLE inspection_responses ADD COLUMN new_question VARCHAR(255);

-- Migrate existing data
UPDATE inspection_responses SET new_question = NULL WHERE inspection_type_id = ?;

-- Modify validations
-- Update application code
-- Deploy new version
```

**Consequences:**
- Frequent migrations
- System downtime
- Increasing schema complexity
- Increasingly sparse tables (with many NULLs)

### With NoSQL

**Solution:**

```javascript
// Version 1.0
{
  "version": "1.0",
  "fields": [
    { "id": "question1", "label": "...", "type": "text" }
  ]
}

// Version 2.0 (6 months later)
{
  "version": "2.0",
  "fields": [
    { "id": "question1", "label": "...", "type": "text" },
    { "id": "new_question", "label": "...", "type": "number" }
  ]
}
```

**Advantages:**
- No schema migrations
- No downtime
- Old inspections maintain their version
- New inspections use the updated version
- Backward compatibility guaranteed

---

## 4. Lifecycle Integrity and Traceability

The relational model ensures strict control over state transitions.

### Flow Control in SQL

```sql
-- State history table
CREATE TABLE inspection_state_history (
  id INT PRIMARY KEY AUTO_INCREMENT,
  inspection_id VARCHAR(50),
  old_state VARCHAR(20),
  new_state VARCHAR(20),
  changed_by INT,
  changed_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  FOREIGN KEY (inspection_id) REFERENCES inspections(id),
  FOREIGN KEY (changed_by) REFERENCES users(id)
);
```

**Each transition is recorded:**

| inspection_id | old_state | new_state | changed_by | changed_at |
|---------------|-----------|-----------|------------|------------|
| INS-123 | DRAFT | ASSIGNED | ADM-01 | 2026-02-14 09:15:00 |
| INS-123 | ASSIGNED | IN_PROGRESS | TEC-05 | 2026-02-15 10:30:00 |
| INS-123 | IN_PROGRESS | UNDER_REVIEW | TEC-05 | 2026-02-15 14:45:00 |
| INS-123 | UNDER_REVIEW | APPROVED | SUP-02 | 2026-02-16 08:20:00 |

**Advantages:**
- Complete audit
- Identification of responsible parties
- Execution time analysis
- Regulatory compliance

### Immutable Content in NoSQL

Once an inspection is approved:

```javascript
db.inspection_results.updateOne(
  { inspection_id: "INS-123" },
  { $set: { locked: true, approved_at: ISODate() } }
)
```

**Result:**  
The document is locked for modifications.

**Separation guarantees:**
- **SQL:** Structural lifecycle integrity
- **NoSQL:** Complete execution content
- **Both:** Clear traceability between identity and inspection

---

## 5. Performance and Scalability Considerations

### Optimized SQL Queries

**Frequent queries:**
- Filter inspections by company
- Get inspections assigned to a technician
- Search inspections by state
- Generate compliance reports

**Example:**
```sql
SELECT COUNT(*) 
FROM inspections 
WHERE company_id = ? AND status = 'APPROVED' AND completed_at >= '2026-01-01'
```

**Why SQL:**
- Optimized indexes
- Efficient joins
- Fast aggregations

### Optimized NoSQL Queries

**Frequent queries:**
- Get complete document of an inspection
- Search specific responses within documents
- Query multimedia evidence

**Example:**
```javascript
db.inspection_results.findOne({ inspection_id: "INS-123" })
```

**Why NoSQL:**
- Direct document access
- No complex joins
- Horizontal scaling

**Result:**
- Reduced relational complexity
- Faster queries
- Independent scalability by data type

---

## 6. Risk Mitigation Through Hybrid Strategy

### If the system were entirely relational (SQL)

**Risk 1: Schema Rigidity**

Each change in inspection structure would require:
- `ALTER TABLE`
- Data migrations
- Potential downtime

**Risk 2: Structural Complexity**

Dynamic forms would require:
- Multiple relational tables
- Complex joins
- Sparse tables (with many NULLs)

**Risk 3: Degraded Performance**

Complete form queries would require:
- Joins of 10+ tables
- Slow queries
- Processing overhead

---

### If the system were entirely document-based (NoSQL)

**Risk 1: Loss of Referential Integrity**

Without foreign keys:
- Possible references to non-existent technicians
- Inconsistencies between companies and inspections
- Difficulty validating relationships

**Risk 2: Weak Multi-Tenant Isolation**

Without relational structure:
- Risk of exposing other companies' data
- More complex permission validations
- Larger security attack surface

**Risk 3: Difficulty with Analytical Queries**

Without relational capabilities:
- Hard to generate consolidated reports
- Impossible to make complex aggregations
- Need for external processing (ETL)

---

### The Hybrid Architecture Balances Both

| Aspect | SQL | NoSQL | Result |
|---------|-----|-------|-----------|
| Referential integrity | ✅ | ❌ | Guaranteed control |
| Schema flexibility | ❌ | ✅ | Evolution without migrations |
| Relational queries | ✅ | ❌ | Reports and analysis |
| Horizontal scalability | ⚠️ | ✅ | Sustainable growth |
| ACID transactions | ✅ | ⚠️ | Critical consistency |
| Complex documents | ❌ | ✅ | Dynamic forms |

---

## 7. Consistency Between Use Cases, Flow, and Architecture

The hybrid architecture directly responds to business use cases.

### Use Case: Create Inspection
- **Actor:** Administrator
- **Architecture:** SQL only
- **Why:** Structural and relational data

### Use Case: Execute Inspection in Field
- **Actor:** Technician
- **Architecture:** Hybrid (SQL for state, NoSQL for content)
- **Why:** Lifecycle control + form flexibility

### Use Case: Work Without Connection
- **Actor:** Technician
- **Architecture:** NoSQL (local cache) + SQL (later synchronization)
- **Why:** Critical offline support for field work

### Use Case: Review and Approve
- **Actor:** Supervisor
- **Architecture:** Hybrid (SQL for metadata, NoSQL for content)
- **Why:** Unified view of structural and dynamic information

### Use Case: Generate Compliance Reports
- **Actor:** Administrator
- **Architecture:** SQL only
- **Why:** Aggregate queries and complex filters

---

## Conclusion

The hybrid architecture of InspectaPro is not a technological preference, but a **direct consequence of the functional business requirements**.

**Guiding principle:**  
> "Structural, stable, and relational data lives in SQL.  
> Dynamic, flexible, and evolutionary data lives in NoSQL.  
> Both are connected through shared identifiers."

This strategy ensures that the system can:
- Maintain operational integrity
- Evolve without constant refactoring
- Scale sustainably
- Meet audit and traceability requirements

The hybrid architecture aligns technical decisions with business realities.
