## 1. Task
Teams must design a relational model that includes at minimum:

- Companies
- Users
- Roles
- User–Company relationship (mandatory N:M)
- Inspection Types
- Inspections
- Subscriptions or service plans

#### Technical requirements:
- Apply normalization at least up to Third Normal Form (3NF).
- Avoid unnecessary redundancy.
- Properly separate entities.
- Clearly define primary keys and foreign keys.
- Represent cardinalities.
- Explain how normalization was applied and why.
No scripts are required. Only conceptual/logical modeling and the ERD.

## 2. Entity-Relationship Diagram

![Entity-Relationship Diagram](/static/img/SQL/SQL.png)

![Entity-Relationship Diagram](/static/img/SQL/SQL%20(1).png)

## 3. Description

### Key Entity Relationships
- **Company_User (N:M)**: A user can work for multiple companies, and a company has many users. The role_id is placed here because a user might be an "Admin" in Company A but a "Viewer" in Company B.

- **Service_Plans (1:N)**: One plan can be assigned to many companies, but a company typically follows one plan at a time.

- **Inspections (N:1)**: Each inspection belongs to one company, is performed by one user, and falls under one specific "Type" (e.g., Safety, Fire, Electrical).

### Normalization Logic Applied
- **1NF (First Normal Form)**: Every table has a unique Primary Key (PK). Attributes are atomic; for example, Service_Plans stores single values for price and limits rather than lists.

- **2NF (Second Normal Form)**: By creating the Company_User bridge table, we've removed partial dependencies. The attribute joined_at depends on the combination of company_id and user_id, satisfying the requirement for mandatory N:M relationships.

- **3NF (Third Normal Form)**: * Eliminating Transitive Dependencies: I separated Service_Plans from Companies. If a plan's price changes, we update one row in Service_Plans rather than thousands of rows in Companies.

- **Separating Roles**: Roles are abstracted from the User table. This allows a user to have different roles across different companies (e.g., Manager in Company A, Staff in Company B) without duplicating user data.
