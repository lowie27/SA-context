# P1 Design

Pure element catalog for P1, alphabetically sorted. Each entry uses the report's catalog fields (description, super-components, sub-components or sub-modules, provided interfaces, required interfaces). Deployment and diagram references are intentionally omitted here. See the VP model for those.

Unprefixed interfaces (`SensorDataMgmt`, `PatientRecordMgmt`, `OtherDataMgmt`, `LaunchRiskEstimation`, `ClinicalModelCacheMgmt`, `healthAPI`) are inherited from the initial architecture. They appear below only when P1 modifies them or wires a new caller or provider into them.

Custom data types referenced (defined in `datatypes_reference.md`): `CorrelationId`, `PhysicianId`, `Notification`, `NotificationId`, `NotificationSeverity`, `SubscriberId`, `SubscriptionId`, `FilterCriteria`, `RiskEvent`, `Priority`.

---

## 1. Components (P1)

#### P1HISAdapter

- **description**: Outbound adapter for the external HIS healthAPI. Used by P1EHRProxyModule for UC16 reads and UC17 writes.
- **super-components**: None
- **sub-components**: None
- **provided interfaces**: P1IHISAccess
- **required interfaces**: healthAPI

#### P1NotificationDispatcher

- **description**: UC5 dispatcher. Subscribes to P1IRiskEvents and fans matching events out to the registered physicians. Priority queue orders red over yellow over green. Drops green first when over 100 per minute. Persists red and yellow to a durable log so they survive a crash.
- **super-components**: None
- **sub-modules**: P1DurableNotificationLog, P1NotificationInboxModule, P1PriorityBuffer, P1RecipientRegistry, P1RiskEventSubscriber
- **provided interfaces**: P1INotificationInbox, P1IRiskEventListener (how can this be provided twice, not possible)
- **required interfaces**: Av2EmergencyDispatch, P1IRiskEvents

#### P1PatientDataService

- **description**: Owns raw sensor data, patient status, and the HIS EHR proxy. Takes over the three interfaces OtherFunctionality exposed.
- **super-components**: None
- **sub-modules**: P1EHRProxyModule, P1PatientStatusModule, P1SensorIngestModule
- **provided interfaces**: OtherDataMgmt, P1IRiskEvents, PatientRecordMgmt, SensorDataMgmt
- **required interfaces**: LaunchRiskEstimation, P1IHISAccess, P1IRiskEventListener

#### P1PatientGatewayCommander

- **description**: Outbound channel from PMS to patient gateways for UC9 on-demand sensor-data fetches. Synchronous with a bounded timeout.
- **super-components**: None
- **sub-components**: None
- **provided interfaces**: P1IOnDemandSensorFetch
- **required interfaces**: None (in current VP model; resolved during Av2 merge)

#### P1PatientQueryService

- **description**: Handles UC6 patient-status reads. Tiered priority queue (high, medium, low) feeds the 2 / 5 / 10 s SLA tiers. The tier for an incoming query comes from an in-memory patient-to-tier index kept current by subscribing to P1IRiskEvents, so the tier decision is O(1) on the hot path. Unknown patients default to the high tier so the SLA fails safe.
- **super-components**: None
- **sub-components**: None
- **provided interfaces**: P1IPatientQuery, P1IRiskEventListener (how can this be provided twice, not possible)
- **required interfaces**: P1IPatientStatusRead, P1IRiskEvents

#### P1PatientStatusCache

- **description**: Read-through cache for patient status. Pins high-risk patients to keep status reads off the slow path.
- **super-components**: None
- **sub-components**: None
- **provided interfaces**: P1IPatientStatusRead
- **required interfaces**: OtherDataMgmt

#### P1PhysicianCommandService

- **description**: Handles physician write and command flows for UC7 (configure risk assessment), UC8 (update risk level), and UC9 (on-demand consultation).
- **super-components**: None
- **sub-modules**: P1CommandRouter, P1ConfigurationHandler, P1CorrelationTracker, P1OnDemandConsultationHandler, P1RiskLevelHandler
- **provided interfaces**: P1IPhysicianCommand, P1IRiskEventListener (how can this be provided twice, not possible)
- **required interfaces**: ClinicalModelCacheMgmt, LaunchRiskEstimation, OtherDataMgmt, P1ClinicalConfigMgmt, P1IConsultationResultPush, P1IOnDemandSensorFetch, P1IRiskEvents, PatientRecordMgmt, SensorDataMgmt (TODO implement in vpp) 

