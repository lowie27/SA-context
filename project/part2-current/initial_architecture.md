# Initial PMS Architecture — Reference Extract

Source: `SA_project_part2_inital_architecture_including_rationale.pdf` (39 pp, 692 KB).
This file is the lookup-free reference; if something below contradicts the PDF, the PDF wins.

The architecture is **incomplete on purpose**: a gray `OtherFunctionality` component and a `TODONode` placeholder mark the parts the QAS extensions (P1, Av2) must replace.

---

## 1. Context

- System: Patient Monitoring Service (PMS) — monitors cardiovascular patients via wearable sensors, runs risk estimation, notifies physicians.
- Two QASs already decided in the initial design: **P2 (Performance — risk-estimation throughput)** and **M2 (Modifiability — clinical model updates at runtime)**. Both are baked into the architecture below.
- The remaining QASs (P1, Av2, …) are addressed by the Part-2 extensions in `p1_current.md` (mine) and the teammate's Av2 work.

---

## 2. Pre-existing QAS decisions

### 2.1 P2 — Risk estimation by clinical models (Medium importance)

- **Stimulus**: new sensor data arriving faster than risk estimation can process it.
- **Response measure**: normal mode = FIFO; system enters **overload mode** when throughput > 20 risk estimations/min. In overload, no starvation: every job initiated within a hard 10 min deadline.
- **Overload-mode deadlines (Table 1.1)** — applied as earliest-deadline-first:

  | Risk level | Deadline (min) |
  | --- | --- |
  | high | 2 |
  | medium | 5 |
  | low | 8 |

- **AD1**: Parallel & concurrent risk estimation — multiple `RiskEstimationProcessor` instances across multiple nodes, multiple instances per node. Jobs are commands (no shared state).
- **AD2**: Centralized scheduling in `RiskEstimationScheduler` — monitors throughput, switches FIFO ↔ dynamic priority scheduling.
- **Sensitivity**: latency/throughput are highly sensitive to (i) per-job data-access cost and (ii) data locality. Remote lookups on the critical path would undermine P2.
- **Trade-off**: distribution complexity vs. predictable QoS — accepts a single scheduler chokepoint for deterministic prioritization.

### 2.2 M2 — New or optimized clinical models (High importance)

- **Stimulus**: a clinical model can be improved.
- **Response measure**: deploy ≤30 min, no downtime, no impact on running risk calculations.
- **AD1**: `ClinicalModelDB` is **append-only**; models fetched on demand at runtime.
- **AD2**: `ClinicalModelCache` with **explicit invalidation** (not TTL). Running jobs see a stable snapshot; new jobs see the new version.
- **Sensitivity**: correctness depends on a running job observing a *stable* model version end-to-end.
- **Trade-off**: prefers run-time modifiability + availability over instantaneous global consistency.

---

## 3. Four views (chapters A–D in the PDF)

| View | Chapter | Figures | What it shows |
| --- | --- | --- | --- |
| Client-Server (C&C) | A | A.1 context, A.2 primary | All runtime components inside `eHealth Platform`, with ball-and-socket interfaces. The unfinished part is the gray `OtherFunctionality` component. |
| Decomposition | B | B.1 context, B.2 `MLModelManager`, B.3 `RiskEstimationProcessor` | Internal modules of the two decomposed components. Other components are leafs in this view. |
| Deployment | C | C.1 context, C.2 primary, C.3 Pilot, C.4 Dev | Nodes and component placement. Pilot = 200 patients / 5 models; Dev = 20 patients / 3 models. |
| Process | D | D.1–D.6 (six sequences) | Risk-estimation flow end-to-end + ML model sync/update flows. |

### 3.1 Process View — sequence diagrams (one-liners)

