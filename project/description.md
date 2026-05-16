# PMS Application Case — Reference Extract

Source: `SA_project_description.pdf` (14 pp, 424 KB). Part 1: Application case description.
This file is the lookup-free reference; if something below contradicts the PDF, the PDF wins.

> Read this once to ground yourself in **why** the PMS exists and **what** it must do. Pairs with `lectures/L1/SA_L01_2_PMS_intro.pdf` (same domain, lecture-format).

---

## 1. Context

### Cardiovascular diseases (CVDs)

- CVDs = heart / blood-vessel diseases (most often arteriosclerosis → heart attacks, strokes; also hypertension, congestive heart failure).
- Precursors start in adolescence; CVDs develop over decades, become acute with age.
- Prevention is critical (diet, blood pressure, smoking).
- Scale: in 2015 CVD cost the EU **€210 bn** (€111 bn health care, €45 bn informal care, €32 bn early mortality, €23 bn lost productivity).
- Treatment priorities: (i) primary prevention + timely diagnosis, (ii) management — prediction/prevention of malignant events (e.g., heart failure).

### E-health and telemedicine

- **E-health** = electronically supported health care (knowledge mgmt, scheduling, data management).
- **EHR** (Electronic Health Record) — centralizes patient data across providers.
- **Telemedicine** = remote treatment + monitoring; cuts hospitalization cost.
- A **patient monitoring service** is one form of telemedicine: not active treatment, but continuous up-to-date data → physician predicts/prevents malignant events.

### Positioning

- Perspective: a **startup company** entering market with a novel PMS for CVD management.
- Pilot with one specific hospital first, then expand.
- Sold **as a service** (not as software) → company hosts/maintains the infra, well-integrated with hospital HIS.
- Integrates with HIS for: patient records, decision support to cardiologist, invoicing, EHR sync.
- **Key innovation**: flexible ML-based risk estimation (vs. static rule-based competitors).

---

## 2. Problem domain

### 2.1 Overall system goals

Timeline of a CVD patient (Fig. 1): precursors → primary diagnosis (T2) → monitoring setup (T3–T4) → prediction of malignant event (T5) → diagnosis + treatment adjustment (T6) → end of monitoring (T7).

Four goal dimensions:

1. **Continuous and remote monitoring** — 24/7, no hospital confinement.
2. **Timely decision making** — physicians get an up-to-date picture, fewer in-person visits.
3. **Prediction of malignant events** — preventive notifications (e.g., heart attack risk).
4. **Low invasiveness** — patient not bound to one location; minimal day-to-day disruption.

### 2.2 Continuous monitoring

#### Wearable unit
- Bundles sensors for CVD; worn close to the heart (e.g., chest band).
- Battery life **≥ 8 h**, minimal size.
- Medical device — measures the following parameters:

| Parameter | Normal range (healthy adult) | Notes |
|---|---|---|
| Heart rate | 60–100 bpm | correlate with activity level |
| ECG | — | extract max ventricular rate, ventricular arrhythmias, ischemia signs |
| Respiratory rate | 12–20 breaths/min | correlate with activity level |
| Blood oxygen | 90–100% | — |
| Blood pressure | 85–140 mmHg systolic | — |
| Activity level | accelerometer (single number) | — |
| Body temperature | 35–37 °C | — |

#### Additional inputs
- Optional: smart scale (weight), GPS (location), daily/weekly questionnaires, third-party lifestyle trackers.

#### Patient gateway
- Sensors don't talk to backend directly — they go through the **patient gateway** (= patient's smartphone running the PMS app).
- Gateway aggregates raw sensor data and **transmits processed packages** (not raw) to the backend.
- Transmission rate is **configurable per-patient and per-parameter**; default depends on patient risk level.

**Default transmission rates for HIGH-risk patients (Table 1):**

| Parameter | Transmission rate | Derived measure |
|---|---|---|
| Heart rate | 48/day | avg, max, min, variance |
| ECG | 24/day | full ECG of 30 s |
| Respiratory rate | 48/day | avg, max, min |
| Oxygen level | 24/day | avg, max, min |
| Blood pressure | 48/day | avg, max, min |
| Activity level | 4/day | single value |
| Temperature | 6/day | avg |

- Gateway also supports **on-demand** transmission (physician pulls latest data for remote consultation).
- Gateway app additionally provides patient UI: device mgmt (battery), data/risk consultation, treatment follow-up, GP contact, optional social network.

