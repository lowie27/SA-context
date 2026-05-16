# Appendix A — Functional Requirements Reference Extract

Source: `SA_project_appendixA_requirements.pdf`. Part 1: Appendix A — actors + the 19 textual use cases.
This file is the lookup-free reference; if something below contradicts the PDF, the PDF wins.

> **Related**: API operations behind UC16 / UC17 (HIS interaction) are defined in `appendixB_APIs.md`.

## Actors (Fig. 1)

- **User** (abstract) — parent of Patient, Physician (abstract: Cardiologist, GP), Trained nurse, Telemedicine operator.
- **Emergency call center**, **Hospital Information System (HIS)**, **ML model provider** — external systems.

## Use case index

| UC | Name | Primary actor | Includes |
|---|---|---|---|
| UC1 | log in | User | — |
| UC2 | log off | User | — |
| UC3 | send emergency notification | Patient (via gateway) | UC17 |
| UC4 | send sensor data | Patient (via gateway) | UC12 |
| UC5 | notify registered parties | PMS | — |
| UC6 | consult patient status | Physician or Patient | — |
| UC7 | configure patient risk assessment | Cardiologist | UC12 |
| UC8 | update risk level | Cardiologist | UC17 |
| UC9 | perform on-demand consultation | Cardiologist | UC12 |
| UC10 | register patient | Trained nurse | — |
| UC11 | deregister patient | Trained nurse | — |
| UC12 | estimate risk level | PMS | UC16, UC5, UC17 |
| UC13 | retrieve new ML models | Telemedicine operator | UC14 |
| UC14 | download ML model | PMS | — |
| UC15 | update ML model | Telemedicine operator | — |
| UC16 | consult patient record | PMS | — |
| UC17 | update patient record | PMS | — |
| UC18 | consult monitoring summary | Patient | — |
| UC19 | request access to personal data | User | UC1, UC5 |

---

## UC1: log in
- **Primary actor:** User
- **Pre:** User is registered and has credentials.
- **Post:** User is authenticated.
- **Main:** User indicates intent → PMS asks for credentials → User provides → PMS verifies and authenticates.
- **Alt:** 4b. Incorrect credentials → resume at step 2.

## UC2: log off
- **Primary actor:** User
- **Pre:** UC1.
- **Post:** User logged off.
- **Main:** User indicates → PMS logs them off.

## UC3: send emergency notification
- **Primary actor:** Patient (via patient gateway)
- **Secondary actor:** Emergency call center
- **Pre:** UC10.
- **Post:** PMS received the emergency notification, verified it, and informed Emergency call center.
- **Main:**
    1. Patient gateway receives sensor data, identifies a potential emergency, packages as emergency notification; supplies proof of patient identity (e.g., API token); sends to PMS.
    2. PMS applies a dedicated (fast) emergency estimation model to confirm.
    3. PMS confirms emergency, sends notification to Emergency call center with all relevant data, and updates patient record (Include UC17).
- **Alt:** 3b. Verification rejects emergency → no notification but event marked in patient record (Include UC17).

## UC4: send sensor data
- **Primary actor:** Patient (via patient gateway)
- **Pre:** UC10.
- **Post:** Sensor data registered and being processed.
- **Main:**
    1. Patient gateway sends packaged sensor data + identity proof at the configured transmission rate.
    2. PMS receives, stores, and schedules processing (Include UC12).

## UC5: notify registered parties
- **Primary actor:** PMS
- **Pre:** PMS has received sensor data (UC4), processed it (UC12), and determined notifications should be sent.
- **Post:** Registered parties received the notification.
- **Main:**
    1. PMS looks up the parties interested in notifications (Physicians, Cardiologist, Patient).
    2. PMS prepares notification tailored per recipient and sends them.
    3. Registered party receives the notification.

## UC6: consult patient status
- **Primary actor:** Physician or Patient
- **Pre:** UC1.
- **Post:** Primary actor has consulted the status; PMS has logged it.
- **Main:**
    1. Actor indicates intent to consult.
    2. PMS looks up the info and presents it, tailored to expertise (Cardiologist gets detail; Patient gets summary).
    3. PMS logs the event.

