# P1 Notes

Reasoning, sensitivity points, trade-offs, open questions, and coordination with the Av2 teammate. The pure element catalog lives in `p1_design.md`; this file is everything that's *about* the design rather than the design itself.

---

## 1. QAS P1 (verbatim from instructions PDF)

- **Source**: Physician
- **Stimulus**:
    - A physician consults a patient's status (UC6 : consult patient status).
    - A cardiologist requests a patient's current data (UC9 : perform on-demand consultation).
    - A cardiologist (re)configures the risk assessment for a patient (UC7 : configure patient risk assessment).
    - A cardiologist updates the risk level of a patient (UC8 : update risk level).
    - The PMS looks up the appropriate recipients and sends a notification about a patient's status (UC5 : notify registered parties).
- **Artifact**: The subsystem(s) responsible to handle requests from physicians and send out notifications.
- **Environment**: Normal mode
- **Response**:
    1. The system is able to process all requests for patient information in a timely fashion.
    2. Any risk assessment triggered by an on-demand consultation or risk assessment configuration is initiated within a short time frame of the request. In case of remote on-demand consultations, the results of the risk assessment are provided to the physician in a timely fashion.
    3. The system is able to process all changes to a patient's risk level in a timely fashion.
    4. The system is able to send notifications concerning (changes to) a patient's status to all appropriate recipients in a timely fashion, as long as the number of notifications to send stays within a threshold.
    5. When the number of notifications to send exceeds the threshold the system prioritizes more urgent notifications.
    6. A large number of notifications to be sent or requests arriving to the system should have no impact on (a) the processing of incoming raw sensor reading data; or (b) ongoing risk estimations.
- **Response measure**:
    1. At minimum 25 requests for patient information per minute (UC6). High-risk in 2 s, medium in 5 s, low in 10 s.
    2. At least 20 on-demand consultation / risk assessment configuration requests per minute (UC9, UC7). UC9 risk assessments prioritised over scheduled ones; each UC9 estimation initiated within 3 minutes; UC9 result returned within 1 minute after completion.
    3. 20 risk-level updates per minute (UC8); EHR forwarded within 1 minute.
    4. ≥100 notifications per minute (UC5); red ≤ 10 s, yellow ≤ 30 s, green ≤ 1 min.
    5. Over the 100/min threshold: red > yellow > green; no red or yellow may be lost.

---

## 2. Approach in plain words

Split `OtherFunctionality` across **two separate nodes** so physician-facing work cannot contend with sensor ingest or risk estimation (this is what Response item 6 demands):

1. **PhysicianNode** — a thin `P1PhysicianGateway` plus four back-end services (`P1PhysicianCommandService`, `P1PatientQueryService`, `P1NotificationDispatcher`, `P1PatientGatewayCommander`). All physician traffic terminates here. Tiered SLAs (2/5/10 s queries, 10/30/60 s notifications) are realised by priority queues.
2. **PatientDataNode** — a `P1PatientDataService` that takes over the ingest / store side of `OtherFunctionality`: it receives raw sensor data, exposes `SensorDataMgmt`, `PatientRecordMgmt`, `OtherDataMgmt` to the existing risk-estimation pipeline, and writes patient status back when `RiskEstimationCombiner` finishes a job. `P1PatientStatusCache` and `P1HISAdapter` also live here.

The two nodes communicate over the network. Because they don't share OS resources, a spike in physician requests or notifications cannot starve the ingest path or the risk-estimation processors. `P1PatientStatusCache` sits on `PatientDataNode` next to `P1PatientDataService`, so cache misses are local; cross-node traffic from PhysicianNode is bounded by physician-initiated read/write rate, not by cache misses.

> **Status note**: per the current VP model, `OtherFunctionality` and `TODONode` are NOT yet retired and `ClinicalModelDB` is still on `TODONode`. The structural intent above is the target; the model still carries the placeholders.

---

## 3. Why this satisfies P1

### 3a. The 2 s response-measure budget

The 2 s ceiling for high-risk queries (Response measure 1) is generous — the real constraint is **tiered SLAs** under contention. With `P1PatientStatusCache` co-located with `P1PatientDataService` on `PatientDataNode`, the worst case is one cross-node hop into PatientDataNode and then a same-node miss-fallback inside it.

