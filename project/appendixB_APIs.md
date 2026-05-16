# Appendix B — healthAPI Reference Extract

Source: `SA_project_appendixB_APIs.pdf` (8 pp, 189 KB). Part 1: Appendix B: APIs.
This file is the lookup-free reference; if something below contradicts the PDF, the PDF wins.

> Consult this when designing anything that touches the **HIS boundary** — specifically UC16 (consult patient record) and UC17 (update patient record). For the P1 extension, this is the seam through which physician-facing data must flow.

## What this API is

`healthAPI` is the interface the PMS uses to interoperate with the Hospital Information System (HIS). It is a **reduced subset of HL7 FHIR** (Fast Healthcare Interoperability Resources), based on the HAPI FHIR Java implementation:

- FHIR standard: `https://hl7.org/fhir/`
- HAPI FHIR Swagger UI (OpenAPI): `https://hapi.fhir.org/baseR4/swagger-ui/`
- HAPI FHIR data-type Javadoc: `https://hapifhir.io/hapi-fhir/apidocs/hapi-fhir-structures-r5/org/hl7/fhir/r5/model/package-summary.html`

The three core resources are **Patient**, **Observation**, **RiskAssessment**. Everything else is supporting data types.

---

## 1. Operations on `healthAPI`

All operations may throw `AuthorizationException` and/or `AuthenticationException` unless noted.

### Patient

| Operation | Effect | Returns |
|---|---|---|
| `Patient getPatient(string id)` | Retrieve a patient record by id. | The `Patient`. |
| `Patient savePatient(Patient patient)` | Create new `Patient` instance or update existing one. | Updated `Patient`. Throws `AuthorizationException` only. |
| `void deletePatient(string id)` | Delete `Patient` by id. | — |
| `List<Patient> searchPatient(...)` | Search by 22 query criteria (see below). | Matching list. |

`searchPatient` query parameters: `birthDate, deceased, addressState, administrativeGender, link, language, addressCountry, deathDate, phonetic, telecom, addressCity, email, given, identifier, address, generalPractitioner, active, addressPostalCode, phone, organizationCustodian, name, family` — all wrapped as `Query<T>`.

### Observation

| Operation | Effect | Returns |
|---|---|---|
| `Observation getObservation(string id)` | Retrieve an observation by id. | The `Observation`. |
| `Observation saveObservation(Observation obs)` | Create new or update existing. | The `Observation`. |
| `void deleteObservation(string id)` | Delete by id. | — |
| `List<Observation> searchObservation(...)` | Search by 14 query criteria. | Matching list. Throws `AuthorizationException` only. |

`searchObservation` query parameters: `date, dataAbsentReason, subject, valueConcept, valueDate, derivedFrom, patient, valueQuantity, identifier, performer, method, category, device, status`.

### RiskAssessment

| Operation | Effect | Returns |
|---|---|---|
| `RiskAssessment getRiskAssessment(string id)` | Retrieve by id. | The `RiskAssessment`. Throws `AuthorizationException` only. |
| `RiskAssessment saveRiskAssessment(RiskAssessment ra)` | Create new or update existing. | Updated `RiskAssessment`. |
| `void deleteRiskAssessment(string id)` | Delete by id. | — |
| `List<RiskAssessment> searchRiskAssessment(...)` | Search by 9 query criteria. | Matching list. |

`searchRiskAssessment` query parameters: `date, identifier, performer, method, probability (Range), subject, condition, patient, risk (BigDecimal)`.

> All operations report **None** for "Sequence Diagrams" in the PDF — the source doesn't bind specific UCs to specific calls. The mapping to UCs is **implicit**: UC16 = read-side calls (`get*`, `search*`), UC17 = write-side (`save*`, `delete*`).

---

## 2. Exceptions

| Exception | Parent | Meaning |
|---|---|---|
| `SecurityException` | — | Parent class for security exceptions. |
| `AuthenticationException` | `SecurityException` | Missing or invalid credentials. |
| `AuthorizationException` | `SecurityException` | Requestor is not authorized to perform the action. |

---

## 3. Data types

### Core resources

#### `Patient`
Demographics + admin info about an individual receiving care.
Attributes: `List<Identifier> identifier`, `boolean active`, `List<String> name`, `String administrativeGender`, `Date birthDate`, `Date deceased`, `List<Address> address`, `boolean multipleBirth`, `List<Identifier> generalPractitioner`, `Identifier managingOrganization`, `ContactComponent contact`, `long serialVersionUID`.

#### `Observation`
Measurements and simple assertions about a patient, device, or other subject.
Key attributes: `List<Identifier> identifier`, `List<Identifier> partOf` (larger event this Observation is part of, e.g., a procedure), `ObservationStatus status`, `List<CodeableConcept> category` (e.g., LOINC `vital-signs`), `CodeableConcept code` (what was observed — e.g., LOINC `8867-4` = heart rate, `85354-9` = BP panel), `Patient subject`, `Period effective`, `Date issued`, `List<Identifier> performer`, `String value`, `CodeableConcept dataAbsentReason`, `CodeableConcept interpretation` (e.g., `H` = high), `List<Annotation> note`, `CodeableConcept bodySite` (SNOMED — e.g., `344001` = ankle), `CodeableConcept method` (SNOMED — e.g., `46973005` = BP procedure), `String device`, `List<Observation> hasMember`, `List<Observation> derivedFrom`, `ObservationReferenceRangeComponent referenceRange`, `ObservationComponentComponent component`.