#### P1PhysicianGateway

- **description**: Single external entry point for physicians. Thin router with no business logic. Also dispatches UC9 result pushes over the physician's already-open client channel.
- **super-components**: None
- **sub-components**: None
- **provided interfaces**: P1IConsultationResultPush (why is this here? what does it do TODO add this), P1IPhysicianAPI
- **required interfaces**: P1INotificationInbox, P1IPatientQuery, P1IPhysicianCommand

---

## 2. Modules (P1)

#### P1CommandRouter

- **description**: Provides P1IPhysicianCommand at the component boundary. Thin dispatcher that routes each operation to the matching handler.
- **super-components**: P1PhysicianCommandService
- **sub-modules**: None
- **provided interfaces**: P1IPhysicianCommand
- **required interfaces**: P1IConfigCommand, P1IOnDemandCommand, P1IRiskLevelCommand

#### P1ConfigurationHandler

- **description**: UC7 handler. Overwrites the per-patient ClinicalModelConfiguration via P1ClinicalConfigMgmt, invalidates the clinical-model cache via ClinicalModelCacheMgmt, then launches a normal-priority risk-estimation job via LaunchRiskEstimation so Response item 2 (re-assessment after config) is met. The launch carries no fresh sensor data, so the scheduler runs the job against the patient's latest stored package.
- **super-components**: P1PhysicianCommandService
- **sub-modules**: None
- **provided interfaces**: P1IConfigCommand
- **required interfaces**: ClinicalModelCacheMgmt, LaunchRiskEstimation, P1ClinicalConfigMgmt

#### P1CorrelationTracker

- **description**: Mints CorrelationIds for UC9 requests and matches incoming RiskEvents back to the originating handler. Single locus an Av2 failover must preserve so UC9 result delivery survives.
- **super-components**: P1PhysicianCommandService
- **sub-modules**: None
- **provided interfaces**: P1ICorrelationTracking
- **required interfaces**: None

#### P1DurableNotificationLog

- **description**: Write-ahead log. Red and yellow notifications are persisted here before being acked from P1IRiskEvents so they survive a crash.
- **super-components**: P1NotificationDispatcher
- **sub-modules**: None
- **provided interfaces**: P1INotificationLog
- **required interfaces**: None

#### P1EHRProxyModule

- **description**: Fronts HIS for EHR reads and writes via PatientRecordMgmt. Stale-tolerant cache on reads. Cache is invalidated structurally on PMS-initiated writes only. External updates made directly on the HIS are not observed, which is acceptable in normal mode and flagged as a sensitivity point.
- **super-components**: P1PatientDataService
- **sub-modules**: None
- **provided interfaces**: PatientRecordMgmt
- **required interfaces**: P1IHISAccess

#### P1NotificationInboxModule

- **description**: Provides P1INotificationInbox at the component boundary. Owns per-physician subscription state and the pull surface used by P1PhysicianGateway (getPendingNotifications and acknowledgeNotification).
- **super-components**: P1NotificationDispatcher
- **sub-modules**: None
- **provided interfaces**: P1INotificationInbox
- **required interfaces**: P1INotificationLog, P1IPriorityBuffer, P1IRecipientRegistry

#### P1OnDemandConsultationHandler

- **description**: UC9 orchestrator. Mints a CorrelationId via P1CorrelationTracker, fetches sensor data via P1IOnDemandSensorFetch, stores it via `SensorDataMgmt.addSensorData(..., triggerEstimation=false)` so the implicit scheduled job is suppressed, then launches a priority risk job via LaunchRiskEstimation carrying the CorrelationId. When the matching RiskEvent arrives on P1IRiskEventListener, the handler pushes the result back to the originating physician via P1IConsultationResultPush. No polling.
- **super-components**: P1PhysicianCommandService
- **sub-modules**: None
- **provided interfaces**: P1IOnDemandCommand, P1IRiskEventListener (TODO add this)
- **required interfaces**: LaunchRiskEstimation, P1IConsultationResultPush, P1ICorrelationTracking, P1IOnDemandSensorFetch, P1IRiskEvents, SensorDataMgmt

