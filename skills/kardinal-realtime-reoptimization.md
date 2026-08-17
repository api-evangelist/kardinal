---
name: Re-optimize a running Kardinal plan after a field disruption
description: Absorb a mid-shift change — a new urgent order, a cancellation, an absent customer, a broken-down vehicle — into a plan that is already running, without rebuilding it from scratch. Use when reacting to live operational events rather than building a plan from nothing.
api: openapi/kardinal-aro-openapi-original.yml
generated: '2026-08-17'
method: generated
source: Grounded in operationIds verified against openapi/kardinal-aro-openapi-original.yml and https://developers.kardinal.ai/guides/real-time-reoptimization
operations:
  - getPlan
  - putPlanOrder
  - deletePlanOrder
  - putPlanResource
  - putPlanResourceState
  - putForbidResourceStop
  - putPlanRunning
  - putPlanMode
  - checkPlanManualMode
  - getPlanStatus
  - getPlanSolution
---

# Re-optimize a running plan

This is what "Always-on" means at Kardinal: a plan is not a batch job you rerun. It keeps optimizing, and you feed changes into it while it runs.

**The core rule: apply the smallest change you can.** Do not re-PUT the whole plan to add one order. Kardinal exposes per-entity endpoints precisely so a disruption costs one call, not a full resubmit.

## Read current state first (`getPlan`, `getPlanStatus`)

`GET /agencies/{agencyId}/plans/{planId}` for the plan, `GET .../status` for the stage it is in (`waitingRoom`, `creation`, `optimization`, `waitingTraffic`). Know whether the engine is actually running before you change anything under it.

## Handle each disruption with the narrowest call

### A new urgent order arrives — `putPlanOrder`

`PUT /agencies/{agencyId}/plans/{planId}/orders/{orderId}`

Same upsert semantics as everything else: your `orderId`, create-or-update, `201` on first write and `200` on update. The engine folds it into the running solution.

### An order is cancelled — `deletePlanOrder`

`DELETE /agencies/{agencyId}/plans/{planId}/orders/{orderId}` → `204`.

### A vehicle changes — `putPlanResource` / `deletePlanResource`

`PUT /agencies/{agencyId}/plans/{planId}/resources/{resourceId}` to change a working time window, capacity or profile mid-shift. `DELETE` when a vehicle drops out entirely.

### A driver has already done some of the route — `putPlanResourceState`

`PUT /agencies/{agencyId}/plans/{planId}/resources/{resourceId}/state`

This is the one that keeps re-optimization honest. Push where the vehicle actually is and what it has actually completed, so the engine re-plans the *remainder* instead of re-planning stops that are already served. `getPlanResourceState` reads it back.

### A specific vehicle must not serve a specific stop — `putForbidResourceStop`

`PUT /agencies/{agencyId}/plans/{planId}/resources/{resourceId}/forbid/{stopId}`

The targeted tool for "the customer refused this driver", "this vehicle cannot access this site", or an absent-customer retry that must go to someone else. Use this rather than deleting and recreating orders.

### Pause or resume the engine — `putPlanRunning`

`PUT /agencies/{agencyId}/plans/{planId}/running` stops or restarts optimization. Use it to freeze a solution before dispatch so the routes handed to drivers do not move underneath them.

### Force a human decision — `putPlanMode` + `checkPlanManualMode`

`PUT /agencies/{agencyId}/plans/{planId}/mode` switches the plan from `standard` (solution built entirely by the OR algorithms) to `manual`, where a user can force assignments **that may violate constraints**. `GET .../manual/check` validates the manual state.

Treat `manual` as an escalation, not a default. It deliberately suspends the guarantees the engine otherwise gives you, so surface to the operator that constraints may now be violated.

## Then poll and re-read

Changes restart optimization. Poll `getPlanStatus` again and re-read `getPlanSolution` — **do not assume the solution you already hold is still current.** There is no webhook or push notification; polling is the sanctioned mechanism.

## Retry semantics

Every operation above except `deletePlanOrder`/`deletePlanResource` is a `PUT` on a client-supplied id, so it is safe to replay after an ambiguous network failure — the state converges. But there is no idempotency-key replay cache: the write genuinely re-executes and optimization restarts. Retry once on a transport error; do not retry in a tight loop against a running plan.

## Errors specific to this flow

- `PRECONDITION_FAILED` (400) — you acted on something not in the right state, e.g. an already-deleted object.
- `INVALID_ID_REFERENCE` (400) — you referenced a `resourceId` or `stopId` that does not exist in this plan.
- `NOT_FOUND` (404) — the plan, order or resource does not exist *or* is not visible to you.
- `NOT_IMPLEMENTED` (400, **not** 501) — the behavior is recognized but not available yet.

And the standing rule: **an infeasible outcome is not an error.** After any re-optimization, check `unaffectedStopIds` in the solution. A `200` with the urgent order sitting in `unaffectedStopIds` means it could not be placed — report that, do not report success.
