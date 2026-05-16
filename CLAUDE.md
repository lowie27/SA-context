# CLAUDE.md

This file provides the necessary architectural context and modeling rules for Claude to assist with Part 2 (Architecture Extension) of the Software Architectures course.

## 1. Project Status & Workspace

- Part 1 Status: Finished. Requirements analysis, Utility Tree, and initial ASRs are complete. No further work is needed on Part 1 materials.
- Active Directory: All Part 2 work occurs in part2-current.
- Archived Files: The directory part2-archive is deprecated and not used.

- Goal: Perform a quality-driven extension of the initial Patient Monitoring Service (PMS) architecture.

- QAS ownership: P1 (Performance) is mine — edit freely. Av2 (Availability) is the teammate's — do not edit.
- P1 design draft: `project/part2-current/p1_current.md` holds my **current proposal** for how to extend the architecture to satisfy P1 — not a snapshot of the existing system. Treat it as a living draft to stress-test: anchor on the QAS in §0, then push back on the approach in §1–§3 and surface sensitivity / trade-off points the draft misses. §7 lists the seams with the Av2 teammate.
- Initial architecture reference: `project/part2-current/initial_architecture.md` is a lookup-free extract of the rationale PDF (`SA_project_part2_inital_architecture_including_rationale.pdf`). Read that file when reasoning about existing components, interfaces, nodes, or sequence flows. **Prefer it over the PDF** — only open the PDF if the readme doesn't answer the question.
- Part 2 instructions reference: `project/part2-current/instructions.md` is a lookup-free extract of the assignment brief (`SA_project_part2_instructions.pdf`). Covers the task, "fully supports" criteria, trade-off analysis requirements, required views, report structure, deadline, and all 6 candidate QASs (including P1 and Av2 in full). **Prefer it over the PDF** — open the PDF only if a wording nuance matters.
- Use cases: full textual UCs live in `project/use_cases.md`. Read that file when reasoning about UC↔component mapping. Quick index:

| UC | Name | Primary actor |
|---|---|---|
| UC1 | log in | User |
| UC2 | log off | User |
| UC3 | send emergency notification | Patient (via gateway) |
| UC4 | send sensor data | Patient (via gateway) |
| UC5 | notify registered parties | PMS |
| UC6 | consult patient status | Physician or Patient |
| UC7 | configure patient risk assessment | Cardiologist |
| UC8 | update risk level | Cardiologist |
| UC9 | perform on-demand consultation | Cardiologist |
| UC10 | register patient | Trained nurse |
| UC11 | deregister patient | Trained nurse |
| UC12 | estimate risk level | PMS |
| UC13 | retrieve new ML models | Telemedicine operator |
| UC14 | download ML model | PMS |
| UC15 | update ML model | Telemedicine operator |
| UC16 | consult patient record | PMS |
| UC17 | update patient record | PMS |
| UC18 | consult monitoring summary | Patient |
| UC19 | request access to personal data | User |

## 1.1 Architectural Reasoning Model (ATAM-inspired)

Architectural decisions in Part 2 should follow an ATAM-inspired reasoning approach, focusing on how design choices impact quality attributes.

- Sensitivity points: Architectural decisions or parameters where small changes can have a large impact on one or more quality attributes (e.g., deployment location affecting performance or availability).
- Trade-off points: Architectural decisions that improve one quality attribute while negatively affecting another (e.g., improving modifiability at the cost of performance or predictability). These are often also sensitivity points for multiple attributes.

All design reasoning should explicitly consider these two concepts when evaluating or justifying architectural choices.

## 2. Mandatory Modeling Conventions (UML in Visual Paradigm) All architectural assistance must adhere to these course-specific rules:

- Convention 1: All components, interfaces, and data types must have globally unique names.
- Convention 2: Map each architectural viewpoint to its specific UML diagram type (e.g., Client-Server to Component diagram, Process to Sequence diagram).
- Convention 3: Connect interfaces to components using only Interface Realizations (Balls) or Interface Usages (Sockets).
- Convention 4: No "residual functionality." If a parent component has an interface, at least one sub-component must also have it.
- Convention 5: Internal design units must use the <<module>> stereotype to distinguish them from runtime components.
- Convention 6: On sequence diagrams, an operation call is only valid if the source component has a Required Interface for that service.
- Convention 7: A sequence diagram must contain lifelines for either components or modules, but never both simultaneously.
- Convention 8: All runtime components must be deployed inside physical Nodes (Deployment View).
- Convention 9: Communicating components must reside on the same Node or on separate Nodes connected by a communication path (Association).

- Mandatory Descriptions: Every element (component, module, interface, node) must have a responsibility description in its General tab to populate the final report's element catalog.
- Explicit References: Messages in sequence diagrams must refer to actual model operations rather than using plain text to ensure consistency during renaming.

## 3. Required Architectural Views Documentation must be consistent across four perspectives:

- Client-Server View (C&C): Runtime entities and service interactions via ball-and-socket notation.
- Decomposition View (Module): Internal code structure using the <<module>> stereotype.
- Process View (C&C): Sequence diagrams showing detection of exceptional situations and response measures.
- Deployment View (Allocation): Mapping software components to nodes like PatientSmartphone or PMSBackendNode.

## 4. Strategic Rationale for Part 2

- Replace Placeholders: The gray OtherFunctionality component and TODONode from the initial architecture must be replaced with concrete design elements.
- End-to-End Support: An extension is only complete if the scenario is covered from stimulus to response measure (e.g., 5s detection window for Av2).

- expected output is a markdown document explaining what needs to be changed, and what choices where made during the process

## 5. Background reference material

Don't scan PDFs blindly. Each of these folders has a `README.md` index that maps files to topics and tells you when to consult them — read the index first, then open the specific PDF only if needed.

- `project/README.md` — Course-provided assignment materials: domain description, appendix A (requirements — extracted to `project/use_cases.md`), appendix B (healthAPI / HIS interface), `part1/` (finished, but contains **instructor feedback PDF** worth checking for Part 2), and pointer to the Part 2 instructions PDF.
- `book/README.md` — Bass/Clements/Kazman "Software Architecture in Practice" chapters. Authoritative source for QA definitions and tactics catalogues. Most relevant: **ch4** (QAS structure), **ch8** (Performance — for P1), **ch5** (Availability — for Av2 seam), **ch16** (ATAM — for sensitivity / trade-off reasoning).
- `lectures/README.md` — Course slide decks (L1–L7). Most relevant: **L4** (architectural views), **L5** (UML modeling — backs the conventions in §2), **L7** (ATAM evaluation), **L1 PMS intro** (domain).
- `project/visual-paradigm/README.md` — SAPlugin manual and VP lab session. Reference only; per the hard constraints below, do not invoke the plugin or edit `.vpp` files.

## Hard constraints, user owns these tools

- **Do not** run any LaTeX build. The user compiles locally.
- **Do not** worry about the SAPlugin, i will use it myself and report the output to you if needed
- **Do not** edit or generate `.vpp` files. The user does all UML modeling inside Visual Paradigm 17.3.
- For diagram work, describe the structure in markdown and stop there.