| Hop | Estimated | Note |
| --- | --- | --- |
| Network in (physician client → P1PhysicianGateway) | ~50 ms | TLS-terminated REST/gRPC |
| P1PhysicianGateway route | ~10 ms | thin router |
| P1PatientQueryService dequeue (priority: high) | ~20 ms | bounded queue, head-of-line for high-risk |
| Cross-node call → P1PatientStatusCache (on PatientDataNode) | ~30 ms | LAN within eHealthPlatform |
| Cache lookup (hit) | ~5 ms | |
| Cache lookup (miss) + same-node OtherDataMgmt read | ~50 ms | local fall-through, no extra network hop |
| Return + serialize | ~50 ms | |
| **Total (cache hit)** | **~165 ms** | |
| **Total (cache miss)** | **~210 ms** | well under 2 s; ~1.8 s slack |

Under load, the priority queue ensures high-risk requests jump ahead; medium-risk get up to 5 s, low-risk up to 10 s. Because the cache is on PatientDataNode the miss path is one *local* read rather than another network hop — cheap, but means a flood of queries always goes cross-node first.

**Other response measures (rougher numbers):**

- **UC8 risk-level update + 1-min EHR forward** — write path P1PhysicianGateway → P1PhysicianCommandService → PatientRecordMgmt.updatePatientRecord (cross-node, ~30 ms) → P1EHRProxyModule (cache invalidate + same-node P1HISAdapter call). HIS round-trip dominates; even at multi-second HIS latency, total stays well under the 60 s SLA. Cache coherence is structural — any subsequent UC16 read from RiskEstimationCombiner sees the new value.
- **UC9 on-demand consultation, 3-min init** — P1PhysicianCommandService mints a correlationId, calls `P1IOnDemandSensorFetch` (P1PatientGatewayCommander → patient gateway, sync round-trip, bounded by the commander timeout). On reply: writes the package via `SensorDataMgmt.addSensorData(...)` and calls `LaunchRiskEstimation(priority=HIGH, correlationId)`. **Open**: the current VP `addSensorData` has no `triggerEstimation` flag (§5 OQ5), so a redundant scheduled risk job may fire on top of the priority UC9 one until that is resolved.
- **UC9 result delivery in 1 min** — P1PhysicianCommandService subscribes to `P1IRiskEvents` filtered by correlationId; pushes result back to physician via long-poll or websocket on `P1INotificationInbox`.
- **UC5 100 notifications/min** — P1NotificationDispatcher priority queue. Red bypasses everything (10 s SLA). Green can queue up to 60 s. Under overload, green is dropped first; red/yellow are persisted to `P1DurableNotificationLog` before being acked from `P1IRiskEvents` so they cannot be lost. Red also has access to the Av2 emergency backup channel via the new `Av2EmergencyDispatch` requirement on `P1RiskEventSubscriber`.

### 3b. The "no impact on ingress" argument

Response item 6 is the strongest constraint in the QAS. With the current VP layout (cache on PatientDataNode), the argument has three legs:

1. **Resource separation by node.** PhysicianNode and the risk-estimation tier (RiskEstimationMgmtNode + RiskEstimationProcessorNode) are physically separate nodes (Conv. 8/9). A spike in physician traffic consumes PhysicianNode resources only; it does not contend with the risk-estimation processors or the scheduler.
2. **No synchronous calls into the risk pipeline from the physician side.** `P1IRiskEvents` is **push** (decision recorded in `p1_design.md`). `P1NotificationDispatcher` and `P1PhysicianCommandService` subscribe; they never poll or query `RiskEstimationCombiner`. A notification storm does not generate incoming load on the risk pipeline.
3. **Bounded cross-node read load *on the data path*.** Every UC6 query goes cross-node into PatientDataNode (cache lives there). The cache absorbs the second-level load against `OtherDataMgmt` — i.e., the cache protects the **data store**, not the network link. PhysicianNode → PatientDataNode bandwidth scales with the *query* rate (≤25/min per QAS, bounded). The cache keeps PatientDataNode's CPU off the slow `OtherDataMgmt` path on repeated reads.