#### P1PatientStatusModule

- **description**: Owns patient-status records. On `setEstimatedPatientStatus` that actually changes the stored status, the module pushes a RiskEvent to every matching subscription via P1IRiskEventListener.
- **super-components**: P1PatientDataService
- **sub-modules**: None
- **provided interfaces**: OtherDataMgmt, P1IRiskEvents
- **required interfaces**: P1IRiskEventListener (TODO add this)

#### P1PriorityBuffer

- **description**: In-memory priority queue ordering red over yellow over green. Enforces the 100 per minute threshold and drops green first under overload.
- **super-components**: P1NotificationDispatcher
- **sub-modules**: None
- **provided interfaces**: P1IPriorityBuffer
- **required interfaces**: None

#### P1RecipientRegistry

- **description**: Maps a patient-status event to the physicians who should be notified about it. UC5 recipient resolution lives here. Populated via subscribeToNotifications on P1INotificationInbox.
- **super-components**: P1NotificationDispatcher
- **sub-modules**: None
- **provided interfaces**: P1IRecipientRegistry
- **required interfaces**: None

#### P1RiskEventSubscriber

- **description**: Provides P1IRiskEventListener at the component boundary. On event, queries P1RecipientRegistry, persists red and yellow to P1DurableNotificationLog, then enqueues to P1PriorityBuffer. Owns the outbound Av2EmergencyDispatch socket for backup-channel escalation of red notifications.
- **super-components**: P1NotificationDispatcher
- **sub-modules**: None
- **provided interfaces**: P1IRiskEventListener (TODO add this)
- **required interfaces**: Av2EmergencyDispatch, P1INotificationLog, P1IPriorityBuffer, P1IRecipientRegistry, P1IRiskEvents

#### P1RiskLevelHandler

- **description**: UC8 handler. Writes the new internal risk level via `OtherDataMgmt.setEstimatedPatientStatus`, which fires P1IRiskEvents so the tier index, recipient fan-out, and any UC9 correlations all observe the change. Then forwards the change to the EHR via PatientRecordMgmt.
- **super-components**: P1PhysicianCommandService
- **sub-modules**: None
- **provided interfaces**: P1IRiskLevelCommand
- **required interfaces**: OtherDataMgmt (TODO add this), PatientRecordMgmt

#### P1SensorIngestModule

- **description**: Accepts and persists raw sensor data. Launches a risk-estimation job via LaunchRiskEstimation only when the caller sets `triggerEstimation=true` (the UC4 default). UC9 callers pass `false` to suppress the implicit scheduled job in favour of the priority one P1OnDemandConsultationHandler launches explicitly.
- **super-components**: P1PatientDataService
- **sub-modules**: None
- **provided interfaces**: SensorDataMgmt
- **required interfaces**: LaunchRiskEstimation

---

## 3. Interfaces

> Convention: prefix every parameter and return type with `Datatypes.` in VP. The signatures below omit the prefix for readability.

#### ClinicalModelCacheMgmt

- **provided by**: ClinicalModelCache (unchanged)
- **required by**: P1ConfigurationHandler, P1PhysicianCommandService
- **operations**:
  - `invalidateCacheEntries(patientId: PatientId)`
    - effect: Invalidate all cached items for the patient identified by patientId. If the cache holds nothing for this patient, nothing changes. After invalidation, the next request leads to a fresh fetch from the database and a fresh cache population.

#### healthAPI

- **provided by**: HIS (external, unchanged)
- **required by**: P1HISAdapter
- **operations**: unchanged from §E.3.8 of the initial rationale (FHIR-derived: `getObservation`, `getPatient`, `getRiskAssessment`, `saveObservation`, `savePatient`, `saveRiskAssessment`, `deleteObservation`, `deletePatient`, `deleteRiskAssessment`, `searchObservation`, `searchPatient`, `searchRiskAssessment`). Signatures live in §E.3.8 and are not reproduced here.

#### LaunchRiskEstimation

