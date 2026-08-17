---
name: Create a Kardinal plan and retrieve its optimized solution
description: Submit vehicles, orders and constraints to the Kardinal ARO API as a plan, poll it to completion, and read back the optimized tours. Use when an agent must turn a set of stops and vehicles into a routed, sequenced delivery plan.
api: openapi/kardinal-aro-openapi-original.yml
generated: '2026-08-17'
method: generated
source: Grounded in operationIds verified against openapi/kardinal-aro-openapi-original.yml and the published guides at https://developers.kardinal.ai/
operations:
  - postLogin
  - putPlan
  - getPlanStatus
  - getPlanSolution
  - getPlanSolutionObjectives
---

# Create a plan and retrieve its solution

The Kardinal ARO API has exactly two moving parts: you **PUT a plan**, and you **GET its solution**. Everything else is lifecycle around those two calls.

## Before you start

- **Base URL** is per customer environment: `https://<env>.kardinal.ai/api/v2`. Sandbox and production are different hosts with different credentials and different data. Never assume a single host.
- **You must geocode first.** Kardinal does not resolve addresses. Every stop needs `position: {lon, lat}` in decimal degrees before you call.
- **Ids are yours.** `planId`, `resourceId`, `orderId` and `stopId` are all client-supplied. Choose them deterministically from your own system's keys — that is what makes retries safe.

## Step 1 — Authenticate (`postLogin`)

`POST /login` with `{"username": ..., "password": ...}`. You get back an `access_token` (valid **one hour**) and a `refresh_token`.

Send `Authorization: Bearer <access_token>` on every subsequent call. Refresh proactively on a ~50-minute timer via `postLoginRefresh` rather than waiting for a 401 — see the authenticate-and-refresh skill.

If the account has MFA configured, `postLogin` returns an OTP token instead and you must complete `postLoginOTP`.

## Step 2 — Submit the plan (`putPlan`)

`PUT /agencies/{agencyId}/plans/{planId}`.

This single call is **both create and update**. There is no separate create endpoint and no separate "start optimization" call — optimization begins the moment the plan is accepted.

The body carries:

- `id` and `agencyId` — must match the path.
- `version` — increment it when you resubmit a changed plan.
- `resources[]` — the vehicles. Each needs an `id`, a `vehicleProfile` (`fly` for a crow-fly smoke test; `car` or `truck` for anything real), and a `workingTimeWindow` with ISO 8601 `begin`/`end`. Add `capacities` and skills when the operation has them.
- `orders[]` — the work. Each has an `id` and `stops[]`, and each stop has a `position`, a `kind` (`pickup`/`delivery`), and an `operationDuration` as an ISO 8601 duration (`PT5M30S`).
- `maxOptimizationDuration` — your compute budget, ISO 8601 (`PT1M`). There is no platform cap; the engine also stops early once it stops improving.

Returns **201** on first write, **200** on update.

You may also submit the plan as a file upload (`-F "file=@plan.json"`) or as an XLSX whose column headers match the JSON field names.

### Retry rule

Replaying the same `PUT` with the same `planId` is state-safe: it converges on the same plan rather than creating a duplicate. It is **not** compute-free — each accepted version restarts optimization. Retry on network failure; do not retry in a loop.

## Step 3 — Poll to completion (`getPlanStatus`)

`GET /agencies/{agencyId}/plans/{planId}/status`.

**Kardinal publishes no webhooks.** The spec says so explicitly: poll this field instead of relying on a push mechanism. The status reports which stage the plan is in:

| Stage | Meaning |
|---|---|
| `waitingRoom` | Your agency is at its cap of simultaneously running plans; this one is queued and its `maxOptimizationDuration` clock has not started mattering yet |
| `creation` | The plan is being built |
| `optimization` | The engine is actively improving the solution |
| `waitingTraffic` | Waiting on traffic data |

A small test plan settles in seconds. Size your poll interval against your `maxOptimizationDuration`, not against a fixed guess. `fetchLastPlanState` and `fetchLastNPlanStates` give you the state history if you need to show progress.

## Step 4 — Read the solution (`getPlanSolution`)

`GET /agencies/{agencyId}/plans/{planId}/solution`.

The payload is wrapped in an `item` field:

- `tours[]` — one per resource that was used, each with `resourceId`, `distanceInKm`, `workingDuration`, and `wayPoints[]` carrying `stopId`, `arrivalTime` and `stopKind` in visit order.
- `unaffectedStopIds[]` — stops the engine could not place.
- `unusedResourceIds[]` — vehicles it did not need.

Call `getPlanSolutionObjectives` for the objective breakdown when you need to explain *why* a solution looks the way it does.

## The mistake to avoid

**An infeasible plan is not an error.** If the engine cannot satisfy every constraint you still get a normal `200`/`201`. The failure shows up as populated `unaffectedStopIds` and constraint violations inside the solution — never as an `Error` object. Always inspect `unaffectedStopIds` before reporting success to a user.

## Error handling

Errors come back as `{"errors": [{"code", "message", "properties"}]}` — an array, because one `400` can report several invalid fields at once. **Branch on `code`, never on the HTTP status**: seven distinct codes all return `400`.

The ones this flow produces:

- `INVALID_INPUT` — body is not valid JSON or does not match the schema.
- `ID_NOT_UNIQUE` — you repeated an `id` inside `resources` or `orders`.
- `INVALID_ID_REFERENCE` — something points at an `id` that does not exist.
- `INVALID_VALUE` — a field failed a format/range/allowed-value rule.
- `NOT_AUTHENTICATED` (401) — token missing, malformed, or past its hour.
- `NOT_FOUND` (404) — the plan does not exist *or* is not visible to you; the API deliberately does not distinguish these.

Full table: `errors/kardinal-error-codes.yml`.
