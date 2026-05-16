# Data Types — Lookup Reference

**Provenance:**
- **§1–§5** = types defined in the initial architecture PDF (§E.6 data types, §E.5 exceptions). Course-provided baseline. Do not rename.
- **§7** = types introduced by the P1 extension (mine). Not in the initial docs — added because P1's new interfaces need them. Free to rename / split / merge.

**Before defining a new type, check this list.** Convention 1 requires globally unique names — picking a name already in use will collide.

Grouped by category. Each entry lists attributes (when defined in the source) and a one-line purpose. Types most commonly referenced in interface signatures are marked ⭐.

---

## 1. PMS-specific (risk-estimation domain)

| Type | Attributes | Purpose |
| --- | --- | --- |
| ⭐ `PatientId` | opaque (long/string/URI) | Identifies a patient. |
| ⭐ `PatientStatus` | — | Patient's risk level. |
| ⭐ `PatientRecord` | — (structured EHR contents) | Medication history, treatments, allergies, etc. |
| ⭐ `SensorDataPackage` | `List<SensorDataElement> sensorDataElements` | Bundle of sensor measurements. |
| `SensorDataElement` | `String sensorId, sensorType, measurementType, measurementUnit; BigDecimal measurementValue; String measurementValueString` | One sensor measurement (numeric or string). |
| ⭐ `Timestamp` | — | Date + time-of-day in the system. |
| ⭐ `ClinicalModel` | `String id, name, description; List<String> requiredParameters, optionalParameters, applicableMLModelIds; boolean requiresHistoricalSensorData` | Clinical risk model definition. |
| `ClinicalModelConfiguration` | `Map<String,String> modelParameters` | Per-patient params for a `ClinicalModel`. |
| `ClinicalModelJob` | `ClinicalModel clinicalModel; ClinicalModelJobId jobId; SensorDataPackage sensorData; PatientId patientId; ClinicalModelConfiguration clinicalModelConfiguration; PatientRecord patientRecord; Map<Timestamp,SensorDataPackage> historicalSensorData` | Single computation unit for the risk pipeline. |
| `ClinicalModelJobId` | opaque (long/string/URI) | Identifies a `ClinicalModelJob`. |
| `ClinicalModelJobResult` | — | Result of one `ClinicalModelJob` (risk score on a common scale). |
| `RiskEstimationId` | opaque (long/string/URL) | Identifies a single risk-estimation execution. |
| `MLModel` | `String modelId; MLModelDescription modelDescription` | ML model used for prediction (internal encoding opaque). |
| `MLModelDescription` | `String uuid, name, description, providerName` | MLModel metadata (no model body). |
| `MLModelUpdate` | `MLModelDescription modelDescription` | Incremental update for an MLModel. |
| `Prediction` | — | Output of `MLModel.predict`. |
| `Credential` | — | Auth object for MLaaS requests. |

---

## 2. FHIR-aligned (clinical interop, mostly via `healthAPI` / HIS)

