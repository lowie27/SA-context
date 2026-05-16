# Visual Paradigm — tooling

Materials for the modeling tool used in this course. **Claude should not run or invoke either of these** — per the hard constraints in CLAUDE.md, the user owns all VP work and the SAPlugin. These files exist for human reference.

> **Repo rule: prefer `.md` over `.pdf` everywhere.** No markdown extracts exist for this folder (and none are planned — Claude is not expected to dig into VP internals).

## Files

| File | Purpose | Claude's role |
|---|---|---|
| `SAPluginManual-10.0.0.pdf` | Manual for the custom SAPlugin extension that automates consistency checks against the course modeling conventions and generates LaTeX reports from a `.vpp` model. | **Do not invoke.** The user runs the plugin and pastes output if a check needs to be discussed. Open this PDF only if the user asks a direct question about what a specific plugin check or rule means. |
| `visual-paradigm-lab-session.pdf` | Walkthrough from the VP lab session — how to create components, interfaces (ball/socket), sequence diagrams, deployment nodes in VP 17.3. | Reference only. Useful if the user describes a VP-specific UI step and you need to translate it to the underlying UML concept. |

## Reminders (also in CLAUDE.md, repeated here so this folder is self-contained)

- **Do not edit or generate `.vpp` files.** All UML modeling happens inside Visual Paradigm 17.3, driven by the user.
- **Do not run any LaTeX build.** The SAPlugin's report generation is the user's job.
- For any diagram-related task, describe the structure in markdown (components, interfaces, connectors, nodes) and stop there — the user transcribes it into VP.