The residual cross-node writes from P1PhysicianCommandService (UC7 config to ClinicalModelDB, UC8/UC17 EHR write via PatientRecordMgmt, UC9 fetched-data write via SensorDataMgmt) DO touch PatientDataNode (and TODONode in the case of UC7, since ClinicalModelDB still lives there). Mitigation: these rates are bounded by the QAS itself (≤25 + 20 + 20 + 20 per minute — UC6/7/8/9 combined), orders of magnitude below the sensor-ingest write rate.

**Caveat from cache-on-PatientDataNode placement**: with the cache co-located with `P1PatientDataService`, a flood of physician reads still consumes PatientDataNode CPU. That node also handles sensor ingest (via `SensorDataMgmt`). Non-interference for ingress therefore relies on the cache being cheap enough that physician-driven load on PatientDataNode stays well below ingest-driven load. If that ever flips, this leg of the argument weakens — see §4 sensitivity #1 / #4.

**Caveat on UC9**: the current VP model lacks `triggerEstimation=false` on `addSensorData`, so until that is added the UC9 path fires *two* risk jobs (the implicit one from `P1SensorIngestModule` plus the priority one P1PhysicianCommandService explicitly launches). That doubles load on `RiskEstimationScheduler` for on-demand traffic and partially undermines leg #1 for the UC9 case. See §4 sensitivity #7.

---

## 4. Sensitivity points

Decisions where a small parameter change swings P1 a lot:

1. **P1PatientStatusCache hit ratio + its placement on PatientDataNode.** If the cache hit ratio drops, every UC6 read hits `OtherDataMgmt` on the same node. Because the cache lives on PatientDataNode, miss traffic stays local — but the *cross-node* request volume from PhysicianNode is already proportional to the query rate. Smaller margin than the earlier design (cache on PhysicianNode) would have given.
2. **Push-vs-pull on P1IRiskEvents.** Flipping this from push to pull (e.g., `P1NotificationDispatcher` polling for status changes) immediately violates Response item 6 — load on the risk pipeline becomes proportional to notification volume.
3. **Priority-queue depth / drop policy in P1NotificationDispatcher.** Setting the threshold for "overload" wrong (or letting green starve infinitely) breaks Response item 5.
4. **Co-location of P1PatientDataService with P1PatientStatusCache on PatientDataNode.** PatientDataNode now hosts both ingest-side data (`SensorDataMgmt`) and the physician-facing cache. If physician read load on the cache ever grows comparable to ingest CPU, the non-interference argument's third leg (§3b) weakens. This is the central trade-off introduced by the report's cache placement.
5. **P1PhysicianGateway thinness.** If anyone adds business logic to the gateway (auth caching, request aggregation, rate-limit math), it becomes a single CPU-bound chokepoint for all five UCs.
6. **Patient gateway availability for UC9.** `P1PatientGatewayCommander` makes a synchronous outbound call to the patient gateway during UC9. If the gateway is offline or slow, the 3-min initiation SLA is at risk before any PMS-internal work even begins. The commander's timeout has to leave enough budget for a retry and for the subsequent priority risk job.
7. **`triggerEstimation` discipline on `SensorDataMgmt.addSensorData`.** The current VP model has **no** `triggerEstimation` parameter — so today every UC9 on-demand consultation fires *two* risk jobs (the implicit one from `P1SensorIngestModule` plus the priority one from `P1PhysicianCommandService`). Doubles load on `RiskEstimationScheduler` for on-demand traffic and could starve scheduled jobs. Resolve by adding the flag (or a sibling "store-without-trigger" operation) in VP, then enforcing it at every UC9 call site.

---

## 5. Trade-offs

