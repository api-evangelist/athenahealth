---
name: athenahealth-fhir-patient-summary
description: Assemble a US Core patient summary from the athenahealth FHIR R4 API — demographics,
  conditions, medications, allergies, immunizations, observations, diagnostic reports and documents
  — using cursor pagination and the tenancy and sensitivity parameters the live CapabilityStatement
  declares.
api: athenahealth FHIR R4 API
base: https://api.platform.athenahealth.com/fhir/r4
operations:
  - getCapabilityStatement
  - searchFhirPatients
  - readFhirPatient
  - searchFhirConditions
  - searchFhirMedicationRequests
  - searchFhirAllergies
  - searchFhirImmunizations
  - searchFhirObservations
  - searchFhirDiagnosticReports
  - searchFhirDocumentReferences
  - searchFhirEncounters
  - readFhirEncounter
generated: '2026-08-14'
method: generated
source: https://api.platform.athenahealth.com/fhir/r4/metadata,
  https://api.platform.athenahealth.com/fhir/r4/.well-known/smart-configuration,
  openapi/athenahealth-patient-api-openapi.yml, openapi/athenahealth-condition-api-openapi.yml,
  openapi/athenahealth-observation-api-openapi.yml,
  openapi/athenahealth-medicationrequest-api-openapi.yml,
  openapi/athenahealth-allergyintolerance-api-openapi.yml,
  openapi/athenahealth-immunization-api-openapi.yml,
  openapi/athenahealth-diagnosticreport-api-openapi.yml,
  openapi/athenahealth-documentreference-api-openapi.yml,
  openapi/athenahealth-encounter-api-openapi.yml
---

# Build a patient summary from athenahealth FHIR R4

This is the read-only half of athenahealth and it is the well-behaved half. The server describes
itself: `getCapabilityStatement` (`GET /metadata`) answers **anonymously**, HTTP 200,
`application/fhir+json`. Start there.

## Discover before you assume

1. **`getCapabilityStatement`** (`GET /metadata`). Confirms `fhirVersion 4.0.1`, lists the 32
   resource types this server supports, and — critically — lists the exact `searchParam` names each
   resource accepts. athenahealth supports **more** resources than any of its documentation
   enumerates. Read this, do not guess.

2. **Fetch `/.well-known/smart-configuration`** (also anonymous). It tells you the authorization
   and token endpoints, that PKCE `S256` is required, and that **both** SMART v1 (`patient/*.read`)
   and v2 (`patient/*.rs`) scope syntaxes are accepted (`permission-v1` and `permission-v2` are both
   in `capabilities`).

## Steps

3. **Find the patient.** `searchFhirPatients` (`GET /Patient`) with `identifier`, or
   `family` + `given` + `birthdate`. Then `readFhirPatient` (`GET /Patient/{id}`).

4. **Pull each clinical domain**, all scoped by `patient`:
   - `searchFhirConditions` — `GET /Condition?patient={id}&category=...`
   - `searchFhirMedicationRequests` — `GET /MedicationRequest?patient={id}`
   - `searchFhirAllergies` — `GET /AllergyIntolerance?patient={id}`
   - `searchFhirImmunizations` — `GET /Immunization?patient={id}`
   - `searchFhirObservations` — `GET /Observation?patient={id}&category=vital-signs|laboratory`
   - `searchFhirDiagnosticReports` — `GET /DiagnosticReport?patient={id}`
   - `searchFhirDocumentReferences` — `GET /DocumentReference?patient={id}`
   - `searchFhirEncounters` — `GET /Encounter?patient={id}`, then `readFhirEncounter` for detail

5. **Join in one round trip where you can.** The server declares `_include` targets such as
   `DiagnosticReport:result`, `DiagnosticReport:performer`, `MedicationRequest:medication`,
   `DocumentReference:author` and `DocumentReference:custodian`. Use them instead of N+1 reads.

## Rules that are not optional

- **Page with `cursor`, not offsets.** Every searchable resource declares `_count` and `cursor` as
  search parameters. Follow `Bundle.link[relation=next]`. Do not compute page numbers — there are
  none.
- **Scope your tenancy explicitly.** athenahealth declares five custom tenancy search parameters:
  `ah-practice`, `ah-brand`, `ah-department`, `ah-provider-group` and `ah-chart-sharing-group`. The
  last one is a chart *sharing* boundary. Omitting it can return records from practices you did not
  mean to span.
- **Respect `_security`.** `_security` is a declared search parameter on nearly every resource, and
  `Condition` additionally declares `ah-redact-inline-security`. These govern restricted content
  (behavioral-health and other protected categories). Never strip a sensitivity filter to "get more
  data", and never surface redacted content you obtained by widening one.
- **Ask for provenance when attribution matters.** 22 of the 32 resource types support
  `_revinclude=Provenance:target`, which returns who recorded the data and from where, in the same
  Bundle. If your output will be read clinically, include it.
- **Errors are `OperationOutcome`, not Problem Details.** Parse `issue[].severity`, `issue[].code`
  and `issue[].diagnostics` (`errors/athenahealth-problem-types.yml`). There is no
  `application/problem+json` anywhere on this API.
- **This skill is read-only by design.** Every operation listed is a `read` or `search-type`
  interaction. If a task asks you to write, stop and use a different, explicitly confirmed flow.
