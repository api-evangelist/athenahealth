# athenahealth GraphQL Schema

## Overview

athenahealth does not currently offer a public GraphQL API. The platform provides over 800 REST endpoints through its athenaOne proprietary API and FHIR R4 standards-based APIs. This conceptual GraphQL schema models the athenahealth clinical and administrative data domain using types derived from the FHIR R4 data model, athenaOne API resources, and standard healthcare interoperability patterns (HL7, SMART on FHIR).

This schema is intended for integration architects, EHR interoperability engineers, and healthcare application developers who want to understand the shape of athenahealth data and how it maps to a GraphQL surface.

## Provider

- **Name**: athenahealth
- **Parent**: Veritas Capital (private equity, acquired from Elliott Management 2022)
- **Products**: athenaOne (cloud EHR/PM), athenaPractice, athenaFlow, athenaCollector
- **REST API Base**: `https://api.athenahealth.com/v1/`
- **FHIR R4 Base**: `https://api.athenahealth.com/fhir/r4/`
- **Developer Docs**: https://docs.athenahealth.com/api/
- **GitHub**: https://github.com/athenahealth
- **Status**: https://status.athenahealth.com/

## Authentication

All athenahealth APIs use OAuth 2.0. Supported grant types:
- Authorization Code (provider-facing and patient-facing apps)
- Client Credentials (server-to-server integrations)

SMART on FHIR scopes are used for FHIR endpoints. API keys are issued through the Developer Portal at https://www.athenahealth.com/developer-portal.

## Schema Design Notes

This schema covers the full clinical, financial, and operational data surface of the athenahealth platform. Types are organized into the following domains:

### Clinical Domain
Patient demographics, encounters, clinical notes, problem lists, diagnoses, allergies, immunizations, medications, prescriptions, labs, vitals, referrals, and care coordination.

### Scheduling Domain
Appointments, providers, departments, practices, schedules, slots, and reminders.

### Financial Domain
Insurance, claims, charges, payments, ERA/remittance, prior authorizations, and revenue cycle operations.

### Administrative Domain
Documents, document types, custom fields, portal users, messages, subscriptions, webhooks, API clients, and tokens.

## Root Operations

### Query (Read)
- `patient(id: ID!)` — fetch a single patient by athenahealth patient ID
- `patients(practiceId: ID!, filter: PatientFilter)` — search patients within a practice
- `encounter(id: ID!)` — fetch a clinical encounter
- `appointment(id: ID!)` — fetch a scheduled appointment
- `appointments(filter: AppointmentFilter!)` — search appointments by date, provider, or patient
- `provider(id: ID!)` — fetch a provider record
- `providers(practiceId: ID!)` — list providers at a practice
- `practice(id: ID!)` — fetch a practice
- `department(id: ID!)` — fetch a department
- `claim(id: ID!)` — fetch a billing claim
- `labResult(id: ID!)` — fetch a lab result
- `document(id: ID!)` — fetch a clinical document
- `schedule(providerId: ID!, date: Date!)` — fetch open schedule slots

### Mutation (Write)
- `createAppointment(input: CreateAppointmentInput!)` — book an appointment
- `cancelAppointment(id: ID!, reason: String)` — cancel an appointment
- `updatePatient(id: ID!, input: UpdatePatientInput!)` — update patient demographics
- `createClinicalNote(input: CreateClinicalNoteInput!)` — create a clinical note
- `createMedicationOrder(input: MedicationOrderInput!)` — order a medication
- `createLabOrder(input: LabOrderInput!)` — place a lab order
- `createReferral(input: ReferralInput!)` — create a referral
- `submitClaim(input: ClaimInput!)` — submit a billing claim
- `createSubscription(input: SubscriptionInput!)` — register a webhook subscription

### Subscription (Real-time)
- `appointmentUpdated(practiceId: ID!)` — real-time appointment status changes
- `labResultReceived(patientId: ID!)` — incoming lab results
- `claimStatusChanged(practiceId: ID!)` — claim adjudication updates
- `patientMessageReceived(patientId: ID!)` — portal messages

## Type Count

This schema defines **75 named types** covering the full athenahealth data domain.

## Mapping to athenaOne REST API

| GraphQL Type | athenaOne REST Resource |
|---|---|
| Patient | `/patients` |
| Encounter | `/encounters` |
| Appointment | `/appointments` |
| Provider | `/providers` |
| Department | `/departments` |
| Practice | `/practices` |
| Insurance | `/insurances` |
| Claim | `/claims` |
| Diagnosis | `/diagnoses` |
| Medication | `/medications` |
| Lab | `/labs` |
| LabResult | `/labresults` |
| Vital | `/vitals` |
| Allergy | `/allergies` |
| Immunization | `/immunizations` |
| Document | `/documents` |
| Referral | `/referrals` |
| Message | `/patientcases` |
| Charge | `/charges` |
| Payment | `/payments` |

## FHIR R4 Mapping

| GraphQL Type | FHIR R4 Resource |
|---|---|
| Patient | Patient |
| PatientDemographic | Patient.name, Patient.address |
| Encounter | Encounter |
| Appointment | Appointment |
| Provider | Practitioner |
| Practice | Organization |
| Department | Location |
| Insurance | Coverage |
| Claim | Claim |
| Diagnosis | Condition |
| Medication | Medication |
| MedicationOrder | MedicationRequest |
| Prescription | MedicationRequest |
| Lab | DiagnosticReport |
| LabResult | Observation |
| LabOrder | ServiceRequest |
| Vital | Observation |
| Allergy | AllergyIntolerance |
| Immunization | Immunization |
| ProblemList | Condition |
| ClinicalNote | DocumentReference |
| Document | DocumentReference |
| Referral | ServiceRequest |
| Authorization | Coverage.preAuthRef |
| Schedule | Schedule |
| Slot | Slot |

## References

- athenahealth Developer Docs: https://docs.athenahealth.com/api/
- athenaOne API Reference: https://docs.athenahealth.com/api/docs/athenaone-apis
- FHIR R4 API: https://docs.athenahealth.com/api/docs/fhir-apis
- FHIR Subscriptions GitHub: https://github.com/athenahealth/aone-fhir-subscriptions
- athenaFlex GitHub: https://github.com/athenahealth/apiserver-athenaFlex
- SMART on FHIR: https://smarthealthit.org/
- HL7 FHIR R4: https://hl7.org/fhir/R4/
