# Part 2 Instructions — Reference Extract

Source: `SA_project_part2_instructions.pdf` (15 pp, 240 KB).
This file is the lookup-free reference; if something below contradicts the PDF, the PDF wins.

Academic year **2025–2026**. Deadline: **19 May 2026**.

---

## 1. Task overview (§1)

Extend the initial PMS architecture in a **quality-driven** way (not feature-driven). Four tasks:

1. **Select two QASs** from the list in §7 below — one **High** priority and one **Medium** priority.
2. Design + document an architectural extension that fully supports the two selected QASs **and** the QASs already baked into the initial architecture (P2, M2).
3. Critically analyze how the extension **interacts with, modifies, or conflicts with** decisions in the initial architecture and within the extension itself.
4. Motivate every design decision with architectural rationale and quality-attribute reasoning.

You **may** change original decisions, but if you do, explicitly state which decision changed, the impact, and why the change was necessary. Mind QAS priorities.

### 1.1 ATAM-inspired reasoning (not a full ATAM)

You must identify and reason about:

- **Sensitivity points** — decisions/parameters where small changes have large impact on one or more QAs (e.g., deployment locality vs. performance/availability).
- **Trade-off points** — decisions that improve one QA while degrading another; usually also sensitivity points for multiple attributes.

Full ATAM is **not** required, but the report must (i) identify sensitivity points introduced/affected, (ii) discuss QA trade-offs explicitly, (iii) justify why the trade-offs are acceptable in the PMS context.

---

## 2. What "fully supports a requirement" means (§1.1)

Coverage must be **end-to-end** across all QAS elements. Partial coverage = insufficient.

| QAS element | What the extension must do |
|---|---|
| **Stimulus** | Clearly identify the triggering event/condition; must be realistic and consistent with the PMS context. |
| **Artifact** | Specify which part(s) of the system are affected (service, DB, channel, pipeline) and how they are protected/adapted/controlled. |
| **Environment** | Describe operational conditions (normal, degraded, peak, failure, maintenance). Justify decisions w.r.t. that environment, not an idealized one. |
| **Responses** | Address **all** required responses (functional + quality-related: detection, isolation, recovery, degradation, notification, enforcement, prevention). Omitting any response = incomplete. |
| **Response measures** | For each response, explain how the architecture meets the quantitative/qualitative constraint (time bounds, availability %, retention, detection latency, recovery time). Mechanism responsible for each measure must be explicit. |

All responses + response measures must be traceable to architectural elements and illustrated via sequence diagrams where appropriate.

---

## 3. Trade-off analysis requirements (§1.2)

The report must include a **structured discussion** that:

1. **Identifies impacted architectural decisions from the initial architecture** — explicitly reference the ADs and rationale documented for the initial design. State which assumptions, constraints, or tactics are affected (e.g., centralized processing, synchronous comms, shared DBs, trust assumptions).
2. **Explains the relationship to existing decisions** — for each impacted AD, classify the extension as one of:
   - reinforces / strengthens (same tactic, extended scope)
   - weakens / constrains (reduced flexibility, tighter coupling, stricter timing)
   - requires the decision to be revised or partially abandoned
   This must be argued explicitly, not implicitly assumed.
3. **Justifies acceptability of the trade-offs** — argue why introduced trade-offs are acceptable given QAS priorities, stakeholder concerns, and operational environment. Risks/limitations must be acknowledged explicitly.

Rationale must match the spirit and depth of the initial architecture's rationale sections. Arguments grounded in architectural principles + QA reasoning, traceable to concrete changes in diagrams and sequence diagrams.

---

## 4. Required architectural views (§1.3)

The extension must be reflected **consistently** across these views. Decisions not visible in diagrams (components, interfaces, responsibilities, deployment) are insufficiently documented. Changes in one view must show up in the others.

| View | Purpose |
|---|---|
| **System context diagram** | PMS as a single boundary + external actors/systems and high-level interactions. Use to make explicit what is outside your control and where stimuli originate. |
| **Client–Server view** (C&C) | Main runtime components/services, provided/required interfaces, who-calls-whom. Use to show how the extension changes service interactions, responsibilities, and interface contracts. |
| **Process view** (sequence diagrams) | Runtime behavior for the chosen QAS, including normal flow + quality-related responses. Make detection and reaction to quality-relevant stimuli explicit (failures, overload, violations) and show response-measure enforcement (timeouts, retries, prioritization, isolation, recovery, auditing). |
| **Decomposition view** | Internal structure of a component into modules + responsibilities + dependencies. Use to explain *how* a component achieves the QAS responses (scheduler module, cache, policy module, circuit breaker, etc.). |
| **Deployment view** | Mapping of software elements to hardware/virtual nodes + comm links. Use to argue QAs that depend on deployment choices (availability via replication, performance via locality, failure isolation, scaling). |

