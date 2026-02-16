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
