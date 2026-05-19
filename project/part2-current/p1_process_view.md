# P1 Process View — Sequence Diagrams

One sequence diagram per P1 response (instructions.md §4.1, §7.3). Each diagram must show **detection** of the quality-relevant stimulus, the **response**, and the **mechanism enforcing the response measure** — not just normal functional flow.

## Conventions checklist (apply to every diagram)

- **Conv. 6**: every message originates from a lifeline whose component (or module) has the matching **Required Interface** for that operation.
- **Conv. 7**: a single diagram uses **components only** or **modules only**, never both. External actors and external systems are exempt. *Pragmatic reading for module diagrams below*: components that have no sub-modules (`P1PhysicianGateway`, `P1HISAdapter`, `P1PatientGatewayCommander`, `P1PatientQueryService`, `P1PatientStatusCache`) appear as boundary lifelines when crossed — there is no module-level alternative, so the rule's intent (no mixing of decomposed vs. undecomposed views of the **same** element) is preserved.
- **Mandatory descriptions**: every lifeline and every operation must have a description in VP's General tab.
- **Explicit references**: every message refers to an actual model operation (not free text). Internal state mutations (cache populate, queue dispatch, tier-index upsert, drop policy) are **notes** on a lifeline, not messages.
- **Datatypes prefix**: parameter/return types are prefixed with `Datatypes.` in the VP model. Omitted below for readability.

## Diagram inventory

| # | Diagram | Response covered | Granularity |
|---|---|---|---|
| 1 | UC6 — Consult patient status (high-risk tier) | Response 1 | components |
| 2 | UC8 — Update risk level + EHR forward | Response 3 | modules |
| 3 | UC5 — Notification fan-out (normal) | Response 4 | modules |
| 4 | UC5 — Notification overload (drop-green-first) | Response 5 | modules |
| 5 | UC9 — On-demand consultation (correlated, prioritized) | Response 2 | modules |

Response 6 (no impact on ingest / risk estimation) is argued in the rationale and shown by the Deployment view + the absence of shared-path edges across diagrams 1–5; no dedicated sequence diagram is needed.

---

## 1. Diagram 1 — UC6 Consult Patient Status (Response 1)

### 1.1 Purpose

Show the tiered SLA (high → 2 s, medium → 5 s, low → 10 s) being enforced for a patient-info request at ≥ 25 req/min.

- **Stimulus**: physician calls `consultPatientStatus(patientId)` for a high-risk patient.
- **Detection**: in-memory tier index in `P1PatientQueryService` classifies the request as HIGH in O(1) (unknown patients fall back to HIGH so the SLA fails safe).
- **Response**: request routed to the high-tier priority queue; served via cache-first read.
- **Mechanism enforcing 2 s**:
  - Priority tier queue (high / medium / low) prevents head-of-line blocking by lower tiers.
  - High-tier worker pool sized for the 25 req/min × HIGH share, with a 2 s deadline on each dequeued job.
  - `P1PatientStatusCache` pins high-risk patients, so the hot path is cache-served.
  - Tier index kept fresh by a parallel subscription to `P1IRiskEvents` (concurrent scenario below).

### 1.2 Lifelines (components — Conv. 7)

| Role | Lifeline | Type |
|---|---|---|
| Physician (actor) | `:Physician` | actor |
| Gateway | `gw : P1PhysicianGateway` | component |
| Query service | `qry : P1PatientQueryService` | component |
| Status cache | `cache : P1PatientStatusCache` | component |
| Patient data service | `data : P1PatientDataService` | component |

`P1PatientStatusModule` is a sub-module of `P1PatientDataService` — at component granularity it sits behind the `OtherDataMgmt` socket on `data`.

### 1.3 Message flow

**Main scenario — HIGH tier, cache hit**

| # | From | To | Message (operation) | Interface used |
|---|---|---|---|---|
| 1 | Physician | gw | `consultPatientStatus(patientId)` | `P1IPhysicianAPI` (gw provides) |
| 2 | gw | qry | `consultPatientStatus(patientId)` | `P1IPatientQuery` (gw requires) |
| 3 | qry | cache | `getPatientStatus(patientId)` | `P1IPatientStatusRead` (qry requires) |
| 4 | cache | qry | *return* `status : PatientStatus` | (return arrow) |
| 5 | qry | gw | *return* `status : PatientStatus` | (return arrow) |
| 6 | gw | Physician | *return* `status : PatientStatus` | (return arrow) |