- **D.1 Risk estimation process**: `OtherFunctionality → RiskEstimationScheduler.launchRiskEstimation` → `ClinicalJobCreator.createClinicalModelJobsForPatient` → `ClinicalModelCache` (read-through to `ClinicalModelDB` on miss) → optional `OtherFunctionality.getAllSensorDataOfPatientBefore` (if model needs historical data) → loop `RiskEstimationCombiner.addJobSet` per ClinicalModelJob.
- **D.2 Compute clinical model result**: inside `RiskEstimationProcessor` — `JobProcessor.getNextJob` → `JobMLModelSelector.determineMLModelForClinicalModel` → `MLModelRetrieval.fetchMLModel` → `SensorDataRiskAssessment.performModelPrediction` (optionally fetches/stores extra data via `KVStore`) → `Results.addClinicalModelJobResults`.
- **D.3 MLModel synchronization**: `MLModelMgmt.synchronizeModels` → `MLModelStorageManager → MLModelUpdateProcessor.fetchNewMLModels` → MLaaS via `MLaaSAPI.authenticate` + `MLaaSService.getAvailableModels` → for each: check `isAvailable`, otherwise `retrieveModel` and store.
- **D.4 Processing MLModel updates**: `MLModelMgmt.updateModel` → if available → `MLModelUpdateProcessor.processMLModelUpdates` → fetches updates from MLaaS → loop `ModelUpdater.applyUpdate` + `verify` → `storeModel`.
- **D.5 Incoming sensor data (incomplete)**: `OtherFunctionality.launchRiskEstimation` → [ref Risk estimation process] → `RiskEstimationCombiner.setEstimatedPatientStatus` back to `OtherFunctionality`. **Critical**: this is the `setEstimatedPatientStatus` callback path my P1 relies on.
- **D.6 Combining ClinicalModelJob results**: `RiskEstimationProcessor.addClinicalModelJobResults` → `RiskEstimationCombiner.combineResults` (opt: all JobSet results received) → `OtherFunctionality.setEstimatedPatientStatus`.

---

## 4. Components (§E.1) — 12 entries

| # | Component | Provides | Requires | Deployed (Pilot) | Deployed (Dev) | Role |
| --- | --- | --- | --- | --- | --- | --- |
| E.1.1 | `ClinicalJobCreator` | `ClinicalJobMgmt` | `FetchClinicalModels`, `OtherDataMgmt`, `SensorDataMgmt` | RiskEstimationManagementNode | RiskEstimationManagementNode | Constructs `ClinicalModelJob` objects, populates them with patient health data for `RiskEstimationProcessor`. |
| E.1.2 | `ClinicalModelCache` | `ClinicalModelCacheMgmt`, `FetchClinicalModels` | `ClinicalModelStorage`, `OtherDataMgmt` | RiskEstimationManagementNode | RiskEstimationManagementNode | Read-through cache for ClinicalModels+configs; **explicit invalidation only**, items don't expire. Located close to RiskEstimationProcessor / Combiner. |
| E.1.3 | `ClinicalModelDB` | `ClinicalModelStorage` | — | **TODO Node** | RiskEstimationManagementNode | Stores ClinicalModels **and per-patient configurations**. **Append-only for models** (per M2 AD1). |
| E.1.4 | `eHealth Platform` (parent) | — | `KVStore`, `MLaaSService` | spans many | spans many | Container component for everything inside the PMS deployment boundary. |
| E.1.5 | `HIS` (external) | `healthAPI` | — | — (Nowhere) | — | Hospital Information System — external EHR provider. |
| E.1.6 | `KVStorageService` | `KVStore` | — | RiskEstimationProcessorNode | RiskEstimationProcessorNode | Generic key-value storage. |
| E.1.7 | `MLaaS` (external) | `MLaaSService` | — | MLaaSCloud / MLaaS Service nodes | MLaaSCloud / MLaaS Service | External MLaaS provider. |
| E.1.8 | `MLModelManager` (decomposed) | `MLModelMgmt` | `MLaaSService` | MLaaS Conn / MLaaSConnectorNode | RiskEstimationManagementNode | Manages local MLModels, syncs with MLaaS. |
| E.1.9 | `OtherFunctionality` ⚠️ gray | `OtherDataMgmt`, `PatientRecordMgmt`, `SensorDataMgmt` | `ClinicalModelCacheMgmt`, `LaunchRiskEstimation` | **TODO Node** | RiskEstimationManagementNode | **Placeholder** — every unmodeled responsibility lives here. Replaced by P1's `PatientDataService` + physician-side services. |
| E.1.10 | `RiskEstimationCombiner` | `JobMgmt`, `Results` | `OtherDataMgmt`, `PatientRecordMgmt` | RiskEstimationMgmtNode | RiskEstimationManagementNode | Combines ClinicalModelJob results into final risk estimation; calls `setEstimatedPatientStatus` (via OtherDataMgmt). |
| E.1.11 | `RiskEstimationProcessor` (decomposed) | — | `FetchJobs`, `KVStore`, `MLModelMgmt`, `Results` | RiskEstimationProcessorNode (×5 in Pilot, ×2 in Dev) | RiskEstimationProcessorNode | Computes individual ClinicalModelJobs in parallel; multiple instances per node. |
| E.1.12 | `RiskEstimationScheduler` | `FetchJobs`, `LaunchRiskEstimation` | `ClinicalJobMgmt`, `FetchClinicalModels`, `JobMgmt`, `OtherDataMgmt`, `SensorDataMgmt` | RiskEstimationMgmtNode | RiskEstimationManagementNode | Queues new risk-estimation jobs. **Switches FIFO → EDF priority on overload (>20/min)** per P2 AD2. |