#### `RiskAssessment`
Assessment of likely outcomes for a patient + their likelihood.
Attributes: `List<Identifier> identifier`, `ObservationStatus status`, `CodeableConcept method` (algorithm used), `CodeableConcept code` (type of risk assessment — e.g., SNOMED `709510001` = assessment of risk for disease), `Patient subject`, `Date occurrence`, `String condition` (simplified ref to existing condition), `Identifier performer`, `List<CodeableConcept> reason`, `List<Observation> basis`, `String mitigation`, `List<Annotation> note`, `RiskAssessmentPredictionComponent prediction`.

#### `RiskAssessmentPredictionComponent`
The actual prediction inside a `RiskAssessment`.
Attributes: `String outcome` (e.g., remission, death, condition), `Range probability`, `String qualitativeRisk` (low/medium/high), `BigDecimal relativeRisk`, `Date when`, `String rationale`. Outcome + qualitativeRisk are actually `CodeableConcept` but simplified to strings here.

#### `ObservationComponentComponent`
Used when an Observation has sub-observations (e.g., systolic + diastolic for BP).
Attributes: `String code`, `Range value`, `String dataAbsentReason`, `List<String> interpretation`, `ObservationReferenceRangeComponent referenceRange`.

#### `ObservationReferenceRangeComponent`
Guidance on how to interpret a value against a normal/recommended range. Multiple ranges = OR-combined for distinct target populations.
Attributes: `Quantity low`, `Quantity high`, `CodeableConcept type` (e.g., `normal`, `recommended`), `List<CodeableConcept> appliesTo` (target population — e.g., SNOMED `248152002` = female), `Range age`, `String text` (fallback when quantitative range not appropriate).

### Identity / contact

#### `Identifier`
Attributes: `IdentifierUse use`, `String type` (coded purpose), `String system` (namespace URL), `String value` (unique within system), `Period period` (validity), `String assigner` (org that issued/manages).

#### `Address`
Attributes: `AddressUse use`, `AddressType type`, `String text` (full address as displayed), `List<String> line` (house no., street, P.O., …), `String city, district, state, postalCode, country`, `Period period`.

#### `ContactComponent`
A contact party (guardian, partner, friend) for a patient.
Attributes: `List<CodeableConcept> relationship` (e.g., CHILD, MGRMTH), `String name`, `List<ContactPoint> telecom`, `Address address`, `String administrativeGender`, `String organization`, `Period period`.

#### `ContactPoint`
Tech-mediated contact info (phone, email, …).
Attributes: `ContactPointSystem system`, `String value` (the actual phone/email/etc.), `ContactPointUse use`, `int rank` (lower = preferred), `Period period`.

### Terminology / coded values

#### `CodeableConcept`
A concept that may be a formal reference to a terminology/ontology, or just text.
Attributes: `List<Coding> coding`, `String text`.

#### `Coding`
A reference to a code defined by a terminology system.
Attributes: `String system` (terminology URL), `String version`, `CodeType code`, `String display`, `boolean userSelected`.

#### `CodeType`
Primitive FHIR "code" not bound to an enumerated list.
Attributes: `String system`, `String value`.

#### `Annotation`
Text note + who said it and when.
Attributes: `Date time`, `String text`, `String author`.

### Primitives & quantities

#### `Date`
Specific instant in time, ms precision.

#### `Period`
A time interval. Attributes: `Date start`, `Date end`.

#### `Quantity`
A measured (or measurable) amount. Attributes: `BigDecimal value`, `String unit` (human-readable, e.g., `mm[Hg]`), `String system` (e.g., `http://unitsofmeasure.org`), `CodeType code` (machine form).

#### `Range`
A bounded set of ordered Quantities. Attributes: `Quantity low`, `Quantity high`.

#### `BigDecimal`
Immutable arbitrary-precision signed decimal. Unscaled-value integer + 32-bit scale.

#### `Query<T>`
Generic search wrapper used throughout the `search*` operations. Encodes string-based search restrictions for the REST API.
Attributes: `List<T> values`, `List<String> qualifier` (e.g., `lt` = less-than for a date, `exact` = exact string match, `matches` = substring).

### Enumerations

| Enum | Values | Notes |
|---|---|---|
| `ObservationStatus` | registered, preliminary, final, amended, corrected, cancelled, enteredinerror, unknown, null | Reused for `RiskAssessment.status`. |
| `AddressType` | postal, physical, both, null | Physical = visitable; postal = mailable (PO box, care-of). |
| `AddressUse` | home, work, temp, old, billing, null | — |
| `IdentifierUse` | usual, official, temp, secondary, old, null | — |
| `ContactPointSystem` | phone, fax, email, pager, url, sms, other, null | — |
| `ContactPointUse` | home, work, temp, old, mobile, null | — |

---

## Quick map: which call for which UC

| UC | API calls likely involved |
|---|---|
| UC16 — consult patient record | `getPatient`, `searchPatient`, `getObservation`, `searchObservation`, `getRiskAssessment`, `searchRiskAssessment` |
| UC17 — update patient record | `savePatient`, `saveObservation`, `saveRiskAssessment` (occasionally `delete*`) |

The PDF doesn't bind these explicitly — read-side = consult, write-side = update.

## Architectural touchpoints for P1

`healthAPI` is the **synchronous HIS boundary**. For the P1 extension (data exchange with physicians, see `part2-current/instructions.md` §7.3), every operation that crosses this boundary contributes to physician-visible latency. The P1 response measures (UC8 EHR forwarded within 1 min, etc.) rely on these calls completing — patterns like caching, batching, async-fire-and-forget, or circuit-breaking at this seam are valid design moves for P1.
