# M09 — Risk Service I: Using Data You Don't Own

| | |
|---|---|
| **Release** | R3 Disruption Flow |
| **Duration** | 1 week (~23 h) |
| **Depends on** | M08 |
| **Unlocks** | M10 |
| **Tag** | `m09-risk-data` |

---

## 1. Why this milestone exists

The central question of microservices: **how does a service make decisions using data owned by other services?** Risk needs POs (Procurement), stock and reservations (Inventory), and ETAs (Shipment). You'll build it two ways:
1. **Sync queries** (call the owners over REST when a delay arrives). Then break it with latency and outages, and protect it with resilience patterns.
2. **Event-carried state transfer** (keep local projections fed by events). Then show it survives the same outages.

Measuring both is what lets you defend the choice.

---

## 2. Learning objectives

- Explain temporal coupling and how sync call chains multiply failure probability.
- Implement timeouts, retries with jitter, circuit breakers and bulkheads, and explain what each protects against.
- Build projections from events, and rebuild them by replaying topics.
- Explain the cost of event-carried state: storage duplication, staleness, rebuild complexity.
- Explain retention vs compaction and how each affects replay.

---

## 3. Concepts

### 3.1 Temporal coupling and availability math
If Risk synchronously needs Procurement *and* Inventory, each 99.5% available, Risk's best case is 0.995 × 0.995 ≈ **99.0%**. Each added sync dependency lowers availability. Latency adds up too, and a slow dependency is often worse than a dead one (threads pile up).

### 3.2 Stability patterns (Resilience4j)
| Pattern | Protects against | NexusFlow setting (start) |
|---|---|---|
| **Timeout** | Waiting forever on a slow dependency | 500 ms connect+read |
| **Retry + jitter** | Transient blips | 2 retries, 100–400 ms jittered; **only idempotent GETs** |
| **Circuit breaker** | Hammering a failing dependency; fail fast | Open at 50% failures over 20 calls; half-open after 10 s |
| **Bulkhead** | One slow dependency exhausting all threads | Max 10 concurrent calls per dependency |
| **Fallback** | Total failure | Degraded result + flag (or reject explicitly) |
Order matters: Retry wraps CircuitBreaker wraps Bulkhead wraps TimeLimiter (decide and document).

### 3.3 Event-carried state transfer
Producers publish events with enough state (e.g. `StockLevelChanged{onHand, reserved}`). Consumers maintain **local projections** with just the fields they need, in *their own* schema.
**Gains:** autonomy (works when the owners are down), fast local reads, no fan-out calls.
**Costs:** duplicated data, **eventual consistency** (a staleness window), rebuild/replay machinery, and schema coupling via events.

### 3.4 Projections and replay
A projection is a derived, disposable read model. **Rebuild** = truncate + reset the consumer-group offset to the earliest + replay. That requires the topic to retain enough history:
- **Time retention** (e.g. 7 days) can't rebuild beyond the window.
- **Log compaction** keeps the latest event per key forever. Good for "current state" topics, but needs state-carrying events.
Decide per topic; document it in an ADR. (For learning: use long retention locally, and discuss compaction.)

### 3.5 When sync is still right
At **decision time with high stakes** (M11: Recovery checks current availability before proposing), fresh data beats autonomy, *with* resilience patterns and an explicit failure behaviour.

---

## 4. Scope

**In:** the sync-call implementation behind a feature flag, Toxiproxy, Resilience4j, projections from `procurement.events` + `inventory.events` + `shipment.events`, a rebuild command, a staleness metric, measurements.
**Out:** the actual risk calculation (M10). For now, the handler gathers the data and logs "inputs ready."

---

## 5. Technical specification

### 5.1 Sync path (`nexusflow.risk.data-source=sync`)
On `ShipmentDelayed`: GET the PO (Procurement) → for each line GET inventory for (product, destination warehouse) and GET reservations → build `RiskInputs`. The service uses a client-credentials token (M04).

