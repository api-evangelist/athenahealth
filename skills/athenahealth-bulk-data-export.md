---
name: athenahealth-bulk-data-export
description: Run a FHIR Bulk Data ($export) job against athenahealth for a Group, poll it to
  completion and cancel it cleanly — the one athenahealth flow that carries a real async runtime
  contract (Prefer respond-async, 202, Content-Location).
api: athenahealth FHIR Bulk Data Access API
base: https://api.platform.athenahealth.com/fhir/r4
operations:
  - getCapabilityStatement
  - groupBulkExport
  - getBulkExportStatus
  - cancelBulkExport
generated: '2026-08-14'
method: generated
source: openapi/athenahealth-bulk-data-api-openapi.yml,
  openapi/_original/athenahealth-fhir-bulk-data-api-openapi.yml,
  https://api.platform.athenahealth.com/fhir/r4/metadata,
  https://api.platform.athenahealth.com/fhir/r4/.well-known/smart-configuration
---

# Export a population from athenahealth with FHIR Bulk Data

athenahealth's live CapabilityStatement declares conformance to the HL7 FHIR Bulk Data Access IG at
**both 1.0.1 and 2.0.0**, and the `Group` resource declares the `$export` operation. This is the
only place in the entire athenahealth contract where a runtime signal is actually published, so it
is the flow most worth automating.

## Authentication is different here

Bulk Data uses **SMART Backend Services**, not the interactive flow:

- Grant: `client_credentials`
- Client auth: `private_key_jwt` (the SMART configuration lists it, and the capabilities array
  includes `client-confidential-asymmetric`)
- Token endpoint: `https://api.platform.athenahealth.com/oauth2/v1/token`
- Scope: `system/*.read` (see `scopes/athenahealth-scopes.yml`)

There is no user, no launch context and no refresh token. If your client is holding a
`patient/` or `user/` scope, you are on the wrong flow.

## Steps

1. **`getCapabilityStatement`** (`GET /metadata`) — confirm `Group` still declares `$export` and
   that the Bulk Data IG versions have not moved.

2. **Kick off the export.** `groupBulkExport` — `GET /Group/{id}/$export`
   - Header `Prefer: respond-async` — **required**
   - Header `Accept: application/fhir+json`
   - Query `_type` to restrict resource types, `_since` for an incremental pull,
     `_outputFormat` for the NDJSON flavour
   - On success you get **202 Accepted** with a **`Content-Location`** header. That header is the
     job handle. Persist it; it is the only thing that lets you find the job again.

3. **Poll.** `getBulkExportStatus` — `GET /bulk/status/{jobid}`, where `jobid` comes from
   `Content-Location`. Poll on a conservative interval. When the job completes you get a manifest
   listing the NDJSON output files.

4. **Cancel when you must.** `cancelBulkExport` — `DELETE /bulk/status/{jobid}`. Do this on any
   abandoned job. A dangling export against a large Group is a real load event on a shared tenant.

## Rules that are not optional

- **`_since` is how you stay cheap.** A full Group export against a national-network practice is
  large. Use `_since` on every run after the first and keep the high-water mark.
- **`_type` is how you stay legal.** Export only the resource types your agreement covers. The
  server will happily give you everything your scope permits.
- **Cancel is not idempotent-protected either.** No athenahealth operation accepts an idempotency
  key. `cancelBulkExport` on an already-finished job is a normal failure, not an error to retry.
- **Do not poll faster than you need to.** No `Retry-After` and no `RateLimit-*` headers are
  declared anywhere on this API (`rate-limits/athenahealth-rate-limits.yml`), so the server will not
  tell you that you are polling too fast — it will just start refusing you.
- **Handle `OperationOutcome`.** Job-level errors come back as FHIR `OperationOutcome`, not RFC 9457
  Problem Details.
- **Test on preview first.** `https://api.preview.platform.athenahealth.com/fhir/r4` is a separate
  environment with its own credentials (`sandbox/athenahealth-sandbox.yml`).