### 4.1 Quality-driven sequence diagrams

For **each major response** of the selected QAS, at least one sequence diagram that:

- Goes beyond normal functional flow.
- Explicitly shows **detection** of the exceptional situation (overload, failure, violation), the **system's response**, and the **mechanism enforcing the response measure** (timeouts, deadlines, retries, isolation, etc.).

---

## 5. Report structure (§2)

### Design approach

Either systematic (e.g., Attribute-Driven Design) or exploratory/ad hoc — both allowed. Either way, plan carefully: unstructured trial-and-error and repeated backtracking are flagged as signs of insufficient architectural reasoning. Identify early which parts of the initial architecture the extension will touch and which ADs will interact/conflict.

### Required sections

1. **Introduction** (optional) — position the QAS, scope the extension.
2. **Architectural decisions for the selected QAS** (core section, one per chosen QAS):
   - (a) Brief overview of principal decisions + tactics/patterns used.
   - (b) Detailed rationale: how + why the decisions support the QAS, explicitly referencing architectural elements (components, interfaces, nodes) and views. Must be a **self-contained, complete explanation** of how the QAS is addressed, including all responses and response measures. Use `nameref` for diagram/catalog references.
   - (c) Short discussion of **alternatives considered** and why they were rejected (insufficient QAS coverage, unacceptable trade-offs, complexity).
   - **Goal of this section is to explain decisions + rationale, NOT to describe/paraphrase diagrams.** Diagrams are evidence of decisions, not a substitute for rationale.
3. **Architectural trade-offs and sensitivity points** — impact of the extension on the initial architecture; key trade-offs and sensitivity points and how they influenced design decisions. Must connect to the rationale in §2.
4. **Final architecture** — final diagrams + element catalog, generated from the final Visual Paradigm project using the SAPlugin.

### Formatting rules

- Use the provided **LaTeX template** (auto-exported by SAPlugin).
- Diagrams + element catalog **must** be SAPlugin-generated from the `.vpp`.
- Use `nameref` for consistent references to architectural elements and diagrams.

### Delivery

Deadline **19/05** (= 19 May 2026). Two artifacts required:
1. The architecture report (PDF), incl. diagrams + element catalog.
2. The Visual Paradigm Project (`.vpp`) source file.

---

## 6. Selection constraint (recap)

Must pick **one High + one Medium** priority QAS from §7. Priority is shown in parentheses next to each QAS name.

| Priority | QASs available |
|---|---|
| High (H) | Av1, Av2 |
| Medium (M) | Sec1, M1, P1, Priv1 |

> **Active selection in this project**: P1 (M) = mine, Av2 (H) = teammate's. ✓ Satisfies the one-H + one-M rule.

---

## 7. QAS catalogue (§3)

All six QASs from the assignment, verbatim. Two are selected per team (see §6).

### 7.1 Sec1 — Patient identification (M)

PMS needs to reliably identify the patient across information flows: incoming sensor data → wearable identity → patient identity; gateway → backend identity; outgoing notifications → correct recipient.

- **Source**: Unregistered user or patient gateway / Registered patient gateway.
- **Stimulus**:
  - Malicious actor presents crafted/falsified sensor data (UC4: send sensor data) to the patient gateway.
  - Malicious actor presents crafted/falsified sensor data (UC4) to the PMS backend services.
  - Malicious actor falsely sends an emergency notification (UC3) via an unregistered/tampered patient gateway.
  - New sensor data (UC4) or emergency notification (UC3) for a patient is received via a gateway registered to *another* patient.
- **Artifact**: Subsystems that process incoming requests.
- **Environment**: Normal operation.
- **Responses**:
  1. Patient registration links wearable/sensor device IDs to the patient's identity.
  2. PMS app access is subject to authentication (username + password).
  3. Gateway↔backend interaction is authenticated per-gateway before any operation.
  4. Patient info is access-controlled (patient sees own data; only treating physician sees clinical data).
  5. Confidentiality and integrity preserved on the wire between app and backend.
  6. Every backend interaction is logged (timestamp, requester ID (IP if pre-auth), operation, granted/denied).
- **Response measures**:
  1. Computationally infeasible to guess auth info (proper crypto keys; long random IDs).
  2. Computationally infeasible to bypass auth. Passwords 8–64 chars, not commonly-used (verify with `zxcvbn` or `haveibeenpwned`). Patient identity verified during registration.
  3. Computationally infeasible to spoof a registered app (public-key crypto or secure API tokens).
  4. Computationally infeasible to bypass access control. Each app only gets data about its associated patient.
  5. Computationally infeasible to breach transit confidentiality or modify data without detection.
  6. Every request logged before processing. Logs kept ≥ 2 years.

### 7.2 M1 — Include third-party lifestyle tracker data (M)