### 5.2 Projection path (`data-source=projection`)
```
po_projection(po_id, tenant_id, supplier_id, status, expected_date, destination_warehouse_id, last_version)
po_line_projection(po_id, product_id, quantity)
stock_projection(warehouse_id, product_id, tenant_id, on_hand, reserved, safety_stock, last_version)
reservation_projection(reservation_id, warehouse_id, product_id, quantity, required_by_date, status, commitment_ref, last_version)
shipment_projection(shipment_id, po_id, planned_eta, current_eta, status, last_version)
demand_history(product_id, warehouse_id, day, quantity)      -- from StockMovement-derived events (used in M10)
```
Consumers use `IdempotentEventHandler` + the version guard (M07/M08).
Port: `RiskInputsProvider` with two adapters, `RestRiskInputsProvider` and `ProjectionRiskInputsProvider` (the hexagonal payoff from M00).

### 5.3 Rebuild
`POST /admin/projections/rebuild` (SERVICE role): pause listeners → truncate projections + processed/version tables for those groups → reset offsets to the earliest (AdminClient) → resume. Report the duration and row counts.

### 5.4 Staleness metric
`nexusflow_projection_lag_seconds{projection=…}` = now − `occurredAt` of the last applied event.

---

## 6. Tasks

| ID | Task | Est | Concept | P |
|---|---|---|---|---|
| M09-T01 | `RiskInputsProvider` port + REST adapter with client-credentials auth | 3h | API composition | P0 |
| M09-T02 | Toxiproxy in Compose (`chaos` profile) in front of Inventory | 1h | Fault injection | P0 |
| M09-T03 | Drill: 3 s latency on Inventory, no resilience; watch consumer stalls and `max.poll.interval` issues | 1.5h | Cascading failure | P0 |
| M09-T04 | Resilience4j: timeout, retry, circuit breaker, bulkhead + metrics | 3h | Stability patterns | P0 |
| M09-T05 | Projection consumers + tables (§5.2) | 5h | Event-carried state | P0 |
| M09-T06 | Projection adapter; switch the default | 1.5h | Adapter swap | P0 |
| M09-T07 | Rebuild command + test | 3h | Replayability | P0 |
| M09-T08 | Staleness metric | 1h | Measuring eventual consistency | P0 |
| M09-T09 | Measurements: latency to "inputs ready" for sync vs projection; behaviour with an owner down | 2h | Evidence-based decisions | P0 |
| M09-T10 | ADRs: *Event-carried state vs sync queries for Risk*, *Topic retention vs compaction* | 1h | — | P0 |

---

## 7. Break tests

| # | Drill | Expected |
|---|---|---|
| B1 | Sync mode, Inventory stopped, delay arrives | Without the breaker: retries/timeouts pile up. With the breaker: it opens, fails fast, the event goes to retry → DLT (document the behaviour) |
| B2 | Sync mode, 3 s latency, 20 delays at once | Without a bulkhead: thread exhaustion. With one: bounded concurrency, the rest fail fast |
| B3 | Projection mode, Inventory **and** Procurement stopped | Inputs still ready from projections |
| B4 | Rebuild projections | Row-for-row identical to before (compare with a checksum query) |
| B5 | Produce 1000 stock events quickly | Staleness metric spikes, then returns to ~0 |
| B6 | Replay with duplicates and stale versions | Projections are unchanged |

---

## 8. Pitfalls

- Retrying non-idempotent calls.
- A circuit breaker without a timeout (slow calls never count as failures).
- Projection tables that copy the owner's whole schema. Copy only what you need.
- Forgetting tenant scoping in projections (they hold multi-tenant data too).
- A rebuild while listeners are still running produces a mixed state.

## 9. Deliverables

Two `RiskInputsProvider` adapters, Resilience4j config, projections + rebuild, measurements in `docs/benchmarks/risk-data-source.md`, 2 ADRs, tag.

## 10. Self-check

1. Compute the availability of a service with 3 sync dependencies at 99.9% each.
2. What does a bulkhead protect against that a circuit breaker doesn't?
3. What are the three costs of event-carried state transfer?
4. Retention vs compaction for rebuilding projections: trade-offs?
5. When is a sync call the right choice even in an event-driven system?

## 11. References (when stuck)

*Release It!* 2e ch. 4–5 (stability antipatterns and patterns) · Resilience4j docs · Martin Fowler, *What do you mean by "Event-Driven"?* · DDIA ch. 11 (derived data, change streams) · Kafka docs *Log compaction*

## 12. Exit checklist

- [ ] B1–B6 done; B3, B4, B6 automated
- [ ] Measurements documented
- [ ] ADRs written · tag `m09-risk-data`