| Type | Attributes | Purpose |
| --- | --- | --- |
| `Patient` | `List<Identifier> identifier; boolean active; List<String> name; String administrativeGender; Date birthDate, deceased; List<Address> address; boolean multipleBirth; List<Identifier> generalPractitioner; Identifier managingOrganization; ContactComponent contact; long serialVersionUID` | Demographic + admin info for an individual. |
| `Observation` | (long; see PDF §E.6) `List<Identifier> identifier, partOf; ObservationStatus status; List<CodeableConcept> category; CodeableConcept code; Patient subject; Period effective; Date issued; List<Identifier> performer; String value; CodeableConcept dataAbsentReason, interpretation; List<Annotation> note; CodeableConcept bodySite, method; String device; List<Observation> hasMember, derivedFrom; ObservationReferenceRangeComponent referenceRange; ObservationComponentComponent component` | Measurement/assertion about a subject. |
| `ObservationComponentComponent` | `String code; Range value; String dataAbsentReason; List<String> interpretation; ObservationReferenceRangeComponent referenceRange` | Sub-observation (e.g. systolic/diastolic). |
| `ObservationReferenceRangeComponent` | `Quantity low, high; CodeableConcept type; List<CodeableConcept> appliesTo; Range age; String text` | Normal/recommended range for interpretation. |
| `RiskAssessment` | `List<Identifier> identifier; ObservationStatus status; CodeableConcept method, code; Patient subject; Date occurrence; String condition; Identifier performer; List<CodeableConcept> reason; List<Observation> basis; String mitigation; List<Annotation> note; RiskAssessmentPredictionComponent prediction` | FHIR-style risk assessment record. |
| `RiskAssessmentPredictionComponent` | `String outcome; Range probability; String qualitativeRisk; BigDecimal relativeRisk; Date when; String rationale` | One predicted outcome inside a `RiskAssessment`. |
| `Identifier` | `IdentifierUse use; String type, system, value; Period period; String assigner` | Business identifier with namespace/validity. |
| `CodeableConcept` | `List<Coding> coding; String text` | Concept defined by terminology refs or text. |
| `Coding` | `String system, version; CodeType code; String display; boolean userSelected` | Code in a terminology system. |
| `CodeType` | `String system, value` | FHIR primitive `code` (when unbound). |
| `Address` | `AddressUse use; AddressType type; String text; List<String> line; String city, district, state, postalCode, country; Period period` | Postal address. |
| `ContactComponent` | `List<CodeableConcept> relationship; String name; List<ContactPoint> telecom; Address address; String administrativeGender, organization; Period period` | Contact party (guardian, partner, …) for a patient. |
| `ContactPoint` | `ContactPointSystem system; String value; ContactPointUse use; int rank; Period period` | Phone/email/etc. contact detail. |
| `Annotation` | `Date time; String text, author` | Free-text note + author/time. |
| `Quantity` | `BigDecimal value; String unit, system; CodeType code` | Measured amount with unit. |
| `Range` | `Quantity low, high` | Quantity interval. |
| `Period` | `Date start, end` | Time interval. |
| `Query<T>` | `List<T> values; List<String> qualifier` | Search restriction for the REST health API (`lt`, `matches`, `exact`, …). |

---

## 3. Primitives / general

| Type | Purpose |
| --- | --- |
| `Date` | Instant in time, ms precision. |
| `BigDecimal` | Arbitrary-precision signed decimal. |
| `Object` | Root for things stored in the KV store — anything cached via `KVStore` must inherit. |

---

## 4. Enums

| Enum | Values |
| --- | --- |
| `AddressType` | `postal, physical, both, null` |
| `AddressUse` | `home, work, temp, old, billing, null` |
| `ContactPointSystem` | `phone, fax, email, pager, url, sms, other, null` |
| `ContactPointUse` | `home, work, temp, old, mobile, null` |
| `IdentifierUse` | `usual, official, temp, secondary, old, null` |
| `ObservationStatus` | `registered, preliminary, final, amended, corrected, cancelled, enteredinerror, unknown, null` |

---

## 5. Exceptions (§E.5)

Hierarchy:

- `Exception` (base)
  - `AuthenticationException`
  - `AuthorizationException`
  - `InvalidAPIKeyException`
  - `InvalidMLModelException`
  - `IOException`
  - `NoSuchMLModelException`
  - `NoSuchPatientException`
  - `NoSuchPatientRecordException`
- `SecurityException` (separate root)
  - `AuthenticationException2`
  - `AuthorizationException2`

---

## 6. Quick "do I need a new type?" checklist

Before introducing a new type:

1. **Identifier-like?** → reuse `PatientId`, `ClinicalModelJobId`, `RiskEstimationId`, or `Identifier` (FHIR business id).
2. **Time / interval?** → `Timestamp`, `Date`, `Period`.
3. **Numeric measurement?** → `BigDecimal`, `Quantity` (with unit), `Range`.
4. **Sensor payload?** → `SensorDataPackage` / `SensorDataElement` — don't invent a parallel shape.
5. **Patient-status / risk** → `PatientStatus`, `RiskAssessment` (FHIR), `ClinicalModelJobResult`.
6. **EHR content** → `PatientRecord` (opaque blob) or `Patient` + `Observation` (FHIR-structured).
7. **Search/query** → `Query<T>`.
8. **KV-store payload** → must extend `Object`.
9. **Auth artifact** → `Credential` (MLaaS-style).

