# States, Decisions and Transitions

## 1. Inspection Lifecycle Overview

Each inspection in InspectaPro follows a structured lifecycle.

The lifecycle ensures that inspections move through controlled stages, preventing inconsistencies and guaranteeing operational integrity.

The process is based on defined states and controlled transitions between them.

---

## 2. Inspection States

The main states of an inspection are:

- Pending
- Assigned
- In Progress
- Under Review
- Approved
- Rejected
- Closed

---

## 3. State Description

### Pending
The inspection has been created but no technician has been assigned yet.

### Assigned
A technician has been assigned to the inspection, but execution has not started.

### In Progress
The technician is actively performing the inspection.

### Under Review
The technician has submitted the results and the inspection is waiting for administrative validation.

### Approved
The administrator has reviewed and validated the inspection results.

### Rejected
The administrator has reviewed the inspection and requested corrections.

### Closed
The inspection has been finalized. No further modifications are allowed.

---

## 4. Decision Points

The most important decision occurs during the validation phase.

The Company Administrator decides whether:

- The inspection meets all requirements (Approved)
- The inspection requires corrections (Rejected)

This decision determines the next state transition.

---

## 5. Transition Rules

State transitions are controlled by the system to ensure process consistency.

Examples of transition rules:

- An inspection cannot move to "Assigned" without being created first.
- An inspection cannot move to "In Progress" without being assigned.
- An inspection cannot move to "Under Review" unless execution is completed.
- An inspection cannot be closed unless it has been approved.
- A rejected inspection must return to execution before it can be validated again.

These transition rules guarantee data integrity and prevent invalid process flows.

---

## 6. Process Integrity

The controlled lifecycle ensures:

- Clear operational responsibility
- Structured validation
- Prevention of unauthorized changes
- Traceability of decisions

This structured state management model is essential for maintaining inspection reliability and business accountability.
