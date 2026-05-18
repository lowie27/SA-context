# P1 Design

Pure element catalog for P1, alphabetically sorted. Each entry uses the report's catalog fields (description, super-components, sub-components/sub-modules, provided interfaces, required interfaces). Deployment and diagram references are intentionally omitted here — see the VP model for those.

Unprefixed interfaces (`SensorDataMgmt`, `PatientRecordMgmt`, `OtherDataMgmt`, `LaunchRiskEstimation`, `ClinicalModelCacheMgmt`, `healthAPI`) are inherited from the initial architecture; they appear below only when P1 modifies them or wires a new caller/provider into them.

Custom data types referenced (defined in `datatypes_reference.md`): `ConsultationId`, `CorrelationId`, `PhysicianId`, `Notification`, `NotificationId`, `NotificationSeverity`, `SubscriberId`, `SubscriptionId`, `FilterCriteria`, `RiskEvent`, `Priority`.

---

## 1. Components (P1)

#### P1HISAdapter

- **description**: outbound adapter for the external HIS healthAPI; used by P1EHRProxyModule for UC16 reads and UC17 writes.
- **super-components**: None
- **sub-components**: None
- **provided interfaces**: P1IHISAccess
- **required interfaces**: healthAPI

#### P1NotificationDispatcher

- **description**: UC5 dispatcher; subscribes to P1IRiskEvents; red > yellow > green priority queue; drops green first under overload; persists red/yellow so they cannot be lost.
- **super-components**: None
- **sub-modules**: P1DurableNotificationLog, P1NotificationInboxModule, P1PriorityBuffer, P1RecipientRegistry, P1RiskEventSubscriber
- **provided interfaces**: P1INotificationInbox
- **required interfaces**: Av2EmergencyDispatch, P1IRiskEvents

#### P1PatientDataService

- **description**: owns raw sensor data, patient status, and the HIS EHR proxy; takes over the three interfaces OtherFunctionality exposed.
- **super-components**: None
- **sub-modules**: P1EHRProxyModule, P1PatientStatusModule, P1SensorIngestModule
- **provided interfaces**: OtherDataMgmt, P1IRiskEvents, PatientRecordMgmt, SensorDataMgmt
- **required interfaces**: LaunchRiskEstimation, P1IHISAccess

#### P1PatientGatewayCommander

- **description**: outbound channel from PMS to patient gateways for UC9 on-demand sensor-data fetches; synchronous with a bounded timeout.
- **super-components**: None
- **sub-components**: None
- **provided interfaces**: P1IOnDemandSensorFetch
- **required interfaces**: None (in current VP model)

#### P1PatientQueryService

- **description**: handles UC6 patient-status reads; tiered priority queue (high > medium > low) feeds the 2/5/10 s SLA tiers. The tier for an incoming query is taken from an in-memory patient→risk-tier index kept current by subscribing to `P1IRiskEvents` (every status change updates the index before the notification path sees it); unknown patients default to the high tier so the SLA fails safe rather than slow.
- **super-components**: None
- **sub-components**: None
- **provided interfaces**: P1IPatientQuery
- **required interfaces**: P1IPatientStatusRead, P1IRiskEvents

#### P1PatientStatusCache

- **description**: read-through cache for patient status; pins high-risk patients to keep status reads off the slow path.
- **super-components**: None
- **sub-components**: None
- **provided interfaces**: P1IPatientStatusRead
- **required interfaces**: OtherDataMgmt

#### P1PhysicianCommandService

- **description**: handles physician write/command flows — UC7 (configure risk assessment), UC8 (update risk level), UC9 (on-demand consultation).
- **super-components**: None
- **sub-modules**: P1CommandRouter, P1ConfigurationHandler, P1CorrelationTracker, P1OnDemandConsultationHandler, P1RiskLevelHandler
- **provided interfaces**: P1IPhysicianCommand
- **required interfaces**: ClinicalModelCacheMgmt, LaunchRiskEstimation, P1ClinicalConfigMgmt, P1IOnDemandSensorFetch, P1IRiskEvents, PatientRecordMgmt, SensorDataMgmt

#### P1PhysicianGateway

- **description**: single external entry point for physicians; thin router with no business logic.
- **super-components**: None
- **sub-components**: None
- **provided interfaces**: P1IPhysicianAPI
- **required interfaces**: P1INotificationInbox, P1IPatientQuery, P1IPhysicianCommand

---

## 2. Modules (P1)

#### P1CommandRouter

