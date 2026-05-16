# Project — assignment materials

Course-provided project documents and team workspace. This file documents what lives at the top of `/project/`; each subfolder has its own `README.md`.

> **Prefer `.md` over `.pdf`. PDFs are last resort** — open a source PDF only when a figure or wording nuance matters that the markdown doesn't preserve.

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
| `part1/` | Finished Part 1 deliverables + instructor feedback PDF. | `part1/README.md` |
| `part2-current/` | **Active workspace** — instructions extract, initial-architecture extract, P1 design draft, source PDFs. | `part2-current/README.md` |
| `visual-paradigm/` | SAPlugin manual + VP lab session. Tooling — Claude must not invoke. | `visual-paradigm/README.md` |
