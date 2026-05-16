# Part 2 — active workspace

This is where all Part 2 work happens. Three markdown files carry the substantive content; two PDFs are the source-of-truth originals.

> **Prefer `.md` over `.pdf`. PDFs are last resort** — open only when a wording nuance or figure matters that the markdown doesn't preserve. If you find drift between an `.md` and its source PDF, the PDF wins.

## Files

| File | Kind | When to read |
|---|---|---|
| `instructions.md` | **MD extract** of `SA_project_part2_instructions.pdf` | Default reference for the assignment brief: task, "fully supports" criteria, trade-off analysis requirements, the 5 required views, report structure, deadline (**19 May 2026**), and all 6 candidate QASs in full (incl. P1 §7.3 and Av2 §7.5). |
| `initial_architecture.md` | **MD extract** of `SA_project_part2_inital_architecture_including_rationale.pdf` | Default reference for the starting architecture: pre-existing QAS decisions (P2, M2), the four views (C&C / Decomposition / Deployment / Process), component responsibilities, sequence flows, and the placeholders (`OtherFunctionality`, `TODONode`) that Part 2 must replace. |
| `p1_current_design.md` | **Living design draft** for the P1 (Performance — Data exchange with physicians) extension. Mine; edit freely. | Read when reasoning about the P1 proposal. Sections: **§0** QAS verbatim, **§1** approach in plain words, **§2** changes vs. initial architecture (2a components, 2b interfaces, 2c deployment), **§3** how P1 is satisfied (3a budget tables, 3b "no impact" argument), **§4** sensitivity points, **§5** trade-offs, **§6** open questions, **§7** coordination with teammate (Av2 seams). Treat as a draft to stress-test — §4/§5/§6 are the richest anchors for surfacing weaknesses. |
| `av2_current_design.md` | **My review notes** of the teammate's Av2 (Availability — High) design. My file; teammate's design lives elsewhere. | Fill in while reading the teammate's Av2 design. Sections: **§0** Av2 QAS verbatim, **§1** approach in my own words, **§2** coverage check vs. response items/measures, **§3** modeling-conventions check, **§4** required-views check, **§5** Part-1-feedback weaknesses, **§6** seams with P1 (mirrors `p1_current_design.md §7`), **§7** ATAM sensitivity / trade-off quality, **§8** red flags, **§9** questions for teammate, **§10** items to fold back into P1, **§11** verdict. |
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

## Strategic constraints for Part 2

These shape every Part 2 design decision:

- **Replace placeholders.** The gray `OtherFunctionality` component and `TODONode` from the initial architecture must be replaced with concrete design elements. See `initial_architecture.md` for what they currently stand in for.
- **End-to-end QAS support.** An extension is only complete if its scenario is covered from stimulus to response measure — every response and every response measure, not just the main flow. Partial coverage = insufficient (see `instructions.md` §2). Examples of "the response measure that anchors the design":
  - **P1**: 2 s / 5 s / 10 s tiered response for status queries (and the "no impact on ingress / risk estimation" clause).
  - **Av2**: 5 s detection window for internal communication subsystem failures.