- **description**: provides P1IPhysicianCommand at the component boundary; thin dispatcher that routes each operation to the matching handler.
- **super-components**: P1PhysicianCommandService
- **sub-modules**: None
- **provided interfaces**: P1IPhysicianCommand
- **required interfaces**: P1IConfigCommand, P1IOnDemandCommand, P1IRiskLevelCommand

#### P1ConfigurationHandler

- **description**: UC7 — overwrites the per-patient ClinicalModelConfiguration, then invalidates the clinical-model cache.
- **super-components**: P1PhysicianCommandService
- **sub-modules**: None
- **provided interfaces**: P1IConfigCommand
- **required interfaces**: ClinicalModelCacheMgmt, P1ClinicalConfigMgmt

#### P1CorrelationTracker

- **description**: mints CorrelationIds for UC9 requests and matches incoming RiskEvents back to the originating handler; the single locus an Av2 failover must preserve so UC9 result delivery survives.
- **super-components**: P1PhysicianCommandService
- **sub-modules**: None
- **provided interfaces**: P1ICorrelationTracking
- **required interfaces**: None

#### P1DurableNotificationLog

- **description**: write-ahead log; red and yellow notifications are persisted here before being acked from P1IRiskEvents so they survive a crash.
- **super-components**: P1NotificationDispatcher
- **sub-modules**: None
- **provided interfaces**: P1INotificationLog
- **required interfaces**: None

#### P1EHRProxyModule

- **description**: fronts HIS for EHR reads and writes via PatientRecordMgmt; stale-tolerant cache on reads, invalidated structurally on writes.
- **super-components**: P1PatientDataService
- **sub-modules**: None
- **provided interfaces**: PatientRecordMgmt
- **required interfaces**: P1IHISAccess

#### P1NotificationInboxModule

- **description**: provides P1INotificationInbox at the component boundary; owns per-physician subscription state and the pull surface used by P1PhysicianGateway (getPendingNotifications / acknowledgeNotification).
- **super-components**: P1NotificationDispatcher
- **sub-modules**: None
- **provided interfaces**: P1INotificationInbox
- **required interfaces**: P1INotificationLog, P1IPriorityBuffer, P1IRecipientRegistry

#### P1OnDemandConsultationHandler

- **description**: UC9 orchestration — mints a CorrelationId via P1CorrelationTracker, fetches sensor data via P1IOnDemandSensorFetch, stores it via `SensorDataMgmt.addSensorData(..., triggerEstimation=false)` so the implicit scheduled job is suppressed, launches a priority risk job carrying the CorrelationId, then stashes the matching `P1IRiskEvents` payload indexed by ConsultationId so `pollConsultationResult` can return it.
- **super-components**: P1PhysicianCommandService
- **sub-modules**: None
- **provided interfaces**: P1IOnDemandCommand
- **required interfaces**: LaunchRiskEstimation, P1ICorrelationTracking, P1IOnDemandSensorFetch, P1IRiskEvents, SensorDataMgmt

#### P1PatientStatusModule

- **description**: owns patient-status records; fires P1IRiskEvents when setEstimatedPatientStatus actually changes the stored status.
- **super-components**: P1PatientDataService
- **sub-modules**: None
- **provided interfaces**: OtherDataMgmt, P1IRiskEvents
- **required interfaces**: None

#### P1PriorityBuffer

- **description**: in-memory priority queue (red > yellow > green); enforces the 100/min threshold and drops green first under overload.
- **super-components**: P1NotificationDispatcher
- **sub-modules**: None
- **provided interfaces**: P1IPriorityBuffer
- **required interfaces**: None

#### P1RecipientRegistry

- **description**: maps a patient-status event to the physicians who should be notified about it; UC5's "looks up the appropriate recipients" lives here. Populated via subscribeToNotifications on P1INotificationInbox.
- **super-components**: P1NotificationDispatcher
- **sub-modules**: None
- **provided interfaces**: P1IRecipientRegistry
- **required interfaces**: None

#### P1RiskEventSubscriber

- **description**: requires P1IRiskEvents at the component boundary; on event, queries P1RecipientRegistry, persists red/yellow to P1DurableNotificationLog, then enqueues to P1PriorityBuffer. Also owns the outbound Av2EmergencyDispatch socket for backup-channel escalation of red notifications.
- **super-components**: P1NotificationDispatcher
- **sub-modules**: None
- **provided interfaces**: None
- **required interfaces**: Av2EmergencyDispatch, P1INotificationLog, P1IPriorityBuffer, P1IRecipientRegistry, P1IRiskEvents

