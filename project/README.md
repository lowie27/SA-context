# Project — assignment materials

Course-provided project documents and team workspace. This file documents what lives at the top of `/project/`; each subfolder has its own `README.md`.

> **Prefer `.md` over `.pdf`. PDFs are last resort** — open a source PDF only when a figure or wording nuance matters that the markdown doesn't preserve.

## Top-level files

Each course-provided PDF has a paired `.md` extract. **Use the `.md` by default**, open the PDF only as last resort (figure or wording nuance).

| MD extract | Paired PDF | Topic | When to consult |
|---|---|---|---|
| `description.md` | `SA_project_description.pdf` | **Domain description** — context (CVDs, e-health, positioning), problem domain (monitoring / risk estimation / notifications / emergencies), stakeholders, key building blocks (HIS, GDPR, security). | Read once to ground yourself in why the PMS exists and what it must do. Pairs with `lectures/L1/SA_L01_2_PMS_intro.pdf`. |
| `appendixA_requirements.md` | `SA_project_appendixA_requirements.pdf` | **Functional requirements** — actors + all 19 textual use cases (pre / post / main / alt) + QAS-P1 touchpoint table. | Default reference for any UC↔component reasoning. |
| `appendixB_APIs.md` | `SA_project_appendixB_APIs.pdf` | **healthAPI specification** — reduced FHIR subset for HIS interop; Patient/Observation/RiskAssessment resources + supporting data types. | Consult for anything that touches the HIS boundary — UC16 (consult patient record), UC17 (update patient record). Relevant for the P1 extension's HIS-seam decisions (caching, batching, async). |

## Subdirectories

| Path | What's there | Indexed by |
|---|---|---|
| `part1/` | Finished Part 1 deliverables + instructor feedback PDF. | `part1/README.md` |
| `part2-current/` | **Active workspace** — instructions extract, initial-architecture extract, P1 design draft, source PDFs. | `part2-current/README.md` |
| `visual-paradigm/` | SAPlugin manual + VP lab session. Tooling — Claude must not invoke. | `visual-paradigm/README.md` |