⚠️ `OtherFunctionality` and `TODONode` are the two placeholders that Part 2 extensions must replace.

---

## 5. Modules (§E.2) — 10 entries

Modules use the `<<module>>` stereotype and live inside a parent component.

| # | Module | Parent | Provides | Requires |
| --- | --- | --- | --- | --- |
| E.2.1 | `JobMLModelSelector` | RiskEstimationProcessor | `ApplyClinicalModel` | — |
| E.2.2 | `JobProcessor` | RiskEstimationProcessor | — | `ApplyClinicalModel`, `FetchJobs`, `FetchMLModel`, `Results`, `SensorDataAnalysis` |
| E.2.3 | `MLaaSLibrary` | (eHealth Platform leaf) | `MLaaSAPI` | — |
| E.2.4 | `MLLibrary` | (eHealth Platform leaf) | `MLAPI` | — |
| E.2.5 | `MLModelRepository` | MLModelManager | `MLModelStorage` | — |
| E.2.6 | `MLModelRetrieval` | RiskEstimationProcessor | `FetchMLModel` | `MLModelMgmt` |
| E.2.7 | `MLModelStorageManager` | MLModelManager | `MLModelMgmt` | `MLModelStorage`, `MLModelSync` |
| E.2.8 | `MLModelUpdateProcessor` | MLModelManager | `MLModelSync` | `MLaaSAPI`, `MLaaSService`, `MLModelStorage`, `MLModelUpdateMgmt` |
| E.2.9 | `ModelUpdater` | MLModelManager | `MLModelUpdateMgmt` | `MLAPI` |
| E.2.10 | `SensorDataRiskAssessment` | RiskEstimationProcessor | `SensorDataAnalysis` | `KVStore`, `MLAPI` |

Decomposed components: `MLModelManager` (B.2) and `RiskEstimationProcessor` (B.3). Everything else is a leaf in the Decomposition View.

---

## 6. Interfaces (§E.3) — 23 entries with key operations