- **provided by**: RiskEstimationScheduler (extended by P1 to gain priority and correlationId parameters so UC9 jobs can jump ahead of scheduled ones, and to allow callers to omit `newSensorData` for UC7 re-assessments)
- **required by**: P1ConfigurationHandler, P1OnDemandConsultationHandler, P1PatientDataService, P1PhysicianCommandService, P1SensorIngestModule
- **operations**:
  - `launchRiskEstimation(patientId: PatientId, priority: Priority, correlationId: CorrelationId [0..1], newSensorData: SensorDataPackage [0..1], receivedAt: Timestamp [0..1])`
    - effect: Submit a risk-estimation job for the patient. When `newSensorData` is supplied (UC4, UC9), the job runs against that package. When omitted (UC7 reconfigure), the scheduler runs the job against the patient's most recent stored sensor data. The scheduler orders the job by `priority` (UC9 HIGH jumps ahead of scheduled jobs) and carries `correlationId` through to the resulting P1IRiskEvents event so subscribers can match it. Scheduling is otherwise unchanged: FIFO in normal mode, dynamic priority (earliest-deadline-first with tighter deadlines for high-risk patients) in overload mode.

#### OtherDataMgmt

- **provided by**: P1PatientStatusModule (was OtherFunctionality), also at the P1PatientDataService boundary
- **required by**: ClinicalJobCreator, ClinicalModelCache, P1PatientStatusCache, P1PhysicianCommandService, P1RiskLevelHandler, RiskEstimationCombiner, RiskEstimationScheduler
- **operations**:
  - `getPatientStatus(patientId: PatientId): PatientStatus`
    - effect: Fetch the patient's most recent persisted status.
    - returns: The patient's most recent persisted status.
  - `setEstimatedPatientStatus(patientId: PatientId, status: PatientStatus, timestamp: Timestamp)`
    - effect: Update the patient status and the time of estimation. If the patient's estimated risk changed, the appropriate parties are notified via P1IRiskEvents (this is the emission point).

#### P1ClinicalConfigMgmt

- **provided by**: ClinicalModelDB (new interface alongside the existing read-only ClinicalModelStorage)
- **required by**: P1ConfigurationHandler, P1PhysicianCommandService
- **operations**:
  - `setClinicalModelConfigForPatient(patientId: PatientId, config: ClinicalModelConfiguration)`
    - effect: Overwrite the per-patient ClinicalModelConfiguration in ClinicalModelDB. The caller is expected to invalidate ClinicalModelCache after a successful write.

#### P1IConfigCommand

- **provided by**: P1ConfigurationHandler
- **required by**: P1CommandRouter
- **operations**:
  - `configurePatientRiskAssessment(patientId: PatientId, config: ClinicalModelConfiguration)`
    - effect: Internal delegate. P1CommandRouter forwards the UC7 request to P1ConfigurationHandler, which writes the config, invalidates the cache, and launches a normal-priority risk-estimation job so the new configuration takes effect immediately.

#### P1IConsultationResultPush

- **provided by**: P1PhysicianGateway
- **required by**: P1OnDemandConsultationHandler, P1PhysicianCommandService
- **operations**:
  - `deliverConsultationResult(physicianId: PhysicianId, correlationId: CorrelationId, status: PatientStatus)`
    - effect: P1OnDemandConsultationHandler pushes the UC9 result to the Gateway, which routes it over the originating physician's already-open client channel. Removes the polling round-trip and lets the 1-minute result-delivery SLA be met without busy-wait.

#### P1ICorrelationTracking

- **provided by**: P1CorrelationTracker
- **required by**: P1OnDemandConsultationHandler
- **operations**:
  - `newCorrelationId(): CorrelationId`
    - effect: Mint a fresh CorrelationId and record it in the pending set so a future P1IRiskEvents push can be matched back to this UC9 request.
    - returns: The freshly minted CorrelationId.
  - `consumeCorrelation(correlationId: CorrelationId): boolean`
    - effect: Resolve a CorrelationId by removing it from the pending set if present. Report whether the event matches a tracked UC9 request.
    - returns: True if the CorrelationId was pending and has now been consumed. False otherwise.

#### P1IHISAccess