| Attribute | Impact | Why |
| --- | --- | --- |
| Modifiability | **−** moderate | ~8 new top-level P1 components + 2 new nodes (PhysicianNode, PatientDataNode) plus the still-present TODONode. Future changes to the physician-side API ripple through Gateway + CommandRouter + handlers. Mitigated by clear seams between query / command / notification / outbound-fetch. |
| Availability | **−** small | More moving parts → more failure modes. PhysicianNode is a single point of failure for *all* physician UCs (Av2's domain — flag for teammate). P1NotificationDispatcher buffering means red/yellow notifications must survive a crash — backed by P1DurableNotificationLog. |
| Performance (other QASs) | **+** small | The split helps P2/M2: physician load no longer competes with risk-estimation throughput. Slightly less win than the original design intended because the cache is on PatientDataNode (shared with ingest), not isolated on the physician side. |
| Cost / complexity | **−** moderate | Two new nodes to operate. Cross-node calls add latency budget pressure (manageable given the 2 s ceiling). |
| Predictability | **+** | Resource isolation makes worst-case behaviour easier to reason about; non-interference is structural, not best-effort. |

The hardest trade-off is between **modifiability** (more components) and **predictability** (resource isolation). The QAS demands predictability (the "no impact" clause), so the modifiability hit is the right cost to pay — this is the central architectural trade-off point.

---

## 6. Open questions

1. ~~Where does the notification recipient registry live?~~ Resolved: `P1RecipientRegistry` module inside `P1NotificationDispatcher`.
2. **How is P1PatientStatusCache populated initially / kept warm for high-risk patients?** Need a policy: write-through on `setEstimatedPatientStatus`? Background pre-warm of all high-risk patients on startup?
3. **HIS integration auth model.** `healthAPI` is provided by an external HIS; how does `P1HISAdapter` authenticate? Out-of-scope for P1 but blocks UC8's EHR forwarding deliverable.
4. **Patient gateway auth model for outbound PMS → gateway calls.** UC9 introduces a new outbound direction. The existing inbound channel (UC4) authenticates via the gateway's token. How does the *PMS* authenticate to the gateway? Mirror of OQ3 for HIS — out of P1 scope but blocks UC9 deliverable. Currently the VP model has no required interface on `P1PatientGatewayCommander` at all — pick a stub interface name and wire it.
5. **`triggerEstimation` representation in VP.** The current model lacks any way to suppress the implicit risk-estimation trigger from `addSensorData`. Options: add a flag parameter (foot-gun, see §4 sensitivity #7), add a sibling `addSensorDataWithoutEstimation` operation, or live with the redundant UC9 job. Decide and update VP + design.
6. **ConsultationId vs CorrelationId.** Report E.6 notes `ConsultationId: TODO maybe delete/merge into CorrelationId`. Decide whether to keep two distinct types (semantic clarity: external handle vs internal trace) or collapse to one.
7. **OtherFunctionality / TODONode / ClinicalModelDB retirement.** Currently still present in VP. Decide the cutover: at what point does `ClinicalModelDB` move from TODONode to PatientDataNode, and `OtherFunctionality` get deleted?

---

## 7. Coordination with teammate (Av2 overlap)

Anything P1 changes that also touches Av2's surface. Flag and confirm before finalising the VP model — these are the seams where the two extensions either compose cleanly or fight each other.

### Hard overlaps (need explicit agreement)

1. **`RiskEstimationScheduler` interface extension.** P1 extends `LaunchRiskEstimation` with `(priority, correlationId)` so UC9 jobs can jump ahead of scheduled ones. The scheduler is *the* P2 component, and Av2 likely adds HA behaviour around it. Need to agree:
   - Is the scheduler still a single instance, or does Av2 replicate it? If replicated, how is `correlationId` preserved across a failover so UC9's result delivery still finds its subscriber?
   - Priority lattice: P2 already uses dynamic priority in overload mode. UC9 adds an above-normal-priority lane with a 3-min initiation budget *independent of patient risk level*. Confirm UC9 priority sits above all three P2 tiers and doesn't starve them.

2. **`RiskEstimationCombiner → setEstimatedPatientStatus` callback path.** P1's `P1IRiskEvents` push relies on `P1PatientStatusModule` firing the event whenever `setEstimatedPatientStatus` actually changes the stored status. If Av2 modifies this callback path, the P1IRiskEvents emission point shifts. Confirm Av2 preserves the "appropriate parties are notified" contract from §E.3.19.

3. **`P1NotificationDispatcher` durable queue for red/yellow.** P1 promises red/yellow notifications survive a crash (§4 sensitivity #3, Response measure 5). Currently backed by `P1DurableNotificationLog` (internal write-ahead log). If Av2 prefers P1 to socket onto an Av2-provided durable queue facility instead, the `P1INotificationLog` interface becomes the seam to externalise.

4. **`P1NotificationDispatcher` requires `Av2EmergencyDispatch`** (per report E.1.41 / E.3.16). This is a **new** seam not in the original design. P1 now consumes the Av2 emergency channel — likely so red notifications can also dispatch via the SMS backup when the primary path is degraded. Need to agree:
   - What exact semantics does P1 expect (best-effort? fan-out copy? failover only)?
   - `P1RiskEventSubscriber` owns the required socket inside the dispatcher.

5. **`PhysicianNode` is a SPOF for every physician UC.** Already flagged in §5 Trade-offs. UC5/6/7/8/9 all fail closed if `PhysicianNode` is unavailable. Av2 should treat this node as a replication target if their QAS demands availability of physician-side flows.

### Soft overlaps (touch shared structure; usually compatible)

6. **`ClinicalModelCache` deployment + invalidation reach.** P1's UC7 path calls `ClinicalModelCacheMgmt.invalidateCacheEntries(patientId)` after writing config. If Av2 replicates the cache across multiple nodes, the invalidation must reach *all* replicas.

7. **`P1PatientDataService` consolidates three legacy interfaces into one component.** `SensorDataMgmt`, `OtherDataMgmt`, and `PatientRecordMgmt` all live in the same runtime component on `PatientDataNode`. Av2 might prefer independent failure boundaries — that would split `P1PatientDataService` further. If Av2 needs that split, P1's argument in §3a/§3b stays valid as long as the modules end up on the same node.

8. **`PatientRecordMgmt` extension (`updatePatientRecord`).** P1 adds a write operation to §E.3.20's interface. If Av2 adds retry / failover behaviour on `getPatientRecord`, it should apply symmetrically to `updatePatientRecord` so UC8 EHR forwards survive HIS hiccups.

9. **`SensorDataMgmt.addSensorData` — `triggerEstimation` not yet in VP.** Open per §6 OQ5. Until resolved, UC9 fires a redundant scheduled risk job. Any Av2 redundancy/replication of `P1SensorIngestModule` needs to know the chosen resolution.

### Decisions Av2 should know about (no compromise needed, just FYI)

10. **`OtherFunctionality` and `TODONode` are *not yet* retired.** Still present in VP. The intent remains to retire both once `P1PatientDataService` + the PhysicianNode components fully cover their responsibilities.
11. **`P1IRiskEvents` is push, not pull.** Sensitivity #2: flipping it breaks the "no impact on ingress" argument. Av2 should not introduce polling here even if it would simplify a failover scheme.
12. **`P1PatientGatewayCommander` adds a brand-new outbound direction (PMS → patient gateway).** No existing component in the initial architecture talks outbound to the gateway. Av2 should know this path exists in case it affects gateway-side failure handling. Currently no required interface is wired in VP (see §6 OQ4).
13. **Cache lives on PatientDataNode, not PhysicianNode.** Changes the resource-isolation story slightly: PatientDataNode now handles both ingest and physician-driven cache reads. See §4 sensitivity #1 / #4.

---

## 8. VP entry conventions

These are reminders for entering elements into Visual Paradigm so the SAPlugin checks pass.

- **Components** need a one-sentence **description** in the General tab.
- **Interfaces** need **operations**. For each operation fill in: `name`, parameter `classifier` (prefix with `Datatypes.` in VP — e.g. `Datatypes.PatientId`), `visibility` (`+` for public on a provided interface), `return type` (also `Datatypes.`-prefixed), `operation description`, and the **return-type description** (separate field, required by `[Gen.desc.05]`).
- **Signature format**:
  - `getPatientStatus(patientId: Datatypes.PatientId): Datatypes.PatientStatus`
  - `setEstimatedPatientStatus(patientId: Datatypes.PatientId, status: Datatypes.PatientStatus, timestamp: Datatypes.Timestamp)` *(void — no `: ReturnType`)*
- **Modules** (`<<module>>` stereotype) live inside the decomposition view of their parent component. They share the parent's process and are NOT independently deployed. Per VP lab §4.4 and SAPlugin manual §4.2: components represent run-time units; modules represent design/implementation units (libraries, code groupings, in-process collaborators).
- **Sequence diagrams** must use either component lifelines OR module lifelines (Conv. 7, `[Diag.seq.04]`), not both in the same diagram.
- **Convention 4**: if a parent component requires/provides an interface, at least one sub-component must require/provide it too. **Convention 5**: same for modules inside a component.
