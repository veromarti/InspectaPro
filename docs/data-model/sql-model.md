# SQL Data Model

## 1. Introduction

The relational database in InspectaPro is responsible for storing all structured and stable information.

We use SQL because this type of data requires strong relationships, consistency, and control over business rules.

The relational model ensures that companies, users, roles, and inspections are properly connected and validated.

---

## 2. Main Entities

The SQL model includes the following main entities:

- Company  
- User  
- Role  
- CompanyUser (Many-to-Many relationship)  
- Subscription  
- InspectionType  
- Inspection  

Each entity has a clear responsibility within the system.

---

## 3. Entity Explanation

### Company
Represents an organization that uses the platform.  
Each company has its own users, inspections, and subscription.

### User
Represents a person who interacts with the system.  
A user can belong to one or more companies.

### Role
Defines the permissions of a user inside a company (for example: Administrator or Technician).

### CompanyUser
Resolves the many-to-many relationship between Users and Companies.  
It also allows assigning different roles per company.

### Subscription
Represents the service plan associated with a company.  
It controls system access and operational limits.

### InspectionType
Defines the general structure of an inspection.  
It stores metadata but not the dynamic form structure.

### Inspection
Represents a specific inspection assigned to a technician.  
It stores the lifecycle state, timestamps, and references to related entities.

---

## 4. Normalization

The relational model follows normalization principles up to Third Normal Form (3NF).

This means:

- There is no unnecessary data duplication.
- Each table has a clear responsibility.
- All attributes depend only on the primary key.
- Relationships are managed using foreign keys.

Normalization helps prevent data anomalies and keeps the structure clean and maintainable.

---

## 5. Why SQL is Necessary

SQL is used because:

- Companies and users require strong relational integrity.
- Inspections must reference valid users and inspection types.
- State transitions must follow controlled rules.
- Multi-tenant isolation must be enforced.

A document-based model alone would not guarantee referential integrity between companies, users, and inspections.

For this reason, the relational model is essential for business control and structural consistency.