- **provided by**: P1HISAdapter
- **required by**: P1EHRProxyModule, P1PatientDataService
- **operations**:
  - `getPatientRecord(patientId: PatientId): PatientRecord`
    - effect: Read the patient's record from the external HIS by combining the relevant healthAPI reads into one record-level view.
    - returns: The aggregated PatientRecord (demographics, observations, and risk assessments per FHIR) retrieved from the HIS.
  - `updatePatientRecord(patientId: PatientId, record: PatientRecord)`
    - effect: Write the patient record back to the HIS by dispatching the appropriate healthAPI save operations.

#### P1INotificationInbox

- **provided by**: P1NotificationInboxModule, also at the P1NotificationDispatcher boundary
- **required by**: P1PhysicianGateway
- **operations**:
  - `subscribeToNotifications(physicianId: PhysicianId)`
    - effect: Add the physician to the recipient registry so future patient-status events fan out to them.
  - `unsubscribeFromNotifications(physicianId: PhysicianId)`
    - effect: Remove the physician from the recipient registry. Notifications already queued are still delivered.
  - `getPendingNotifications(physicianId: PhysicianId): List<Notification>`
    - effect: Return the physician's queued notifications in priority order (red, yellow, green) without removing them from the buffer.
    - returns: The list of pending Notifications for the physician, ordered by severity then arrival.
  - `acknowledgeNotification(notificationId: NotificationId)`
    - effect: Confirm delivery of the notification so the dispatcher can remove it from the priority buffer and the durable log.

#### P1INotificationLog

- **provided by**: P1DurableNotificationLog
- **required by**: P1NotificationInboxModule, P1RiskEventSubscriber
- **operations**:
  - `persist(notification: Notification)`
    - effect: Write the notification to the durable log before it enters the in-memory buffer so a crash cannot lose a red or yellow notification.
  - `markDelivered(notificationId: NotificationId)`
    - effect: Remove the notification from the durable log once the physician has acked delivery, bounding storage growth.
  - `recoverPending(): List<Notification>`
    - effect: On startup, return notifications that were persisted but never acked so P1NotificationInboxModule can replay them into the priority buffer.
    - returns: The list of unacked Notifications recovered from the durable log.

#### P1IOnDemandCommand

- **provided by**: P1OnDemandConsultationHandler
- **required by**: P1CommandRouter
- **operations**:
  - `requestOnDemandConsultation(patientId: PatientId, physicianId: PhysicianId): CorrelationId`
    - effect: Internal delegate. P1CommandRouter forwards the UC9 request to P1OnDemandConsultationHandler, which mints a CorrelationId, fetches sensor data, launches a priority risk job, and (when the matching RiskEvent arrives) pushes the result back to physicianId via P1IConsultationResultPush.
    - returns: The CorrelationId minted for this UC9 request.

#### P1IOnDemandSensorFetch

- **provided by**: P1PatientGatewayCommander
- **required by**: P1OnDemandConsultationHandler, P1PhysicianCommandService
- **operations**:
  - `requestCurrentSensorData(patientId: PatientId, correlationId: CorrelationId): SensorDataPackage`
    - effect: Synchronously ask the patient gateway to push the current sensor reading. Bounded by the commander timeout sized inside the 3-minute UC9 initiation budget.
    - returns: The fresh SensorDataPackage supplied by the patient gateway for this UC9 request.

#### P1IPatientQuery

- **provided by**: P1PatientQueryService
- **required by**: P1PhysicianGateway
- **operations**:
  - `consultPatientStatus(patientId: PatientId): PatientStatus`
    - effect: Read the patient's current status. The request is dispatched to the priority tier (high, medium, low) matching the patient's last-known risk level from the in-memory tier index inside P1PatientQueryService, so the tier decision is O(1) on the hot path. Unknown patients fall back to the high tier so the SLA fails safe.
    - returns: The patient's most recent status.

#### P1IPatientStatusRead

- **provided by**: P1PatientStatusCache
- **required by**: P1PatientQueryService
- **operations**:
  - `getPatientStatus(patientId: PatientId): PatientStatus`
    - effect: Cache-first read of the patient's status. On miss, the cache fetches via OtherDataMgmt and populates itself for subsequent reads.
    - returns: The patient's most recent status.

#### P1IPhysicianAPI