Allow PMS to consume data from third-party fitness/lifestyle trackers (e.g., Strava, Google Fit) for opt-in patients, to contextualize clinical model computation.

- **Source**: Research department.
- **Stimulus**: Wishes to improve clinical model accuracy by incorporating additional data types.
- **Artifact**: Clinical model computation, patient dashboard, communication subsystem(s).
- **Environment**: At design time or runtime.
- **Responses**:
  1. Patient app easily extensible to let patients grant permission + set auth tokens for third-party retrieval.
  2. PMS easily extensible with dedicated communication subsystem(s) per third-party backend, incl. periodic sync.
  3. No impact on clinical model computation subsystem beyond *possibly* a new model accepting the new data types.
  4. No change to existing interface(s) for sensor data and HIS info.
  5. No change to existing storage subsystems for raw sensor data or patient profiles.
  6. Performance / availability of third-party services must not affect performance / availability of risk estimation.
- **Response measures**:
  1. App extension ≤ 1 person-month to develop/test/deploy.
  2. New communication subsystem ≤ 3 person-months to develop/test/deploy.
  3. Once deployed, adding another third-party service ≤ 2 person-weeks.

### 7.3 P1 — Data exchange with physicians (M)  ← mine

PMS exchanges data with physicians (cardiologist on-demand consultations; notifications to registered physicians when risk level changes). These exchanges must not hinder medical services and must be timely.

- **Source**: Physician.
- **Stimulus**:
  - Physician consults patient status (UC6).
  - Cardiologist requests current data (UC9: on-demand consultation).
  - Cardiologist (re)configures risk assessment (UC7).
  - Cardiologist updates risk level (UC8) after reviewing a notification.
  - PMS looks up recipients and sends a notification about patient status (UC5).
- **Artifact**: Subsystem(s) handling requests from physicians and sending notifications.
- **Environment**: Normal mode.
- **Responses**:
  1. System processes all patient-info requests in a timely fashion.
  2. Risk assessment triggered by on-demand consultation / config initiated in a short time frame; remote consultation results delivered timely.
  3. Risk-level changes processed in a timely fashion.
  4. Notifications about status changes sent to all appropriate recipients in a timely fashion, as long as count stays below threshold.
  5. When notification count exceeds the threshold, the system prioritizes more urgent ones.
  6. Large notification/request volume must **not** impact (a) raw sensor data ingest or (b) ongoing risk estimations.
- **Response measures**:
  1. ≥ 25 patient-info requests/min (UC6):
     - High-risk → 2 s
     - Medium-risk → 5 s
     - Low-risk → 10 s
  2. ≥ 20 on-demand consultations and/or risk-assessment configs per minute (UC9 / UC7):
     - On-demand-consultation risk assessments **prioritized over** scheduled risk estimations on new sensor data.
     - Each on-demand-triggered risk assessment (UC12) initiated within 3 min of request, regardless of patient risk level.
     - On-demand results delivered to physician within 1 min after completion.
  3. 20 risk-level updates/min (UC8): EHR forwarded within 1 min.
  4. ≥ 100 notifications/min (UC5):
     - Red → all recipients within 10 s
     - Yellow → all recipients within 30 s
     - Green → all recipients within 1 min
  5. When > 100 notifications/min: red prioritized over yellow, yellow over green. **No red or yellow notifications may be lost.**

### 7.4 Av1 — Internal PMS database failure (H)

The internal raw-sensor-data store fails/crashes. Significant downtime would impede risk estimation that relies on historical data + persistence of new data.

- **Source**: Internal.
- **Stimulus**: Internal subsystem storing raw sensor data readings fails or crashes.
- **Artifact**: Internal subsystem.
- **Environment**: Normal execution.
- **Responses**:
  1. Other persistent-data availability (registered parties, pairings, auth credentials, risk results, notifications) is unaffected.
  2. Crash causes no loss of already-stored sensor data, no inconsistency, no loss of integrity.
  3. **Prevention**: storage subsystem has a minimal guaranteed up-time.
  4. **Detection**: PMS admin notified on crash; PMS detects + goes into degraded mode. During downtime: incoming readings buffered elsewhere then added on recovery; clinical computation requests for sensor data fail gracefully (reschedule or inform user).
  5. **Resolution**: Admin replaces failed hardware / restarts software / reverts to backup. PMS automatically backfills the missed readings after recovery.
- **Response measures**:
  1. **Prevention** (raw sensor DB availability, measured per month):
     - High-risk patients: ≥ 99%
     - Medium-risk: ≥ 95%
     - Low-risk: ≥ 90%
  2. **Detection**: admin notified within 5 min of detection; hardware/software failure detected within 1 min of crash. For high/medium-risk patients: seamless transition to temporary backup storage (no disruption, no data updates omitted regardless of size/frequency). For low-risk: at most 2 of the regular updates may be omitted.
  3. **Resolution**: storage restart within 1 min; rollback to previous/backup state within 5 min.