If none fit, define a new type — and add it back here (in §7 if it's a P1 addition) so future lookups stay fast.

---

## 7. P1 additions (NOT in initial docs — introduced by `p1_current_design.md`)

These types appear in P1's new interface signatures (`IPhysicianAPI`, `INotificationInbox`, `IRiskEvents`, `IOnDemandSensorFetch`, extended `LaunchRiskEstimation`). They must be created in VP as new data-type elements (Convention 1 — globally unique names).

> **VP entry convention used in this section.** Everything below is copy-paste ready into Visual Paradigm:
> - **Description** for the class/enum in VP's *General* tab → use the **Purpose** line verbatim.
> - **Attributes** of a structured payload → one line per attribute, format `+ name: Type` (UML standard: visibility, name, type). Default visibility is `+` (public) for everything in a data type. Optional fields are written `+ name: Type [0..1]` (UML multiplicity), or in shorthand notes here as `Type?`.
> - **Literals** of an enum → one identifier per line, no commas.
> - **Stereotype** — leave blank for plain data classes; use `«enumeration»` for enums (VP applies this automatically when you create the element as an Enumeration).

### 7.1 Identifiers (opaque, like `PatientId`)

These are simple identifier classes — model each as a data type with **no attributes** (the actual encoding — long, string, URI — is intentionally left open, same convention as `PatientId` in §E.6). The Purpose line is the class description.

| Type | Used in | Purpose (paste into VP description) |
| --- | --- | --- |
| ⭐ `CorrelationId` | `IOnDemandSensorFetch.requestCurrentSensorData`, extended `LaunchRiskEstimation`, `RiskEvent` payload, `FilterCriteria` | Cross-component tag that ties a UC9 on-demand consultation to its eventual risk-estimation result. |
| `ConsultationId` | `IPhysicianAPI.requestOnDemandConsultation` return | UC9 handle returned to the physician. **Open question (§6 OQ in design): probably the same thing as `CorrelationId` — consider merging into one type.** |
| `PhysicianId` | `IPhysicianAPI.subscribeToNotifications`, `INotificationInbox.subscribe/unsubscribe/getPendingNotifications` | Identifies a physician for notification routing. |
| `NotificationId` | `INotificationInbox.acknowledgeNotification`, inside `Notification` | Identifies one queued notification (used for ack/drop). |
| `SubscriberId` | `IRiskEvents.subscribe` | Identifies a *component-level* subscriber (NotificationDispatcher, PhysicianCommandService). Distinct from `PhysicianId`. |
| `SubscriptionId` | `IRiskEvents.subscribe` return, `unsubscribe` arg | Handle for an active subscription. |

### 7.2 Structured payloads

Each one is a data-type class. Use **Purpose** as the description, paste the **Attributes** block line-by-line into VP's Attributes table.

#### `Notification`
- **Purpose:** UC5 payload returned to physicians via `INotificationInbox.getPendingNotifications`.
- **Attributes:**
  ```
  + id: NotificationId
  + patientId: PatientId
  + status: PatientStatus
  + severity: NotificationSeverity
  + ts: Timestamp
  ```

#### `RiskEvent`
- **Purpose:** Payload pushed to `IRiskEvents` subscribers when `setEstimatedPatientStatus` actually changes the stored status.
- **Attributes:**
  ```
  + patientId: PatientId
  + status: PatientStatus
  + ts: Timestamp
  + correlationId: CorrelationId [0..1]
  ```

#### `FilterCriteria`
- **Purpose:** Optional filter passed to `IRiskEvents.subscribe`. Empty for NotificationDispatcher (wants every event); carries a correlationId for PhysicianCommandService UC9 result lookup.
- **Attributes:**
  ```
  + correlationId: CorrelationId [0..1]
  ```
  *(Extensible — add more optional fields here if future filters need them.)*

### 7.3 Enums

Each one is an **Enumeration** in VP (right-click → New → Enumeration). Use **Purpose** as the description; paste the literals one per line into the Enumeration Literals table.

#### `NotificationSeverity`
- **Purpose:** Severity of a `Notification`. Drives the dispatcher's priority queue + drop policy (Response measure 5: red > yellow > green; green dropped first under overload).
- **Literals:**
  ```
  red
  yellow
  green
  ```

#### `Priority`
- **Purpose:** Priority of a launched risk-estimation job. UC9 uses `HIGH`; `NORMAL` is the default for legacy schedule-driven jobs.
- **Literals:**
  ```
  HIGH
  NORMAL
  ```
- **Open question:** if this enum is shared with P2's EDF tiers, it also needs `MEDIUM` and `LOW` — see `p1_current_design.md` §7 hard overlap #1 (teammate coordination).

### 7.4 Quick "are you sure this is new?" check

Before adding to §7, double-check it's not already in §1–§5 under a different name:

- A new opaque id → can you reuse `Identifier` (FHIR business id) instead? Usually no for PMS-internal ids, but worth checking.
- A new struct carrying patient + status → reuse `PatientStatus` if you only need risk level; reuse `RiskAssessment` if you need full FHIR shape.
- A new enum for severity → `NotificationSeverity` is intentionally separate from `PatientStatus` because they overlap conceptually but serve different roles (one indexes a queue, the other indexes a patient state).