- **provided by**: P1PhysicianGateway
- **required by**: (external; physicians calling into the PMS)
- **operations**:
  - `consultPatientStatus(patientId: PatientId): PatientStatus`
    - effect: Return the named patient's current status. SLA tier (2 / 5 / 10 s) is picked from the patient's last-known risk level via the tier index inside P1PatientQueryService.
    - returns: The patient's most recent status: risk level, supporting data, and timestamp.
  - `requestOnDemandConsultation(patientId: PatientId, physicianId: PhysicianId): CorrelationId`
    - effect: Initiate an on-demand consultation. Fetches fresh sensor data, launches a priority risk estimation, and (when the result is ready) pushes it back to physicianId via P1IConsultationResultPush. No polling.
    - returns: The CorrelationId minted for this request, used by the client to match the eventual push.
  - `configurePatientRiskAssessment(patientId: PatientId, config: ClinicalModelConfiguration)`
    - effect: Overwrite the per-patient ClinicalModelConfiguration, invalidate the clinical-model cache, and launch a normal-priority risk-estimation job so the new configuration takes effect immediately.
  - `updatePatientRiskLevel(patientId: PatientId, status: PatientStatus)`
    - effect: Update the patient's internal risk level (which fires P1IRiskEvents) and forward the change to the EHR via the HIS proxy.
  - `subscribeToNotifications(physicianId: PhysicianId)`
    - effect: Register a physician to receive notifications about subsequent patient-status changes.
  - `unsubscribeFromNotifications(physicianId: PhysicianId)`
    - effect: Remove the physician from the recipient registry. Notifications already queued for them are still delivered.
  - `getPendingNotifications(physicianId: PhysicianId): List<Notification>`
    - effect: Return the physician's queued notifications in priority order (red, yellow, green) without removing them from the buffer.
    - returns: The list of pending Notifications for the physician, ordered by severity then arrival.
  - `acknowledgeNotification(notificationId: NotificationId)`
    - effect: Confirm delivery of the notification so the dispatcher can remove it from the priority buffer and the durable log.

#### P1IPhysicianCommand

- **provided by**: P1CommandRouter, also at the P1PhysicianCommandService boundary
- **required by**: P1PhysicianGateway
- **operations**:
  - `configurePatientRiskAssessment(patientId: PatientId, config: ClinicalModelConfiguration)`
    - effect: Overwrite the per-patient ClinicalModelConfiguration, invalidate the clinical-model cache, and launch a normal-priority risk-estimation job so the new configuration takes effect immediately.
  - `updatePatientRiskLevel(patientId: PatientId, status: PatientStatus)`
    - effect: Update the patient's internal risk level (which fires P1IRiskEvents) and forward the change to the EHR via the HIS proxy.
  - `requestOnDemandConsultation(patientId: PatientId, physicianId: PhysicianId): CorrelationId`
    - effect: Initiate an on-demand consultation. Mints a CorrelationId, fetches fresh sensor data, launches a priority risk job, and pushes the result back to physicianId via P1IConsultationResultPush when ready.
    - returns: The CorrelationId minted for this UC9 request.

#### P1IPriorityBuffer

- **provided by**: P1PriorityBuffer
- **required by**: P1NotificationInboxModule, P1RiskEventSubscriber
- **operations**:
  - `enqueue(physicianId: PhysicianId, notification: Notification)`
    - effect: Add the notification to the physician's pending set. Severity drives priority ordering and the drop-green-first policy when the 100 per minute threshold is exceeded.
  - `peekPending(physicianId: PhysicianId): List<Notification>`
    - effect: Return the physician's pending notifications in priority order (red, yellow, green) without removing them from the buffer.
    - returns: The list of pending Notifications for the physician, ordered by severity then arrival.
  - `removePending(notificationId: NotificationId)`
    - effect: Remove the notification from the buffer after the physician acks it through P1INotificationInbox.

#### P1IRecipientRegistry

- **provided by**: P1RecipientRegistry
- **required by**: P1NotificationInboxModule, P1RiskEventSubscriber
- **operations**:
  - `subscribe(physicianId: PhysicianId)`
    - effect: Add the physician to the notification recipient set.
  - `unsubscribe(physicianId: PhysicianId)`
    - effect: Remove the physician from the notification recipient set.
  - `findRecipients(patientId: PatientId): List<PhysicianId>`
    - effect: Look up the physicians who should be notified about events for the patient. Backs UC5 recipient resolution.
    - returns: The list of PhysicianIds subscribed to events for this patient.

