---
name: Kardinal
description: Use when building route optimization integrations, modeling logistics problems from client data, creating delivery plans with constraints, or retrieving optimized solutions. Agents should reach for this skill when working with vehicle routing, order assignment, capacity planning, time windows, or real-time route reoptimization.
metadata:
    mintlify-proj: kardinal
    version: "1.0"
---

# Kardinal Skill

## Product summary

Kardinal is an Always-on Route Optimization (ARO) API that solves vehicle routing problems. You submit a **plan** (resources/vehicles, orders/stops, constraints) via REST API, and the engine continuously optimizes it, returning a **solution** with assigned tours, arrival times, and unserviceable stops. The API uses JWT authentication (username/password exchanged for access_token), operates on a per-environment basis (sandbox and production are separate), and works through a PUT/GET pattern: PUT to create/update plans, GET to retrieve solutions and status.

**Key endpoints:**
- `PUT /agencies/{agencyId}/plans/{planId}` — create or update a plan
- `GET /agencies/{agencyId}/plans/{planId}/solution` — retrieve the optimized solution
- `GET /agencies/{agencyId}/plans/{planId}` — fetch plan details and status
- `PUT /agencies/{agencyId}/plans/{planId}/resources/{resourceId}` — add/update a vehicle
- `PUT /agencies/{agencyId}/plans/{planId}/orders/{orderId}` — add/update an order

**Primary docs:** https://developers.kardinal.ai

## When to use

Reach for this skill when:
- Building a logistics integration from client data (spreadsheets, CRM exports, verbal descriptions)
- Modeling delivery or pickup operations with multiple vehicles, orders, and constraints
- Handling capacity constraints (weight, volume, custom units), time windows, or driver skills
- Implementing real-time reoptimization (disruptions, cancellations, new orders mid-shift)
- Retrieving optimized routes with arrival times, distances, and cost metrics
- Deciding between hard constraints (must-obey) and soft constraints (prefer-but-flexible)
- Calibrating optimization duration for problem size (small plans vs. 100+ stops/10+ vehicles)

Do not use this skill for authentication-only tasks, dashboard operations, or account management — those are outside the API's scope.

## Quick reference

### Authentication
```bash
curl -X POST "https://<env>.kardinal.ai/api/v2/login" \
  -H "Content-Type: application/json" \
  -d '{"username":"<user>","password":"<pass>"}'
```
Returns `access_token` (valid 1 hour) and `refresh_token` (valid 30 days). Use `Authorization: Bearer <access_token>` on all subsequent requests.

### Minimal plan structure
```json
{
  "id": "plan-id",
  "agencyId": "<your-agency>",
  "version": 1,
  "resources": [
    {
      "id": "vehicle-1",
      "vehicleProfile": { "type": "car", "withTraffic": true },
      "workingTimeWindow": { "begin": "2024-01-15T08:00:00Z", "end": "2024-01-15T18:00:00Z" },
      "capacities": { "weight": 1000 }
    }
  ],
  "orders": [
    {
      "id": "order-1",
      "stops": [
        {
          "type": "single",
          "id": "stop-1",
          "position": { "lat": 48.85, "lon": 2.35 },
          "kind": "delivery",
          "operationDuration": "PT10M",
          "capacities": { "weight": 100 }
        }
      ]
    }
  ],
  "maxOptimizationDuration": "PT1H",
  "objectives": [
    "maximizeMandatoryStops",
    "minimizeDelay",
    "minimizeCosts",
    "minimizeResources",
    "minimizeWorkingDuration",
    "minimizeDistance"
  ]
}
```

### Vehicle profiles
| Type | Use case | Key fields |
|------|----------|-----------|
| `fly` | Smoke tests, crow-fly distance only | `kmph` |
| `car` | Standard delivery vehicles | `withTraffic`, `avoidTollRoad` |
| `truck` | Heavy vehicles, hazmat | `grossWeight`, `height`, `width`, `length`, `shippedHazardousGoods` |
| `pedestrian`, `bicycle`, `scooter`, `motorbike` | Alternative modes | Mode-specific avoidance flags |

### Stop kinds and capacity direction
| Kind | Capacity effect | Use for |
|------|-----------------|---------|
| `pickup` | Adds to load | Collections, reloads |
| `delivery` | Deducts from load | Drop-offs |
| `acknowledgement` | No change | Interventions without cargo |

**Critical:** The real-world direction is determined by the *sign* of `capacities`, not by `kind`. A `pickup` with negative capacities is a drop-off; a `delivery` with positive capacities is a collection.

### Time windows
- **`authorizedTimeWindows`** — hard constraint; stop is unserviceable if missed
- **`preferredTimeWindows`** — soft constraint; late arrival costs delay but stop still gets served
- Default to `preferredTimeWindows` for contractual windows unless the client explicitly states a site is closed outside the window

