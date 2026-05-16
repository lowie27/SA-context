# Av2 — Teammate's Proposed Approach (my capture for review)

> **Not active yet.** Don't worry about Av2 for now — I'll start filling this in once my
> P1 work is finished. Until then, this file is just a skeleton; ignore it when reasoning
> about the current design.
>
> My working capture of the teammate's Av2 (Availability — High) design.
> Structure mirrors `p1_current_design.md` so the two can be compared side-by-side.
> I do **not** edit teammate's files (per CLAUDE.md). Findings + questions go in §6 and §7;
> coordination outcomes flow back into `p1_current_design.md §7`.

---

## 0. QAS Av2 (verbatim from instructions PDF)

> Paste the Av2 QAS exactly as written in the instructions PDF. Do not paraphrase.

- **Source:** TODO
- **Stimulus:** TODO
- **Artifact:** TODO
- **Environment:** TODO
- **Response:** TODO
- **Response measure:** TODO

---

## 1. Approach in plain words

> Two or three paragraphs summarising the teammate's approach in my own words. If I
> can't summarise it concisely, that's a finding (motivation depth — per Part 1 feedback).

TODO — core idea.

TODO — main structural change vs. initial architecture (which placeholder(s) replaced, which nodes added, which existing components touched).

TODO — key mechanism(s) — replication? heartbeats? leader-election? hot-standby? circuit breaker? — and why they fit the QAS.

---

## 2. What changes vs. the initial architecture

### 2a. Components

**VP entry rules — components get descriptions, interfaces get operations.** (Same as P1 §2a.)
- Components need a **description** only: one short, concise sentence (≤ ~20 words). Source = the `description:` line below.
- Interfaces (§2b) carry the operations.