#### P1RiskLevelHandler

- **description**: UC8 — writes the updated risk level via PatientRecordMgmt; the EHR forward is structural through P1EHRProxyModule.
- **super-components**: P1PhysicianCommandService
- **sub-modules**: None
- **provided interfaces**: P1IRiskLevelCommand
- **required interfaces**: PatientRecordMgmt

#### P1SensorIngestModule

- **description**: accepts and persists raw sensor data; on arrival launches a risk-estimation job via LaunchRiskEstimation **only when the caller sets `triggerEstimation=true`** (the UC4 default). UC9 callers pass `false` to suppress the implicit scheduled job in favour of the priority one P1OnDemandConsultationHandler launches explicitly.
- **super-components**: P1PatientDataService
- **sub-modules**: None
- **provided interfaces**: SensorDataMgmt
- **required interfaces**: LaunchRiskEstimation

---

## 3. Interfaces

> Convention: prefix every parameter and return type with `Datatypes.` in VP. The signatures below omit the prefix for readability.

#### ClinicalModelCacheMgmt

- **provided by**: ClinicalModelCache (unchanged)
- **required by**: P1ConfigurationHandler
- **operations**:
    - `invalidateCacheEntries(patientId: PatientId)`
        - effect: Invalidate (remove) all items in the cache for the patient identified by patientId. If the cache does not contain any items for this patient, nothing is changed. After invalidating the cached items for a certain patient, the next request for them will lead to fetching them from the database and storing them in the cache again.

#### healthAPI

- **provided by**: HIS (external, unchanged)
- **required by**: P1HISAdapter
- **operations**: unchanged from §E.3.8 of the initial rationale (FHIR-derived: `getObservation`, `getPatient`, `getRiskAssessment`, `saveObservation`, `savePatient`, `saveRiskAssessment`, `deleteObservation`, `deletePatient`, `deleteRiskAssessment`, `searchObservation`, `searchPatient`, `searchRiskAssessment`). Signatures live in §E.3.8 — not reproduced here.

#### LaunchRiskEstimation

- **provided by**: RiskEstimationScheduler (extended by P1 — gains a priority and correlationId parameter so UC9 jobs can jump ahead of scheduled ones)
- **required by**: P1OnDemandConsultationHandler, P1SensorIngestModule
- **operations**:
    - `launchRiskEstimation(patientId: PatientId, newSensorData: SensorDataPackage, receivedAt: Timestamp, priority: Priority, correlationId: CorrelationId)`
        - effect: Submits a risk-estimation job for the patient against the supplied sensor data; the scheduler orders the job by `priority` (UC9 HIGH jumps ahead of scheduled jobs) and carries `correlationId` through to the resulting P1IRiskEvents event so subscribers can match it. Scheduling otherwise unchanged: in normal mode FIFO; in overload mode the system switches to dynamic priority (earliest-deadline-first with tighter deadlines for high-risk patients).

#### OtherDataMgmt

- **provided by**: P1PatientStatusModule (was OtherFunctionality)
- **required by**: ClinicalJobCreator, ClinicalModelCache, P1PatientStatusCache, RiskEstimationCombiner, RiskEstimationScheduler
- **operations**:
    - `getPatientStatus(patientId: PatientId): PatientStatus`
        - effect: Fetch and return the status of the patient identified by the patientId.
        - returns: The patient's most recent persisted status.
    - `setEstimatedPatientStatus(patientId: PatientId, status: PatientStatus, timestamp: Timestamp)`
        - effect: Update the patient status estimation of the patient identified by patientId to the given value and update the time of estimation. If the patient's estimated risk changed, the appropriate parties are notified via P1IRiskEvents (this is the emission point).

#### P1ClinicalConfigMgmt

- **provided by**: ClinicalModelDB (new interface alongside the existing read-only ClinicalModelStorage)
- **required by**: P1ConfigurationHandler
- **operations**:
    - `setClinicalModelConfigForPatient(patientId: PatientId, config: ClinicalModelConfiguration)`
        - effect: Overwrites the per-patient ClinicalModelConfiguration in ClinicalModelDB. The caller is expected to invalidate ClinicalModelCache after a successful write.

#### P1IConfigCommand

- **provided by**: P1ConfigurationHandler
- **required by**: P1CommandRouter
- **operations**:
    - `configurePatientRiskAssessment(patientId: PatientId, config: ClinicalModelConfiguration)`
        - effect: Internal delegate — P1CommandRouter forwards the UC7 request to P1ConfigurationHandler, which writes the config and triggers the cache invalidation.