| # | Interface | Provided by | Required by | Key operations |
| --- | --- | --- | --- | --- |
| E.3.1 | `ApplyClinicalModel` | JobMLModelSelector | JobProcessor | `determineMLModelForClinicalModel(ClinicalModel, ClinicalModelConfiguration) → String` |
| E.3.2 | `ClinicalJobMgmt` | ClinicalJobCreator | RiskEstimationScheduler | `createClinicalModelJobsForPatient(PatientId, Tuple<SensorDataPackage,Timestamp>) → List<Tuple<ClinicalModelJob, Double>>` |
| E.3.3 | `ClinicalModelCacheMgmt` | ClinicalModelCache | **OtherFunctionality** ⚠️ | `invalidateCacheEntries(PatientId)` — removes all cached entries for a patient |
| E.3.4 | `ClinicalModelStorage` | ClinicalModelDB | ClinicalModelCache | `getAssignedClinicalModelsAndConfigForPatient(PatientId) → Map<ClinicalModel, Tuple<ClinicalModelConfiguration,Double>>` — **READ ONLY** |
| E.3.5 | `FetchClinicalModels` | ClinicalModelCache | ClinicalJobCreator, RiskEstimationScheduler | Same signature as E.3.4 but cache-aware |
| E.3.6 | `FetchJobs` | RiskEstimationScheduler | JobProcessor, RiskEstimationProcessor | `getNextJob() → ClinicalModelJob` |
| E.3.7 | `FetchMLModel` | MLModelRetrieval | JobProcessor | `fetchModel(String modelId) → MLModel` |
| E.3.8 | `healthAPI` | HIS (external) | — (legacy: nothing inside the PMS) | FHIR-derived: getObservation, getPatient, getRiskAssessment, saveObservation, savePatient, saveRiskAssessment, deleteObservation, deletePatient, deleteRiskAssessment, searchObservation, searchPatient, searchRiskAssessment |
| E.3.9 | `JobMgmt` | RiskEstimationCombiner | RiskEstimationScheduler | `addJobSet(RiskEstimationId, List<Tuple<ClinicalModelJobId,Double>>)` |
| E.3.10 | `KVStore` | KVStorageService | RiskEstimationProcessor, SensorDataRiskAssessment, eHealth Platform | `retrieve(String) → Object`, `store(String, Object)` |
| E.3.11 | `LaunchRiskEstimation` | RiskEstimationScheduler | **OtherFunctionality** ⚠️ | `launchRiskEstimation(PatientId, SensorDataPackage, Timestamp)` — fetches clinical models via ClinicalModelCache, notifies RiskEstimationCombiner, schedules jobs (FIFO normal / EDF overload). |
| E.3.12 | `MLaaSAPI` | MLaaSLibrary | MLModelUpdateProcessor | `authenticate() → Credential`, `delete`, `getModelDescription`, `getModelId`, `register`, `setAPIKey` |
| E.3.13 | `MLaaSService` | MLaaS (external) | MLModelManager, MLModelUpdateProcessor, eHealth Platform | `fetchModelUpdates`, `getAvailableModels`, `pushModelUpdates`, `retrieveModel` |
| E.3.14 | `MLAPI` | MLLibrary | ModelUpdater, SensorDataRiskAssessment | `applyUpdate`, `predict(SensorDataPackage) → Prediction`, `verify` |
| E.3.15 | `MLModelMgmt` | MLModelManager, MLModelStorageManager | MLModelRetrieval, RiskEstimationProcessor | `fetchModel`, `getAvailableModels`, `synchronizeModels`, `updateModel` |
| E.3.16 | `MLModelStorage` | MLModelRepository | MLModelStorageManager, MLModelUpdateProcessor | `isAvailable`, `retrieveModel`, `storeModel` |
| E.3.17 | `MLModelSync` | MLModelUpdateProcessor | MLModelStorageManager | `fetchNewMLModels`, `processMLModelUpdates` |
| E.3.18 | `MLModelUpdateMgmt` | ModelUpdater | MLModelUpdateProcessor | `processMLModelUpdates(MLModel, List<MLModelUpdate>) → MLModel` |
| E.3.19 | `OtherDataMgmt` | **OtherFunctionality** ⚠️ | ClinicalJobCreator, ClinicalModelCache, RiskEstimationCombiner, RiskEstimationScheduler | `getPatientStatus(PatientId) → PatientStatus`, `setEstimatedPatientStatus(PatientId, PatientStatus, Timestamp)` — **the setEstimatedPatientStatus path notifies "appropriate parties"** (D.5, D.6) |
| E.3.20 | `PatientRecordMgmt` | **OtherFunctionality** ⚠️ | RiskEstimationCombiner | `getPatientRecord(PatientId) → PatientRecord` — **READ ONLY**; fronts HIS with a stale-tolerant cache fallback. |
| E.3.21 | `Results` | RiskEstimationCombiner | JobProcessor, RiskEstimationProcessor | `addClinicalModelJobResults(ClinicalModelJobId, ClinicalModelJobResult)` |
| E.3.22 | `SensorDataAnalysis` | SensorDataRiskAssessment | JobProcessor | `performModelPrediction(ClinicalModelJob, MLModel)` |
| E.3.23 | `SensorDataMgmt` | **OtherFunctionality** ⚠️ | ClinicalJobCreator, RiskEstimationScheduler | `addSensorData(PatientId, SensorDataPackage, Timestamp)`, `getAllSensorDataOfPatient`, `getAllSensorDataOfPatientBefore` |

