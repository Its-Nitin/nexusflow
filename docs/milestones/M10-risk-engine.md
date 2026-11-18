# M10 — Risk Service II: Deterministic Risk Engine

| | |
|---|---|
| **Release** | R3 Disruption Flow |
| **Duration** | 1 week (~22 h) |
| **Depends on** | M09 |
| **Unlocks** | M11 |
| **Tag** | `m10-risk-engine` |

---

## 1. Why this milestone exists

In an asynchronous system, a decision is made *somewhere, sometime*, from *some version* of the data. Weeks later someone asks: "Why did the system flag P-104 as critical?" If you can't reproduce the answer, you can't trust the system. This milestone builds the risk calculation as **pure, deterministic, versioned domain logic** that is fully separated from messaging and stores its inputs, so every assessment can be explained and replayed.

---

## 2. Learning objectives

- Keep complex business logic pure (no I/O) inside a hexagonal service, and test it exhaustively.
- Explain why determinism and input snapshots matter in event-driven systems (replays, audits, debugging).
- Version calculation logic, and explain how to handle a logic change for events that are replayed later.
- Separate commands from queries (a what-if simulation has no side effects).
- Design behind a stable internal contract (`DemandSource`) that is swapped later (M19).

---

## 3. Concepts

### 3.1 Functional core, imperative shell
The calculation is a pure function `assess(RiskInputs, today, CalculationParams) → RiskAssessment`. Messaging, persistence and time are the "shell." **Why:** pure functions are trivially testable, deterministic under replay, and safe to run in simulations.

### 3.2 Determinism under replay
If you replay `shipment.events` (M09 rebuild, M08 DLT replay), the same inputs must give the same assessment. That rules out `now()` inside logic, random numbers, unordered iteration when output order matters, and floating-point quantities or money.

### 3.3 Input snapshot + calculation version
Store the exact inputs (`input_snapshot jsonb`) and `calculationVersion` (e.g. `risk-calc@1.2.0`) with every assessment. **Why:** explainability ("show your working"), reproducibility, and safe evolution. When the logic changes, old assessments still say which logic produced them.

### 3.4 Idempotency of derived outputs
The same `ShipmentDelayed` event (duplicated or replayed) must not create two assessments. Key the assessment by `(sourceEventId, productId, warehouseId)` with a unique constraint, or derive a deterministic assessment ID from those fields (UUID v5).

### 3.5 CQS: simulation vs real assessment
`POST /simulations` runs the same pure function with hypothetical inputs and **publishes nothing and changes nothing** except storing the simulation run. Same logic, different shell.

---

## 4. Scope

**In:** `DemandSource` (deterministic), the risk calculator, severity and reason codes, persistence with snapshot + version, `RiskAssessmentCompleted` via the outbox, `RiskResolved` when a shipment arrives, simulations API, 20+ scenario tests, queries API.
**Out:** recovery options (M11), forecasts (M19).

---

## 5. Technical specification

### 5.1 Inputs (assembled from projections)
For delayed shipment S → PO P → destination warehouse W → for each line (product X, inbound qty Q):
- `onHand(X,W)`, `reserved(X,W)`, `safetyStock(X)`
- reservations of X at W with `requiredByDate` and qty
- `avgDailyDemand(X,W)` from `DemandSource`
- `plannedEta`, `revisedEta`, today (from `Clock`)

### 5.2 `DemandSource` v1 (deterministic)
Average daily ISSUE quantity over the last 28 days from `demand_history` (zero-filled for days without movements). If there are fewer than 7 days of history → use the product's configured `defaultDailyDemand`, and mark `demandSource=DEFAULT`.

### 5.3 Projected position
For each day `t` from today to `revisedEta + 7 days`:
```
position(t) = onHand
            + inbound arriving on or before t          (Q arrives at the ETA being evaluated)
            − Σ reservations with requiredByDate ≤ t    (committed demand)
            − avgDailyDemand × (t − today)               (uncommitted demand)
```
Compute it twice: with **plannedEta** (baseline) and with **revisedEta** (disrupted). The *impact of the delay* is the difference.

### 5.4 Outputs and rules
| Output | Rule |
|---|---|
| `stockoutDate` | first `t` where position(t) < 0 under the disrupted scenario |
| `affectedCommitments` | reservations whose `requiredByDate` falls where the disrupted position < 0 **and** the baseline position ≥ 0 (caused by the delay) |
| `shortfallQty` | −min(position(t)) over the window, if negative |
| `reasonCodes` | `INBOUND_DELAYED` always; `STOCKOUT_BEFORE_ARRIVAL`; `COMMITMENT_AT_RISK`; `BELOW_SAFETY_STOCK` |
| `severity` | `CRITICAL`: a commitment at risk within 3 days · `HIGH`: commitment at risk · `MEDIUM`: below safety stock but commitments met · `LOW`: no material impact |

