# Flows and States Images - InspectaPro

This folder contains flow and state diagrams for the system.

## Required Images

Please save the following images in this folder:

### 1. diagrama-estados-inspeccion.png
**Description:** Inspection State Diagram  
**Content:** Shows the 7 main states (DRAFT, ASSIGNED, IN_PROGRESS, SYNCING, UNDER_REVIEW, APPROVED, REJECTED) with their transitions and decisions.

**This is the image with:**
- Initial black circle
- States in rounded rectangular boxes
- Arrows with transition labels
- Final black circle

### 2. flujo-vida-inspeccion.png
**Description:** Inspection Lifecycle Flow - InspectaPro  
**Content:** Complete flow divided into 4 phases:
- **Configuration (SQL):** Create inspection, select type, assign technician, define date
- **Execution (Hybrid):** Technician receives notification, opens app, fills form
- **Closure and Synchronization (Hybrid):** Completes inspection, saves document, updates state
- **Review (SQL):** Supervisor reviews, approves or rejects

**This is the blue image with:**
- Initial black circle
- Rounded blue rectangles for each step
- Border rectangles for phases
- Explanatory notes in yellow boxes
- Decision points (diamonds)

## References in Documentation

These images are referenced in:
- `docs/flows/main-process-flow.md` → flujo-vida-inspeccion.png
- `docs/flows/states-decisions-transitions.md` → diagrama-estados-inspeccion.png
- `docs/flows/functional-use-cases.md` → both images

## Recommended Format

- **Format:** PNG (preferred) or JPG
- **Resolution:** Minimum 1200px width for readability
- **Background:** White or transparent
- **Exact name:** Use the names specified above (lowercase, with hyphens)

---

**Note:** If you change the file names, you will need to update the references in the markdown files mentioned above.