#### Impact on day-to-day life
- Low-power wearable + mobile network → patient is not bound to one place.
- Privacy concerns covered in §4.3 (GDPR).

### 2.3 Risk estimation and notifications

- Backend continuously estimates patient's CVD health status and its evolution → predicts upcoming malignant events → notifies relevant parties (GP, cardiologist, etc.).
- **Risk levels**: low, medium, high. Physician sets initial level at primary diagnosis.
- **Clinical risk models** combine data mining, ML, probabilistic models (Bayesian networks), ontologies.
- Inputs: (i) new gateway data, (ii) old data, (iii) patient pathologies, (iv) current risk level. Optional: artifacts from prior classifications (per-patient trained classifiers).

#### Simple threshold-based example (Table 2 — systolic BP, mmHg)

| Risk level | Normal | Worrisome | Dangerous |
|---|---|---|---|
| Low | 105–120 | 90–105, 120–150 | 80–90, 150–190 |
| Medium | 95–135 | 90–95, 135–160 | 80–90, 160–190 |
| High | 90–150 | 85–90, 150–170 | 80–85, 170–190 |

> Higher-risk patients have **wider "normal" ranges** because their baseline cardiovascular condition is already elevated.

- Other example: HeartScore (general-population tool, useful for patient self-tracking).
- **No single clinical model fits all CVD pathologies** → system supports multiple, possibly per-patient. Overall risk = combination of individual model outcomes.
- Some models are versatile/generic (deployed system-wide), others are very specific to certain pathologies or even per-patient.
- **All risk estimations are executed by PMS** (not by clinical model provider).
- Advanced models are **ML classifiers** provided by a third-party research institute; PMS uses an ML framework to run them; provider releases updates over time.
- **Decision-making authority**: only the **physician** can change a patient's risk level. The system notifies the physician of suspected changes; the physician approves / revises / declines.

#### Notification levels
- **Green** — status reverts to a lower risk level.
- **Yellow** — status changes non-critically but stays potentially dangerous.
- **Red** — status changes significantly, immediate / urgent attention.

### 2.4 Physician decision support

- Same medical models also feed the physician's decision-support UI during a consultation: structured summary of the most important and relevant info (current risk + relevant context such as recent notifications, ECG, BP graphs).
- Physician can also request any additional info on demand.

### 2.5 Patient support

- Patient sees their own status via the gateway — same data, but **less technical** (e.g., "% improvement since last week" graphs).

### 2.6 Medical emergencies

- In addition to green/yellow/red, the system can send **emergency notifications** to the **Hospital Emergency services** for life-threatening events (heart attack).
- Emergency notifications cover both **ongoing** and **predicted/anticipated** emergencies.
- Regular notifications are computed in the **backend** using clinical risk models. But emergencies need to be faster than the 48/day max transmission cadence.
- Therefore the **gateway itself** runs a lightweight emergency model (threshold-based, like §2.3) to detect emergencies locally.
- Gateway alerts backend with the necessary data → backend re-checks with more extensive models → sends emergency notification if needed.
- **Not a life-saving device.** Timely delivery of emergency notification is **not guaranteed in every case.**

---

## 3. Main stakeholders

| Stakeholder | Role / interest |
|---|---|
| **Patients** | Already diagnosed with CVD, now being monitored. Want help with treatment in the most comfortable (least invasive) way. |
| **Physicians** | Cardiologists, thoracic surgeons, vascular surgeons, neurologists, radiologists, **GPs** (holistic view). Want structured/efficient info; notifications of important status changes; clean status graphs. Not all in same location — GP often has independent practice; specialists may practice outside the hospital too. |
| **Nurses** | Not part of the system at the hospital — **except** a specialized team that does sensor setup / configuration after diagnosis. |
| **Emergency call center** | Receives emergency notifications. Wants as much info as possible to dispatch appropriately. Also needs the event entered in the system so other parties (specialist, GP) are kept up to date. |
| **Telemedicine operators** | Tech support for the platform: sensor malfunctions, sysadmin issues. Also do **system management**: integrate risk models, deal with model updates, provision resources. Receive alerts when sensors have not sent data, or when data looks anomalous. |
| **The hospital** | Pilot client. Wants improved CVD treatment efficiency + frees up beds (monitor at home instead of in-hospital). |
| **EHR service** | Collects health records across organizations, provides physicians/patients with access. PMS data must eventually integrate into patient's EHR via HIS. |
| **Legal departments** | Both company's and hospital's. Want provable compliance during operation, not just by-design. |
| **Hospital financial dept.** | Wants clear usage/billing reports for invoicing patients. |
| **Telecom operators** | Provide channels for sensors/gateway↔system comms. Bound by SLA (capabilities, availability). |
| **ML model provider** | External research institute. Trains/prepares specialized ML models for specific CVD pathologies, releases trained classifiers + updates. Also provides the ML framework that PMS integrates to run the models. |