### Optimization duration starting points
| Problem size | Starting `maxOptimizationDuration` |
|--------------|-----------------------------------|
| ≤10 stops, no advanced constraints | `PT10M` |
| 50–100 stops, 5–15 resources | `PT1H` to `PT4H` |
| 100+ stops or advanced constraints | `PT4H` or longer, then recalibrate empirically |

Empirical calibration: start with a generous ceiling (e.g., `PT1H`), poll the solution's objective values, and note when improvements stop. Use that observed convergence time + margin as your steady-state duration.

## Decision guidance

### When to use hard vs. soft time windows

| Scenario | Use | Reasoning |
|----------|-----|-----------|
| Site is physically closed outside window (gate locked, staff absent) | `authorizedTimeWindows` | Stop is genuinely unserviceable |
| Contractual SLA or customer preference, but late delivery is still acceptable | `preferredTimeWindows` | Stop gets served; delay is tracked as a cost |
| Data doesn't specify; client hasn't confirmed | `preferredTimeWindows` | Safer default; flag as open question |

### When to use single-loop vs. multi-trip (reload)

| Condition | Pattern | Action |
|-----------|---------|--------|
| Total daily demand ≤ total fleet capacity in one loop | Single-loop | Model each delivery as one `delivery` stop |
| Total demand > fleet capacity; client confirms depot reload | Multi-trip | Pair each delivery with a depot `pickup` in the same order |
| Total demand > fleet capacity; client hasn't confirmed | Neither yet | Surface as open question; state numbers (demand vs. capacity) |
| Individual order demand > largest vehicle capacity | Split order | Decompose into multiple pickup+delivery legs before submitting |

### When to use polling vs. static vs. iterative optimization

| Pattern | When to use | Trade-off |
|---------|------------|----------|
| **Polling** (recommended) | Real-time integrations; you want to decide when solution is "good enough" | Poll status/objectives; decide yourself when to stop |
| **Static** | Batch jobs; you can wait for full convergence | Set long `maxOptimizationDuration`; wait for it to elapse |
| **Iterative** | Manual restarts needed; short initial duration | Use short duration, restart with `PUT /running` if needed |

## Workflow

### 1. Understand the client's operation
- Identify the fleet (vehicles, capacities, working hours, skills)
- Identify the orders (stops, quantities, time windows, precedence)
- Identify constraints (capacity shortfalls, reload patterns, hard vs. soft windows)
- **Do not guess:** if the data is ambiguous, list open questions explicitly (see Common gotchas)

### 2. Check capacity feasibility
- Sum total demand across all stops for the day
- Compare against total fleet capacity in one loop
- If demand > capacity: confirm with client whether reload is intended, or if they need more vehicles/days
- For each order: verify its demand fits in at least one vehicle (not just the fleet total)

### 3. Geocode all positions
- Every `position` (resource departure/arrival, every stop) must be a real `{lat, lon}` pair
- Use a geocoding service (Google Maps, Mapbox, BAN for French addresses) beforehand
- Do not submit placeholder coordinates (town centroids, jittered points)
- If you lack real positions, flag this explicitly; do not ship a payload with fabricated coordinates

### 4. Build the plan payload
- Set `id`, `agencyId`, `version` (start at 1)
- Add `resources` with `id`, `vehicleProfile`, `workingTimeWindow`, `capacities`
- Add `orders` with `id`, `stops` (each with `id`, `position`, `kind`, `operationDuration`, `capacities`)
- Set `maxOptimizationDuration` based on problem size (see Quick reference)
- Set `objectives` list in priority order (default sequence covers most cases)
- Use `tz` if datetimes lack explicit UTC offsets
- Set `type: "single"` explicitly on every stop (even though it defaults)

