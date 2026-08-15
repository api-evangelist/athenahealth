---
name: athenahealth-fhir-subscriptions-webhook
description: Register, verify and tear down a FHIR Subscription against athenahealth so an agent
  gets pushed rest-hook notifications instead of polling — including the two-second acknowledgement
  deadline and the id-only payload that forces a follow-up read.
api: athenahealth FHIR Subscriptions API
base: https://api.platform.athenahealth.com/fhir/r4
operations:
  - searchSubscriptions
  - createSubscription
  - readSubscription
  - getSubscriptionStatus
  - deleteSubscription
generated: '2026-08-14'
method: generated
source: openapi/athenahealth-subscription-api-openapi.yml,
  asyncapi/athenahealth-fhir-subscriptions-asyncapi.yml,
  https://github.com/athenahealth/aone-fhir-subscriptions,
  https://api.platform.athenahealth.com/fhir/r4/.well-known/smart-configuration
---

# Subscribe to athenaOne events with FHIR Subscriptions

athenahealth's event surface is FHIR Subscriptions on the **rest-hook** channel, using the R5
Backport on R4. athenahealth publishes a reference receiver in Java at
`github.com/athenahealth/aone-fhir-subscriptions` — read it before you build one. athenahealth
labels the framework **Alpha**; treat the contract as movable.

## Authentication

`client_credentials` with SMART Backend Services (`private_key_jwt`). Scopes:
`system/Subscription.read` and `system/Subscription.write`
(`scopes/athenahealth-scopes.yml`).

## Steps

1. **Stand up the receiver first.** It must be publicly reachable over HTTPS and it must return a
   **2xx within 2 seconds**. See `asyncapi/athenahealth-fhir-subscriptions-asyncapi.yml`. Two
   seconds means: accept, enqueue, return. Do not do any work inline — not a database write, not a
   FHIR read, nothing.

2. **Check what is already registered.** `searchSubscriptions` (`GET /Subscription`). Subscriptions
   are tenant-scoped and duplicates are easy to create, because — as everywhere on this API — there
   is no idempotency key.

3. **Register.** `createSubscription` (`POST /Subscription`) with the rest-hook channel pointed at
   your endpoint and the criteria for the resource/event you want.

4. **Verify it took.** `readSubscription` (`GET /Subscription/{id}`) and `getSubscriptionStatus`
   (`GET /Subscription/{id}/$status`). `$status` is the operation that tells you whether athenahealth
   believes your endpoint is healthy — check it after deployments, not only at registration.

5. **Handle the notification.** Payload is **id-only**. You receive a FHIR Bundle naming what
   changed; you do **not** receive the clinical content. Fetch the actual resource with the
   read/search operations in `athenahealth-fhir-patient-summary`, under your own scopes.

6. **Tear down.** `deleteSubscription` (`DELETE /Subscription/{id}`) whenever a receiver is
   decommissioned. A dead rest-hook target that keeps failing is athenahealth's problem to retry and
   yours to have caused.

## Rules that are not optional

- **Two seconds is the contract, not a guideline.** Acknowledge, then process asynchronously.
- **The payload is not the data.** id-only is deliberate — it keeps PHI out of the webhook channel.
  Do not log the notification body and then treat that log as a clinical record; it is not one.
- **Search before you create.** No idempotency key means a retried `createSubscription` produces a
  second subscription and you will receive every event twice.
- **Alpha means versioned churn.** athenahealth labels this framework Alpha in its own repository.
  Pin nothing, monitor `getSubscriptionStatus`, and watch
  `https://docs.athenahealth.com/api/resources/release-notes-and-change-logs` — noting that page is
  JS-rendered and cannot be monitored by machine (`changelog/athenahealth-changelog.yml`).
- **Errors are `OperationOutcome`.** Not Problem Details.