(These rules are a starting point. Refine them, and bump `calculationVersion` whenever you do.)

### 5.5 Persistence
```
risk_assessment(id, tenant_id, source_event_id, shipment_id, product_id, warehouse_id, severity, reason_codes text[],
                stockout_date, shortfall_qty, affected_commitments jsonb, input_snapshot jsonb,
                calculation_version, demand_source, assessed_at, status[OPEN|RESOLVED|SUPERSEDED],
                UNIQUE(source_event_id, product_id, warehouse_id))
simulation_run(id, tenant_id, scenario jsonb, result jsonb, calculation_version, created_by, created_at)
```
A newer assessment for the same (shipment, product, warehouse) marks the older one `SUPERSEDED`. `ShipmentArrived` → `RESOLVED` + `RiskResolved` event.

### 5.6 API
`GET /api/v1/risks?severity=&status=&productId=` · `GET /api/v1/risks/{id}` (includes evidence) · `POST /api/v1/simulations` `{shipmentId, hypotheticalDelayDays}` · `GET /api/v1/simulations/{id}`

### 5.7 Event `RiskAssessmentCompleted` (payload)
`assessmentId, shipmentId, purchaseOrderId, productId, warehouseId, severity, reasonCodes, stockoutDate, shortfallQty, affectedCommitments[{reservationId, commitmentRef, requiredByDate, qty}], calculationVersion`

---

## 6. Tasks

| ID | Task | Est | Concept | P |
|---|---|---|---|---|
| M10-T01 | `DemandSource` port + deterministic adapter; consume ISSUE movements into `demand_history` | 2.5h | Stable internal contract | P0 |
| M10-T02 | Pure calculator (domain): position series, outputs, severity, reason codes | 5h | Functional core | P0 |
| M10-T03 | Scenario test suite: ≥ 20 parameterised cases (single line, multi-line PO, high/low demand, no reservations, commitment exactly on revisedEta, zero history, delay shortened (ETA earlier), very long delay, multiple warehouses…) | 4h | Deterministic verification | P0 |
| M10-T04 | Shell: handle `ShipmentDelayed` → inputs → assess → persist with snapshot → outbox event (one transaction) | 2.5h | Imperative shell | P0 |
| M10-T05 | Idempotent assessments (unique key / UUID v5) + supersede logic | 1.5h | Idempotent derived outputs | P0 |
| M10-T06 | `ShipmentArrived` → resolve + `RiskResolved` | 1h | Lifecycle of derived state | P0 |
| M10-T07 | Simulations API (no events) | 2h | CQS | P0 |
| M10-T08 | Queries API + authorisation + tenant | 1.5h | — | P0 |
| M10-T09 | Replay determinism test: rebuild + replay all shipment events → compare assessments | 1.5h | Determinism under replay | P0 |
| M10-T10 | ADR *Risk calculation model & versioning* | 0.5h | — | P0 |

---

## 7. Break tests

| # | Drill | Expected |
|---|---|---|
| B1 | Flagship scenario (5-day delay) | P-104 assessed HIGH/CRITICAL; affected commitments = the 12 Nov reservation (verify by hand calculation in the journal) |
| B2 | The same delay event delivered twice | One assessment |
| B3 | Full replay | Identical assessments (same IDs, severities, snapshots) |
| B4 | Change a rule, bump the version, replay one event | The new assessment carries the new version; the old one stays explainable |
| B5 | Simulation of a 10-day delay | Result returned; no event on `risk.events`; no assessment rows |
| B6 | Fixed `Clock` in tests across midnight/time zones | No off-by-one (all date maths in UTC) |

---

## 8. Pitfalls

- Calling `LocalDate.now()` inside the calculator (breaks determinism; ArchUnit should catch it).
- Using `double` for quantities or money.
- Iterating a `HashMap` to build `affectedCommitments`: nondeterministic order breaks snapshot comparison. Sort explicitly.
- Building an over-clever model. A clear, explainable formula beats an opaque accurate one, especially here.

## 9. Deliverables

Risk engine, scenario suite (`src/test/resources/scenarios/*.json`), simulations API, ADR, tag.

## 10. Self-check

1. Why must the calculator be pure for replay to be safe?
2. What is the input snapshot for, and what would you lose without it?
3. How do you keep the same event from creating two assessments?
4. How do you change the risk logic without making past assessments unexplainable?
5. Walk through the flagship calculation by hand.

## 11. References (when stuck)

Gary Bernhardt, *Functional Core, Imperative Shell* (talk) · *Domain Modeling Made Functional* (Wlaschin) · DDIA ch. 12 (derived data) · RFC 9562 (UUID v5)

## 12. Exit checklist

- [ ] 20+ scenarios green; B1–B6 pass
- [ ] Flagship hand calculation in the journal matches the output
- [ ] ADR written · tag `m10-risk-engine`