new components (fill in from teammate's design)

- `TODO_ComponentName`
    - description: TODO (one short sentence)
    - node: TODO
    - provides:
        - `TODO_InterfaceName`
    - requires:
        - `TODO_InterfaceName`
- `TODO_ComponentName`
    - description: TODO
    - node: TODO
    - provides:
        - `TODO_InterfaceName`
    - requires:
        - `TODO_InterfaceName`
- `TODO_DecomposedComponent`
    - description: TODO
    - node: TODO
    - provides:
        - `TODO_InterfaceName`
    - requires:
        - `TODO_InterfaceName`
    - `decomposed` into modules (decomposition view — Conv. 5):
        - `TODO_ModuleName`
            - description: TODO
            - provides:
                - `TODO_InterfaceName`
            - requires:
                - `TODO_InterfaceName`
        - `TODO_ModuleName`
            - description: TODO
            - provides:
                - `TODO_InterfaceName`
            - requires:
                - `TODO_InterfaceName`

retire / relocate

- TODO — list any components from the initial architecture that Av2 retires, relocates, or extends. (Convention 4 reminder: if a parent's interface is retired, at least one child must still expose it.)

modified existing components

- `TODO_ExistingComponent` — TODO what changed (extra interface? replicated? failover added?)

### 2b. Interfaces

new interfaces (fill in from teammate's design)

- `TODO_InterfaceName`
    - provided by: `TODO_Component`
    - required by:
        - `TODO_Component`
    - operations (all `+`):
        - `TODO_operationName(ParamType) → ReturnType` — TODO purpose

- `TODO_InterfaceName`
    - provided by: `TODO_Component`
    - required by:
        - `TODO_Component`
    - operations (all `+`):
        - `TODO_operationName(ParamType) → ReturnType` — TODO purpose

existing interfaces extended by Av2 (signatures originate in §E.3 of `initial_architecture.md`)

- `TODO_ExistingInterface`
    - provided by: `TODO`
    - required by:
        - TODO
    - operations (mark which are new / changed):
        - existing: `TODO_operationName(...) → ...` — §E.3.X
        - **extended by Av2**: `TODO_operationName(...) → ...` — TODO what Av2 added and why

existing interfaces reused verbatim (no change vs. initial architecture)

- TODO — list them with one line each (provided by / required by). Refer to §E.3 for signatures.

### 2c. Deployment changes

> Mirror the P1 §2c sketch: which nodes are added/retired/relocated, which communication paths are added.

- New nodes: TODO
- Retired/relocated nodes: TODO
- Communication paths (Conv. 9):
    - TODO ↔ TODO (interfaces crossing the link)
    - TODO ↔ TODO (interfaces crossing the link)

---

## 3. How this satisfies Av2

> Av2 is High-priority Availability. The "satisfies" argument is structurally different from P1's
> latency budgets — but the bones are the same: take each response item / measure and show the
> mechanism that achieves it.

### 3a. Detection-and-recovery argument

One subsection per response measure that has a quantitative ceiling (e.g., "5 s detection window
for internal communication subsystem failures" per `README.md` §Strategic constraints).

| Step | Mechanism | Budgeted time | Note |
| --- | --- | --- | --- |
| Failure occurs | TODO | t = 0 | |
| Failure detected | TODO (heartbeat? timeout? health check?) | TODO | |
| Recovery decision | TODO (leader election? watchdog?) | TODO | |
| Recovery action | TODO (failover? restart? reroute?) | TODO | |
| Service restored | TODO | TODO | **must be ≤ QAS ceiling** |

### 3b. Coverage of remaining response items

One bullet per response item not anchored to a numeric measure. Each bullet: the response item, the mechanism, and *why it is sufficient* (not just plausible — per Part 1 feedback "necessary ≠ sufficient").

- TODO
- TODO
- TODO

### 3c. Non-regression of existing QASs

Av2 must not break P2 (risk-estimation throughput) or M2 (clinical-model modifiability) — per the README's existing-QAS constraint. Brief argument that it doesn't.

- TODO

---

## 4. Sensitivity points

> Av2 decisions where a small parameter change swings availability a lot. Mirror P1 §4 in style.

1. TODO — e.g., heartbeat interval. Set too short → false-positive failovers. Set too long → exceeds 5 s detection ceiling.
2. TODO
3. TODO

---

## 5. Trade-offs

> Av2 buys availability with something — latency? cost? modifiability? predictability? Make the trade-off explicit.
> Mirror P1 §5's table format.

| Attribute | Impact | Why |
| --- | --- | --- |
| Performance (P1, P2) | TODO | TODO |
| Modifiability (M2) | TODO | TODO |
| Cost / complexity | TODO | TODO |
| Predictability | TODO | TODO |

The central trade-off: TODO — "we accept X regression in QA Y to achieve the Z s detection ceiling, because [reason]." This sentence is what the grader looks for.

---

## 6. Open questions (mine, for the teammate)

> Concrete, numbered. Things I want to ask before the VP model is finalised.

1. TODO
2. TODO
3. TODO

---

## 7. Coordination with P1 (inverse of `p1_current_design.md §7`)

> My P1 §7 lists what P1 needs Av2 to know about. This section captures what Av2 needs P1 to
> know about — fill in as you read the teammate's design.

### Hard overlaps (need explicit agreement)

1. TODO — e.g., "Av2 replicates `RiskEstimationScheduler` for failover. P1's added `priority` + `correlationId` parameters on `LaunchRiskEstimation` must survive a failover so UC9's correlationId still maps to the eventual `IRiskEvents` event."
2. TODO
3. TODO

### Soft overlaps (touch shared structure; usually compatible)

4. TODO
5. TODO

### FYI (decisions P1 should know about; no compromise needed)

6. TODO
7. TODO

---

## 8. Review notes (my checks against course constraints)

> Lightweight self-check while reading Av2. Doesn't replace ATAM-style reasoning above — just a tickbox.

### 8a. Modeling-conventions check (CLAUDE.md §2)

| # | Convention | Status | Notes |
| --- | --- | --- | --- |
| 1 | Globally unique names | ☐ | TODO |
| 2 | Viewpoint → correct UML diagram type | ☐ | TODO |
| 3 | Balls/sockets only for interface wiring | ☐ | TODO |
| 4 | No residual functionality | ☐ | TODO |
| 5 | Modules use `<<module>>` | ☐ | TODO |
| 6 | Sequence calls only via required interfaces | ☐ | TODO |
| 7 | Sequence diagrams: components OR modules, not both | ☐ | TODO |
| 8 | Runtime components inside nodes | ☐ | TODO |
| 9 | Communicating components share node or have comm path | ☐ | TODO |
| — | Every element has a description | ☐ | TODO |
| — | Sequence messages refer to model operations | ☐ | TODO |

### 8b. Required-views check

| View | Present? | Cross-view consistent? | Notes |
| --- | --- | --- | --- |
| 1. System context | ☐ | ☐ | TODO |
| 2. Client–Server | ☐ | ☐ | TODO |
| 3. Process (sequence) | ☐ | ☐ | TODO |
| 4. Decomposition | ☐ | ☐ | TODO |
| 5. Deployment | ☐ | ☐ | TODO |

### 8c. Part-1-feedback weaknesses to watch

| Weakness | Triggered? | Where | Notes |
| --- | --- | --- | --- |
| Motivation depth (no *why*) | ☐ | TODO | |
| Necessary ≠ sufficient | ☐ | TODO | |
| Mechanism vs. restatement | ☐ | TODO | |
| Precision (vague terms) | ☐ | TODO | |

---

## 9. Items to fold back into my own P1 design

> If reviewing Av2 reveals a P1 gap (e.g., I assumed a single scheduler instance that Av2
> contradicts), capture it here and propagate to `p1_current_design.md` §7 (or wherever it belongs).

- TODO
- TODO

---

## 10. Verdict

> One paragraph. Strong / partial / weak coverage of Av2's QAS. Top 2–3 things Av2 must
> address before submission.

TODO
