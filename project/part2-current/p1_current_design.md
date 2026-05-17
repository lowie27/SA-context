# P1 — My Proposed Approach

> Living draft. Update freely; Claude will stress-test against the QAS, not rewrite.
> Sections / lines marked `<!-- [claude] -->` are additions by Claude. Push back where they don't match your intent.

---

## 0. QAS P1 (verbatim from instructions PDF)

- **Source:**: Physician
- **Stimulus:**:
    - A physician consults a patient's status (cf. UC6 : consult patient status), e.g., during a consultation with the patient.
    - A cardiologist requests a patient's current data (cf. UC9 : perform on-demand consultation), e.g., during a consultation with the patient.
    - A cardiologist (re)configures the risk assessment for a patient (cf. UC7 : configure patient risk assessment).
    - A cardiologist updates the risk level of a patient (cf. UC8 : update risk level), for example, after reviewing a received notification.
    - The pms looks up the appropriate recipients and sends a notification about a patient's status (cf.UC5 : notify registered parties).
- **Artifact:**: The subsystem(s) responsible to handle requests from physicians and send out notifications.
- **Environment:**: Normal mode
- **Response:**:
    - 1. The system is able to process all requests for patient information in a timely fashion.
    - 2. Any risk assessment triggered by an on-demand consultation or risk assessment configuration is initiated within a short time frame of the request. In case of remote on-demand consultations, the results of the risk assessment are provided to the physician in a timely fashion.
    - 3. The system is able to process all changes to a patient's risk level in a timely fashion.
    - 4. The system is able to send notifications concerning (changes to) a patient's status to all appropriate recipients in a timely fashion, as long as the number of notifications to send stays within a threshold.
    - 5. When the number of notifications to send exceeds the threshold the system prioritizes more urgent notifications.
    - 6. A large number of notifications to be sent or requests arriving to the system should have no impact on the (a) the processing of incoming raw sensor reading data; or (b) ongoing risk estimations.
- **Response measure:**:
    - 1. The system is able to process at minimum 25 requests for patient information per minute (cf. UC6 : consult patient status)
        - Requests for high-risk patients are processed in 2 seconds.
        - Requests for medium-risk level patients are processed in 5 seconds.
        - Requests for low-risk patients are processed in 10 seconds.
    - 2. The system is able to process at least 20 requests for an on-demand consultation (cf. UC9 : perform on-demand consultation) and/or risk assessment configurations per minute (cf. UC7 : configure patient risk assessment).
        - The risk assessments triggered in an on-demand consultation are prioritized over regular risk estimations triggered by scheduled arrival of new sensor data.
        - Each risk assessment (cf. UC12 : estimate risk level) triggered in an on-demand consultation is initiated within 3 minutes of receiving the request from the physician, independent of the patient's risk level.
        - In case of an on-demand consultation, the results of the risk assessment are provided to the physician within 1 minute after its completion.
    - 3. The system is able to process 20 risk-level updates per minute (cf. UC8 : update risk level).
        - Updates to a patient's ehr are forwarded within 1 minute.
    - 4. The system should be able to process and send at least 100 notifications per minute (cf. UC5 : notify registered parties).
        - A red notification should be sent to all recipients within 10 seconds.
        - A yellow notification should be sent to all recipients within 30 seconds.
        - A green notification should be sent to all recipients within 1 minute.
    - 5. When the system must send more than 100 notifications (cf. UC5 : notify registered parties) in a minute, it prioritizes red notifications over yellow and green ones, and yellow notifications over green ones. No red or yellow notifications may be lost.

---

## 1. Approach in plain words

I don't really understand what I actually did but I'll try to reason about it

Physicians interact with the PMS via a sigle point, and this gateway is interacting with the correct backend services replacing the otherfunctionality.
Patients need to be queried fast so there has to be caching in place so results are quick depending on the urgency

[claude] Expanded version of the same intuition, organized around the QAS:

Replace `OtherFunctionality` by splitting it into two halves on **two separate nodes**, so the
physician-facing work cannot contend with sensor ingest or risk estimation (this is what
Response item 6 demands):

1. **PhysicianAccessNode** — a thin `PhysicianGateway` plus five back-end services
   (`PhysicianCommandService`, `PatientQueryService`, `NotificationDispatcher`, `HISAdapter`,
   `PatientStatusCache`). All physician traffic terminates here. Tiered SLAs (2/5/10s queries,
   10/30/60s notifications) are realised by priority queues + a hot cache for high-risk
   patient status.

2. **PatientDataNode** — a `PatientDataService` that takes over the ingest / store side of
   `OtherFunctionality`: it receives raw sensor data, exposes `SensorDataMgmt`, `PatientRecordMgmt`,
   `OtherDataMgmt` to the existing risk-estimation pipeline, and writes patient status back
   when `RiskEstimationCombiner` finishes a job. `ClinicalModelDB` also moves here, so
   `TODONode` retires.

The two nodes communicate over the network. Because they don't share OS resources, a spike
in physician requests or notifications cannot starve the ingest path or the risk-estimation
processors. The `PatientStatusCache` on the physician side absorbs most read load and keeps
the cross-node traffic bounded.

i like this suggestions

---

## 2. What changes vs. the initial architecture

### 2a. Components

**VP entry rules — components get descriptions, interfaces get operations.**

