---
name: athenahealth-find-and-book-appointment
description: Find a patient in athenaOne, list open appointment slots for the right department and
  provider, and complete a scheduling change (reschedule or cancel) safely on an API that offers no
  idempotency key.
api: athenahealth athenaOne v1 REST API
base: https://api.platform.athenahealth.com/v1/{practiceid}
operations:
  - getPracticeInfo
  - listDepartments
  - listProviders
  - searchPatients
  - getPatient
  - getOpenAppointmentSlots
  - searchAppointments
  - getAppointment
  - rescheduleAppointment
  - cancelAppointment
generated: '2026-08-14'
method: generated
source: openapi/athenahealth-practice-api-openapi.yml,
  openapi/athenahealth-departments-api-openapi.yml,
  openapi/athenahealth-providers-api-openapi.yml,
  openapi/athenahealth-patients-api-openapi.yml,
  openapi/athenahealth-appointments-api-openapi.yml
---

# Find and change an athenaOne appointment

## Before you start

**Bind the practice first.** Every athenaOne v1 URL contains the practice as a path segment:
`https://api.platform.athenahealth.com/v1/{practiceid}/...`. `practiceid` is a *server variable*,
not a request parameter. If you have not resolved it, you cannot build a valid URL.

**Get a token.** OAuth 2.0 against
`https://api.platform.athenahealth.com/oauth2/v1/token`. Both `authorization_code` and
`client_credentials` are supported. The athenaOne v1 surface uses the
`athena/service/Athenanet.MDP.*` scope; see `scopes/athenahealth-scopes.yml`.

**Use preview while you develop.** Swap the host for
`api.preview.platform.athenahealth.com` — it is a genuinely separate environment with its own
issuer and its own credentials (`sandbox/athenahealth-sandbox.yml`). Credentials do not carry
across.

## Steps

1. **Confirm the tenant.** `getPracticeInfo` (`GET /practiceinfo`) — confirms the token can see this
   practice before you touch patient data.

2. **Resolve the department.** `listDepartments` (`GET /departments`). `departmentid` is required by
   the slot search and by most scheduling calls; do not assume a default.

3. **Resolve the provider** (when the request names one). `listProviders` (`GET /providers`).

4. **Find the patient.** `searchPatients` (`GET /patients`) with `firstname`, `lastname`, `dob`
   and/or `departmentid`. Then `getPatient` (`GET /patients/{patientid}`) to confirm you have the
   right person before acting. **Never act on a search result with more than one match** — an
   ambiguous patient match in an EHR is a patient-safety event, not a UX inconvenience.

5. **Find the appointment or the slot.**
   - Existing appointment: `searchAppointments` (`GET /appointments`) filtered by `patientid`,
     `departmentid`, `startdate`/`enddate`; then `getAppointment`
     (`GET /appointments/{appointmentid}`).
   - New time: `getOpenAppointmentSlots` (`GET /appointments/open`) with `departmentid`,
     `providerid` and a date range.

6. **Make the change.**
   - `rescheduleAppointment` — `PUT /appointments/{appointmentid}/reschedule`
   - `cancelAppointment` — `PUT /appointments/{appointmentid}/cancel`

## Rules that are not optional

- **There is no idempotency key.** No athenaOne operation accepts `Idempotency-Key` or any
  equivalent (`conventions/athenahealth-conventions.yml`). If a write times out, **do not retry**.
  Re-run `getAppointment` and decide from the current state. A blind retry can cancel a
  re-booked slot or double-act on a patient's calendar.
- **Confirm before every write.** All three write operations here change a real patient's real
  appointment. Surface the patient name, the current time, the proposed time and the department to
  a human, and get an explicit yes.
- **You will not get a useful error.** The athenaOne v1 specification declares zero 4xx and zero 5xx
  responses (`errors/athenahealth-problem-types.yml`). Treat any non-2xx as unknown-outcome, not as
  a classified failure, and re-read state before doing anything else.
- **Paging is undocumented.** The captured list responses are bare JSON arrays with no envelope and
  no declared `limit`/`offset`. Do not assume you received a complete result set; narrow the query
  instead of paging blind.
- **Rate limits are undocumented and unsignalled.** No `RateLimit-*` or `Retry-After` header is
  declared anywhere. Back off conservatively on 429.
