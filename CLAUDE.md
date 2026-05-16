# CLAUDE.md

This file provides the necessary architectural context and modeling rules for Claude to assist with Part 2 (Architecture Extension) of the Software Architectures course.

## 1. Project Status & Workspace

- **Part 1**: finished. Requirements analysis, Utility Tree, and initial ASRs are complete. Instructor feedback extracted to `project/part1/feedback.md` — recurring weaknesses to avoid in Part 2 are listed there.
- **Part 2**: active. All work happens in `project/part2-current/` — see `project/part2-current/README.md` for the file map (instructions extract, initial-architecture extract, P1 design draft, source PDFs).
- **Goal**: quality-driven extension of the initial Patient Monitoring Service (PMS) architecture.
- **QAS ownership**:
  - **P1 (Medium)** — mine. Design lives in `project/part2-current/p1_current_design.md`.
  - **Av2 (High)** — teammate's. **Do not edit** teammate's files. Coordination notes on the P1↔Av2 seams live in **my** `p1_current_design.md` §7.

Full use-case text + index: `project/appendixA_requirements.md`. The folder-level READMEs (see §5) point at every other reference.

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

## 3. Required Architectural Views

The Part 2 assignment requires **five** views, fully specified (with "when to use each") in `project/part2-current/instructions.md` §4. Always cross-check against that file before producing a diagram. Quick recall:

1. System context — PMS boundary + external actors/systems.
2. Client–Server (C&C) — runtime components, ball-and-socket interfaces.
3. Process — sequence diagrams of quality-driven runtime behavior.
4. Decomposition — internal module structure with `<<module>>` stereotype.
5. Deployment — components mapped to nodes + comm paths.

Decisions not visible in the diagrams are considered insufficiently documented. Cross-view consistency is required.

## 4. Output expectations

Design work persists as markdown documents explaining what was changed and what choices were made during the process. Don't reason inline in conversation only — capture decisions in the relevant `.md` file (e.g., `p1_current_design.md` for P1) so they survive across sessions.

## 5. Background reference material

> **Rule: prefer `.md` over `.pdf`. PDFs are last resort** — only open a source PDF when a wording nuance or figure matters that the markdown extract doesn't preserve. Every folder below has a `README.md` index; read the index first, then open files on demand.

Folder indexes (each one is self-contained):

- `project/README.md` — Course-provided assignment materials at top of `/project/`. All three appendices have markdown extracts: `description.md`, `appendixA_requirements.md`, `appendixB_APIs.md`.
- `project/part2-current/README.md` — Active workspace: `instructions.md`, `initial_architecture.md`, `p1_current_design.md`, source PDFs.
- `project/part1/README.md` — Finished Part 1 deliverables. Includes `feedback.md` — markdown extract of the instructor's graded feedback (68%, 20.4/30). **Read this before any Part 2 design or rationale work** — it surfaces the four recurring weaknesses (motivation depth, necessary ≠ sufficient, mechanism vs. restatement, precision) likely to resurface if not actively avoided.
- `project/visual-paradigm/README.md` — SAPlugin manual + VP lab session. Reference only; per hard constraints below, do not invoke the plugin or edit `.vpp` files.
- `book/README.md` — Bass/Clements/Kazman *Software Architecture in Practice* chapters. Authoritative source for QA definitions and tactics catalogues. Most relevant: **ch4** (QAS structure), **ch8** (Performance — for P1), **ch5** (Availability — for Av2 seam), **ch16** (ATAM — for sensitivity / trade-off reasoning).
- `lectures/README.md` — Course slide decks (L1–L7). Most relevant: **L4** (architectural views), **L5** (UML modeling — backs the conventions in §2), **L7** (ATAM evaluation), **L1** (PMS domain intro).

## Hard constraints, user owns these tools

- **Do not** run any LaTeX build. The user compiles locally.
- **Do not** worry about the SAPlugin, i will use it myself and report the output to you if needed
- **Do not** edit or generate `.vpp` files. The user does all UML modeling inside Visual Paradigm 17.3.
- For diagram work, describe the structure in markdown and stop there.
