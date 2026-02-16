## 1. Scope Description
This use case diagram represents the complete business lifecycle of an inspection within the InspectaPro system, from company registration to inspection closure.

The diagram models interactions between organizational actors and the system, ensuring alignment with the underlying SQL and NoSQL data models.

The purpose of this diagram is to illustrate:

- The actors involved in the inspection process
- The responsibilities of each role
- The full lifecycle of inspection management
- The relationship between operational actions and system validation

## 2. Use Case Diagram

![Use Case Diagram](/static/img/use-cases/use-case-diagram.png)

## 3. Actors Description

### 3.1 Company 

Represents the organization that owns inspections, users, and inspection data.
Although not a human actor, it is included to represent the business entity interacting with the system at an organizational level.

#### Responsibilities

- Register company
- Manage company profile

### 3.2 Company Administrator

Responsible for configuring and supervising the inspection process.

#### Responsibilities

- Define inspection types
- Create inspections
- Assign inspections to technicians
- Review inspection results
- Close inspections
- Manage users

### 3.3 Technician

Responsible for executing inspections assigned by the administrator.

#### Responsibilities

- View assigned inspections
- Start inspection
- Execute inspection
- Submit inspection results

### 3.4 System

Represents automated system responsibilities necessary for enforcing business rules.

#### Responsibilities

- Validate state transitions
- Store inspection results

The system actor ensures governance and traceability across the inspection lifecycle.

## 4. Full Inspection Lifecycle Representation

The use case diagram covers the following lifecycle stages:

- Company registration and configuration
- Inspection type definition
- Inspection creation
- Inspection assignment
- Inspection execution
- Results submission
- Administrative review
- Inspection closure

This guarantees that the model represents the complete business process.