#### P1ICorrelationTracking

- **provided by**: P1CorrelationTracker
- **required by**: P1OnDemandConsultationHandler
- **operations**:
    - `newCorrelationId(): CorrelationId`
        - effect: Mints a fresh CorrelationId and records it in the pending set so a future P1IRiskEvents can be matched back to this UC9 request.
        - returns: The freshly minted CorrelationId.
    - `consumeCorrelation(correlationId: CorrelationId): boolean`
        - effect: Resolves a CorrelationId by removing it from the pending set if present; reports whether the event matches a tracked UC9 request.
        - returns: True if the CorrelationId was pending (and has now been consumed); false otherwise.

#### P1IHISAccess

- **provided by**: P1HISAdapter
- **required by**: P1EHRProxyModule, P1PatientDataService
- **operations**:
    - `getPatientRecord(patientId: PatientId): PatientRecord`
        - effect: Reads the patient's record from the external HIS by combining the relevant healthAPI reads into one record-level view.
        - returns: The aggregated PatientRecord (demographics + observations + risk assessments per FHIR) retrieved from the HIS.
    - `updatePatientRecord(patientId: PatientId, record: PatientRecord)`
        - effect: Writes the patient record back to the HIS by dispatching the appropriate healthAPI save operations.

#### P1INotificationInbox

- **provided by**: P1NotificationInboxModule (also at the parent boundary on P1NotificationDispatcher)
- **required by**: P1PhysicianGateway
- **operations**:
    - `subscribeToNotifications(physicianId: PhysicianId)`
        - effect: Adds the physician to the recipient registry so future patient-status events fan out to them.
    - `unsubscribeFromNotifications(physicianId: PhysicianId)`
        - effect: Removes the physician from the recipient registry; notifications already queued are still delivered.
    - `getPendingNotifications(physicianId: PhysicianId): List<Notification>`
        - effect: Returns the physician's queued notifications in priority order (red > yellow > green) without removing them from the buffer.
        - returns: The list of pending Notifications for the physician, ordered by severity then arrival.
    - `acknowledgeNotification(notificationId: NotificationId)`
        - effect: Confirms delivery of the notification so the dispatcher can remove it from the priority buffer and the durable log.

#### P1INotificationLog

- **provided by**: P1DurableNotificationLog
- **required by**: P1NotificationInboxModule, P1RiskEventSubscriber
- **operations**:
    - `persist(notification: Notification)`
        - effect: Writes the notification to the durable log before it enters the in-memory buffer so a crash cannot lose a red or yellow notification.
    - `markDelivered(notificationId: NotificationId)`
        - effect: Removes the notification from the durable log once the physician has acked delivery, bounding storage growth.
    - `recoverPending(): List<Notification>`
        - effect: On startup, returns notifications that were persisted but never acked so P1NotificationInboxModule can replay them into the priority buffer.
        - returns: The list of unacked Notifications recovered from the durable log.

#### P1IOnDemandCommand

- **provided by**: P1OnDemandConsultationHandler
- **required by**: P1CommandRouter
- **operations**:
    - `requestOnDemandConsultation(patientId: PatientId): ConsultationId`
        - effect: Internal delegate — P1CommandRouter forwards the UC9 request to P1OnDemandConsultationHandler, which mints a CorrelationId, fetches sensor data, launches a priority risk job, and awaits the matching event.
        - returns: The ConsultationId minted for this UC9 request.
    - `pollConsultationResult(consultationId: ConsultationId): PatientStatus`
        - effect: Internal delegate — P1CommandRouter forwards the poll to P1OnDemandConsultationHandler, which looks up the stashed result for this consultationId. The result becomes available once the matching P1IRiskEvents event (CorrelationId-filtered) has arrived; callers re-poll until non-null. The handler clears the entry after the result has been returned once.
        - returns: The PatientStatus produced by the UC9 risk estimation, or null if the matching event has not yet arrived.

#### P1IOnDemandSensorFetch

- **provided by**: P1PatientGatewayCommander
- **required by**: P1OnDemandConsultationHandler, P1PhysicianCommandService
- **operations**:
    - `requestCurrentSensorData(patientId: PatientId, correlationId: CorrelationId): SensorDataPackage`
        - effect: Synchronously asks the patient gateway to push the current sensor reading; bounded by the commander timeout sized inside the 3-min UC9 initiation budget.
        - returns: The fresh SensorDataPackage supplied by the patient gateway for this UC9 request.

