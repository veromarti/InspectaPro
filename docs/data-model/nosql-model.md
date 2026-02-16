## 1. Task
#### Teams must design a document collection to store:

- Dynamic inspection responses.
- Evidence links.
- Flexible form structures.
- Relevant metadata.

#### The document model must include:
- A reference to the inspection stored in SQL.
- A clearly defined JSON structure.
- Embedded objects.
- Arrays.
- A form version field.

#### Teams must justify:
- Why this information should not live in SQL.
- Why they chose embedding instead of referencing (if applicable).
- How their structure supports form evolution over time.

No query design is required. Only structural modeling.

## 2. Entity-Relationship Diagram

![Entity-Relationship Diagram](/static/img/NoSQL/NoSQL.png)

## 3. Description

1. **Why this should not live in SQL**: Storing dynamic inspection responses in SQL leads to "Exhaustive Normalization" or "EAV (Entity-Attribute-Value) Hell." * Complexity: In SQL, a single inspection with 50 questions and multiple photos would require joining 4 or 5 tables to reconstruct one report.

   - **Schema Rigidity**: If a new inspection type adds a "GPS Coordinate" or "Multi-select" field, you would have to run a migration on a massive SQL table. NoSQL allows each document to vary without breaking the database.


2. **Why Embedding vs. Referencing**: I chose Embedding for the sections, questions, and evidence links.

   - **Atomic Reads**: Inspections are usually "write once, read many." By embedding the questions and evidence inside the inspection document, we can retrieve the entire report in a single disk read (O(1) lookup).

   - **Data Integrity (Snapshotting)**: If a question's text changes in the master template next year, the "Embedded" response maintains the text as it was at the time of the inspection. Referencing would historically "corrupt" old reports.


3. **Support for Form Evolution**
   - **Form Versioning** : The form_version field is critical. The application code can use this field to determine which "view" or "validation logic" to apply to the document.

   - **Schemaless Nature**: As requirements evolve (e.g., adding a "Risk Score" to each question), we can simply start adding that field to new documents. Old documents remain valid and simply lack that specific field, which the UI can handle gracefully.
