# M03 — Procurement Service (the Reference Service)

| | |
|---|---|
| **Release** | R0 Foundation |
| **Duration** | 1 week (~22 h) |
| **Depends on** | M02 |
| **Unlocks** | M04, M05, M06 (they copy this service's shape) |
| **Tag** | `m03-procurement` |

---

## 1. Why this milestone exists

The first service sets the pattern every other service copies. If it is sloppy (inconsistent errors, no health probes, business rules in controllers), the sloppiness multiplies by eight. Procurement is the **reference implementation** of the M00 standards: contract-first API, hexagonal layout, explicit state machine, real-database tests, correct health semantics.

---

## 2. Learning objectives

- Build a service contract-first and explain why the contract is the *only* allowed coupling between services.
- Model an aggregate with an explicit state machine, ready for later events and sagas.
- Explain liveness vs readiness and why getting them wrong causes cascading restarts.
- Implement graceful shutdown and explain why it matters for rolling deploys and for Kafka consumers later.

---

## 3. Concepts

### 3.1 Contract-first APIs
**What:** write the OpenAPI spec first, then generate server interfaces from it.
**Why:** in microservices, other services and the UI depend on your API, not your code. Contract-first makes the API a deliberate design artefact, enables mocking and contract tests (M17), and prevents accidental breaking changes.
**Trade-off:** generator configuration friction. Worth it.

### 3.2 Explicit state machines for aggregates
**What:** allowed transitions are defined in one place (`CONFIRMED → CANCELLED` allowed; `RECEIVED → DRAFT` forbidden).
**Why (distributed angle):** later, events (`PurchaseOrderConfirmed`) are emitted *on transitions*, and sagas (M12) issue commands that must be rejected if the state doesn't allow them. Ad-hoc `setStatus()` calls make that impossible to reason about.

### 3.3 Liveness vs readiness
| Probe | Question | If it fails | Should check |
|---|---|---|---|
| Liveness | "Is the process broken beyond self-repair?" | Restart | Internal state only (deadlock) |
| Readiness | "Can I serve traffic right now?" | Stop routing traffic | DB connectivity, warm-up |
**Why it matters:** if liveness checks the database and the database blips, *every* instance restarts at once. That's a self-inflicted outage.

### 3.4 Graceful shutdown
On `SIGTERM`: stop accepting new requests → finish in-flight ones → close resources. Later (M06+), Kafka consumers must commit offsets and leave the group cleanly to avoid duplicate processing and slow rebalances.

### 3.5 Consistent error contract (RFC 9457)
Clients (UI, other services, the AI assistant) handle errors uniformly only if every service speaks the same error format, which `platform-web` provides.

---

## 4. Scope

**In:** Supplier, SupplierProduct (catalogue: product, unit price, lead time), PurchaseOrder + lines, the state machine, REST API, validation, Problem Details, correlation IDs, health, graceful shutdown, integration tests, seed data, service README.
**Out:** security (M04), events (M06–M07).

---

## 5. Technical specification

### 5.1 Domain model
```
Supplier(id, tenantId, name, status[ACTIVE|SUSPENDED], defaultLeadTimeDays, reliabilityScore)
SupplierProduct(id, supplierId, productId, unitPrice: Money, leadTimeDays, minOrderQty)
PurchaseOrder(id, tenantId, supplierId, status, expectedDeliveryDate, destinationWarehouseId, version, lines[])
PurchaseOrderLine(id, productId, quantity, unitPrice: Money)
```
`productId` and `warehouseId` are **plain UUIDs** owned by Inventory. No foreign keys.

### 5.2 PO state machine
```
DRAFT ──confirm──▶ CONFIRMED ──receivePartial──▶ PARTIALLY_RECEIVED ──receiveRest──▶ RECEIVED
  │                    │                                │
  └──cancel──▶ CANCELLED ◀──cancel──┘                    (cancel not allowed once anything is received)
```
Invariants: ≥ 1 line to confirm; quantity > 0; the expected date is not in the past at confirm time (using the injected `Clock`); the supplier must be ACTIVE.

### 5.3 API (contract in `contracts/openapi/procurement.yaml`)
| Method | Path | Success | Errors |
|---|---|---|---|
| POST | `/api/v1/suppliers` | 201 + Location | 400 |
| GET | `/api/v1/suppliers?status=&page=&size=` | 200 page | — |
| POST | `/api/v1/suppliers/{id}/products` | 201 | 400, 404 |
| POST | `/api/v1/purchase-orders` | 201 (DRAFT) | 400, 422 (supplier suspended) |
| GET | `/api/v1/purchase-orders/{id}` | 200 + ETag | 404 |
| GET | `/api/v1/purchase-orders?status=&supplierId=` | 200 page | — |
| POST | `/api/v1/purchase-orders/{id}/confirm` | 200 | 409 `PO_INVALID_TRANSITION`, 412 ETag mismatch |
| POST | `/api/v1/purchase-orders/{id}/cancel` | 200 | 409 |
| POST | `/api/v1/purchase-orders/{id}/receipts` | 200 | 409, 422 (over-receipt) |

### 5.4 Configuration
`server.shutdown=graceful`, `spring.lifecycle.timeout-per-shutdown-phase=20s`, `management.endpoint.health.probes.enabled=true`, `spring.jpa.open-in-view=false`, `ddl-auto=validate`, and Hikari with a sane pool size (5) and connection timeout (3 s).

---

## 6. Tasks

| ID | Task | Est | Concept | P |
|---|---|---|---|---|
| M03-T01 | Write `procurement.yaml` (contract-first); Spectral lint passes | 3h | Contract-first | P0 |
| M03-T02 | `openapi-generator-maven-plugin` → interfaces only (`interfaceOnly=true`); controllers implement them | 1.5h | Contract ↔ code sync | P0 |
| M03-T03 | Domain: aggregates, `Money`, state machine, invariants; **unit tests first** | 4h | Aggregates, state machines | P0 |
| M03-T04 | Application: use cases (`CreatePurchaseOrder`, `ConfirmPurchaseOrder`…), ports | 2h | Hexagonal core | P0 |
| M03-T05 | Persistence adapter: Flyway V1, JPA entities, mappers, repository port implementation | 3h | Persistence isolation | P0 |
| M03-T06 | Web adapter: controllers, DTO mapping, Problem Details codes, ETag/If-Match | 3h | Uniform error contract, optimistic concurrency over HTTP | P0 |
| M03-T07 | Health probes, graceful shutdown config | 1h | Liveness vs readiness | P0 |
| M03-T08 | Testcontainers integration tests (repository + full HTTP slice) | 2.5h | Real-infra testing | P0 |
| M03-T09 | Seed script (`scripts/seed/procurement.http` or SQL via API) for the flagship scenario | 1h | Reproducible demos | P0 |
| M03-T10 | Service README (M00 template) | 0.5h | — | P0 |
| M03-T11 | Break tests | 1.5h | — | P0 |

---

## 7. Break tests

| # | Drill | Expected |
|---|---|---|
| B1 | Stop Postgres while the service runs | readiness DOWN, liveness UP; requests → 503 Problem Details; auto-recovers when the DB returns |
| B2 | Start the service *before* Postgres is up | Doesn't crash-loop forever; becomes ready when the DB arrives (decide: fail fast + container restart, or retry; document it) |
| B3 | `SIGTERM` during an artificially slow request (sleep in a test endpoint/profile) | The in-flight request completes; new requests refused |
| B4 | Confirm a RECEIVED PO | 409 with `code=PO_INVALID_TRANSITION` |
| B5 | Two clients update the same PO with the same ETag | One 200, one 412 |
| B6 | ArchUnit: add a controller → repository shortcut | Build fails (M00 rule still guards you) |

---

## 8. Pitfalls

- Business rules leaking into controllers or JPA entities. Keep them in the domain.
- Returning JPA entities from controllers. Always map to DTOs.
- Liveness including the `db` health indicator. Configure the probe groups explicitly.
- Testing with `@SpringBootTest` for everything (slow). Use the pyramid.

## 9. Deliverables

Procurement service, OpenAPI contract, migrations, tests, seed, README, ADR *API error format & versioning* (if not done in M00), tag `m03-procurement`.

## 10. Self-check

1. Why is the API contract the only acceptable coupling between services?
2. What breaks if liveness checks the database?
3. Why model the PO lifecycle as an explicit state machine *before* you add events?
4. How does ETag/If-Match relate to optimistic locking in the database?

## 11. References (when stuck)

Spring Boot docs: *Kubernetes probes*, *Graceful shutdown* · OpenAPI Generator Maven plugin · RFC 9457 · *Building Microservices* 2e ch. 5

## 12. Exit checklist

- [ ] Flagship PO can be created/confirmed via the API from a clean clone
- [ ] B1–B6 pass; all tests + gates green in CI
- [ ] README + contract committed · tag `m03-procurement`
