InspectaPro adopts a hybrid architectural model combining relational (SQL) and document-based (NoSQL) databases.
This decision is not technological preference but a direct consequence of the business model and the nature of inspection data.

## SQL Data

The relational database stores structural and governance-critical data.

### Stored in SQL

- Companies
- Users (Administrators, Technicians)
- Inspection Types (metadata only)
- Inspection Assignments
- Inspection Status
- State transitions history

### Why SQL?

These entities:

1. Have stable structure
2. Require referential integrity
3. Represent organizational hierarchy
4. Demand transactional consistency
5. Must be auditable

#### For example

- An inspection cannot exist without a company.
- An assignment cannot exist without a technician.
- A state transition must be historically traceable.

#### Relational modeling enforces

- Foreign keys
- Constraints
- ACID transactions (Atomicity, Consistency, Isolation, Durability)
- Structured normalization

This guarantees that the organizational backbone of InspectaPro remains consistent and governed.

## NoSQL Data

The document database stores dynamic inspection execution data.

### Stored in NoSQL

- Inspection responses
- Variable question structures
- Media references (optional)
- Dynamic answer formats
- Evolving inspection templates

### Why NoSQL?

#### Inspection content is inherently flexible

- Different inspection types have different questions.
- Questions may change over time.
- Answer formats may vary (boolean, text, numeric, lists).
- Some inspections may include optional sections.

#### Modeling this variability in SQL would require

- Frequent schema migrations
- Complex join structures
- Highly sparse tables
- Reduced performance

#### Document storage allows

- Flexible JSON structures
- Schema evolution without downtime
- Encapsulation of full inspection result in one document
- Natural representation of nested answers

## Risks of a Fully Relational Model

If InspectaPro were entirely relational:

### Risk 1: Schema Rigidity

Each change in inspection structure would require:

- ALTER TABLE operations
- Data migrations
- Potential downtime

This reduces agility and slows business evolution.

### Risk 2: Structural Complexity

Dynamic forms would require:

- Multiple relational tables
- Indirect relationships
- Complex joins

This increases:

- Query complexity
- Maintenance cost

Risk of inconsistency across related tables

### Risk 3: Performance Degradation

Retrieving one inspection result would require:

- Multiple joins
- Reconstruction of nested structures

This is inefficient compared to document retrieval.

## Risks of a Fully Document-Based Model

If InspectaPro were entirely NoSQL:

### Risk 1: Loss of Referential Integrity

There would be no strict enforcement that:

- A technician exists before assignment
- A company is valid
- A state transition is allowed

This would rely entirely on application logic, increasing risk.

### Risk 2: Inconsistent Organizational Data

Document duplication could occur:

- User data repeated in multiple documents
- Company data embedded in inspections

This creates:

- Update anomalies
- Version inconsistency
- Data redundancy

### Risk 3: Weak Transactional Guarantees

Administrative operations require:

- Strong consistency
- Atomic updates

Many document databases offer weaker transactional guarantees compared to relational systems.

Governance requires structure.
Structure requires relational enforcement.

## How Both Models Are Connected

The connection between SQL and NoSQL is controlled via:

- Inspection ID (primary key in SQL)
- Document reference using the same ID in NoSQL

Flow:

- Inspection is created in SQL.
- Assignment and state are managed in SQL.
- Execution results are stored in NoSQL under the same inspection identifier.
- SQL retains the authoritative lifecycle state.

This ensures:

- SQL governs process integrity.
- NoSQL stores operational payload.
- Each inspection has one relational identity and one document representation.

There is separation, but not fragmentation.

## Consistency and Traceability Considerations

Consistency Strategy:

- Strong consistency for structural data (SQL).
- Document-level consistency for inspection results (NoSQL).
- Application-layer validation for cross-database integrity.

Traceability Strategy:

- State transitions recorded in SQL.
- Timestamps maintained in both systems.
- Inspection lifecycle never depends solely on document data.

This guarantees:

- Auditability
- Controlled progression
- Historical reconstruction capability

Traceability is centralized in SQL.
Flexibility is delegated to NoSQL.

## How the Business Model Influences Data Design Decisions

InspectaPro is not a static record system.
It is a controlled inspection workflow platform.

The business model requires:

- Organizational governance
- Controlled state transitions
- Role-based responsibility
- Variable inspection structures
- Auditability

These requirements directly shaped architectural decisions:

| Business Need             | Data Design Decision      |
| ------------------------- | ------------------------- |
| Controlled workflow       | SQL-based state model     |
| Flexible inspection forms | Document storage          |
| Organizational structure  | Relational modeling       |
| Audit trail               | Transactional persistence |
| Scalability of results    | NoSQL document separation |

## Final Architectural Position

A hybrid architecture is not optional — it is structurally justified.

Fully relational → too rigid

Fully document-based → too uncontrolled

Hybrid → controlled core + flexible execution

InspectaPro requires both governance and adaptability.

This balance defines its architectural identity.
InspectaPro adopts a hybrid architectural model combining relational (SQL) and document-based (NoSQL) databases. This decision is not technological preference but a direct consequence of the business model and the nature of inspection data.

SQL Data
The relational database stores structural and governance-critical data.

Stored in SQL
Companies
Users (Administrators, Technicians)
Inspection Types (metadata only)
Inspection Assignments
Inspection Status
State transitions history
Why SQL?
These entities:

Have stable structure
Require referential integrity
Represent organizational hierarchy
Demand transactional consistency
Must be auditable
For example
An inspection cannot exist without a company.
An assignment cannot exist without a technician.
A state transition must be historically traceable.
Relational modeling enforces
Foreign keys
Constraints
ACID transactions (Atomicity, Consistency, Isolation, Durability)
Structured normalization
This guarantees that the organizational backbone of InspectaPro remains consistent and governed.

NoSQL Data
The document database stores dynamic inspection execution data.

Stored in NoSQL
Inspection responses
Variable question structures
Media references (optional)
Dynamic answer formats
Evolving inspection templates
Why NoSQL?
Inspection content is inherently flexible
Different inspection types have different questions.
Questions may change over time.
Answer formats may vary (boolean, text, numeric, lists).
Some inspections may include optional sections.
Modeling this variability in SQL would require
Frequent schema migrations
Complex join structures
Highly sparse tables
Reduced performance
Document storage allows
Flexible JSON structures
Schema evolution without downtime
Encapsulation of full inspection result in one document
Natural representation of nested answers