### 5. Submit and poll
```bash
curl -X PUT "https://<env>.kardinal.ai/api/v2/agencies/<agency>/plans/<plan-id>" \
  -H "Authorization: Bearer <token>" \
  -H "Content-Type: application/json" \
  -d @plan.json
```
Optimization starts automatically. Poll the solution:
```bash
curl "https://<env>.kardinal.ai/api/v2/agencies/<agency>/plans/<plan-id>/solution" \
  -H "Authorization: Bearer <token>"
```
Check `unaffectedStopIds` (stops that couldn't be served) and `objectives[].value` (solution quality).

### 6. Handle disruptions (real-time reoptimization)
- Resource running late: `PUT` the resource with updated `workingTimeWindow`
- Order cancelled: `DELETE /orders/{orderId}`
- New urgent order: `PUT /orders/{newOrderId}` (creates it in the existing plan)
- Resource can't serve a specific stop: `PUT /resources/{resourceId}/forbid/{stopId}` (forbids the entire order)
- Batch multiple changes into one `PUT` to avoid repeated re-optimization

### 7. Deliver reports
- **Modeling report:** every non-trivial decision (unit choices, field mappings, hard vs. soft trade-offs, assumptions)
- **Data-quality report:** missing values, inconsistent codes, contradictory rows, values outside plausible range
- **Field-status table:** for every optional/ambiguous field, state whether it was set from data, defaulted, or left unset
- **Verification table:** for every validation or check claim, cite the exact file/line/function that implements it

## Common gotchas

- **Token expiry:** `access_token` is valid 1 hour only. Refresh proactively via `refreshToken` before it expires, or re-login.
- **Wrong environment:** sandbox and production are separate. Confirm which environment your token was issued against; a sandbox token fails against production.
- **Positions not geocoded:** submitting addresses instead of `{lat, lon}` pairs produces a valid-looking plan that routes to the wrong place. Always geocode beforehand.
- **Capacity direction confusion:** `kind: "pickup"` with positive capacities is a collection, not a drop-off. The sign of `capacities` determines direction, not `kind`.
- **Contractual windows as hard constraints:** defaulting every SLA window to `authorizedTimeWindows` silently makes stops unserviceable if late. Use `preferredTimeWindows` unless the client explicitly states a site is closed.
- **Skipping capacity checks:** a plan with total demand > fleet capacity will silently drop stops into `unaffectedStopIds`. Check feasibility before submitting.
- **Undersized `maxOptimizationDuration`:** a 10-minute duration for a 100-stop plan produces a poor solution. Calibrate empirically; start generous and observe convergence.
- **Forgetting `type: "single"` on stops:** some validators re-parse the payload and reject stops missing the explicit type tag, even though the API defaults it.
- **Silently defaulting ambiguous fields:** if vehicle type, break rules, or objective order aren't derivable from data, list them as open questions, not silent assumptions.
- **Updating a plan while old version is optimizing:** the engine stops work on the stale version. This is intentional, but means rapid updates can prevent convergence.
- **Not setting `withTraffic: true`:** travel times computed without traffic are systematically optimistic, eroding on-time performance against time windows. Confirm billing with the account team, but default to `true`.

## Verification checklist

Before submitting work:

- [ ] All positions are real geocoded `{lat, lon}` pairs (not addresses, not centroids)
- [ ] Capacity feasibility checked: total demand vs. fleet capacity, and per-order vs. largest vehicle
- [ ] Time windows classified: each window is either `authorizedTimeWindows` (hard) or `preferredTimeWindows` (soft), with reasoning
- [ ] Vehicle profiles set correctly: `car` vs. `truck` based on actual vehicle type or dimensions, not guesses
- [ ] `type: "single"` set explicitly on every stop
- [ ] `operationDuration` computed from business data (not left as placeholder)
- [ ] `maxOptimizationDuration` sized for problem size (not copied from a different case)
- [ ] Objectives list ordered by business priority (not copied unchanged from default)
- [ ] All optional/ambiguous fields have explicit status (set, defaulted, or not set) in a table
- [ ] Open questions listed separately: every ambiguity requiring client confirmation
- [ ] Modeling report delivered: decisions, assumptions, field-status table
- [ ] Data-quality report delivered: missing values, inconsistencies, out-of-range values
- [ ] Verification claims backed by file/line references (not prose alone)
- [ ] No fabricated coordinates, even if flagged as provisional

## Resources

**Comprehensive page listing:** https://developers.kardinal.ai/llms.txt

**Critical pages:**
- [First API call](https://developers.kardinal.ai/getting-started/first-api-call) — minimal working example and authentication
- [Start here if you're an AI agent](https://developers.kardinal.ai/getting-started/agent-modeling-checklist) — mandatory checklist for modeling real client data
- [How the optimization engine works](https://developers.kardinal.ai/concepts/how-the-optimization-engine-works) — objectives, polling patterns, continuous optimization
- [Data model](https://developers.kardinal.ai/reference/data-model) — complete field dictionary for resources, orders, stops, constraints
- [Hard vs soft constraints](https://developers.kardinal.ai/concepts/hard-vs-soft-constraints) — when constraints bend and when they don't
- [Multi-trip tours](https://developers.kardinal.ai/guides/multi-trip-tours) — modeling depot reloads
- [Handling large volumes](https://developers.kardinal.ai/guides/handling-large-volumes) — sizing `maxOptimizationDuration` for large fleets
- [Real-time reoptimization](https://developers.kardinal.ai/guides/real-time-reoptimization) — disruption handling, mid-shift updates
- [Error codes](https://developers.kardinal.ai/reference/error-codes) — business error codes beyond HTTP status

---

> For additional documentation and navigation, see: https://developers.kardinal.ai/llms.txt