Between msg 2 (in) and msg 3 (out), `qry` performs **detection** (tier-index lookup) and **mechanism** (high-tier dispatch with 2 s deadline). These are internal state — see notes in §1.4.

**Cache-miss variant — replaces msg 4 with 4a–4c**

| # | From | To | Message | Interface |
|---|---|---|---|---|
| 4a | cache | data | `getPatientStatus(patientId)` | `OtherDataMgmt` (cache requires) — provided at the `P1PatientDataService` boundary by `P1PatientStatusModule` |
| 4b | data | cache | *return* `status : PatientStatus` | (return arrow) |
| 4c | cache | qry | *return* `status : PatientStatus` | (return arrow) |

After msg 4b, `cache` pins the entry if the patient is HIGH (read-through populate). State mutation — note on lifeline, no message.

**Concurrent scenario — tier-index refresh** (separate trigger; not part of the request flow)

| # | From | To | Message | Interface |
|---|---|---|---|---|
| P1 | data | qry | `onRiskEvent(event)` | `P1IRiskEventListener` (qry provides) |

`data` emits via `P1IRiskEvents` whenever `setEstimatedPatientStatus` actually changes a patient's status. On receipt, `qry` updates its internal tier index. This keeps msg 2's classification O(1) and current.

### 1.4 Notes to attach in VP

Attach each note on the named lifeline, anchored between the surrounding messages.

- **qry, between msg 2 and msg 3** — *"Detection: tier-index lookup is O(1) on the hot path; unknown patients default to HIGH so the SLA fails safe. Tier index is internal state of P1PatientQueryService — not an interface operation."*
- **qry, same gap (second note)** — *"Mechanism: high-tier worker dequeues the request and arms the 2 s deadline. Lower tiers run in parallel queues so they cannot induce head-of-line blocking."*
- **cache, after msg 4b (miss branch)** — *"Cache pins HIGH-risk patients on populate (read-through), keeping the hot path off the slow `OtherDataMgmt` path."*
- **qry, after msg P1** — *"Tier-index upsert keeps the msg 2 lookup O(1) and current."*
- **Diagram-level note** — *"Sensitivity point: if the `P1IRiskEvents` subscription breaks, the tier classification silently degrades. See `p1_notes.md` §4."*

### 1.5 Response-measure budget (2 s for HIGH)

| Segment | Budget | Justification |
|---|---|---|
| Network in (physician → gw) | 200 ms | TLS-terminated client channel; gw co-located with backend (see Deployment) |
| gw routing + msg 2 dispatch | 50 ms | stateless router |
| Tier lookup (qry internal, before msg 3) | < 1 ms | in-memory hash on patientId |
| Queue wait (qry internal, before msg 3) | ≤ 200 ms | HIGH-tier worker pool sized for the 25 req/min × HIGH share |
| Cache hit (msg 3–4) | ≤ 50 ms | in-process read |
| Cache miss fallback (msg 4a–4c) | ≤ 500 ms | adds one `OtherDataMgmt` round-trip; still fits inside 2 s |
| Return path (msg 5–6) | 200 ms | symmetric to ingress |
| **Total (hit path)** | **≈ 500 ms** | well inside 2 s |
| **Total (miss path)** | **≈ 1.0 s** | inside 2 s |
| Headroom for jitter / GC / contention | ≈ 1 s | covers tail latency at 25 req/min |

Budget numbers are first-cut; refine if profiling data appears in Part 3.

### 1.6 What's deliberately NOT on this diagram

- **Medium / Low tier flows.** Same lifelines, same path, different queue and deadline. One diagram is enough to make the mechanism visible; mention M/L variants in the rationale.
- **Tier-index population on startup.** Cold-start path — outside the steady-state QAS environment ("normal mode").
- **Cache eviction.** Implementation detail of `P1PatientStatusCache`; belongs in the Decomposition view of that component, not here.

---

## 2. Diagram 2 — UC8 Update Risk Level + EHR Forward (Response 3)

### 2.1 Purpose

Show the write path for a cardiologist's risk-level update at ≥ 20 updates/min, with the EHR forwarded within 1 minute.

