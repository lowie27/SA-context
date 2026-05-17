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

- **Source:**: external or internal
- **Stimulus:**:
    - the (external) communication channel between a patient gateway and the pms becomes unavailable, e.g., due to failure or (scheduled) maintenance;
    - a patient gateway becomes unavailable, e.g., its battery may have run empty or it has no network connection; or
    - an internal communication subsystem of the pms fails or crashes.
- **Artifact:**: external communication channel(s), external device(s), internal subsystem(s)
- **Environment:**: Normal executions
- **Response:**:
    - 1. Prevention:
        - pms has negotiated a Service-Level Agreement (sla) with the intermediate telecommunication operator that stipulates
            - an availability of at least 99.5% (measured per year) of the communication channel is guaranteed;
            - at least 80% mobile network coverage in the broad region in which the pms will operate; and
            - an average bandwidth of at least 64Kb/s.
        - The app on the patient’s gateway warns the patient in time when the battery of their patient gateway unit is running low.
        – The app on the patient’s gateway warns the patient when they have no or limited network connectivity.
    - 2. Detection:
        - The pms back-end services should be able to autonomously detect any failures based on the lack of sensor data updates from one or more patient gateways.
        – The pms back-end services should be able to autonomously detect any failures of the relevant internal communication subsystems.
        – The patient gateway app should be able autonomously detect any communication failures and goes into degraded mode, which involves temporarily storing sensor data and notifications for later synchronization, systematic retrying to complete the communications, and use of possible back-up communication channels.
        – The pms keeps track of how long there was a lack of communication.
    - 3. Resolution:
        – The pms proactively notifies the relevant stakeholders of the detected problem.
            - In case of a failing communication subsystem, the system administrator must redeploy the failing component(s), or revert it to a previously working state.
            - In case of a failing communication channel, the system administrator must contact the telecommunication operator to resolve this.
            - In case of a failing patient gateway, the patient should be contacted to resolve the issues.
        – In degraded mode, the patient gateway application
            - uses a back-up communication channel for emergency notifications, e.g., sms; and
            - keeps retrying to deliver sensor data readings in a proper fashion, e.g., using exponential back-off; and
            - the patient gateway is able to store the sensor data readings (and unsent notifications) for the past 24 hours.
- **Response measure:**:
    - 1. Prevention:
        – The app warns the patient when the battery of their patient gateway has dropped to 15% or less charge.
    - 2. Detection:
        – Detection time depends on the transmission rate configured on the patient gateway (and indirectly on the risk level), but does not exceed this with more than 5 minutes. So, if an expected update does not arrive at expected time T , detection must have taken place before T 5 minutes.
        – The pms detects any failures of internal communication subsystems within 5 seconds after the failure.
    - 3. Resolution:
        – Once detected, in 90% of the cases, the notifications sent to the system administrator arrive within 2 minutes if the patient has a high risk level, within 5 minutes if the patient has a medium risk level, and within 10 minutes in case of a low risk level.
        – Redeployment or roll-back of the communication subsystem does not take longer than 5 minutes.
        – The sla with the telecommunication operator stipulates availability of technical support within 10 minutes and resolution within the hour.

---

## 1. Approach in plain words

> Two or three paragraphs summarising the teammate's approach in my own words. If I
> can't summarise it concisely, that's a finding (motivation depth — per Part 1 feedback).

TODO — core idea.

TODO — main structural change vs. initial architecture (which placeholder(s) replaced, which nodes added, which existing components touched).

TODO — key mechanism(s) — replication? heartbeats? leader-election? hot-standby? circuit breaker? — and why they fit the QAS.

okay because i cant tell you the answer because i did not make it try to reason about the structure based of the components provided

---

## 2. What changes vs. the initial architecture

### 2a. Components

**VP entry rules — components get descriptions, interfaces get operations.** (Same as P1 §2a.)
- Components need a **description** only: one short, concise sentence (≤ ~20 words). Source = the `description:` line below.
- Interfaces (§2b) carry the operations.