⚠️ = the four interfaces tied to the `OtherFunctionality` placeholder. Three are *provided* by OtherFunctionality (OtherDataMgmt, PatientRecordMgmt, SensorDataMgmt); one is *required* by it (ClinicalModelCacheMgmt). Plus `LaunchRiskEstimation` is *required* by OtherFunctionality. **Replacing OtherFunctionality means preserving all five edges.**

---

## 7. Nodes (§E.4)

| # | Node | Holds | Notes |
| --- | --- | --- | --- |
| E.4.1 | `eHealthPlatform` | (container) | Context-boundary node. |
| E.4.2 | `MLaaS Conn (Pilot)` | MLModelManager | Pilot-only. |
| E.4.3 | `MLaaS Service (Dev)` | MLaaS | Dev-only. |
| E.4.4 | `MLaaS Service (Pilot)` | MLaaS | Pilot-only. |
| E.4.5 | `MLaaSCloud` | MLaaS service infrastructure | Cloud envelope. |
| E.4.6 | `MLaaSConnectorNode` | MLModelManager | Handles MLaaS connections. |
| E.4.7 | `RiskEstimationManagementNode (Pilot)` | ClinicalJobCreator, ClinicalModelCache, RiskEstimationCombiner, RiskEstimationScheduler | Pilot: job creation/scheduling/combining. |
| E.4.8 | `RiskEstimationManagmentNode (Dev)` | OtherFunctionality, ClinicalJobCreator, ClinicalModelCache, RiskEstimationScheduler, RiskEstimationCombiner, ClinicalModelDB, MLModelManager | Dev: everything except the processors. |
| E.4.9 | `RiskEstimationMgmtNode` | (abstract) | Generic primary-diagram node for management plane. |
| E.4.10 | `RiskEstimationProcessorNode` | RiskEstimationProcessor (multiple) | Generic processor node (1..*). |
| E.4.11 | `RiskEstimationProcessorNode (Dev)` | 2× RiskEstimationProcessor, KVStorageService | Dev sizing: 2 processors. |
| E.4.12 | `RiskEstimationProcessorNode (Pilot)` | 5× RiskEstimationProcessor, KVStorageService | Pilot sizing: 5 processors. |
| E.4.13 | `TODO Node (Pilot)` | OtherFunctionality, ClinicalModelDB | **Placeholder** — Part 2 must replace. |
| E.4.14 | `TODONode` | — | Primary-diagram placeholder. |

---

## 8. Exceptions & data types (§E.5 / §E.6)

Exception hierarchy: `Exception` (base) → `AuthenticationException`, `AuthorizationException`, `InvalidAPIKeyException`, `InvalidMLModelException`, `IOException`, `NoSuchMLModelException`, `NoSuchPatientException`, `NoSuchPatientRecordException`. Plus a separate `SecurityException` → `AuthenticationException2`, `AuthorizationException2`.

Key data types (only the ones likely to matter for P1):
- `PatientId` — opaque identifier (long/string/URI).
- `PatientStatus` — patient's risk level.
- `PatientRecord` — EHR contents (medication, treatments, allergies).
- `SensorDataPackage` — list of `SensorDataElement`.
- `SensorDataElement` — sensorId, sensorType, measurementType/unit, measurementValue (BigDecimal), measurementValueString.
- `ClinicalModel`, `ClinicalModelConfiguration`, `ClinicalModelJob`, `ClinicalModelJobId`, `ClinicalModelJobResult` — the risk-estimation job model.
- `Timestamp`, `Date`, `Period`, `Range`, `Quantity` — primitives.
- FHIR-aligned: `Patient`, `Observation`, `RiskAssessment`, `CodeableConcept`, `Coding`, `Identifier`, `Address`, `ContactPoint`, `Annotation`.

---

## 9. Unresolved issues in the initial design (§E.7)

The SAPlugin already flags these on the initial model (will still appear until Part 2 fills them in):
- `Comp.06`: a component has no visualizations.
- `Diag.comp.06`: a component may be missing on the C/S context diagram.
- `Diag.depl.03`: a component or its sub-components have not been deployed.
- `If.03`: an interface is not required by any component or module.

---

## 10. Recap — what P1/Av2 must replace