- **Stimulus**: cardiologist calls `updatePatientRiskLevel(patientId, status)`.
- **Detection**: `P1CommandRouter` dispatches the operation to `P1RiskLevelHandler` by operation identity.
- **Response**: update internal status (which fires `P1IRiskEvents`), then forward to the HIS-backed EHR.
- **Mechanism enforcing 1 min**:
  - Write to internal status is a same-node, in-memory mutation — sub-second.
  - `PatientRecordMgmt.updatePatientRecord` is a synchronous call into the EHR proxy; `P1HISAdapter` performs the actual `healthAPI` save. HIS latency is the dominant term.
  - EHR-proxy cache is invalidated structurally on PMS-initiated writes so the next UC16 read sees the new value (coherence is mechanical, not time-based).
  - 20 updates/min × ≤ 2 s HIS round-trip leaves wide slack inside the 60 s budget.

### 2.2 Lifelines (modules — Conv. 7)

| Role | Lifeline | Kind |
|---|---|---|
| Cardiologist (actor) | `:Cardiologist` | actor |
| Gateway boundary | `gw : P1PhysicianGateway` | boundary (no sub-modules) |
| Command router | `router : P1CommandRouter` | module |
| Risk-level handler | `rlh : P1RiskLevelHandler` | module |
| Patient-status module | `psm : P1PatientStatusModule` | module |
| EHR proxy | `ehr : P1EHRProxyModule` | module |
| HIS adapter boundary | `his : P1HISAdapter` | boundary (no sub-modules) |
| External HIS | `:HIS` | external system |

### 2.3 Message flow

**Main scenario — write + EHR forward**

| # | From | To | Message (operation) | Interface used |
|---|---|---|---|---|
| 1 | Cardiologist | gw | `updatePatientRiskLevel(patientId, status)` | `P1IPhysicianAPI` (gw provides) |
| 2 | gw | router | `updatePatientRiskLevel(patientId, status)` | `P1IPhysicianCommand` (gw requires; router provides at the `P1PhysicianCommandService` boundary) |
| 3 | router | rlh | `updatePatientRiskLevel(patientId, status)` | `P1IRiskLevelCommand` (router requires) |
| 4 | rlh | psm | `setEstimatedPatientStatus(patientId, status, timestamp)` | `OtherDataMgmt` (rlh requires; psm provides at the `P1PatientDataService` boundary) |
| 5 | psm | rlh | *return* | (return arrow) |
| 6 | rlh | ehr | `updatePatientRecord(patientId, record)` | `PatientRecordMgmt` (rlh requires; ehr provides at the `P1PatientDataService` boundary) |
| 7 | ehr | his | `updatePatientRecord(patientId, record)` | `P1IHISAccess` (ehr requires) |
| 8 | his | HIS | *(healthAPI save calls — `saveObservation` / `saveRiskAssessment` / `savePatient` as needed)* | `healthAPI` (his requires) |
| 9 | HIS | his | *return* | (return arrow) |
| 10 | his | ehr | *return* | (return arrow) |
| 11 | ehr | rlh | *return* | (return arrow) |
| 12 | rlh | router | *return* | (return arrow) |
| 13 | router | gw | *return* | (return arrow) |
| 14 | gw | Cardiologist | *return* | (return arrow) |

**Concurrent scenario — risk-event fan-out** (asynchronous side-effect of msg 4)

| # | From | To | Message | Interface |
|---|---|---|---|---|
| E1 | psm | (subscribers: `P1RiskEventSubscriber`, `P1PatientQueryService`, `P1OnDemandConsultationHandler`) | `onRiskEvent(event)` | `P1IRiskEventListener` (subscribers provide) |

Fan-out happens only if msg 4 actually changes the stored status. UC5 notification fan-out continues from here (diagram 3).

### 2.4 Notes to attach in VP

- **psm, between msg 4 and msg 5** — *"If `status` differs from the stored value, psm fires `P1IRiskEvents` to all matching subscribers (msg E1). Emission is internal; the interface operation that triggers it is `setEstimatedPatientStatus`."*
- **ehr, between msg 6 and msg 7** — *"Cache invalidate is structural on every PMS-initiated write — next read repopulates from HIS. External HIS edits are not observed (sensitivity point, `p1_notes.md` §4)."*
- **Diagram-level note** — *"Synchronous EHR forward keeps the UC8 result coupled to HIS responsiveness. With ≤ 2 s typical HIS round-trip and 20 writes/min, the 1-min EHR-forward SLA holds with > 50× headroom."*

