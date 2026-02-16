# NoSQL Data Model

## 1. Introduction

The NoSQL database in InspectaPro is used to store dynamic and flexible data related to inspection execution.

Unlike structured organizational data, inspection responses may vary depending on the inspection type. For this reason, a document-based model is more appropriate.

NoSQL allows the system to store inspection forms and responses without modifying the relational schema every time a form changes.

---

## 2. What is Stored in NoSQL

The document database stores:

- Inspection responses
- Dynamic form structures
- Questions and sections
- Evidence links
- Notes
- Numeric values
- Boolean fields
- Form version information

This data is flexible and may change over time.

---

## 3. Document Structure

Each inspection execution is stored as a document.

The document includes:

- A reference to the inspection ID stored in SQL
- The form version used
- A list of questions
- Answers provided by the technician
- Attached evidence (if applicable)
- Additional notes

The document structure may include embedded objects and arrays to represent grouped questions or multiple responses.

---

## 4. Why This Data Does Not Belong in SQL

Inspection responses can vary significantly between inspection types.

If this data were stored in SQL:

- The schema would require frequent changes.
- Many optional columns would be needed.
- The number of tables could increase excessively.
- Maintenance complexity would grow over time.

Using NoSQL avoids schema rigidity and allows inspection forms to evolve naturally.

---

## 5. Embedding Strategy

The system uses embedded structures within each document.

This means that:

- All responses related to one inspection are stored together.
- Questions and answers are nested inside the inspection document.
- Evidence links are stored as arrays within the same document.

Embedding improves performance when retrieving a complete inspection result.

---

## 6. Version Control and Evolution

Each document includes a form version field.

This ensures that:

- Older inspections remain unchanged.
- New inspection types can evolve independently.
- Historical traceability is preserved.

This design supports long-term flexibility while maintaining consistency between relational and document data.