#### P1IRiskEventListener

- **provided by**: P1OnDemandConsultationHandler (also at the P1PhysicianCommandService boundary), P1PatientQueryService, P1RiskEventSubscriber (also at the P1NotificationDispatcher boundary)
- **required by**: P1PatientDataService, P1PatientStatusModule
- **operations**:
  - `onRiskEvent(event: RiskEvent)`
    - effect: Push callback invoked by P1PatientStatusModule for every subscription whose FilterCriteria matches the event. Receiver applies its own handling: P1RiskEventSubscriber fans out to physicians, P1OnDemandConsultationHandler delivers UC9 results, P1PatientQueryService refreshes its tier index.

#### P1IRiskEvents

- **provided by**: P1PatientStatusModule, also at the P1PatientDataService boundary
- **required by**: P1NotificationDispatcher, P1OnDemandConsultationHandler, P1PatientQueryService, P1PhysicianCommandService, P1RiskEventSubscriber
- **operations**:
  - `subscribe(subscriberId: SubscriberId, filter: FilterCriteria): SubscriptionId`
    - effect: Register a subscriber to receive RiskEvent payloads matching the filter. An empty filter receives every event. Delivery happens by callback on the subscriber's P1IRiskEventListener.
    - returns: A SubscriptionId the caller uses to unsubscribe later.
  - `unsubscribe(subscriptionId: SubscriptionId)`
    - effect: Remove the named subscription so no further RiskEvents are pushed to it.

#### P1IRiskLevelCommand

- **provided by**: P1RiskLevelHandler
- **required by**: P1CommandRouter
- **operations**:
  - `updatePatientRiskLevel(patientId: PatientId, status: PatientStatus)`
    - effect: Internal delegate. P1CommandRouter forwards the UC8 request to P1RiskLevelHandler, which writes the new internal risk level via `OtherDataMgmt.setEstimatedPatientStatus` (firing P1IRiskEvents) and then forwards to the EHR via PatientRecordMgmt.

#### PatientRecordMgmt

- **provided by**: P1EHRProxyModule (was OtherFunctionality), also at the P1PatientDataService boundary
- **required by**: P1PhysicianCommandService, P1RiskLevelHandler, RiskEstimationCombiner
- **operations**:
  - `getPatientRecord(patientId: PatientId): PatientRecord`
    - effect: Fetch the EHR record of the patient with given id from the HIS and return it. If the HIS is unavailable, return an older cached copy of the patient record when possible.
    - returns: The PatientRecord fronted by the EHR cache.
  - `updatePatientRecord(patientId: PatientId, record: PatientRecord)`
    - effect: Write the patient record back to HIS via P1HISAdapter and invalidate the EHR cache entry so subsequent reads see the new value.

#### SensorDataMgmt

- **provided by**: P1SensorIngestModule (was OtherFunctionality), also at the P1PatientDataService boundary
- **required by**: Av2DataIngestionService, ClinicalJobCreator, P1OnDemandConsultationHandler, P1PhysicianCommandService, RiskEstimationScheduler
- **operations**:
  - `addSensorData(patientId: PatientId, sensorData: SensorDataPackage, receivedAt: Timestamp, triggerEstimation: boolean)`
    - effect: Store the given sensor data and meta-data. When `triggerEstimation` is true (UC4 default) the ingest module launches a risk-estimation job on the new data. When false the call is store-only. UC9 callers pass `false` because they explicitly launch a priority risk job and would otherwise double the scheduler load for every on-demand consultation.
  - `getAllSensorDataOfPatient(patientId: PatientId): Map<Timestamp, SensorDataPackage>`
    - effect: Fetch and return all sensor data belonging to the patient identified by patientId.
    - returns: The sensor data and the timestamp of arrival in the system.
  - `getAllSensorDataOfPatientBefore(patientId: PatientId, stopTime: Timestamp): Map<Timestamp, SensorDataPackage>`
    - effect: Fetch and return all sensor data belonging to the patient identified by patientId which was received before the specified stopTime.
    - returns: The sensor data and the timestamp of arrival in the system.