### 2.5 Response-measure budget (1 min EHR forward)

| Segment | Budget | Justification |
|---|---|---|
| Network in (cardiologist → gw) | 200 ms | TLS client channel |
| gw → router → rlh (msg 2–3) | 50 ms | same node, stateless dispatch |
| `setEstimatedPatientStatus` (msg 4–5) | ≤ 50 ms | cross-node into PatientDataNode; in-memory write + event emit |
| `updatePatientRecord` (msg 6–11) | ≤ 5 s typical | dominated by HIS round-trip via `healthAPI`; bounded by HIS-side SLA |
| Return path (msg 12–14) | 200 ms | symmetric to ingress |
| **Total typical** | **≈ 5.5 s** | well inside 60 s |
| Headroom for HIS slowness / retries | ≈ 54 s | covers HIS outages within the SLA |

### 2.6 What's deliberately NOT on this diagram

- **Subscriber-side handling of the risk event.** Each subscriber's behaviour has its own diagram (UC5 → diagram 3/4; UC9 correlation → diagram 5; tier-index refresh → diagram 1).
- **UC7 configure-risk-assessment path.** Same router pattern but routes to `P1ConfigurationHandler` (different interface, different mechanism). Not part of Response 3.
- **HIS authentication.** Out of P1 scope (open question OQ3 in `p1_notes.md` §6).

---

## 3. Diagram 3 — UC5 Notification Fan-out, Normal Mode (Response 4)

### 3.1 Purpose

Show the fan-out from a single risk event to all subscribed physicians, with priority-tier SLAs (red ≤ 10 s, yellow ≤ 30 s, green ≤ 1 min) under the 100 notifications/min normal-mode threshold.

- **Stimulus**: `P1PatientStatusModule` fires a `P1IRiskEvents` event after a status change.
- **Detection**: `P1RiskEventSubscriber` receives the event via `P1IRiskEventListener` and looks up recipients.
- **Response**: each recipient receives a notification, persisted (red/yellow) before being enqueued for pull.
- **Mechanism enforcing tier SLAs**:
  - `P1PriorityBuffer` orders red > yellow > green inside each physician's queue.
  - Red and yellow are written to `P1DurableNotificationLog` **before** the `P1IRiskEvents` ack, so a crash cannot lose them.
  - Pull-based delivery via `P1INotificationInbox` — physicians poll the gateway; tier SLAs are met by client poll cadence + buffer ordering.

### 3.2 Lifelines (modules — Conv. 7)

| Role | Lifeline | Kind |
|---|---|---|
| Patient-status module | `psm : P1PatientStatusModule` | module |
| Risk-event subscriber | `sub : P1RiskEventSubscriber` | module |
| Recipient registry | `reg : P1RecipientRegistry` | module |
| Durable log | `log : P1DurableNotificationLog` | module |
| Priority buffer | `buf : P1PriorityBuffer` | module |
| Inbox module | `inbox : P1NotificationInboxModule` | module |
| Gateway boundary | `gw : P1PhysicianGateway` | boundary (no sub-modules) |
| Physician (actor) | `:Physician` | actor |

### 3.3 Message flow

**Scenario A — fan-out path (event → buffer, per notification)**

| # | From | To | Message | Interface |
|---|---|---|---|---|
| 1 | psm | sub | `onRiskEvent(event)` | `P1IRiskEventListener` (sub provides; psm requires) |
| 2 | sub | reg | `findRecipients(patientId)` | `P1IRecipientRegistry` (sub requires) |
| 3 | reg | sub | *return* `List<PhysicianId>` | (return arrow) |

For each recipient (loop / per-recipient frame):

| # | From | To | Message | Interface |
|---|---|---|---|---|
| 4 | sub | log | `persist(notification)` | `P1INotificationLog` (sub requires) — **red/yellow only** |
| 5 | log | sub | *return* | (return arrow) |
| 6 | sub | buf | `enqueue(physicianId, notification)` | `P1IPriorityBuffer` (sub requires) |
| 7 | buf | sub | *return* | (return arrow) |

**Scenario B — pull path (physician reads pending, separate trigger)**

