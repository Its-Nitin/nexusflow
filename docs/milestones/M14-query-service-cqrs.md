# M14 — Control-Tower Query Service (CQRS Read Side)

| | |
|---|---|
| **Release** | R4 Control Tower MVP |
| **Duration** | 1 week (~21 h) |
| **Depends on** | M12 (all events exist), M13 |
| **Unlocks** | M15 |
| **Tag** | `m14-query-cqrs` |

---

## 1. Why this milestone exists

The dashboard needs one screen that combines shipments, POs, stock, risks, plans and audit, which is data owned by **six** services. If the UI calls all six on every page load (API composition), the page is as slow as the slowest service and fails when any one is down. The alternative is a **dedicated read side**: a service that subscribes to every event stream and maintains denormalised views shaped exactly for the screens. That is **CQRS**, and this milestone makes it concrete. It also makes eventual consistency *visible* to users, honestly.

---

## 2. Learning objectives

- Explain CQRS and when it is (and isn't) worth the complexity.
- Compare API composition vs pre-built read models on latency, availability and staleness.
- Build denormalised views from multiple event streams, handling ordering and duplicates.
- Treat read models as disposable: rebuild them from the log.
- Measure and expose staleness (`asOf`) to the UI.

---

## 3. Concepts

### 3.1 CQRS
**Command side:** the owning services validate and change state. **Query side:** separate models, optimised for reading, updated from events.
**Why:** reads and writes have different shapes and scaling needs; cross-service queries become local; reads survive owner outages.
**Costs:** eventual consistency, more infrastructure, rebuild tooling, and "read-your-own-writes" problems in the UI.
**Not worth it:** for a single service's simple CRUD screens.

### 3.2 API composition vs CQRS view
| | API composition (at query time) | CQRS view |
|---|---|---|
| Latency | max(calls) + merge | one local query |
| Availability | product of dependencies | independent |
| Freshness | current | staleness window |
| Complexity | low upfront; N+1 & pagination pain | consumers + rebuild |

### 3.3 Read-your-own-writes
The manager approves a plan (command → Recovery), then the UI reloads from Query, and the view may not have the event yet. Mitigations: return the new state from the command response and show it optimistically; show a "pending" marker until the view's `sourceVersion` ≥ the command's resulting version; push updates via SSE (M13).

### 3.4 Joining streams
A view row like `shipment_overview` needs facts from shipment, procurement and risk events, which arrive in **any order across topics**. Design views as upserts of independent column groups ("shipment part," "risk part"), each guarded by its own version, so the arrival order doesn't matter.

### 3.5 BFF
The Query service is effectively the dashboard's **Backend-for-Frontend** for reads: endpoints are shaped per screen, not per entity.

---

## 4. Scope

**In:** Query service consuming all event topics; views; read endpoints; `asOf`/`sourceVersion`; rebuild; staleness metric; gateway routes.
**Out:** writes (commands always go to the owning services).

---

## 5. Technical specification

### 5.1 Views
```
shipment_overview(shipment_id, tenant_id, po_id, supplier_name, carrier, mode, status, planned_eta, current_eta, delay_days,
                  open_risk_severity, open_risk_count, shipment_version, po_version, risk_version, as_of)
risk_board(assessment_id, tenant_id, product_sku, product_name, warehouse_code, severity, reason_codes, stockout_date,
           shortfall_qty, affected_commitments_count, plan_id, plan_status, as_of)
recovery_timeline(plan_id, tenant_id, seq, at, kind, actor, summary, details jsonb)   -- every plan/saga event in order
audit_log_view(id, tenant_id, at, actor, action, entity_type, entity_id, details)
product_stock_view(product_id, warehouse_id, tenant_id, sku, on_hand, reserved, available, safety_stock, as_of)
```
Reference names (supplier name, SKU) are copied from their events into small lookup tables (`supplier_lookup`, `product_lookup`), never fetched synchronously.

### 5.2 Endpoints (tenant-scoped, paginated)
`GET /api/v1/control-tower/summary` (counts by severity, delayed shipments, open plans) · `/shipments` · `/risks` · `/risks/{id}` (with evidence copied from the event) · `/recovery-plans/{id}/timeline` · `/audit` · `/stock?productId=`

### 5.3 Consumer group & ordering
One consumer group `query` per topic; per-view version guards (§3.4); inbox for dedupe.

---

## 6. Tasks

| ID | Task | Est | Concept | P |
|---|---|---|---|---|
| M14-T01 | Service from the template; consumers for 5 topics | 2h | — | P0 |
| M14-T02 | Lookup tables (supplier, product, warehouse) | 1.5h | Event-carried reference data | P0 |
| M14-T03 | `shipment_overview` with per-part version guards | 3h | Joining streams | P0 |
| M14-T04 | `risk_board`, `product_stock_view` | 3h | Denormalised views | P0 |
| M14-T05 | `recovery_timeline`, `audit_log_view` | 2.5h | Event history as UX | P0 |
| M14-T06 | Read endpoints + gateway routes + security | 3h | BFF | P0 |
| M14-T07 | Rebuild (reuse the M09 mechanism) + checksum compare | 2h | Disposable read models | P0 |
| M14-T08 | `as_of` + staleness metric + `X-Data-As-Of` response header | 1.5h | Visible eventual consistency | P0 |
| M14-T09 | Measure: API-composition prototype for `/summary` (calls 3 services) vs view; latency + behaviour with Risk down | 2h | Evidence | P1 |
| M14-T10 | ADR *CQRS read side for the control tower* | 0.5h | — | P0 |

---

## 7. Break tests

| # | Drill | Expected |
|---|---|---|
| B1 | Drop the Query DB and rebuild | Identical API responses (checksum) |
| B2 | Stop Query, run the flagship flow, start Query | It catches up; the final views are correct |
| B3 | Deliver risk events *before* shipment events for a new shipment | The row converges to the correct state |
| B4 | Stop Procurement, Inventory and Risk | Dashboard reads still work (stale but available, with `as_of`) |
| B5 | Approve a plan, then immediately GET the timeline | Shows pending until the event arrives (measure the gap) |

---

## 8. Pitfalls

- Calling owner services from Query "just for names" (reintroduces coupling). Use lookups.
- A single version column for a row fed by several streams (drops legitimate updates).
- Business logic creeping into the read side (computing risk here). Read models *display*; they don't *decide*.

## 9. Deliverables

Query service, views, endpoints, rebuild, measurements, ADR, tag.

## 10. Self-check

1. When is CQRS worth it, and when is it over-engineering?
2. How do you join three event streams that arrive in arbitrary order?
3. Three ways to handle read-your-own-writes in the UI.
4. Why must the read side never make business decisions?

## 11. References (when stuck)

Martin Fowler *CQRS* · *Microservices Patterns* ch. 7 (querying: API composition & CQRS) · Greg Young's CQRS documents · DDIA ch. 11–12 (derived data)

## 12. Exit checklist

- [ ] B1–B5 pass; B1–B3 automated
- [ ] All dashboard reads come from Query
- [ ] ADR written · tag `m14-query-cqrs`
