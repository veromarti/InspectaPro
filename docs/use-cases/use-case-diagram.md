# Use Case Diagram Explanation

## Overview

The use case diagram shows how the main actors interact with InspectaPro.  
It represents the full lifecycle of an inspection, from creation to completion.

The diagram connects business actions with system behavior.

## Actors

### Company

The Company represents the organization that uses the system.  
It registers in the platform and manages its company profile.

### Company Administrator

The Company Administrator manages the operational structure.  
This actor can:

- Manage users  
- Define inspection types  
- Create inspections  
- Assign inspections  
- Review results  
- Close inspections  

The administrator controls the inspection process.

### Technician

The Technician executes inspections in the field.  
This actor can:

- View assigned inspections  
- Execute inspections  
- Submit inspection results  

The technician performs the operational work.

### System

The System represents automated validations.  
It ensures that:

- Inspection results are stored correctly  
- State transitions are valid  
- Business rules are enforced  

## Inspection Lifecycle

The diagram reflects the following flow:

1. The administrator defines an inspection type.
2. The administrator creates an inspection.
3. The inspection is assigned to a technician.
4. The technician executes the inspection.
5. The technician submits the results.
6. The system validates the state transition.
7. The administrator reviews and closes the inspection.

This sequence ensures that inspections follow a controlled and traceable process.