| # | From | To | Message | Interface |
|---|---|---|---|---|
| 8 | Physician | gw | `getPendingNotifications(physicianId)` | `P1IPhysicianAPI` (gw provides) |
| 9 | gw | inbox | `getPendingNotifications(physicianId)` | `P1INotificationInbox` (gw requires; inbox provides at the `P1NotificationDispatcher` boundary) |
| 10 | inbox | buf | `peekPending(physicianId)` | `P1IPriorityBuffer` (inbox requires) |
| 11 | buf | inbox | *return* `List<Notification>` | (return arrow) |
| 12 | inbox | gw | *return* `List<Notification>` | (return arrow) |
| 13 | gw | Physician | *return* `List<Notification>` | (return arrow) |

**Scenario C — acknowledge path (after delivery)**

| # | From | To | Message | Interface |
|---|---|---|---|---|
| 14 | Physician | gw | `acknowledgeNotification(notificationId)` | `P1IPhysicianAPI` |
| 15 | gw | inbox | `acknowledgeNotification(notificationId)` | `P1INotificationInbox` |
| 16 | inbox | buf | `removePending(notificationId)` | `P1IPriorityBuffer` |
| 17 | inbox | log | `markDelivered(notificationId)` | `P1INotificationLog` |

### 3.4 Notes to attach in VP

- **sub, between msg 3 and the per-recipient loop** — *"Loop iterates over recipients returned by msg 3. Severity drives whether msg 4 runs (red/yellow persist; green skips persist)."*
- **sub, between msg 4 and msg 6** — *"Order matters: persist before enqueue, and only ack the upstream `P1IRiskEvents` event after enqueue returns. This is the durability boundary — see §3.5."*
- **buf, on the enqueue lifeline** — *"Priority ordering is red > yellow > green within each physician's queue. Normal mode: no drops."*
- **Diagram-level note** — *"Pull-based delivery: tier SLA is met by buffer ordering + a poll cadence at the gateway. Push delivery is a future option but out of P1 scope."*

### 3.5 Response-measure budget (red ≤ 10 s, yellow ≤ 30 s, green ≤ 60 s)

| Segment | Budget | Justification |
|---|---|---|
| Event delivery (msg 1) | ≤ 50 ms | same-process callback inside PMS |
| Recipient lookup (msg 2–3) | ≤ 10 ms | in-memory registry |
| Durable persist (msg 4–5, red/yellow) | ≤ 100 ms | local write-ahead log |
| Enqueue (msg 6–7) | < 5 ms | in-memory buffer |
| Poll cadence (physician → buffer head-of-line) | red ≤ 5 s; yellow ≤ 15 s; green ≤ 30 s | client-side poll interval per tier |
| Return path on poll (msg 10–13) | 200 ms | symmetric |
| **Total red** | **≈ 5.5 s** | inside 10 s |
| **Total yellow** | **≈ 15.5 s** | inside 30 s |
| **Total green** | **≈ 30.5 s** | inside 60 s |
| Headroom | ≥ 50 % per tier | covers buffer contention at 100 / min |

### 3.6 What's deliberately NOT on this diagram

- **Overload regime (drop-green-first).** Separate diagram (§4) because the mechanism is different (rate-limited enqueue, drop policy).
- **Av2 emergency dispatch for red.** Handled at the seam (`p1_current_design.md` §7); appears in §4 because it is only triggered under overload + red.
- **Subscription management (`subscribeToNotifications` / `unsubscribeFromNotifications`).** Setup path, not steady-state QAS behaviour.
- **Replay on startup (`recoverPending`).** Recovery path, outside normal mode.

---

## 4. Diagram 4 — UC5 Notification Overload, Drop-Green-First (Response 5)

### 4.1 Purpose

Show the overload policy that preserves red and yellow when the system is asked to send more than 100 notifications/min — drop green first, never drop red or yellow.

- **Stimulus**: incoming notification rate at `P1RiskEventSubscriber` exceeds 100/min (sustained, not instantaneous burst).
- **Detection**: `P1PriorityBuffer` tracks the per-window enqueue rate; on `enqueue` it inspects rate + severity.
- **Response**:
  - Green notifications are dropped at the buffer; red and yellow always proceed.
  - Red gains escalation via `Av2EmergencyDispatch` (backup channel) so it survives even if the inbox path is congested.
- **Mechanism enforcing "no red or yellow lost"**:
  - Red/yellow persisted to `P1DurableNotificationLog` **before** any rate check.
  - Drop policy in `P1PriorityBuffer` keys off severity — only the green class is eligible for drop.
  - Av2 backup channel is independent of `P1IPhysicianAPI` poll path.