## UC7: configure patient risk assessment
- **Primary actor:** Cardiologist
- **Pre:** UC1, UC6 (watching a patient's status).
- **Post:** PMS registered the configuration; processes sensor data accordingly; event logged.
- **Main:**
    1. Cardiologist requests to configure.
    2. PMS shows configurable options + current values (measurements to monitor, thresholds, conditions like hypertension).
    3. Cardiologist (re)configures and confirms.
    4. PMS stores configuration and logs.
    5. PMS schedules recalculation with the updated config (Include UC12).

## UC8: update risk level
- **Primary actor:** Cardiologist
- **Pre:** UC1, optionally received UC5 notification, UC6 watching the patient.
- **Post:** Risk level adapted; logged.
- **Main:**
    1. Cardiologist indicates need to change risk level.
    2. PMS offers low/medium/high.
    3. Cardiologist chooses.
    4. PMS changes risk level and updates patient record (Include UC17).
- **Alt:** 1b. Cardiologist accepts a notification's estimated risk level → forward to step 4.

## UC9: perform on-demand consultation
- **Primary actor:** Cardiologist
- **Pre:** UC1, UC6 (watching a patient's status).
- **Post:** Cardiologist has performed an on-demand consultation.
- **Main:**
    1. Cardiologist requests current data for a patient.
    2. PMS issues a request to the patient's gateway for current sensor data.
    3. Patient gateway responds with current sensor data.
    4. PMS schedules processing (Include UC12) and presents results to Cardiologist.

## UC10: register patient
- **Primary actor:** Trained nurse
- **Pre:** Cardiologist approved registration; UC1.
- **Post:** Patient registered; logged.
- **Main:** Nurse starts process → PMS asks to pair two devices → nurse pairs wearable + patient gateway and links gateway to patient identity → PMS initializes, runs diagnostics, requests first sensor data and shows to nurse → nurse calibrates if needed → PMS lets nurse set patient credentials → nurse enters them → PMS registers patient.

## UC11: deregister patient
- **Primary actor:** Trained nurse
- **Pre:** UC10, UC1, Cardiologist approved deregistration.
- **Post:** Patient deregistered; logged.
- **Main:** Nurse indicates deregistration → PMS deregisters wearable, gateway, stops monitoring, logs.

## UC12: estimate risk level
- **Primary actor:** PMS
- **Pre:** UC4 (new sensor data received), UC7 (risk assessment configured).
- **Post:** Patient status updated; notifications issued if needed.
- **Main:**
    1. PMS looks up monitoring history, current status, patient record (Include UC16), and risk assessment configuration.
    2. PMS applies all models in the risk assessment to the new + known data.
    3. PMS determines estimated risk level should change based on combined model results.
    4. PMS determines interested parties and issues notifications (Include UC5).
    5. PMS determines that new sensor data + risk results should propagate to patient record and sends update (Include UC17).
- **Alt:**
  - 3b. Risk level should NOT change → skip to step 5.
  - 5b. Results should NOT propagate → use case ends.

## UC13: retrieve new ML models
- **Primary actor:** Telemedicine operator
- **Secondary actor:** ML model provider
- **Pre:** UC1.
- **Post:** New ML models retrieved and stored.
- **Main:** Operator triggers check → PMS queries model provider for list → provider replies with available models + metadata → for each new model PMS offers download option (Include UC14).
- **Alt:** 4b. No new models available → inform operator, end.

## UC14: download ML model
- **Primary actor:** PMS
- **Secondary actor:** ML model provider
- **Pre:** None.
- **Post:** Model retrieved and installed.
- **Main:** PMS queries provider with model ID → provider replies → PMS adds to repository.

## UC15: update ML model
- **Primary actor:** Telemedicine operator
- **Secondary actor:** ML model provider
- **Pre:** UC1.
- **Post:** ML model updated.
- **Main:** Operator requests list → PMS shows list → operator selects one → PMS shows detail → operator triggers update → PMS queries provider for updates → provider replies → PMS applies updates → PMS informs operator.
- **Alt:** 2b. No models installed → inform, end. 7b. No updates available → inform, end.

## UC16: consult patient record
- **Primary actor:** PMS
- **Secondary actor:** HIS
- **Pre:** UC10.
- **Post:** PMS has received the patient record from HIS.
- **Main:** PMS requests lookup → HIS performs lookup and returns the patient record.
- **healthAPI calls** (see `appendixB_APIs.md`): `getPatient`, `searchPatient`, `getObservation`, `searchObservation`, `getRiskAssessment`, `searchRiskAssessment` (the read-side surface).

## UC17: update patient record
- **Primary actor:** PMS
- **Secondary actor:** HIS
- **Pre:** PMS is monitoring a patient of the hospital.
- **Post:** PMS provided info for the patient record in HIS.
- **Main:** PMS sends update → HIS accepts and processes.
- **healthAPI calls** (see `appendixB_APIs.md`): `savePatient`, `saveObservation`, `saveRiskAssessment` (occasionally `delete*`).

## UC18: consult monitoring summary
- **Primary actor:** Patient
- **Pre:** UC1.
- **Post:** Patient received a summary.
- **Main:** Patient indicates via gateway → PMS compiles a layperson-suited summary (e.g., graph of risk level changes over time) and sends to gateway → gateway displays as tables/graphs.

## UC19: request access to personal data
- **Primary actor:** User
- **Pre:** None.
- **Post:** Primary actor informed about processing status; provided with downloadable archive.
- **Main:**
    1. Actor indicates intent to perform right-to-access request.
    2. Actor authenticates (Include UC1).
    3. PMS determines whether it processes personal data concerning the actor.
    4. PMS informs actor it processes their data (Include UC5).
    5. PMS gathers all relevant personal data into an archive: profile info (incl. gateway data), all risk estimations, all sensor readings, data used from EHR, patient-specific risk assessment config (e.g., clinical model).
    6. PMS notifies actor that archive is available for download (Include UC5).
- **Alt:**
  - 2b. Actor not registered → provides email; PMS sends verification token; actor returns token; continue at step 3.
  - 3b. No personal data processed → inform, end.
  - 6b. Email-identified actor → PMS emails a download link.

---

## QAS-P1 use case touchpoints (quick reference)

| QAS response | Touched UCs |
|---|---|
| Process all status requests timely | UC6 |
| On-demand consultation init + result | UC9, UC12 |
| Risk-level updates timely | UC8, UC17 |
| Notification volume / SLAs | UC5 |
| Notification overload prioritization | UC5 |
| No impact on ingest / risk estimation | UC4 (ingest), UC12 (estimation) — must NOT be disturbed |
