# Project — assignment materials

Course-provided project documents and team workspace. The subfolders each have their own index (or are covered directly by CLAUDE.md); this file documents what lives at the top of `/project/`.

## Top-level files

| File | Topic | When to consult |
|---|---|---|
| `SA_project_description.pdf` | **Domain description** — high-level context, business objectives, problem space for the PMS. | Read once to ground yourself in why the system exists. Pair with `lectures/L1/SA_L01_2_PMS_intro.pdf`. |
| `SA_project_appendixA_requirements.pdf` | **Functional requirements** — the 19 textual use cases + actors. | **Prefer the markdown extract** at `use_cases.md` (in this folder) — it covers all 19 UCs with pre/post/main/alt. Open the PDF only to check a figure or wording the markdown doesn't preserve. |
| `SA_project_appendixB_APIs.pdf` | **healthAPI specification** — interface the PMS uses to interoperate with the external Hospital Information System (HIS). | Consult when designing anything that touches the HIS boundary — UC16 (consult patient record) and UC17 (update patient record). Relevant if the P1 extension introduces caching, batching, or async patterns at the HIS seam. |
| `use_cases.md` | Lookup-free extract of appendix A: actors + all 19 UCs (pre / post / main / alt) + QAS-P1 touchpoint table. | Default reference for any UC↔component reasoning. Lives at `/project/` (not under `part2-current/`) because it's project-wide reference material, not Part 2-specific work product. |

## Subdirectories

| Path | What's there | Indexed by |
|---|---|---|
| `part1/` | Finished Part 1 deliverables + instructor feedback. Contains `SA_project_part1_instructions.pdf`, `SA_project_part1_template.pdf`, and **`Goddé_Ramaekers_VanVyve_feedback.pdf`** (graded feedback on our submission). Worth skimming the feedback PDF before Part 2 design decisions — it tells us what the instructors flagged as weak in our requirements / utility tree / ASRs, and those weaknesses may resurface in Part 2 if not addressed. | This file. |
| `part2-current/` | **Active workspace.** Contains `initial_architecture.md` (extract of the rationale PDF), `instructions.md` (extract of the assignment brief), `p1_current.md` (P1 design draft), plus the source PDFs (`SA_project_part2_inital_architecture_including_rationale.pdf`, `SA_project_part2_instructions.pdf`). Prefer the `.md` extracts. | Covered directly by CLAUDE.md §1. |
| `visual-paradigm/` | SAPlugin manual and VP lab session — tooling Claude must not invoke. | `visual-paradigm/README.md`. |