1. **`OtherFunctionality`** — owns `SensorDataMgmt`, `OtherDataMgmt`, `PatientRecordMgmt` (all provided) and `LaunchRiskEstimation` + `ClinicalModelCacheMgmt` (required). Convention 4 means any replacement must keep all five interface edges live.
2. **`TODONode` / `TODO Node (Pilot)`** — needs to be split into real physical nodes that host the replacement components.
3. **Patient gateway interactions** are absent from the initial design — UC4 (push sensor data in) is implicit on `OtherFunctionality.SensorDataMgmt`; UC9 (pull current data from gateway) has no path at all. P1 introduces both directions explicitly.
4. **Physician interactions** — UC5/6/7/8/9 have no concrete components in the initial design. P1 introduces the entire `PhysicianAccessNode` cluster.

---

## 11. Critical quotes worth remembering

- *§E.1.3*: "ClinicalModelDB stores all the ClinicalModels and the configurations of these ClinicalModels for the different patients separately from other data. **It only allows appending new ClinicalModels.**"
- *§E.1.2*: "items in the ClinicalModelCache do not expire over time, but should be invalidated explicitly if needed."
- *§E.3.19 `setEstimatedPatientStatus` effect*: "Update the patient status estimation … **If patient's estimated risk changed the appropriate parties are notified.**" — this is the legacy contract my P1 reuses for `IRiskEvents`.
- *§E.3.20 `getPatientRecord` effect*: "fetch the EHR record from HIS … **If HIS is not available, an older (cached) copy of the patient record is returned if possible.**" — defines the proxy + stale-tolerant cache requirement.
- *Table 1.1 (P2 overload deadlines)*: high=2 min, medium=5 min, low=8 min. **My P1 UC9 priority must sit above these or carry its own correlation lane.**
- *P2 AD2 considered alternative — "Always-on priority scheduling"*: rejected because it adds overhead on the scheduler's critical path even in normal mode. Keep FIFO in normal mode; switch on overload.
- *M2 trade-off justification*: "running jobs operate on a snapshot of cached models, while updates only affect jobs started after invalidation." Append-only DB + explicit cache invalidation is the mechanism.

---

## 12. Quick index — figures

| Fig | View | Subject |
| --- | --- | --- |
| A.1 | Client-Server | Context — eHealth Platform with KVStorageService, RiskEstimationProcessor, MLModelManager, MLaaS as anchors. |
| A.2 | Client-Server | Primary — full graph with `OtherFunctionality` (gray) in the center. |
| B.1 | Decomposition | Context — three top-level decomposed components (MLModelUpdateProcessor, SensorDataRiskAssessment, ModelUpdater) and their interfaces to MLaaSAPI / MLAPI. |
| B.2 | Decomposition | `MLModelManager` decomposed into 4 modules. |
| B.3 | Decomposition | `RiskEstimationProcessor` decomposed into 4 modules. |
| C.1 | Deployment | Context — eHealthPlatform ↔ MLaaSCloud via MLaaSConnectorNode. |
| C.2 | Deployment | Primary — TODONode (with OtherFunctionality + ClinicalModelDB) wired to RiskEstimationMgmtNode (ClinicalJobCreator, ClinicalModelCache, RiskEstimationCombiner, RiskEstimationScheduler) and 1..* RiskEstimationProcessorNode (RiskEstimationProcessor + KVStorageService) and 1 MLaaSConnectorNode. |
| C.3 | Deployment (Pilot) | 200 patients, 5 clinical models — 5 RiskEstimationProcessor instances. |
| C.4 | Deployment (Dev) | 20 patients, 3 clinical models — 2 RiskEstimationProcessor instances; OtherFunctionality co-located with management plane. |
| D.1 | Process | Risk estimation process — OtherFunctionality → Scheduler → ClinicalJobCreator → Cache → DB → Combiner. |
| D.2 | Process | Compute clinical model result inside RiskEstimationProcessor. |
| D.3 | Process | MLModel synchronization with MLaaS. |
| D.4 | Process | MLModel update application. |
| D.5 | Process | Incoming sensor data (incomplete) — `setEstimatedPatientStatus` callback from Combiner back to OtherFunctionality. |
| D.6 | Process | Combining ClinicalModelJob results — callback again ends at `setEstimatedPatientStatus`. |
