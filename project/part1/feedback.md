# Part 1 — Instructor Feedback Reference Extract

Source: `Goddé_Ramaekers_VanVyve_feedback.pdf` (3 pp, 149 KB). Part 1: Feedback on your report.
This file is the lookup-free reference; if something below contradicts the PDF, the PDF wins.

> Why this matters for Part 2: every weakness called out below is a pattern to **avoid repeating** in P1 / Av2 deliverables. Architectural Impact and QAS Response specificity are the biggest scoring gaps.

---

## Overall score

**68% — 20.4 / 30.** Verdict: "satisfactory quality" but **potential left on the table** in deeper reasoning, specificity, attention to detail.

---

## Top-level critique (from "General Comments")

The four recurring weaknesses to avoid repeating in Part 2:

1. **Business value / architectural impact motivations are too brief and high-level.** Many ASRs (in particular ASR 2, ASR 13, ASR 15, …) had short / vague motivations.
2. **Don't restate the requirement when explaining its business value — explain *why* it matters.**
   Example called out: ASR 15 said "crucial for billing" but never explained *why* crucial, or what the business impact of failure would be.
3. **Necessary ≠ sufficient.** When listing architectural-impact requirements, the list was often *not comprehensive* enough to actually achieve the desired quality.
   Example: ASR 13 mentioned the data schema but missed the physician-facing configuration interfaces and the device↔backend config sync mechanism.
4. **Don't repeat the ASR when explaining architectural impact — explain *how* it would be realized.**
   Example: ASR 8 description said "support seamless ML model updates"; the architectural-impact section just repeated that. Should have explained *how* seamless updates are achieved within the architecture.

---

## ASR-specific call-outs

| ASR | Issue |
|---|---|
| **ASR 2** | Business value / arch impact too short and vague. |
| **ASR 8** | Architectural impact merely restates the description ("support seamless ML model updates"). Needs to explain the mechanism. |
| **ASR 13** | Arch impact lists a *necessary* requirement (data schema) but is not *sufficient* — missing the config-entry interfaces and device-sync mechanism. |
| **ASR 14** | **Needs full revision.** Unclear what was being argued. Per the case description PMS is a standalone system; this ASR seems to claim a HIS dependency. Just restating that adds no value. |
| **ASR 15** | "Crucial for billing" stated without explaining *why* crucial or what the business impact would be if unmet. |

---

## QAS-specific call-outs

Overall quality of QAS was good, but specificity should improve:

| QAS | Issue |
|---|---|
| **QAS 1** | Response says "the system detects…" — **which system exactly?** Be precise about which subsystem/component does the detection. |
| **QAS 2** | Response measures are unclear about *how* to measure. Reads like "there should be no errors" + "follow HL7 FHIR spec" but doesn't define a measurement. When an error is retried: are you counting only errors that don't succeed after retry, or every error including those whose retry succeeded? |
| **QAS 3** | Responses are vague and high-level. Be precise about *what kind* of instances/nodes the system should provision (storage? processing?). Possibly missing responses entirely. |

---

## Quality-dimension scores (from the boxplots in §1.2 and §2)

Approximated from the PDF's boxplot figures. Use as *relative weakness signal*, not absolute numbers.

### Per-ASR descriptors (§1.2)

| Dimension | Median | Spread | Notable |
|---|---|---|---|
| Description | ~0.75 | ~0.5–1.0 | Solid. |
| Business Value | ~0.75 | ~0.5–1.0 | Solid. |
| **Architectural Impact** | **~0.5** | **~0.25–1.0** | **Weakest dimension**, plus an **outlier at 0**. This is where the General Comments concentrate. |

### Per-QAS dimensions (§2)

| Dimension | Median | Spread | Notable |
|---|---|---|---|
| Quality / Concreteness | ~0.9 | ~0.75–1.0 | Best dimension. |
| Responses suitability | ~0.6 | ~0.5–0.75 | Weakest QAS dimension. Matches the QAS-specific call-outs above. |
| Response realism | ~1.0 | tight at 1.0 | Strongest — responses were realistic. |

---

## ASR breadth and coverage (§1.1)

> "The coverage of your ASRs is **very good**. You have a lot of variation in different qualities in your report to obtain a diverse set of ASRs."

Coverage was not flagged as a weakness — depth was.

---

## Applied to Part 2 (P1 / Av2)

The feedback pattern below should shape how Part 2 deliverables are written. Each ↦ links a Part 1 weakness to a Part 2 risk:

- **Arch Impact thinness ↦ Trade-off discussions.** The Part 2 instructions (`part2-current/instructions.md` §3) demand a *structured discussion* of impacted ADs, relationship classification, and trade-off acceptability. If the rationale stays at the depth of Part 1's arch-impact sections, this section will lose points.
- **"Necessary ≠ sufficient" ↦ End-to-end QAS coverage.** Part 2 instructions §2 require *all responses and response measures* to be covered. Listing tactics that satisfy some responses but miss others is exactly the Part 1 failure mode.
- **"Don't restate, explain the mechanism" ↦ Architectural rationale sections.** Instructions §5 explicitly say the rationale section's goal is to *explain decisions and rationale*, NOT to describe diagrams. Diagrams are evidence. Same trap as ASR 8.
- **"Which system exactly?" ↦ Component-level precision in sequence diagrams.** Instructions §4.1 requires sequence diagrams to show *which component* detects each exceptional situation. Vague "the system detects" in our QAS responses would be the same anti-pattern at the design level.

The current P1 draft (`part2-current/p1_current_design.md`) already addresses some of these (e.g., §3a budget table, §3b "no impact" argument, §6 open questions). The risk areas to watch are §5 (trade-offs — make sure each row explains *why*, not just states the direction) and §2 (component descriptions — make sure each component's role explains the *mechanism*, not just the responsibility name).