new components (fill in from teammate's design)

- `SensorDataRepository`
    - description: Persistent store for incoming sensor data packages and their arrival timestamps. Provides the SensorDataMgmt interface for appending new readings and for querying historical sensor data needed by clinical models that require historical context.
    - node: TODO
    - provides:
        - `SensorDataMgmt`
    - requires:
        - none

- `DataIngestionService`
    - description: Consumes validated sensor data and emergency notifications from the MessageBroker and persists them through the SensorDataRepository. Acts as the single write path for incoming patient data into the backend persistence layer, decoupling the ingestion rate from the downstream risk-estimation pipeline.
    - node: TODO
    - provides:
        - none
    - requires:
        - `SensorDataMgmt`, `QueuedSensorDataMgmt`

- `EmergencyBackupReceiver`
    - description: Receives emergency notifications transmitted via the backup SMS gateway when the primary channel is unavailable. Authenticates the sending patient gateway by a pre-registered device token embedded in the SMS payload, and forwards validated emergencies to the MessageBroker for the same downstream processing as primary-channel emergencies.
    - node: TODO
    - provides:
        - `BackupEmergencyIngress`
    - requires:
        - `QueuedSensorDataMgmt`

- `MessageBroker`
    - description: Buffers validated sensor-data packages and emergency notifications between the CommunicationGateway and the DataIngestionService. Provides durability and decoupling so that transient failures of downstream services do not lead to data loss, and exposes a health-check endpoint for the InternalFailureDetector.
    - node: TODO
    - provides:
        - `InternalHealthCheck`, `BrokerCommitNotification`
    - requires:
        - `BufferedDataDispatch`


- `AcknowledgementService`
    - description: After RoutingManager confirms downstream delivery (e.g., MessageBroker accepts the message), DeliveryAcknowledger emits an acknowledgement back to the originating PatientGateway, allowing the gateway's RetryAndSynchronizationService to clear delivered entries from LocalBufferingRepository.
    - node: TODO
    - provides:
        - `DeliveryAck`
    - requires:
        - `BrokerCommitNotification`

- `CommunicationGateway`
    - description: The single backend ingress point for all primary-channel patient gateway traffic. Authenticates incoming transmissions, routes them to downstream services (MessageBroker, EmergencyNotificationService), detects communication failures, coordinates retries with exponential backoff, tracks per-gateway heartbeats for missing-data detection, and acknowledges successfully dispatched messages back to the originating gateway.
    - node: TODO
    - provides:
        - `BufferedDataDispatch`, `AvailabilityMonitoring`
    - requires:
        - `GatewayDataIngress`

- `MonitoringService`
    - description: Central observability and recovery-orchestration component. Detects (a) missing sensor data, (b) internal subsystem crashes, (c) SLA violations of the telecom channel; classifies failures by affected patient risk level; notifies administrators and patient contacts within Av2 deadlines; records communication-gap durations; and coordinates recovery with the RecoverySyncService and RedeploymentCoordinator.
    - node: TODO
    - provides:
        - `RecoveryCoordination`, `NotificationDispatch`
    - requires:
        - `AvailabilityMonitoring`

- `PatientGateway`
    - description: The patient-facing edge component running on the wearable/handheld device. Acquires raw sensor readings, transmits them to the backend CommunicationGateway over the primary telecom channel, monitors device battery and network health, manages local buffering during connectivity loss, and dispatches emergency notifications via the backup channel when the primary channel is unavailable. It is the only component that physically resides with the patient.
    - node: TODO
    - provides:
        - `EmergencyDispatch`, `GatewayDataIngress`
    - requires:
        - `DeliveryAck`, `BackupEmergencyIngress`, `GatewayResyncCommand`

- `RecoverySyncService`
    - description: Coordinates post-failure recovery. When RecoveryTracker signals a failure has been resolved, RecoverySyncService (a) drains any sensor data queued in the MessageBroker that arrived during degraded mode, and (b) instructs the affected PatientGateway(s) to flush their LocalBufferingRepository through a GatewayResyncCommand.
    - node: TODO
    - provides:
        - `GatewayResyncCommand`
    - requires:
        - `RecoveryCoordination`, `QueuedSensorDataMgmt`

- `RedeploymentCoordinator`
    - description: TODO
    - node: TODO
    - provides:
        - `RedeploymentControl`
    - requires:
        - `RecoveryCoordination`


- `AdminPortal`
    - description: Provides the system administrator with a web-based interface for monitoring the operational status of the PMS, receiving failure notifications, and triggering redeployment or rollback of failed internal subsystems. It consumes the RedeploymentControl interface to initiate and monitor redeployment jobs, and receives notifications from the MonitoringService via the NotificationDispatch interface. It is the only component through which a system administrator can interact with the RedeploymentCoordinator.
    - node: TODO
    - provides:
        - none
    - requires:
        - `RedeploymentControl`, `NotificationDispatch`

- `TODO`
    - description: TODO
    - node: TODO
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

okay actually it is this convention so stick to this convention please:

- `TODO_InterfaceName`
    - provided by: `TODO_Component`
    - required by:
        - `TODO_Component`
    - operations (all `+`):
        - `TODO_operationName(paramname: ParamType, ...): ReturnType`
    - purpose: TODO

all these need to be changed to comply:

- `QueuedSensorDataMgmt`
    - provided by: `MessageBroker`
    - required by:
        - `EmergencyBackupReceiver`, `RecoverySyncService`, `DataIngestionService`
    - operations (all `+`) NO OPERATIONS YET:
        - `TODO_operationName(ParamType) → ReturnType` — TODO purpose

- `BrokerCommitNotification`
    - provided by: `MessageBroker`
    - required by:
        - `AcknowledgementService`
    - operations (all `+`) NO OPERATIONS YET:
        - `TODO_operationName(ParamType) → ReturnType` — TODO purpose

- `BufferedDataDispatch`
    - provided by: `TODO_Component`
    - required by:
        - `TODO_Component`
    - operations (all `+`) NO RETURN TYPES??:
        - `dispatchSensorData(Datatypes.GatewayId, Datatypes.SensorDataPackage, Datatypes.Timestamp)` — TODO purpose
        - `dispatchEmergencyNotification(Datatypes.GatewayId, Datatypes.EmergencyMessage, Datatypes.Timestamp)` — TODO purpose

- `InternalHealthCheck` also  mistake 2 provided interface not possible right?
    - provided by: `MessageBroker`, `CommunicationGateway`
    - required by:
        - `TODO_Component`
    - operations (all `+`) IS Datatypes.HealthStatus MADE FOR THIS OR IS IT WRONG PURPOSE:
        - `ping() → Datatypes.HealthStatus` — TODO purpose
        - `getSubSystem() → Map<String, Datatypes.HealthStatus>` — TODO purpose

- `DeliveryAck`
    - provided by: `AcknowledgementService`
    - required by:
        - `PatientGateway`
    - operations (all `+`) NO OPERATIONS YET:
        - `TODO_operationName(ParamType) → ReturnType` — TODO purpose

- `BackupEmergencyIngress`
    - provided by: `EmergencyBackupReceiver`
    - required by:
        - `PatientGateway`
    - operations (all `+`) NO OPERATIONS YET:
        - `TODO_operationName(ParamType) → ReturnType` — TODO purpose

- `EmergencyDispatch`
    - provided by: `PatientGateway`
    - required by:
        - none
    - operations (all `+`):
        - `sendEmergencyNotification(Datatypes.EmergencyMessage)` — TODO purpose

- `GatewayDataIngress`
    - provided by: `TODO_Component`
    - required by:
        - `TODO_Component`
    - operations (all `+`) TODO TransmissionAck not a type yet create this:
        - `transmitSensorData(Datatypes.GatewayId, Datatypes.SensorDataPackage, Datatypes.Timestamp) → TransmissionAck` — TODO purpose
        - `transmitEmergencyNotification(Datatypes.GatewayId, Datatypes.EmergencyMessage, Datatypes.Timestamp) → TransmissionAck` — TODO purpose

- `AvailabilityMonitoring`
    - provided by: `CommunicationGateway`
    - required by:
        - `MonitoringService`
    - operations (all `+`):
        - `registerGateway(Datatypes.GatewayId, int)` — TODO purpose
        - `recordHeartBeat(Datatypes.GatewayId, Datatypes.Timestamp)` — TODO purpose
        - `getLastHeartbeat(Datatypes.GatewayId)` — TODO purpose
        - `getExpectedInterval(Datatypes.GatewayId)` — TODO purpose

- `RecoveryCoordination`
    - provided by: `MonitoringService`
    - required by:
        - `RecoverySyncService`
        - `RedeploymentCoordinator`
    - operations (all `+`) NO OPERATIONS YET:
        - `TODO_operationName(ParamType) → ReturnType` — TODO purpose

- `GatewayResyncCommand`
    - provided by: `RecoverySyncService`
    - required by:
        - `PatientGateway`
    - operations (all `+`) NO OPERATIONS YET:
        - `TODO_operationName(ParamType) → ReturnType` — TODO purpose

- `NotificationDispatch`
    - provided by: `MonitoringService`
    - required by:
        - `AdminPortal`
    - operations (all `+`):
        - `notifyAdministrator(failure: Datatypes.ClassifiedFailure, message: string)` — TODO purpose
        - `notifyPatientContact(Datatypes.PatientId, Datatypes.ClassifiedFailure, message: string)` — TODO purpose

- `RedeploymentControl`
    - provided by: `RedeploymentCoordinator`
    - required by:
        - `AdminPortal`
    - operations (all `+`):
        - `triggerRedeployment(subsystemId: string) → Datatypes.RedeploymentJob` — TODO purpose
        - `triggerRollback(subsystemId: string, targetVersion: string) → Datatypes.RedeploymentJob` — TODO purpose
        - `getRedeploymentStatus(id: Datatypes.RedeploymentJobId) → Datatypes.RedeploymentStatus` — TODO purpose


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