### 7.5 Av2 — Communication between patient gateways and PMS backend (H)  ← teammate

PMS depends on gateway↔backend communication for sensor data, emergency notifications, and on-demand consultations. Failures: (1) intermediate telecom infra, (2) gateway, (3) internal comm subsystem.

- **Source**: External or internal.
- **Stimulus**:
  - External comm channel between gateway and PMS becomes unavailable (failure or scheduled maintenance).
  - Patient gateway becomes unavailable (battery, network).
  - Internal communication subsystem of PMS fails/crashes.
- **Artifact**: External communication channel(s), external device(s), internal subsystem(s).
- **Environment**: Normal executions.
- **Responses**:
  1. **Prevention**:
     - PMS has negotiated SLA with telecom operator: channel ≥ 99.5%/year availability, ≥ 80% mobile coverage in operating region, average ≥ 64 Kb/s bandwidth.
     - Gateway app warns patient when battery low.
     - Gateway app warns patient when network connectivity is missing/limited.
  2. **Detection**:
     - PMS backend autonomously detects failures based on lack of sensor data updates from one or more gateways.
     - PMS backend autonomously detects failures of relevant internal comm subsystems.
     - Gateway app autonomously detects comm failures and goes into degraded mode: temporarily store sensor data + notifications for later sync; systematic retries; possible backup channels.
     - PMS tracks duration of communication outages.
  3. **Resolution**:
     - PMS proactively notifies relevant stakeholders of the detected problem:
       - Failing comm subsystem → admin must redeploy failing component(s) or revert.
       - Failing comm channel → admin must contact telecom operator.
       - Failing patient gateway → patient should be contacted.
     - In degraded mode, gateway app uses a back-up channel for emergency notifications (e.g., SMS) and keeps retrying sensor data delivery with exponential back-off; stores readings + unsent notifications for the past 24 h.
- **Response measures**:
  1. **Prevention**: app warns patient when gateway battery ≤ 15%.
  2. **Detection**:
     - Detection time depends on configured transmission rate (and indirectly on risk level) but does **not** exceed it by more than 5 min. If an expected update fails to arrive at time *T*, detection happens before *T* + 5 min.
     - PMS detects failures of internal comm subsystems within **5 s** after the failure.  ← **5 s is the canonical Av2 detection figure**
  3. **Resolution**:
     - In 90% of cases, admin notification arrives within 2 min (high risk), 5 min (medium), 10 min (low) after detection.
     - Comm-subsystem redeploy or rollback ≤ 5 min.
     - SLA with telecom operator: support response within 10 min, resolution within the hour.

### 7.6 Priv1 — Right of access requests by patients (M)

GDPR-style "right of access": users can obtain a copy of personal data processed by the system + info about processing purpose, recipients. Requires collecting data from multiple subsystems and possibly external systems (HIS) — must comply without hurting core functionality (e.g., risk estimation).

- **Source**: User.
- **Stimulus**: A user requests access to their personal data being collected and processed by the PMS.
- **Artifact**: Internal subsystem.
- **Environment**: Normal mode.
- **Responses**:
  1. Multiple right-of-access requests processable simultaneously.
  2. All received requests processed in a reasonable time frame.
  3. Returned archive covers all relevant personal data processed up to (at least) the time the request was received: patient profile incl. data kept on patient Gateway; all past risk estimations; all processed sensor readings; data used from EHR; patient-specific risk-assessment config (e.g., tailored clinical model).
  4. For each personal-data item in the archive, also include: purpose(s) of processing; any additional recipients; retention duration; source (if not collected from the patient themselves).
  5. The constructed archive remains available for download for a sufficiently long time.
  6. Collecting personal/additional data to build the archive has **no impact** on: UC4 (send sensor data), UC12 (estimate risk level), UC3 (send emergency notification), UC9 (on-demand consultation).
  7. System verifies user identity prior to processing the request and prior to download.
- **Response measures**:
  1. ≥ 3 right-of-access requests processable simultaneously.
  2. Per request:
     - Reply on whether personal data is/has been processed: within 1 h.
     - Archive available: within 48 h.
  3. Generated archive remains downloadable for ≥ 1 week.

---

## 8. Quick map for this project

| Selected QAS | Section | Owner | File where the design lives |
|---|---|---|---|
| **P1 — Data exchange with physicians (M)** | §7.3 | mine | `p1_current_design.md` |
| **Av2 — Gateway↔backend comm (H)** | §7.5 | teammate | (teammate's workspace — do not edit) |

Initial architecture already covers **P2** (Performance — risk-estimation throughput) and **M2** (Modifiability — clinical model updates). See `initial_architecture.md` §2.