### 4.2 Lifelines (modules — Conv. 7)

| Role | Lifeline | Kind |
|---|---|---|
| Patient-status module | `psm : P1PatientStatusModule` | module |
| Risk-event subscriber | `sub : P1RiskEventSubscriber` | module |
| Recipient registry | `reg : P1RecipientRegistry` | module |
| Durable log | `log : P1DurableNotificationLog` | module |
| Priority buffer | `buf : P1PriorityBuffer` | module |
| Av2 emergency dispatch | `av2 : Av2EmergencyDispatch` | external interface (Av2 teammate) |

### 4.3 Message flow

**Scenario A — green notification under overload (drop)**

| # | From | To | Message | Interface |
|---|---|---|---|---|
| 1 | psm | sub | `onRiskEvent(event)` *(severity = green)* | `P1IRiskEventListener` |
| 2 | sub | reg | `findRecipients(patientId)` | `P1IRecipientRegistry` |
| 3 | reg | sub | *return* `List<PhysicianId>` | (return arrow) |
| 4 | sub | buf | `enqueue(physicianId, notification)` *(per recipient)* | `P1IPriorityBuffer` |
| 5 | buf | sub | *return* | (return arrow) |

Between msg 4 and msg 5, `buf` inspects rate + severity. If rate > 100/min **and** severity = green, the notification is dropped. State mutation — note on `buf` (§4.4).

**Scenario B — red notification under overload (persist + enqueue + escalate)**

| # | From | To | Message | Interface |
|---|---|---|---|---|
| 6 | psm | sub | `onRiskEvent(event)` *(severity = red)* | `P1IRiskEventListener` |
| 7 | sub | reg | `findRecipients(patientId)` | `P1IRecipientRegistry` |
| 8 | reg | sub | *return* `List<PhysicianId>` | (return arrow) |

For each recipient:

| # | From | To | Message | Interface |
|---|---|---|---|---|
| 9 | sub | log | `persist(notification)` | `P1INotificationLog` |
| 10 | log | sub | *return* | (return arrow) |
| 11 | sub | buf | `enqueue(physicianId, notification)` | `P1IPriorityBuffer` |
| 12 | buf | sub | *return* | (return arrow) |
| 13 | sub | av2 | *(escalate red — operation defined on `Av2EmergencyDispatch`; signature owned by Av2)* | `Av2EmergencyDispatch` (sub requires) |

Yellow follows scenario B without msg 13 (no escalation).

### 4.4 Notes to attach in VP

- **buf, between msg 4 and msg 5** — *"Drop policy: if windowed enqueue rate > 100/min and notification.severity == green, the entry is dropped at the buffer. Red and yellow are never eligible. Drop counters are exported for observability."*
- **sub, between msg 9 and msg 11** — *"Durability invariant: red and yellow are persisted in the log before they enter the buffer. The upstream `P1IRiskEvents` ack happens **after** msg 11 returns. A crash between msg 9 and msg 11 is replayable via `recoverPending`."*
- **sub, on msg 13** — *"Red-only escalation over Av2's backup channel. Independent of the `P1INotificationInbox` poll path so a stalled inbox cannot delay a red. Signature owned by Av2 — wire in VP once Av2 publishes it."*
- **Diagram-level note** — *"Sensitivity point: setting the rate window or the 'green only' policy wrong breaks Response 5. The threshold (100/min) is a sensitivity parameter — see `p1_notes.md` §4."*

### 4.5 Response-measure budget

| Segment | Budget | Justification |
|---|---|---|
| Event delivery (msg 1 / msg 6) | ≤ 50 ms | same-process callback |
| Recipient lookup | ≤ 10 ms | in-memory |
| Persist (msg 9–10, red/yellow only) | ≤ 100 ms | local WAL |
| Enqueue + drop check (msg 4 or msg 11) | < 5 ms | rate counter + severity test |
| Av2 escalation (msg 13, red only) | ≤ 500 ms | seam latency owned by Av2; budget guess pending Av2 numbers |
| **Red end-to-end (overload)** | **≈ 700 ms** | well inside 10 s SLA |
| **Yellow end-to-end (overload)** | **≈ 200 ms to buffer**, then pull cadence (≤ 15 s) | inside 30 s SLA |
| **Green** | dropped under sustained overload — does not contribute to SLA |
| Headroom for transient bursts | large | drop policy is the safety valve |