#### P1IPatientQuery

- **provided by**: P1PatientQueryService
- **required by**: P1PhysicianGateway
- **operations**:
    - `consultPatientStatus(patientId: PatientId): PatientStatus`
        - effect: Reads the patient's current status. The request is dispatched to the priority tier (high > medium > low) matching the patient's last-known risk level from the in-memory tier index inside P1PatientQueryService — so the tier decision is O(1) on the hot path, not a chicken-and-egg lookup against the same status read. Unknown patients fall back to the high tier so the SLA fails safe.
        - returns: The patient's most recent status.

#### P1IPatientStatusRead

- **provided by**: P1PatientStatusCache
- **required by**: P1PatientQueryService
- **operations**:
    - `getPatientStatus(patientId: PatientId): PatientStatus`
        - effect: Cache-first read of the patient's status; on miss the cache fetches via OtherDataMgmt and populates itself for subsequent reads.
        - returns: The patient's most recent status.

#### P1IPhysicianAPI

- **provided by**: P1PhysicianGateway
- **required by**: (external — physicians calling into the PMS)
- **operations**:
    - `consultPatientStatus(patientId: PatientId): PatientStatus`
        - effect: Returns the named patient's current status; SLA tier (2/5/10 s) is picked from the patient's last-known risk level via P1PatientQueryService's in-memory risk-tier index.
        - returns: The patient's most recent status — risk level, supporting data, and timestamp.
    - `requestOnDemandConsultation(patientId: PatientId): ConsultationId`
        - effect: Initiates an on-demand consultation — fetches fresh sensor data, launches a priority risk estimation, and returns an id the caller uses to correlate the eventual result.
        - returns: The ConsultationId minted for this request.
    - `pollConsultationResult(consultationId: ConsultationId): PatientStatus`
        - effect: Retrieves the result of the UC9 risk estimation identified by consultationId; null until the priority risk job has completed and its event has been observed.
        - returns: The PatientStatus from the matching UC9 risk estimation, or null if pending.
    - `configurePatientRiskAssessment(patientId: PatientId, config: ClinicalModelConfiguration)`
        - effect: Overwrites the per-patient ClinicalModelConfiguration and invalidates the clinical-model cache.
    - `updatePatientRiskLevel(patientId: PatientId, status: PatientStatus)`
        - effect: Updates the patient's risk level and forwards the change to the EHR via the HIS proxy.
    - `subscribeToNotifications(physicianId: PhysicianId)`
        - effect: Registers a physician to receive notifications about subsequent patient-status changes.
    - `unsubscribeFromNotifications(physicianId: PhysicianId)`
        - effect: Removes the physician from the recipient registry; notifications already queued for them are still delivered.
    - `getPendingNotifications(physicianId: PhysicianId): List<Notification>`
        - effect: Returns the physician's queued notifications in priority order (red > yellow > green) without removing them from the buffer.
        - returns: The list of pending Notifications for the physician, ordered by severity then arrival.
    - `acknowledgeNotification(notificationId: NotificationId)`
        - effect: Confirms delivery of the notification so the dispatcher can remove it from the priority buffer and the durable log.

#### P1IPhysicianCommand

- **provided by**: P1CommandRouter (also at the parent boundary on P1PhysicianCommandService)
- **required by**: P1PhysicianGateway
- **operations**:
    - `configurePatientRiskAssessment(patientId: PatientId, config: ClinicalModelConfiguration)`
        - effect: Overwrites the per-patient ClinicalModelConfiguration and invalidates the clinical-model cache.
    - `updatePatientRiskLevel(patientId: PatientId, status: PatientStatus)`
        - effect: Updates the patient's risk level and forwards the change to the EHR via the HIS proxy.
    - `requestOnDemandConsultation(patientId: PatientId): ConsultationId`
        - effect: Initiates an on-demand consultation — fetches fresh sensor data, launches a priority risk estimation, and returns an id correlating the eventual result.
        - returns: The ConsultationId minted for this UC9 request.
    - `pollConsultationResult(consultationId: ConsultationId): PatientStatus`
        - effect: Returns the result of the UC9 risk estimation identified by consultationId; null until the matching P1IRiskEvents event has been received and stashed by P1OnDemandConsultationHandler.
        - returns: The PatientStatus from the matching UC9 risk estimation, or null if pending.

#### P1IPriorityBuffer

