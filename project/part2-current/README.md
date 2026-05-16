# Part 2 — active workspace

This is where all Part 2 work happens. Three markdown files carry the substantive content; two PDFs are the source-of-truth originals.

> **Prefer `.md` over `.pdf`. PDFs are last resort** — open only when a wording nuance or figure matters that the markdown doesn't preserve. If you find drift between an `.md` and its source PDF, the PDF wins.

## Files

| File | Kind | When to read |
|---|---|---|
| `instructions.md` | **MD extract** of `SA_project_part2_instructions.pdf` | Default reference for the assignment brief: task, "fully supports" criteria, trade-off analysis requirements, the 5 required views, report structure, deadline (**19 May 2026**), and all 6 candidate QASs in full (incl. P1 §7.3 and Av2 §7.5). |
| `initial_architecture.md` | **MD extract** of `SA_project_part2_inital_architecture_including_rationale.pdf` | Default reference for the starting architecture: pre-existing QAS decisions (P2, M2), the four views (C&C / Decomposition / Deployment / Process), component responsibilities, sequence flows, and the placeholders (`OtherFunctionality`, `TODONode`) that Part 2 must replace. |
| `p1_current_design.md` | **Living design draft** for the P1 (Performance — Data exchange with physicians) extension. Mine; edit freely. | Read when reasoning about the P1 proposal. §0 = QAS, §1–§3 = approach, §7 = seams with the teammate's Av2 work. Treat as a draft to stress-test — surface sensitivity / trade-off points the draft misses. |
| `SA_project_part2_instructions.pdf` | Source PDF for `instructions.md` | Last resort only. |
| `SA_project_part2_inital_architecture_including_rationale.pdf` | Source PDF for `initial_architecture.md` | Last resort only. |

## Ownership reminder

- **P1 (Medium priority)** — mine. All P1 design work goes in `p1_current_design.md`.
- **Av2 (High priority)** — teammate's. Do not edit. Coordination notes live in `p1_current_design.md` §7 (seams).

The H+M priority split (P1=M, Av2=H) satisfies the assignment's "one High + one Medium" selection rule (see `instructions.md` §6).

## Existing-architecture QASs (already covered, don't redo)

Per `initial_architecture.md` §2:
- **P2** — Risk-estimation throughput (Performance, Medium).
- **M2** — New/optimized clinical models (Modifiability, High).

The P1 + Av2 extensions must not regress these.