- **Components** (§2a) need a **description** only — one short, concise sentence (≤ ~20 words) in VP's General tab. The `description:` line under each component is the source — paste it into VP. No operations on the component itself.
- **Interfaces** (§2b) need **operations**. For each operation, fill in these fields in VP:
    - **name** — e.g. `getPatientStatus`
    - **classifier** — the type of each parameter, prefixed with `Datatypes.`, e.g. `Datatypes.PatientId`, `Datatypes.SensorDataPackage`. ("Classifier" is just UML jargon for "type"; a parameter's classifier = its type.)
    - **visibility** — `+` public, `-` private, `#` protected, `~` package. Anything exposed on a provided interface is `+`.
    - **return type** — what the operation returns, also prefixed with `Datatypes.`, e.g. `Datatypes.PatientStatus`. Use `void` if nothing.
    - **operation description** — one short sentence describing the *effect* of calling the operation (what it does, what changes, or what it answers). This is the rule-code [Gen.desc.04] checks against. The inline note after the `—` in each signature below is the source — paste it into VP's General/Documentation tab for the operation.
    - **return-type description** — when the operation has a return type, VP exposes a *separate* description field on the return type itself. Fill it with one short sentence describing what the returned value represents (e.g. "The current status of the patient, including risk level and timestamp."). This is what rule [Gen.desc.05] checks — leaving it blank is what produced the error `Operation getPatientRecord of interface IHISAccess has a return type but does not describe it`. Void operations skip this.
- **Signature format to copy** (colon return type, named parameters; void operations omit the trailing `: ReturnType`):
    - `getPatientStatus(patientId: Datatypes.PatientId): Datatypes.PatientStatus`
    - `setEstimatedPatientStatus(patientId: Datatypes.PatientId, status: Datatypes.PatientStatus, timestamp: Datatypes.Timestamp)` *(no `: ReturnType` = void)*

new components, like your style of defining components very clear!

#### PhysicianGateway

- `PhysicianGateway`
    - description: single external entry point for physicians; thin router with no business logic.
    - node: PhysicianAccessNode
    - provides:
        - `IPhysicianAPI`
    - requires:
        - `IPhysicianCommand`
        - `IPatientQuery`
        - `INotificationInbox`
#### PhysicianCommandService

- `PhysicianCommandService`
    - description: handles physician write/command flows: UC7 (configure risk assessment), UC8 (update risk level), UC9 (on-demand consultation).
    - node: PhysicianAccessNode
    - provides:
        - `IPhysicianCommand`
    - requires:
        - `PatientRecordMgmt` (UC8 EHR write — routes via EHRProxyModule for cache coherence)
        - `ClinicalConfigMgmt` (UC7 write — new interface on ClinicalModelDB)
        - `IRiskEvents` (UC9 result delivery)
        - `LaunchRiskEstimation` (UC9 priority launch with correlation)
        - `ClinicalModelCacheMgmt` (UC7 invalidation after the config write)
        - `IOnDemandSensorFetch` (UC9 gateway fetch)
        - `SensorDataMgmt` (UC9 store fetched data with triggerEstimation=false)
    - **decomposed — see §2c**

#### PatientQueryService

- `PatientQueryService`
    - description: handles UC6 patient-status reads; tiered priority queue (high > medium > low) feeds the 2/5/10 s SLA tiers.
    - node: PhysicianAccessNode
    - provides:
        - `IPatientQuery`
    - requires:
        - `IPatientStatusRead`
#### PatientStatusCache

- `PatientStatusCache`  <!-- [claude] new -->
    - description: read-through cache for patient status; pins high-risk patients to bound cross-node traffic.
    - node: PhysicianAccessNode
    - provides:
        - `IPatientStatusRead`
    - requires:
        - `OtherDataMgmt` (miss fallback + invalidation; reuses the existing patient-status interface per §E.3.19)

#### NotificationDispatcher

- `NotificationDispatcher`
    - description: UC5 dispatcher; subscribes to IRiskEvents; red > yellow > green priority queue; drops green first under overload; persists red/yellow so they cannot be lost.
    - node: PhysicianAccessNode
    - provides:
        - `INotificationInbox`
    - requires:
        - `IRiskEvents`
    - **decomposed — see §2c**

#### PatientGatewayCommander

- `PatientGatewayCommander`  <!-- [claude] new — closes UC9 gateway-fetch gap. Treats the patient gateway as an external system (same pattern as HISAdapter for HIS). -->
    - description: outbound channel from PMS to patient gateways for UC9 on-demand sensor-data fetches; synchronous with a bounded timeout.
    - node: PhysicianAccessNode
    - provides:
        - `IOnDemandSensorFetch`
    - requires:
        - `patientGatewayAPI` (external interface exposed by the patient gateway; new — see §6 OQ4 for auth model)

#### HISAdapter

- `HISAdapter`
    - description: outbound adapter for the external HIS healthAPI; used by EHRProxyModule for UC16 reads and UC17 writes.
    - node: PatientDataNode  <!-- [claude] kept on PatientDataNode: co-located with EHRProxyModule, which fronts every HIS call. PhysicianCommandService's UC8 write reaches HIS via PatientRecordMgmt.updatePatientRecord (one cross-node hop) but with structural cache coherence. -->
    - provides:
        - `IHISAccess`
    - requires:
        - `healthAPI`

#### PatientDataService

- `PatientDataService`  <!-- [claude] new — replaces OtherFunctionality -->
    - description: owns raw sensor data, patient status, and the HIS EHR proxy; takes over the three interfaces OtherFunctionality exposed.
    - node: PatientDataNode
    - provides:
        - `SensorDataMgmt`
        - `OtherDataMgmt`
        - `PatientRecordMgmt`
        - `IRiskEvents`  <!-- [claude] IRiskEvents now provided here (not by RiskEstimationCombiner) — fired internally when setEstimatedPatientStatus actually changes status, matching the existing OtherFunctionality contract ("appropriate parties are notified") -->
    - requires:
        - `LaunchRiskEstimation`
        - `IHISAccess`  <!-- [claude] required by EHRProxyModule because PatientRecordMgmt is a HIS proxy per §E.3.20 -->
    - **decomposed — see §2c**

#### Retired / relocated

- `OtherFunctionality` — **retired**; replaced by PatientDataService (ingest + store side) and the six physician-side components on PhysicianAccessNode. Convention 4 satisfied because PatientDataService still exposes the three legacy interfaces (SensorDataMgmt, PatientRecordMgmt, OtherDataMgmt) via its modules.
- `TODONode` — **retired**; replaced by PhysicianAccessNode + PatientDataNode.
- `ClinicalModelDB` — **relocated** from TODONode to PatientDataNode (responsibilities unchanged).

### 2b. Interfaces

> **Custom-type marker (`†`)** — types not defined in initial-architecture §E.6 are marked with `†` on first use within each interface block. Full definitions in `datatypes_reference.md` §7.
> Custom types used in P1: `ConsultationId`†, `CorrelationId`†, `PhysicianId`†, `Notification`†, `NotificationId`†, `NotificationSeverity`† *(implied, lives inside `Notification`)*, `SubscriberId`†, `SubscriptionId`†, `FilterCriteria`†, `RiskEvent`†, `Priority`†.
> Everything *not* marked with `†` already exists in §E.6 / §E.5 — reuse verbatim.
> **VP entry convention (`Datatypes.` prefix)** — when entering operation parameters or return types in Visual Paradigm, prefix every type with `Datatypes.` (e.g. `Datatypes.PatientId`, `Datatypes.PatientStatus`). The signatures below already use this prefix so you can paste them into VP 1:1. Primitives like `boolean` stay unprefixed. The `†` marker is documentation only — it does NOT appear in the VP classifier.
>
> **Interface entry convention** — every interface entry follows this template:
> - operations use `paramname: ParamType, ...): ReturnType` (colon syntax for return type; void operations omit the trailing `: ReturnType`)
> - every interface ends with a `purpose:` line summarising its role
>
> ```
> - `InterfaceName`
>     - provided by: `Component`
>     - required by:
>         - `Component`
>     - operations (all `+`):
>         - `operationName(paramname: ParamType, ...): ReturnType` — inline note (UC refs, §E refs, behaviour)
>     - purpose: one-line explanation
> ```

new interfaces

#### IPhysicianAPI

- `IPhysicianAPI`
    - provided by: `PhysicianGateway`
    - required by: external (physicians calling into the PMS)
    - operations (all `+`):
        - `consultPatientStatus(patientId: Datatypes.PatientId): Datatypes.PatientStatus` — UC6
        - `requestOnDemandConsultation(patientId: Datatypes.PatientId): Datatypes.ConsultationId†` — UC9; `ConsultationId` is the correlationId used to match the eventual result
        - `configurePatientRiskAssessment(patientId: Datatypes.PatientId, config: Datatypes.ClinicalModelConfiguration)` — UC7
        - `updatePatientRiskLevel(patientId: Datatypes.PatientId, status: Datatypes.PatientStatus)` — UC8
        - `subscribeToNotifications(physicianId: Datatypes.PhysicianId†)` — UC5; lets the gateway deliver notifications back to this physician
    - purpose: external physician-facing surface; thin façade routing UC5–UC9 commands and queries into the back-end services

#### IPhysicianCommand

- `IPhysicianCommand`
    - provided by: `PhysicianCommandService`
    - required by:
        - `PhysicianGateway`
    - operations (all `+`):
        - `configurePatientRiskAssessment(patientId: Datatypes.PatientId, config: Datatypes.ClinicalModelConfiguration)` — UC7
        - `updatePatientRiskLevel(patientId: Datatypes.PatientId, status: Datatypes.PatientStatus)` — UC8
        - `requestOnDemandConsultation(patientId: Datatypes.PatientId): Datatypes.ConsultationId†` — UC9
    - purpose: physician write/command surface (UC7, UC8, UC9) consumed by PhysicianGateway

#### IPatientQuery

- `IPatientQuery`
    - provided by: `PatientQueryService`
    - required by:
        - `PhysicianGateway`
    - operations (all `+`):
        - `consultPatientStatus(patientId: Datatypes.PatientId): Datatypes.PatientStatus` — UC6; the service picks the priority tier (high/med/low) from the patient's current risk level
    - purpose: physician read surface backing UC6 with tiered 2/5/10 s SLAs

#### INotificationInbox

- `INotificationInbox`
    - provided by: `NotificationDispatcher`
    - required by:
        - `PhysicianGateway`
    - operations (all `+`):
        - `subscribeToNotifications(physicianId: Datatypes.PhysicianId†)` — register a physician to receive notifications
        - `unsubscribeFromNotifications(physicianId: Datatypes.PhysicianId†)`
        - `getPendingNotifications(physicianId: Datatypes.PhysicianId†): List<Datatypes.Notification†>` — gateway pulls queued notifications for a physician (poll model; long-poll or websocket as an implementation choice). `Notification` carries a `NotificationSeverity†` (red/yellow/green).
        - `acknowledgeNotification(notificationId: Datatypes.NotificationId†)` — gateway confirms delivery so the dispatcher can drop it from its durable queue
    - purpose: notification inbox surface for the gateway; subscribe/unsubscribe, poll pending, and ack delivery

#### IHISAccess

- `IHISAccess`
    - provided by: `HISAdapter`
    - required by:
        - `EHRProxyModule` (UC16 reads, UC17 writes — sole consumer; UC8 writes route through PatientRecordMgmt for cache coherence)
    - operations (all `+`):
        - `getPatientRecord(patientId: Datatypes.PatientId): Datatypes.PatientRecord` — UC16; wraps the relevant `healthAPI` reads (getPatient / getObservation / getRiskAssessment)
        - `updatePatientRecord(patientId: Datatypes.PatientId, record: Datatypes.PatientRecord)` — UC17; wraps the relevant `healthAPI` writes (savePatient / saveObservation / saveRiskAssessment)
    - purpose: adapter façade over the external HIS healthAPI, exposing record-level read/write

#### ClinicalConfigMgmt

- `ClinicalConfigMgmt`  <!-- [claude] new interface alongside the existing read-only ClinicalModelStorage on ClinicalModelDB. UC7 needs a write surface; this is it. ClinicalModelCache still sockets onto ClinicalModelStorage so it cannot accidentally gain write access. -->
    - provided by: `ClinicalModelDB` (new interface alongside the existing ClinicalModelStorage)
    - required by:
        - `PhysicianCommandService` (UC7)
    - operations (all `+`):
        - `setClinicalModelConfigForPatient(patientId: Datatypes.PatientId, config: Datatypes.ClinicalModelConfiguration)` — UC7; overwrites the per-patient config
    - purpose: write surface for per-patient ClinicalModelConfiguration (UC7), alongside the existing read-only ClinicalModelStorage on ClinicalModelDB
    - note: caller (PhysicianCommandService) must call `ClinicalModelCacheMgmt.invalidateCacheEntries(patientId)` after a successful write — the cache won't otherwise notice the change. Mirrors the legacy `OtherFunctionality → ClinicalModelCacheMgmt` invalidation edge.

#### IRiskEvents

- `IRiskEvents`  <!-- [claude] push, provided by PatientDataService — fired internally when setEstimatedPatientStatus changes the stored status. Matches the existing OtherFunctionality contract ("appropriate parties are notified") without modifying RiskEstimationCombiner. -->
    - provided by: `PatientDataService` (via PatientStatusModule)
    - required by:
        - `NotificationDispatcher` (UC5 notify)
        - `PhysicianCommandService` (UC9 result delivery)
    - operations (all `+`):
        - `subscribe(subscriberId: Datatypes.SubscriberId†, filter: Datatypes.FilterCriteria†): Datatypes.SubscriptionId†` — register a subscriber; `FilterCriteria` may carry a correlationId (UC9 result lookup) or be empty (NotificationDispatcher gets every event)
        - `unsubscribe(subscriptionId: Datatypes.SubscriptionId†)`
    - purpose: push channel for patient-status risk events; consumed by NotificationDispatcher (UC5) and PhysicianCommandService (UC9 result delivery)
    - note: event payload delivered to subscribers is `RiskEvent†(PatientId, PatientStatus, Timestamp, CorrelationId†?)`. Fired by PatientStatusModule when `setEstimatedPatientStatus` actually changes the stored status.

#### IPatientStatusRead

- `IPatientStatusRead`
    - provided by: `PatientStatusCache`
    - required by:
        - `PatientQueryService`
    - operations (all `+`):
        - `getPatientStatus(patientId: Datatypes.PatientId): Datatypes.PatientStatus` — cache-first read; on miss the cache falls back to `OtherDataMgmt.getPatientStatus` and populates itself
    - purpose: cache-first patient status read; miss-fallback into OtherDataMgmt

#### IOnDemandSensorFetch

- `IOnDemandSensorFetch`
    - provided by: `PatientGatewayCommander`
    - required by:
        - `PhysicianCommandService`
    - operations (all `+`):
        - `requestCurrentSensorData(patientId: Datatypes.PatientId, correlationId: Datatypes.CorrelationId†): Datatypes.SensorDataPackage` — synchronous with timeout sized inside the 3-min UC9 initiation budget
    - purpose: outbound on-demand sensor fetch from PMS to a patient gateway (UC9)

existing interfaces reused verbatim from rationale PDF §E.3

#### SensorDataMgmt

- `SensorDataMgmt`
    - provided by: `PatientDataService` (was OtherFunctionality)
    - required by:
        - `ClinicalJobCreator`
        - `RiskEstimationScheduler` (per §E.3.23)
        - `PhysicianCommandService` (UC9 — calls `addSensorData(..., triggerEstimation=false)` after gateway fetch)
    - operations (all `+`):
        - `addSensorData(patientId: Datatypes.PatientId, package: Datatypes.SensorDataPackage, timestamp: Datatypes.Timestamp, triggerEstimation: boolean = true)` — §E.3.23 plus the new optional `triggerEstimation` flag (default `true` preserves legacy behaviour; UC9 passes `false`)
        - `getAllSensorDataOfPatient(patientId: Datatypes.PatientId): Map<Datatypes.Timestamp, Datatypes.SensorDataPackage>` — §E.3.23 (Map instead of list)
        - `getAllSensorDataOfPatientBefore(patientId: Datatypes.PatientId, before: Datatypes.Timestamp): Map<Datatypes.Timestamp, Datatypes.SensorDataPackage>` — §E.3.23 (Map instead of list)
    - purpose: append/query raw sensor data; ingest write path with optional trigger of the risk pipeline

#### PatientRecordMgmt

- `PatientRecordMgmt`
    - provided by: `PatientDataService` (was OtherFunctionality)
    - required by:
        - `RiskEstimationCombiner` (UC16 reads per §E.3.20)
        - `PhysicianCommandService` (UC8/UC17 writes via the new `updatePatientRecord` operation)
    - operations (all `+`):
        - `getPatientRecord(patientId: Datatypes.PatientId): Datatypes.PatientRecord` — §E.3.20; stale-tolerant cached read fronted by EHRProxyModule
        - `updatePatientRecord(patientId: Datatypes.PatientId, record: Datatypes.PatientRecord)` — UC17 write (new op on §E.3.20); EHRProxyModule calls HISAdapter then invalidates its cache entry
    - purpose: EHR-proxy surface for record reads (UC16) and writes (UC8 / UC17)

#### OtherDataMgmt

- `OtherDataMgmt`
    - provided by: `PatientDataService` (was OtherFunctionality)
    - required by:
        - `ClinicalJobCreator`
        - `ClinicalModelCache`
        - `RiskEstimationCombiner`
        - `RiskEstimationScheduler` (per §E.3.19)
        - `PatientStatusCache` (miss fallback)
    - operations (all `+`, both from §E.3.19, unchanged):
        - `getPatientStatus(patientId: Datatypes.PatientId): Datatypes.PatientStatus`
        - `setEstimatedPatientStatus(patientId: Datatypes.PatientId, status: Datatypes.PatientStatus, timestamp: Datatypes.Timestamp)` — fires `IRiskEvents` when the stored status actually changes
    - purpose: patient-status read/write; `setEstimatedPatientStatus` is the IRiskEvents emission point

#### LaunchRiskEstimation

- `LaunchRiskEstimation`
    - provided by: `RiskEstimationScheduler` (extended — gains a priority/correlation parameter so UC9 jobs can jump ahead of scheduled ones; coordinate with teammate on Av2 since this touches their scheduler surface)
    - required by:
        - `PatientDataService`.SensorIngestModule
        - `PhysicianCommandService` (UC9 priority launch with correlationId)
    - operations (all `+`):
        - `launchRiskEstimation(patientId: Datatypes.PatientId, package: Datatypes.SensorDataPackage, timestamp: Datatypes.Timestamp, priority: Datatypes.Priority† = NORMAL, correlationId: Datatypes.CorrelationId†? = null)` — §E.3.11 plus two new optional parameters. `priority=HIGH` (above all three P2 tiers) for UC9; `correlationId` lets PhysicianCommandService match the eventual `IRiskEvents` event back to the originating UC9 request. Default values preserve legacy callers.
    - purpose: scheduler entry point; extended with priority + correlationId for UC9 priority launches

#### ClinicalModelCacheMgmt

- `ClinicalModelCacheMgmt`
    - provided by: `ClinicalModelCache` (unchanged)
    - required by:
        - `PhysicianCommandService` (UC7 config change → cache invalidation)
    - operations (`+`, unchanged from §E.3.3):
        - `invalidateCacheEntries(patientId: Datatypes.PatientId)` — removes all cached entries for a patient
    - purpose: invalidate ClinicalModelCache entries after a per-patient config change (UC7)

#### healthAPI

- `healthAPI`
    - provided by: `HIS` (external to eHealthPlatform, unchanged)
    - required by:
        - `HISAdapter`
    - operations: unchanged from §E.3.8 (FHIR-derived: `getObservation`, `getPatient`, `getRiskAssessment`, `saveObservation`, `savePatient`, `saveRiskAssessment`, `deleteObservation`, `deletePatient`, `deleteRiskAssessment`, `searchObservation`, `searchPatient`, `searchRiskAssessment`). Signatures live in §E.3.8 — not reproduced here since P1 doesn't change them.

### 2c. Decomposed components

> Each subsection below is a decomposition view (Conv. 5) for one parent component listed in §2a. Modules carry the `<<module>>` stereotype in VP. Internal interfaces (the ones with names like `IConfigCommand`, `IPriorityBuffer`, etc.) are defined in §2d and do not appear on the C&C diagram.

#### PhysicianCommandService

- `decomposed` into modules (decomposition view — Conv. 5):
    - `CommandRouter`
        - description: provides IPhysicianCommand at the component boundary; thin dispatcher that routes each operation to the matching handler.
        - provides:
            - `IPhysicianCommand`
        - requires:
            - `IConfigCommand` (internal — UC7 → ConfigurationHandler)
            - `IRiskLevelCommand` (internal — UC8 → RiskLevelHandler)
            - `IOnDemandCommand` (internal — UC9 → OnDemandConsultationHandler)
    - `ConfigurationHandler`
        - description: UC7 — overwrites the per-patient ClinicalModelConfiguration, then invalidates the clinical-model cache.
        - provides:
            - `IConfigCommand` (internal to PhysicianCommandService; consumed by CommandRouter)
        - requires:
            - `ClinicalConfigMgmt`
            - `ClinicalModelCacheMgmt`
    - `RiskLevelHandler`
        - description: UC8 — writes the updated risk level via PatientRecordMgmt; the EHR forward is structural through EHRProxyModule.
        - provides:
            - `IRiskLevelCommand` (internal to PhysicianCommandService; consumed by CommandRouter)
        - requires:
            - `PatientRecordMgmt`
    - `OnDemandConsultationHandler`
        - description: UC9 orchestration — mints a CorrelationId via CorrelationTracker, fetches sensor data, stores it with `triggerEstimation=false`, launches a priority risk job carrying the CorrelationId, then waits for the matching IRiskEvents.
        - provides:
            - `IOnDemandCommand` (internal to PhysicianCommandService; consumed by CommandRouter)
        - requires:
            - `ICorrelationTracking` (internal — mints + resolves CorrelationIds)
            - `IOnDemandSensorFetch`
            - `SensorDataMgmt`
            - `LaunchRiskEstimation`
            - `IRiskEvents`
    - `CorrelationTracker`  <!-- [claude] makes §7 point #1 structural — Av2 must preserve this state across failover -->
        - description: mints CorrelationIds for UC9 requests and matches incoming RiskEvents back to the originating handler; the single locus an Av2 failover must preserve so UC9 result delivery survives.
        - provides:
            - `ICorrelationTracking` (internal to PhysicianCommandService; consumed by OnDemandConsultationHandler)
        - requires:
            - (none)

#### NotificationDispatcher

- `decomposed` into modules (decomposition view — Conv. 5):
    - `NotificationInboxModule`
        - description: provides INotificationInbox at the component boundary; owns per-physician subscription state and the pull surface used by PhysicianGateway (getPendingNotifications / acknowledgeNotification).
        - provides:
            - `INotificationInbox`
        - requires:
            - `IPriorityBuffer` (internal — peek/remove pending notifications for a physician)
            - `INotificationLog` (internal — recover unacked red/yellow on startup; mark delivered after ack)
            - `IRecipientRegistry` (internal — record subscribe/unsubscribe)
    - `PriorityBuffer`  <!-- [claude] Response measure 5 + sensitivity #3 -->
        - description: in-memory priority queue (red > yellow > green); enforces the 100/min threshold and drops green first under overload.
        - provides:
            - `IPriorityBuffer` (internal to NotificationDispatcher; written by RiskEventSubscriber, read by NotificationInboxModule)
        - requires:
            - (none)
    - `DurableNotificationLog`  <!-- [claude] Response measure 5 "may not be lost" + Av2 seam §7 point #3 -->
        - description: write-ahead log; red and yellow notifications are persisted here before being acked from IRiskEvents so they survive a crash.
        - provides:
            - `INotificationLog` (internal to NotificationDispatcher; written by RiskEventSubscriber, read by NotificationInboxModule on recovery)
        - requires:
            - (none)
    - `RecipientRegistry`  <!-- [claude] resolves OQ1 — registry lives here as a module, not a separate component -->
        - description: maps a patient-status event to the physicians who should be notified about it; UC5's "looks up the appropriate recipients" lives here. Populated via subscribeToNotifications on INotificationInbox.
        - provides:
            - `IRecipientRegistry` (internal to NotificationDispatcher; queried by RiskEventSubscriber, written by NotificationInboxModule on subscribe/unsubscribe)
        - requires:
            - (none)
    - `RiskEventSubscriber`  <!-- [claude] required-interface owner per Conv. 4; preserves §7 point #2's emission-point contract -->
        - description: requires IRiskEvents at the component boundary; on event, queries RecipientRegistry, persists red/yellow to DurableNotificationLog, then enqueues to PriorityBuffer.
        - provides:
            - (none — reacts to IRiskEvents push; no inbound calls from siblings)
        - requires:
            - `IRiskEvents`
            - `IRecipientRegistry` (internal — UC5 recipient lookup)
            - `INotificationLog` (internal — persist red/yellow before enqueue)
            - `IPriorityBuffer` (internal — enqueue for dispatch)

#### PatientDataService

- `decomposed` into modules (decomposition view — Conv. 5):
    - **no internal interfaces** — the three modules are parallel (independent responsibilities, each tied to its own data store: sensor data, patient status, EHR cache); all collaboration is at the component boundary, so the decomposition diagram has no ball-and-socket connections between modules. Contrast with PhysicianCommandService and NotificationDispatcher decompositions, where modules collaborate internally.
    - `SensorIngestModule`
        - description: accepts and persists raw sensor data; triggers risk estimation unless caller opts out via `triggerEstimation=false` (used by UC9).
        - provides:
            - `SensorDataMgmt`
        - requires:
            - `LaunchRiskEstimation`
    - `PatientStatusModule`
        - description: owns patient-status records; fires IRiskEvents when setEstimatedPatientStatus actually changes the stored status.
        - provides:
            - `OtherDataMgmt`
            - `IRiskEvents`
        - requires:
            - (none)
    - `EHRProxyModule`  <!-- [claude] handles both reads and writes against HIS so cache invalidation is structural -->
        - description: fronts HIS for EHR reads and writes via PatientRecordMgmt; stale-tolerant cache on reads, invalidated structurally on writes.
        - provides:
            - `PatientRecordMgmt`
        - requires:
            - `IHISAccess`

### 2d. Internal interfaces

> These interfaces exist purely inside a single component's decomposition view (see §2c). They do **not** appear on the C&C diagram and have no direct callers from outside their parent component. They are listed here so that each module-to-module call in §2c has a backing required interface per Conv. 3 and Conv. 6.

PhysicianCommandService — internal interfaces

#### IConfigCommand

- `IConfigCommand`
    - provided by: `ConfigurationHandler` (module of PhysicianCommandService)
    - required by:
        - `CommandRouter` (module of PhysicianCommandService)
    - operations (all `+`):
        - `configurePatientRiskAssessment(patientId: Datatypes.PatientId, config: Datatypes.ClinicalModelConfiguration)` — UC7 delegation; mirrors the same op on IPhysicianCommand
    - purpose: internal — CommandRouter → ConfigurationHandler delegation for UC7

#### IRiskLevelCommand

- `IRiskLevelCommand`
    - provided by: `RiskLevelHandler` (module of PhysicianCommandService)
    - required by:
        - `CommandRouter` (module of PhysicianCommandService)
    - operations (all `+`):
        - `updatePatientRiskLevel(patientId: Datatypes.PatientId, status: Datatypes.PatientStatus)` — UC8 delegation; mirrors the same op on IPhysicianCommand
    - purpose: internal — CommandRouter → RiskLevelHandler delegation for UC8

#### IOnDemandCommand

- `IOnDemandCommand`
    - provided by: `OnDemandConsultationHandler` (module of PhysicianCommandService)
    - required by:
        - `CommandRouter` (module of PhysicianCommandService)
    - operations (all `+`):
        - `requestOnDemandConsultation(patientId: Datatypes.PatientId): Datatypes.ConsultationId†` — UC9 delegation; mirrors the same op on IPhysicianCommand
    - purpose: internal — CommandRouter → OnDemandConsultationHandler delegation for UC9

#### ICorrelationTracking

- `ICorrelationTracking`  <!-- [claude] makes §7 point #1 (Av2 failover state) structural -->
    - provided by: `CorrelationTracker` (module of PhysicianCommandService)
    - required by:
        - `OnDemandConsultationHandler` (module of PhysicianCommandService)
    - operations (all `+`):
        - `newCorrelationId(): Datatypes.CorrelationId†` — mints a fresh CorrelationId and records it as pending
        - `consumeCorrelation(correlationId: Datatypes.CorrelationId†): boolean` — returns true if the CorrelationId was pending (and removes it); called by OnDemandConsultationHandler when an IRiskEvents arrives, to decide whether the event matches a pending UC9
    - purpose: internal — CorrelationId minting and matching for UC9 result delivery (Av2 failover state lives here per §7 point #1)

NotificationDispatcher — internal interfaces

#### IPriorityBuffer

- `IPriorityBuffer`
    - provided by: `PriorityBuffer` (module of NotificationDispatcher)
    - required by:
        - `RiskEventSubscriber` (module of NotificationDispatcher; write side — enqueue)
        - `NotificationInboxModule` (module of NotificationDispatcher; read side — peek/remove)
    - operations (all `+`):
        - `enqueue(physicianId: Datatypes.PhysicianId†, notification: Datatypes.Notification†)` — adds a pending notification for a specific physician; severity drives priority + drop-green-first overload policy (Response measure 5)
        - `peekPending(physicianId: Datatypes.PhysicianId†): List<Datatypes.Notification†>` — returns pending notifications for a physician, in priority order; called from getPendingNotifications
        - `removePending(notificationId: Datatypes.NotificationId†)` — removes a notification after the physician acks it via INotificationInbox.acknowledgeNotification
    - purpose: internal — priority-ordered notification buffer (red > yellow > green) with drop-green-first overload policy

#### INotificationLog

- `INotificationLog`  <!-- [claude] backs Response measure 5 "may not be lost" -->
    - provided by: `DurableNotificationLog` (module of NotificationDispatcher)
    - required by:
        - `RiskEventSubscriber` (module of NotificationDispatcher; persist before enqueue)
        - `NotificationInboxModule` (module of NotificationDispatcher; mark delivered + recover on startup)
    - operations (all `+`):
        - `persist(notification: Datatypes.Notification†)` — write-ahead log entry for red/yellow notifications before they enter the buffer; survives a crash
        - `markDelivered(notificationId: Datatypes.NotificationId†)` — removes from log after physician ack so storage doesn't grow unbounded
        - `recoverPending(): List<Datatypes.Notification†>` — on startup, returns notifications that were persisted but never acked; NotificationInboxModule replays them into the buffer
    - purpose: internal — durable write-ahead log so red/yellow notifications survive a crash (Response measure 5)

#### IRecipientRegistry

- `IRecipientRegistry`  <!-- [claude] resolves OQ1 structurally -->
    - provided by: `RecipientRegistry` (module of NotificationDispatcher)
    - required by:
        - `NotificationInboxModule` (module of NotificationDispatcher; subscribe/unsubscribe writes from INotificationInbox)
        - `RiskEventSubscriber` (module of NotificationDispatcher; UC5 recipient lookup on each event)
    - operations (all `+`):
        - `subscribe(physicianId: Datatypes.PhysicianId†)` — register a physician to receive notifications
        - `unsubscribe(physicianId: Datatypes.PhysicianId†)` — remove a physician
        - `findRecipients(patientId: Datatypes.PatientId): List<Datatypes.PhysicianId†>` — UC5 lookup; returns the physicians who should be notified about events for this patient. The mapping rule (which physicians watch which patient) is not yet specified — see §6 OQ1 follow-up.
    - purpose: internal — physician↔patient subscription registry used for UC5 recipient lookup

### 2e. Deployment changes

I only did primary diagram of the client server view, this is fine for now, will move on to other diagrams later

<!-- [claude] Sketch for when you do get to deployment (needed for §3b and Conv. 8):

- **eHealthPlatform** (existing parent node) gains two new physical nodes and drops TODONode:
  - **PhysicianAccessNode** (new): PhysicianGateway, PhysicianCommandService, PatientQueryService, PatientStatusCache, NotificationDispatcher, PatientGatewayCommander.  <!-- [claude] HISAdapter no longer here (moved to PatientDataNode); PatientGatewayCommander added for UC9 outbound fetch. -->
  - **PatientDataNode** (new): PatientDataService, ClinicalModelDB, HISAdapter.
  - TODONode: retired.
- Communication paths (Conv. 9):
  - PhysicianAccessNode ↔ PatientDataNode (PatientStatusCache → OtherDataMgmt for miss fallback; PhysicianCommandService → PatientRecordMgmt for UC8/UC17 writes; PhysicianCommandService → SensorDataMgmt for UC9 fetched data; PhysicianCommandService → ClinicalConfigMgmt on UC7)  <!-- [claude] ClinicalConfigMgmt is on ClinicalModelDB which lives on PatientDataNode. -->
  - PhysicianAccessNode ↔ RiskEstimationMgmtNode (PhysicianCommandService → LaunchRiskEstimation with priority+correlation; NotificationDispatcher ← IRiskEvents; PhysicianCommandService ← IRiskEvents; PhysicianCommandService → ClinicalModelCacheMgmt for UC7 invalidation)  <!-- [claude] ClinicalModelCacheMgmt is on ClinicalModelCache; per §E.1.2 the cache is sited close to RiskEstimationProcessor/Combiner — assumed to deploy on RiskEstimationMgmtNode post-TODONode-retirement (verify when wiring the deployment view). -->
  - PatientDataNode ↔ RiskEstimationMgmtNode (PatientDataService → LaunchRiskEstimation; PatientDataService.PatientStatusModule ← PatientRecordMgmt callbacks from RiskEstimationCombiner)
  - PatientDataNode ↔ RiskEstimationProcessorNode (SensorDataMgmt and OtherDataMgmt are pulled by ClinicalJobCreator which lives on RiskEstimationMgmtNode actually — double-check in figure C.2)
  - PatientDataNode → HIS (external, via healthAPI on HISAdapter)
  - **PhysicianAccessNode → patient gateway (external, via patientGatewayAPI on PatientGatewayCommander)**  <!-- [claude] new path for UC9: outbound request to the patient gateway. The reply (sensor data) is fed back into SensorDataMgmt by PhysicianCommandService, so no separate inbound path is needed beyond the existing UC4 channel. -->

The point of the split: **PhysicianAccessNode has its own CPU/memory/threads.** A surge of 25 status queries + 100 notifications/min stays inside this node. Sensor data and risk jobs don't pass through it. That's what makes Response item 6 (no impact on ingress / risk estimation) defensible.
-->

---

## 3. How this satisfies P1

### 3a. The 2s response-measure budget

[claude] The 2s ceiling for high-risk queries (Response measure 1) is generous — the real
constraint is **tiered SLAs** under contention. Budget for the *worst* case (cache miss):

| Hop | Estimated | Note |
| --- | --- | --- |
| Network in (physician client → PhysicianGateway) | ~50 ms | TLS-terminated REST/gRPC |
| PhysicianGateway route + auth | ~10 ms | thin router |
| PatientQueryService dequeue (priority: high) | ~20 ms | bounded queue, head-of-line for high-risk |
| PatientStatusCache lookup (miss) | ~10 ms | |
| Cross-node call → PatientDataService.PatientRecordModule | ~30 ms | LAN within eHealthPlatform |
| Store read (indexed by patientId) | ~50 ms | |
| Return to cache + populate | ~10 ms | |
| Serialize + network out | ~50 ms | |
| **Total (cache miss)** | **~230 ms** | well under 2 s; ~1.7 s slack |
| **Total (cache hit)** | **~150 ms** | |

The slack matters: under load, the priority queue ensures high-risk requests jump ahead;
medium-risk get up to 5 s (room for queueing) and low-risk up to 10 s (more queueing). The
cache is what keeps the **average** path far below the SLA — without it, a flood of cache
misses against the store would saturate the cross-node link.

**Other response measures (rougher numbers):**

- **UC8 risk-level update + 1-min EHR forward**: write path PhysicianGateway → PhysicianCommandService → PatientRecordMgmt.updatePatientRecord (cross-node to PatientDataNode, ~30 ms) → EHRProxyModule (cache invalidate + same-node HISAdapter call). HIS round-trip dominates; even at multi-second HIS latency, total stays well under the 60 s SLA. Cache coherence is structural — any subsequent UC16 read from RiskEstimationCombiner sees the new value.
- **UC9 on-demand consultation, 3-min init**: PhysicianCommandService mints a correlationId, calls `IOnDemandSensorFetch` (PatientGatewayCommander → patient gateway, sync round-trip — typically a few seconds, bounded by the commander timeout). On reply: writes the package via `SensorDataMgmt.addSensorData(..., triggerEstimation=false)` so the data is stored *without* triggering an estimation, then calls `LaunchRiskEstimation(priority=high, correlationId)`. The scheduler must honour this priority over scheduled jobs (couples to Av/P2 design — coordinate with teammate).
- **UC9 result delivery in 1 min**: PhysicianCommandService subscribes to `IRiskEvents` filtered by correlationId; pushes result back to physician via long-poll or websocket on `INotificationInbox`.
- **UC5 100 notifications/min**: NotificationDispatcher priority queue. Red bypasses everything (10 s SLA). Green can queue up to 60 s. Under overload, green is dropped first; red/yellow are persisted to disk before being acked from `IRiskEvents` so they cannot be lost.


### 3b. The "no impact on ingress" argument

[claude] Response item 6 is the strongest constraint in the QAS. The argument has three legs:

1. **Resource separation by node.** PhysicianAccessNode and PatientDataNode are physically
   separate nodes (Conv. 8/9). The CPU, memory, network stack, OS schedulers used by
   PhysicianGateway + the five physician-side services are disjoint from those used by the
   sensor-ingest path (PatientDataService on PatientDataNode) and the risk-estimation pipeline
   (RiskEstimationMgmtNode, RiskEstimationProcessorNode). A spike in physician traffic
   consumes PhysicianAccessNode resources only.

2. **No synchronous calls into the risk pipeline from the physician side.** `IRiskEvents` is
   **push** (decision recorded in §2b). NotificationDispatcher and PhysicianCommandService
   subscribe; they never poll or query RiskEstimationCombiner. Therefore a notification storm
   does not generate any incoming load on the risk pipeline.

3. **Bounded cross-node read load.** The dominant data flow from PhysicianAccessNode toward the
   ingest/store side is PatientStatusCache → OtherDataMgmt on cache miss. The cache pins
   high-risk patient records (the hottest ones) and uses a write-through invalidation policy
   so cross-node traffic is bounded by the *change* rate, not the *query* rate. A flood of
   physician queries hitting the same handful of high-risk patients translates to ~0
   cross-node calls after warm-up.

The residual cross-node writes from PhysicianCommandService (UC7 config to ClinicalModelDB,
UC8/UC17 EHR write via PatientRecordMgmt, UC9 fetched-data write via SensorDataMgmt) DO touch
PatientDataNode. Mitigation: these rates are bounded by the QAS itself (≤25 + 20 + 20 + 20
per minute — UC6/7/8/9 combined) — orders of magnitude below the sensor-ingest write rate, so
they add negligible load. Crucially, the UC9 `addSensorData` call uses `triggerEstimation=false`
so it does NOT fan out into a redundant risk job on top of the priority one PhysicianCommandService
explicitly launches.

---

## 4. Sensitivity points

[claude] The decisions where a small parameter change swings P1 a lot:

1. **PatientStatusCache hit ratio for high-risk patients.** If pinning policy fails or the
   working set grows, cache misses spike and cross-node traffic dominates — high-risk SLA
   tightens fast (the 2 s margin is large but degrades non-linearly under load).
2. **Push-vs-pull on IRiskEvents.** Flipping this from push to pull (e.g., NotificationDispatcher
   polling for status changes) immediately violates Response item 6 — load on the risk
   pipeline becomes proportional to notification volume.
3. **Priority-queue depth / drop policy in NotificationDispatcher.** Setting the threshold
   for "overload" wrong (or letting green starve infinitely) breaks Response item 5.
4. **Co-location of PatientDataService and the risk-estimation nodes.** If a future deployment
   merges PatientDataNode with RiskEstimationMgmtNode for cost reasons, the non-interference
   guarantee collapses.
5. **PhysicianGateway thinness.** If anyone adds business logic to the gateway (auth caching,
   request aggregation, rate-limit math), it becomes a single CPU-bound chokepoint for all
   five UCs.
6. **Patient gateway availability for UC9.** PatientGatewayCommander makes a synchronous
   outbound call to the patient gateway during UC9. If the gateway is offline or slow, the
   3-min initiation SLA is at risk before any PMS-internal work even begins. The commander's
   timeout has to leave enough budget for a retry and for the subsequent priority risk job;
   set it too generous and UC9 stalls, too tight and we falsely give up.
7. **`triggerEstimation` discipline on `SensorDataMgmt.addSensorData`.** If a future caller
   forgets to pass `triggerEstimation=false` from the UC9 path, every on-demand consultation
   fires *two* risk jobs (the implicit one from SensorIngestModule plus the priority one
   from PhysicianCommandService). Doubles the load on RiskEstimationScheduler for on-demand
   traffic and could starve scheduled jobs. Worth a unit-test invariant.

---

## 5. Trade-offs

[claude]

| Attribute | Impact | Why |
| --- | --- | --- |
| Modifiability | **−** moderate | 8 new components + 2 new nodes instead of 1 placeholder. Future changes to the physician-side API now ripple through Gateway + Service + (sometimes) Cache. Mitigated by clear seams between query/command/notification/outbound-fetch. |
| Availability | **−** small | More moving parts → more failure modes. Specifically: PatientAccessNode is a single point of failure for *all* physician UCs (Av2's domain — flag for teammate). NotificationDispatcher buffering means red/yellow notifications must survive a crash (need persistent queue). |
| Performance (other QASs) | **+** small | The split actually helps P2/M2: physician load no longer competes with risk-estimation throughput. |
| Cost / complexity | **−** moderate | Two new nodes to operate. Cross-node calls add latency budget pressure (manageable given the 2 s ceiling). |
| Predictability | **+** | Resource isolation makes worst-case behaviour easier to reason about; non-interference is structural, not best-effort. |

The hardest trade-off is between **modifiability** (more components) and **predictability**
(resource isolation). The QAS demands predictability (the "no impact" clause), so the
modifiability hit is the right cost to pay — this is the central architectural trade-off
point.

---

## 6. Open questions

<!-- Anything you want Claude to think about, or where you're not sure.
     Listing it here is better than skipping it — unknowns are the point. -->

[claude]
1. **Where does the notification recipient registry live?** UC5 says "looks up the appropriate
   recipients" but there's no recipient store in the design. Options: a `RecipientRegistry`
   module inside NotificationDispatcher, or a separate component on PatientDataNode. Affects
   how UC8 risk-level updates trigger notifications.
2. **How is PatientStatusCache populated initially / kept warm for high-risk patients?**
   Need a policy: write-through on `setEstimatedPatientStatus`? Background pre-warm of all
   high-risk patients on startup?
3. **HIS integration auth model.** `healthAPI` is provided by an external HIS; how does
   HISAdapter authenticate? Out-of-scope for P1 but blocks UC8's EHR forwarding deliverable.
4. **Patient gateway auth model for outbound PMS → gateway calls.** UC9 introduces a new
   outbound direction (`patientGatewayAPI`). The existing inbound channel (UC4) authenticates
   via the gateway's token. How does the *PMS* authenticate to the gateway? Mirror of OQ3
   for HIS — out of P1 scope but blocks UC9 deliverable.
5. **Should `triggerEstimation` be a parameter on `addSensorData` or two separate operations?**
   Parameter is simpler but is a foot-gun (see sensitivity #7). Two operations
   (`addSensorData`, `addSensorDataWithoutEstimation`) make the choice explicit at every
   call site. Marginal — pick when wiring VP.

---

## 7. Coordination with teammate (Av2 overlap)

[claude] Anything P1 changes that also touches Av2's surface. Flag and confirm before
finalising the VP model — these are the seams where the two extensions either compose
cleanly or fight each other.

### Hard overlaps (need explicit agreement)

1. **`RiskEstimationScheduler` interface extension.**
   P1 extends `LaunchRiskEstimation` with `(priority, correlationId)` so UC9 jobs can jump
   ahead of scheduled ones. The scheduler is *the* P2 component, and Av2 likely adds HA
   behaviour around it. Need to agree:
   - Is the scheduler still a single instance, or does Av2 replicate it? If replicated, how
     is `correlationId` preserved across a failover so UC9's result delivery still finds
     its subscriber?
   - Priority lattice: P2 already uses dynamic priority in overload mode (high/med/low,
     2/5/8 min deadlines per Table 1.1 of the initial rationale). UC9 adds an *above-
     normal-priority* lane with a 3-min initiation budget *independent of patient risk
     level*. Confirm UC9 priority sits above all three P2 tiers and doesn't starve them.

2. **`RiskEstimationCombiner → setEstimatedPatientStatus` callback path (Figures D.5, D.6).**
   P1's `IRiskEvents` push relies on `PatientStatusModule` firing the event whenever
   `setEstimatedPatientStatus` actually changes the stored status. If Av2 modifies this
   callback path (e.g., to dedupe writes during a failover, or to fan out to a replica
   before the notifier), the IRiskEvents emission point shifts. Confirm Av2 preserves
   the "appropriate parties are notified" contract from §E.3.19.

3. **`NotificationDispatcher` durable queue for red/yellow.**
   P1 promises red/yellow notifications survive a crash (sensitivity #2 in §4, Response
   measure 5). That's a persistence/HA story that's really an Av2 concern. Either: Av2
   provides a durable queue facility that P1 sockets onto, or P1 declares its own
   on-disk write-ahead log. Pick one.

4. **`PhysicianAccessNode` is a SPOF for every physician UC.**
   Already flagged in §5 Trade-offs. UC5/6/7/8/9 all fail closed if `PhysicianAccessNode`
   is unavailable. Av2 should treat this node as a replication target if their QAS
   demands availability of physician-side flows.

### Soft overlaps (touch shared structure; usually compatible)

5. **`ClinicalModelCache` deployment + invalidation reach.**
   P1's UC7 path calls `ClinicalModelCacheMgmt.invalidateCacheEntries(patientId)` after
   writing config. If Av2 replicates the cache across multiple nodes (e.g., one per
   `RiskEstimationProcessorNode` for locality), the invalidation must reach *all*
   replicas. Today my §2c sketch assumes a single `ClinicalModelCache` on
   `RiskEstimationMgmtNode`.

6. **`PatientDataService` consolidates three legacy interfaces into one component.**
   `SensorDataMgmt` (ingest write path), `OtherDataMgmt` (status read/write), and
   `PatientRecordMgmt` (HIS EHR proxy) all live in the same runtime component on
   `PatientDataNode`. Av2 might prefer independent failure boundaries (ingest can fail
   without status reads failing, etc.) — that would split `PatientDataService` further.
   If Av2 needs that split, P1's "single cross-node hop per request" argument in §3a/§3b
   stays valid as long as the modules end up on the same node.

7. **`PatientRecordMgmt` extension (`updatePatientRecord`).**
   P1 adds a write operation to §E.3.20's interface. The HIS path is shared. If Av2
   adds retry / failover behaviour on `getPatientRecord`, it should apply symmetrically
   to `updatePatientRecord` so UC8 EHR forwards survive HIS hiccups.

8. **`SensorDataMgmt.addSensorData` signature change (`triggerEstimation` param).**
   The sensor ingest path is Av2 territory too. Any redundancy/replication of
   `SensorIngestModule` must respect the `triggerEstimation=false` flag from UC9 to
   avoid double-firing risk jobs (sensitivity #7).

### Decisions Av2 should know about (no compromise needed, just FYI)

9. **`OtherFunctionality` and `TODONode` are retired by P1.** Replacement is
   `PhysicianAccessNode` + `PatientDataNode` + `PatientGatewayCommander`. If Av2's draft
   still references `OtherFunctionality` or `TODONode`, those references need updating.
10. **`IRiskEvents` is push, not pull.** Sensitivity #2: flipping it breaks the "no impact
    on ingress" argument. Av2 should not introduce polling here even if it would simplify
    a failover scheme.
11. **`PatientGatewayCommander` adds a brand-new outbound direction (PMS → patient
    gateway).** No existing component in the initial architecture talks outbound to the
    gateway. Av2 should know this path exists in case it affects gateway-side failure
    handling.