- **provided by**: P1PriorityBuffer
- **required by**: P1NotificationInboxModule, P1RiskEventSubscriber
- **operations**:
    - `enqueue(physicianId: PhysicianId, notification: Notification)`
        - effect: Adds the notification to the physician's pending set; severity drives priority ordering and the drop-green-first policy when the 100/min threshold is exceeded.
    - `peekPending(physicianId: PhysicianId): List<Notification>`
        - effect: Returns the physician's pending notifications in priority order (red > yellow > green) without removing them from the buffer.
        - returns: The list of pending Notifications for the physician, ordered by severity then arrival.
    - `removePending(notificationId: NotificationId)`
        - effect: Removes the notification from the buffer after the physician acks it through P1INotificationInbox.

#### P1IRecipientRegistry

- **provided by**: P1RecipientRegistry
- **required by**: P1NotificationInboxModule, P1RiskEventSubscriber
- **operations**:
    - `subscribe(physicianId: PhysicianId)`
        - effect: Adds the physician to the notification recipient set.
    - `unsubscribe(physicianId: PhysicianId)`
        - effect: Removes the physician from the notification recipient set.
    - `findRecipients(patientId: PatientId): List<PhysicianId>`
        - effect: Looks up the physicians who should be notified about events for the patient; backs UC5 recipient resolution.
        - returns: The list of PhysicianIds subscribed to events for this patient.

#### P1IRiskEvents

- **provided by**: P1PatientStatusModule (also at the parent boundary on P1PatientDataService)
- **required by**: P1NotificationDispatcher, P1OnDemandConsultationHandler, P1PatientQueryService, P1PhysicianCommandService, P1RiskEventSubscriber
- **operations**:
    - `subscribe(subscriberId: SubscriberId, filter: FilterCriteria): SubscriptionId`
        - effect: Registers a subscriber to receive RiskEvent payloads matching the filter; an empty filter receives every event.
        - returns: A SubscriptionId the caller uses to unsubscribe later.
    - `unsubscribe(subscriptionId: SubscriptionId)`
        - effect: Removes the named subscription so no further RiskEvents are pushed to it.

#### P1IRiskLevelCommand

- **provided by**: P1RiskLevelHandler
- **required by**: P1CommandRouter
- **operations**:
    - `updatePatientRiskLevel(patientId: PatientId, status: PatientStatus)`
        - effect: Internal delegate — P1CommandRouter forwards the UC8 request to P1RiskLevelHandler, which writes the new risk level via PatientRecordMgmt.

#### PatientRecordMgmt

- **provided by**: P1EHRProxyModule (was OtherFunctionality)
- **required by**: P1PhysicianCommandService, P1RiskLevelHandler, RiskEstimationCombiner
- **operations**:
    - `getPatientRecord(patientId: PatientId): PatientRecord`
        - effect: Fetches the EHR record of the patient with given id from the HIS and returns it. If the HIS is not available, an older (cached) copy of the patient record is returned if possible.
        - returns: The PatientRecord fronted by the EHR cache.
    - `updatePatientRecord(patientId: PatientId, record: PatientRecord)`
        - effect: Writes the patient record back to HIS via P1HISAdapter and invalidates the EHR cache entry so subsequent reads see the new value.

#### SensorDataMgmt

- **provided by**: P1SensorIngestModule (was OtherFunctionality)
- **required by**: Av2DataIngestionService, ClinicalJobCreator, P1OnDemandConsultationHandler, P1PhysicianCommandService, RiskEstimationScheduler
- **operations**:
    - `addSensorData(patientId: PatientId, sensorData: SensorDataPackage, receivedAt: Timestamp, triggerEstimation: boolean)`
        - effect: Store the given sensor data and meta-data. When `triggerEstimation` is true (UC4 default) the ingest module launches a risk-estimation job on the new data; when false the call is store-only — UC9 callers pass `false` because they explicitly launch a priority risk job and would otherwise double the scheduler load for every on-demand consultation.
    - `getAllSensorDataOfPatient(patientId: PatientId): Map<Timestamp, SensorDataPackage>`
        - effect: Fetch and return all sensor data belonging to the patient identified by patientId.
        - returns: The sensor data and the timestamp of their arrival in the system.
    - `getAllSensorDataOfPatientBefore(patientId: PatientId, stopTime: Timestamp): Map<Timestamp, SensorDataPackage>`
        - effect: Fetch and return all sensor data belonging to the patient identified by patientId which was received before the specified stopTime.
        - returns: The sensor data and the timestamp of their arrival in the system.