### 4.6 What's deliberately NOT on this diagram

- **Normal-mode fan-out and pull / ack paths.** Covered in diagram 3.
- **Av2's internal handling of the escalation.** Owned by the Av2 teammate; this diagram shows the outbound seam call only.
- **Burst handling vs. sustained overload distinction.** The QAS treats overload as a rate condition; transient bursts are absorbed by the buffer. Tuning notes belong in `p1_notes.md` §4.

---

## 5. Diagram 5 — UC9 On-Demand Consultation (Response 2)

### 5.1 Purpose

Show the correlated, prioritized flow for an on-demand consultation: initiation within 3 minutes of the request, result delivered within 1 minute of risk estimation completion, with no polling on the client side.

- **Stimulus**: cardiologist calls `requestOnDemandConsultation(patientId, physicianId)` at up to 20 req/min.
- **Detection**: `P1OnDemandConsultationHandler` mints a `CorrelationId` and tracks it in `P1CorrelationTracker`.
- **Response**:
  - Fetch current sensor data from the patient gateway (synchronous, bounded timeout).
  - Store the data with `triggerEstimation=false` so the implicit scheduled job is suppressed.
  - Launch a priority risk-estimation job carrying the `CorrelationId`.
  - Return the `CorrelationId` immediately (sync part ends here).
  - When the matching `P1IRiskEvents` event arrives, push the result via `P1IConsultationResultPush`.
- **Mechanism enforcing the SLAs**:
  - **3-min initiation**: bounded `P1IOnDemandSensorFetch` timeout + priority lane on `LaunchRiskEstimation`.
  - **1-min result**: priority risk job jumps ahead of scheduled ones; correlation-driven push avoids polling.

### 5.2 Lifelines (modules — Conv. 7)

| Role | Lifeline | Kind |
|---|---|---|
| Cardiologist (actor) | `:Cardiologist` | actor |
| Gateway boundary | `gw : P1PhysicianGateway` | boundary (no sub-modules) |
| Command router | `router : P1CommandRouter` | module |
| On-demand handler | `odh : P1OnDemandConsultationHandler` | module |
| Correlation tracker | `ct : P1CorrelationTracker` | module |
| Patient-gateway commander | `pgc : P1PatientGatewayCommander` | boundary (no sub-modules) |
| Sensor ingest | `ing : P1SensorIngestModule` | module |
| Risk-estimation scheduler | `sched : RiskEstimationScheduler` | boundary (no sub-modules in this scope) |
| Patient-status module | `psm : P1PatientStatusModule` | module |

### 5.3 Message flow

**Scenario A — synchronous initiation (returns within 3 min)**

| # | From | To | Message | Interface |
|---|---|---|---|---|
| 1 | Cardiologist | gw | `requestOnDemandConsultation(patientId, physicianId)` | `P1IPhysicianAPI` |
| 2 | gw | router | `requestOnDemandConsultation(patientId, physicianId)` | `P1IPhysicianCommand` |
| 3 | router | odh | `requestOnDemandConsultation(patientId, physicianId)` | `P1IOnDemandCommand` |
| 4 | odh | ct | `newCorrelationId()` | `P1ICorrelationTracking` |
| 5 | ct | odh | *return* `correlationId : CorrelationId` | (return arrow) |
| 6 | odh | pgc | `requestCurrentSensorData(patientId, correlationId)` | `P1IOnDemandSensorFetch` |
| 7 | pgc | odh | *return* `sensorData : SensorDataPackage` | (return arrow) |
| 8 | odh | ing | `addSensorData(patientId, sensorData, receivedAt, triggerEstimation=false)` | `SensorDataMgmt` |
| 9 | ing | odh | *return* | (return arrow) |
| 10 | odh | sched | `launchRiskEstimation(patientId, priority=HIGH, correlationId, newSensorData=sensorData, receivedAt)` | `LaunchRiskEstimation` |
| 11 | sched | odh | *return* | (return arrow) |
| 12 | odh | router | *return* `correlationId` | (return arrow) |
| 13 | router | gw | *return* `correlationId` | (return arrow) |
| 14 | gw | Cardiologist | *return* `correlationId` | (return arrow) |