---

## 4. Key building blocks and constraints

### 4.1 Hospital Information System (HIS)

- PMS = standalone platform run by the company, **integrates tightly with the existing HIS**.
- Relevant standard: **HL7 FHIR** (Fast Healthcare Interoperability Resources) — `https://hl7.org/fhir/`. The healthAPI extract (`appendixB_APIs.md`) is a reduced subset of FHIR.
- Integration covers:

  - **Patient data**: hospital retains full control of patient records. PMS must integrate measurements into the hospital's patient-record storage. From a regulatory POV the hospital is the **data controller**, PMS is the **data processor**. Some data (very fine-grained, frequent updates) is infeasible to forward to the hospital record in full — so **synchronization** of patient data between PMS and HIS is a core functional aspect.
  - **Physician workstation**: cardiologists already access HIS via personal workstation or remote. PMS adds decision support, remote sensor config, daily follow-up, notifications — must integrate with that workstation.
  - **Patient registration**: trained nurse pairs/links wearable + sensors to patient, initializes, calibrates, tests before sending patient home.
  - **Emergency situation**: PMS must be able to trigger emergency procedures via the hospital's emergency services (e.g., 112) and notify other stakeholders (physician, GP).
  - **Accounting**: hospital must be kept aware day-by-day of operational costs (sensor maintenance, compute, storage, bandwidth, call-center operation).

### 4.2 Financial constraints

- Pilot is costly. Company has negotiated a **€1M contract** with one specific hospital.
- Pilot includes an **SLA** with quality requirements (performance + availability), e.g., upper bound on emergency-notification delivery time.
- Company also has SLAs with **telecom operator** (patient connectivity) and **hardware manufacturer** (acquisition, maintenance, upgrade, repair).

### 4.3 Legal constraints (GDPR)

Health data = personal data under **GDPR**. Guiding principles:

- Health info collected/used only with **explicit patient consent**, and only for continuous CVD treatment — unless processing is required to protect vital interests (emergencies), for payment, or for service administration.
- **Minimize disclosure** of identity info; anonymize/de-identify whenever possible.
- Patient has **right to access, rectify, remove** their personal data.
- Health data retained for legally mandated duration (**30 years in Belgium**, 6 years in the US).
- **Accountability**: any disclosure to a third party must be monitored and logged.

### 4.4 Security (scope: authentication + action logging)

- **User authentication**: ensure identity of every actor; prevent spoofing; prevent unauthorized access. Concrete realization differs per user type — e.g., physicians may authenticate via badge, patients implicitly via wearable linked to their identity.
- **User action logging**: record every action performed by any actor (and by the system itself). Enables compliance audits + incident traceability. Depends on correct authentication.

---

## References (verbatim from PDF §References)

[1] Cowie & Lam, "Remote monitoring and digital health tools in CVD management," *Nature Reviews Cardiology* 18(7), 2021.
[2] Proudfoot et al., "Physical activity and trajectories of cardiovascular health indicators during early childhood," *Pediatrics* 144(1), 2019.
[3] Leal & Burns, "MEP Heart Group — The cost of cardiovascular disease in the European Union," 2017.
[4] "eHealth — Wikipedia."
[5] Nakanishi et al., "Machine learning adds to clinical and CAC assessments in predicting 10-year CHD and CVD deaths," *Cardiovascular Imaging* 14(3), 2021.
[6] Damjathikar et al., "Multiclass ML vs. conventional calculators for stroke/CVD risk assessment," *Int. J. Cardiovascular Imaging* 37(4), 2021.
[7] Kakadiaris et al., "Machine learning outperforms ACC/AHA CVD risk calculator in MESA," *J. American Heart Assoc.* 7(22), 2018.
[8] HeartScore — `http://www.heartscore.org/`.
[9] EU Regulation 2016/679 (GDPR), April 2016.
