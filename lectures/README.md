# Lectures — course slide decks

Lecture slides for the Software Architectures course. The book (`/book`) is the deeper reference; lectures are useful for the course-specific framing, the PMS case study, and instructor emphasis (which tactics / views are expected in the deliverables).

> **Repo rule: prefer `.md` over `.pdf` everywhere.** This folder has no markdown extracts (slides are visual artifacts that don't extract cleanly), so the PDFs are the primary reference here. Use the index below to avoid opening decks blindly.

## Index

| Folder | Files | Topic | When to consult |
|---|---|---|---|
| `L1` | `SA_L01_1_overview_organization.pdf` | Course overview & organization | Skip — administrative. |
| `L1` | `SA_L01_2_PMS_intro.pdf` | **Patient Monitoring System (PMS) case introduction** | Foundational context for the whole project — domain, actors, why the system exists. Pair with `project/description.md`. |
| `L1` | `SA_L01_3_process_requirements.pdf` | Architecture process & requirements gathering | Background for Part 1 (already finished). |
| `L2` | `SA_L02_intro.pdf` | Lecture 2 intro | Short framing deck. |
| `L2` | `SA_L02_software_quality.pdf` | **Software quality & quality attributes** | Definitions of QAs, QAS template, utility tree. Read alongside `book/SA_book_ch4_QAS.pdf`. |
| `L3` | `SA_L03_01_intro.pdf` | Lecture 3 intro | Short framing deck. |
| `L3` | `SA_L03_03_architecture_defined.pdf` | What "architecture" is (and isn't) | Conceptual baseline — elements, relations, properties; structure vs. behavior. |
| `L4` | `SA_L04_01_introduction.pdf` | Lecture 4 intro | Short framing deck. |
| `L4` | `SA_L04_03_architectural_views.pdf` | **Architectural views (4+1 / C&C / Module / Allocation)** | Direct support for the four mandatory views in CLAUDE.md §3: Client-Server (C&C), Decomposition (Module), Process (C&C sequence), Deployment (Allocation). |
| `L5` | `SA_L05_modeling.pdf` | **UML modeling for architecture** | Backs the modeling conventions in CLAUDE.md §2: components vs. modules, ball/socket interfaces, sequence diagrams, deployment nodes. Read this before debating any convention. |
| `L6` | `SA_L06.pdf` | Lecture 6 | Likely tactics / patterns deck (general). |
| `L7` | `SA_L06_part2.pdf` | Lecture 6 continued | Continuation of L6. |
| `L7` | `SA_L07_part1.pdf` | Lecture 7 part 1 | **ATAM / architectural evaluation territory** — sensitivity points, trade-offs, risks. Aligns with the ATAM-inspired reasoning in CLAUDE.md §1.1. |
| `L7` | `SA_L07_part2.pdf` | Lecture 7 part 2 | Continuation of L7. |

## Quick map: task → lecture

- Writing or critiquing a **QAS** → `L2/SA_L02_software_quality.pdf` + `book/SA_book_ch4_QAS.pdf`
- Picking the right **view / diagram type** for a Part 2 deliverable → `L4/SA_L04_03_architectural_views.pdf`
- Resolving a **modeling-convention** question (component vs. module, interface placement, sequence-diagram rules) → `L5/SA_L05_modeling.pdf`
- Justifying a decision with **sensitivity / trade-off** reasoning → `L7/SA_L07_part1.pdf` + `book/SA_book_ch16.pdf`
- Refreshing on the **PMS domain** → `L1/SA_L01_2_PMS_intro.pdf`