**Scenario B — asynchronous result push (later, when estimation finishes)**

The estimation pipeline eventually causes `RiskEstimationCombiner` to call `OtherDataMgmt.setEstimatedPatientStatus`, which fires `P1IRiskEvents` carrying the `correlationId`. The handler receives the matching event:

| # | From | To | Message | Interface |
|---|---|---|---|---|
| 15 | psm | odh | `onRiskEvent(event)` | `P1IRiskEventListener` (odh provides; psm requires) |
| 16 | odh | ct | `consumeCorrelation(correlationId)` | `P1ICorrelationTracking` |
| 17 | ct | odh | *return* `true` | (return arrow) |
| 18 | odh | gw | `deliverConsultationResult(physicianId, correlationId, status)` | `P1IConsultationResultPush` (gw provides) |
| 19 | gw | Cardiologist | *push over open client channel* | (out-of-PMS path) |

If msg 17 returns `false`, the event is not for this handler — the message stops at msg 17 and `odh` does nothing.

### 5.4 Notes to attach in VP

- **odh, between msg 7 and msg 8** — *"`pgc` reaches out to the patient gateway via a stub interface that is not yet wired in VP (open question OQ4 in `p1_notes.md` §6). Synchronous, bounded by the commander timeout."*
- **odh, on msg 8** — *"`triggerEstimation=false` suppresses the implicit scheduled job in `P1SensorIngestModule`. Without this, every UC9 fires two risk jobs (sensitivity point #7 in `p1_notes.md` §4)."*
- **sched, on msg 10** — *"`LaunchRiskEstimation` is extended by P1 with `priority` and `correlationId` parameters. HIGH priority jumps ahead of scheduled jobs; `correlationId` is carried through to the resulting `P1IRiskEvents` event."*
- **odh, between msg 16 and msg 17** — *"`consumeCorrelation` is the matching step: a `true` result confirms this event belongs to a tracked UC9 request; a `false` result means another subscriber will handle it (e.g., the UC5 fan-out)."*
- **Diagram-level note** — *"No polling on the client: msg 14 returns the `correlationId` immediately and msg 19 is a separate push. The Av2 seam must preserve `P1CorrelationTracker` state across failover or the result push silently fails (Av2 coordination point — `p1_current_design.md` §7)."*

### 5.5 Response-measure budget

| Segment | Budget | Justification |
|---|---|---|
| **Initiation budget (3 min total, msg 1–14)** | | |
| Network in (cardiologist → gw) | 200 ms | TLS client channel |
| gw → router → odh (msg 2–3) | 50 ms | same node, stateless |
| Correlation mint (msg 4–5) | < 5 ms | in-memory |
| Sensor fetch (msg 6–7) | ≤ 30 s typical, ≤ 90 s timeout | bounded by commander timeout; dominated by patient-gateway round-trip |
| Sensor store (msg 8–9) | ≤ 100 ms | cross-node into PatientDataNode; in-memory write |
| Scheduler enqueue (msg 10–11) | ≤ 50 ms | priority lane on scheduler |
| Return path (msg 12–14) | 200 ms | symmetric |
| **Total initiation (typical)** | **≈ 30 s** | well inside 3 min |
| **Total initiation (worst, fetch timeout)** | **≈ 90 s** | inside 3 min |
| **Result-delivery budget (1 min from estimation completion, msg 15–19)** | | |
| Event delivery (msg 15) | ≤ 50 ms | same-process callback |
| Correlation consume (msg 16–17) | < 5 ms | in-memory |
| Push to gateway (msg 18) | ≤ 50 ms | cross-node |
| Push to client (msg 19) | ≤ 500 ms | open client channel |
| **Total result delivery** | **≈ 1 s** | well inside 1 min |
| Headroom | large | covers contention on the priority lane |

### 5.6 What's deliberately NOT on this diagram

- **Risk-estimation internals.** The path from `sched` (msg 10) through `RiskEstimationCombiner` back to `psm` (msg 15) traverses inherited components; not part of P1's design responsibility.
- **Patient-gateway side of the sensor fetch.** Bounded by the commander; the outbound stub interface is OQ4.
- **HIS forward of the result.** UC9 result delivery does not include an EHR write — that path belongs to UC8 (diagram 2).
- **Failover preservation of `P1CorrelationTracker`.** Av2 coordination, not P1 process behaviour.
