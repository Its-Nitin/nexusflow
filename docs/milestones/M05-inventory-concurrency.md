# M05 — Inventory Service & Concurrency

| | |
|---|---|
| **Release** | R1 Core Domain |
| **Duration** | 1 week (~23 h) |
| **Depends on** | M03, M04 |
| **Unlocks** | M06, and the saga participant role in M12 |
| **Tag** | `m05-inventory` |

---

## 1. Why this milestone exists

Concurrency bugs are invisible in demos and catastrophic in production. Two requests read "10 available," both reserve 10, and you've promised 20 units you don't have. Inventory is the textbook case. It also introduces **idempotent commands**: in a distributed system clients *will* retry after timeouts, and a retry must not reserve stock twice.

You'll build it the naive way first, watch it oversell, then fix it three different ways and measure them.

---

## 2. Learning objectives

- Explain the lost-update anomaly and which isolation levels do or don't prevent it in Postgres.
- Implement and compare optimistic locking, pessimistic locking and atomic conditional updates.
- Explain the ledger pattern (append-only movements + derived balance).
- Design and implement idempotency keys for HTTP commands, including concurrent duplicate requests.

---

## 3. Concepts

### 3.1 Lost update
Two transactions read the same row, both compute a new value from what they read, both write, and one write silently overwrites the other. Postgres's default **READ COMMITTED** does **not** prevent this for read-modify-write in application code.

### 3.2 Three fixes
| Strategy | How | Good when | Bad when |
|---|---|---|---|
| **Optimistic** | `version` column; `UPDATE … WHERE version = ?`; 0 rows → conflict → retry | Low contention | Hot rows: endless retries |
| **Pessimistic** | `SELECT … FOR UPDATE` locks the row until commit | High contention, short transactions | Long transactions; deadlock risk; lock waits |
| **Atomic conditional** | `UPDATE balance SET reserved = reserved + :q WHERE id = ? AND on_hand - reserved >= :q` (1 row = success) | Simple arithmetic invariants | Complex multi-row rules |
**Why all three:** interviewers ask "how would you prevent overselling?" The senior answer compares them *with numbers*.

### 3.3 Ledger pattern
`StockMovement` rows are **append-only** (RECEIPT, RESERVE, RELEASE, ISSUE (stock shipped to a customer, i.e. consumed demand), ADJUST, TRANSFER_OUT/IN), and `InventoryBalance` is a derived, cached aggregate. **Why:** auditability (every unit is explainable), reconstruction (rebuild balances from movements), and a natural source for events (M07) and demand history (M10).

### 3.4 Idempotency keys
**Problem:** the client sends `POST /reservations`, the server commits, the response is lost (timeout), the client retries, and now there are two reservations.
**Solution:** the client sends `Idempotency-Key: <uuid>`. The server stores `(key, tenant, requestHash, status, response)`:
- First request: insert the key row (`IN_PROGRESS`) → execute → store the response (`COMPLETED`).
- Retry with the same key and body: return the **stored response** without re-executing.
- Same key, *different* body: 422 (a client bug).
- Retry while the first is still `IN_PROGRESS`: 409 (try later).
**Crucial detail:** the key insert and the business change must be in the **same DB transaction**, or a crash between them breaks the guarantee. (The same lesson returns with the outbox in M07.)

### 3.5 Commitments (the gap fixed from the blueprint)
A `Reservation` is stock promised to a **commitment** (a customer order line), with `requiredByDate`. This gives the Risk engine (M10) something concrete to put at risk.

---

## 4. Scope

**In:** Warehouse, Product, InventoryBalance, StockMovement ledger, Reservation with commitments, receive/reserve/release/adjust/transfer APIs, three concurrency strategies, idempotency keys (`platform-idempotency`), concurrency tests, a benchmark.
**Out:** events (M06–M07), saga commands (M12).

---

## 5. Technical specification

### 5.1 Model
```
Warehouse(id, tenantId, code, name, region)
Product(id, tenantId, sku, name, unitOfMeasure, safetyStock)
InventoryBalance(id, tenantId, warehouseId, productId, onHand, reserved, version)   -- available = onHand - reserved
  CHECK (on_hand >= 0), CHECK (reserved >= 0), CHECK (reserved <= on_hand)        -- the DB as the last line of defence
StockMovement(id, tenantId, warehouseId, productId, type, quantity, reference, occurredAt)
Reservation(id, tenantId, warehouseId, productId, quantity, commitmentRef, customerRef, requiredByDate, status[ACTIVE|RELEASED|FULFILLED], version)
IdempotencyRecord(key, tenantId, requestHash, status, responseStatus, responseBody, createdAt, expiresAt)
```

### 5.2 API
| Method | Path | Notes |
|---|---|---|
| POST | `/api/v1/warehouses`, `/api/v1/products` | setup |
| POST | `/api/v1/inventory/receipts` | increases onHand + RECEIPT movement |
| POST | `/api/v1/reservations` | **Idempotency-Key required**; 201 or 409 `INSUFFICIENT_STOCK` |
| POST | `/api/v1/reservations/{id}/release` | idempotent by nature (already released → 200 no-op) |
| POST | `/api/v1/inventory/adjustments` | MANAGER only |
| POST | `/api/v1/inventory/transfers` | between warehouses (used by recovery in M12) |
| GET | `/api/v1/inventory?productId=&warehouseId=` | balances with `available` |
| GET | `/api/v1/inventory/movements?productId=&from=&to=` | ledger |

### 5.3 Strategy switch
`nexusflow.inventory.reservation-strategy = optimistic | pessimistic | atomic` selects a `ReservationLockingStrategy` port implementation. The default after the benchmark is your choice, recorded in an ADR.

---

## 6. Tasks

| ID | Task | Est | Concept | P |
|---|---|---|---|---|
| M05-T01 | Service from the template; model + Flyway with CHECK constraints | 3h | Constraints as last defence | P0 |
| M05-T02 | Ledger: every balance change writes a movement in the same transaction | 2h | Ledger pattern | P0 |
| M05-T03 | **Naive** reserve (read → check → save, no locking); temporarily drop the CHECK constraint to *see* the bug | 1.5h | (sets up the break) | P0 |
| M05-T04 | Concurrency test harness: N threads (`ExecutorService` + `CountDownLatch` start gate) against Testcontainers Postgres | 2h | Reproducing races | P0 |
| M05-T05 | Optimistic strategy + bounded retry | 2h | Optimistic locking | P0 |
| M05-T06 | Pessimistic strategy (`@Lock(PESSIMISTIC_WRITE)` / `FOR UPDATE`) with a lock timeout | 1.5h | Pessimistic locking | P0 |
| M05-T07 | Atomic conditional UPDATE strategy | 1.5h | Atomic updates | P0 |
| M05-T08 | Benchmark: 3 strategies × contention levels (10, 50, 200 threads on 1 SKU); success, conflicts/retries, p50/p95 | 2.5h | Contention cost | P0 |
| M05-T09 | `platform-idempotency`: filter/aspect + table + hash; same-transaction guarantee | 4h | Idempotent commands | P0 |
| M05-T10 | Seed the flagship stock and reservations (P-104 @ W1, commitments due 12 and 14 Nov) | 0.5h | — | P0 |
| M05-T11 | ADR *Inventory concurrency strategy* with your numbers; ADR *Idempotency-key design* | 1.5h | — | P0 |
| M05-T12 | Restore the CHECK constraints; journal | 0.5h | — | P0 |

---

## 7. Break tests

| # | Drill | Expected |
|---|---|---|
| B1 | Naive: 100 threads reserve 1 unit each with 10 available | **Oversells** (more than 10 succeed / negative available). Record it |
| B2 | Each fixed strategy, same test | Exactly 10 succeed, available = 0, never negative |
| B3 | Naive code + CHECK constraint restored | The DB rejects it, but with 500s. Explain why constraints are a safety net, not the design |
| B4 | Same Idempotency-Key ×5 sequentially | 1 reservation; 5 identical responses |
| B5 | Same key ×5 **concurrently** | 1 reservation; others get 409 or the stored response |
| B6 | Same key, different body | 422 |
| B7 | Kill the service mid-request (after commit, before response) → client retries with the same key | Stored response returned; no duplicate |
| B8 | Pessimistic: two transactions lock 2 SKUs in opposite order | Deadlock detected by Postgres; explain it; fix with consistent lock ordering |

---

## 8. Pitfalls

- `@Transactional` on a private method or a self-invocation: no transaction (Spring proxy rule).
- Optimistic retry wrapping *inside* the transaction: a retry must start a **new** transaction.
- Benchmarks without warm-up, or on a single run. Do 3+ runs and report the median.
- Storing the idempotency record in a separate transaction from the business write.

## 9. Deliverables

Inventory service, `platform-idempotency`, concurrency tests in CI, `docs/benchmarks/inventory-locking.md`, 2 ADRs, tag `m05-inventory`.

## 10. Self-check

1. Does READ COMMITTED prevent lost updates? What about REPEATABLE READ in Postgres?
2. When is optimistic locking the wrong choice?
3. Why must the idempotency record share the transaction with the reservation?
4. What's the difference between an idempotent *operation* and an idempotency *key*?
5. Why keep a ledger if you already have a balance?

## 11. References (when stuck)

DDIA ch. 7 (weak isolation, lost updates, write skew) · PostgreSQL docs *Transaction Isolation*, *Explicit Locking* · Stripe engineering blog *Designing robust and predictable APIs with idempotency* · IETF draft *The Idempotency-Key HTTP Header Field*

## 12. Exit checklist

- [ ] B1–B8 done; B2, B4–B7 automated in CI
- [ ] Benchmark doc with measured numbers
- [ ] ADRs written · tag `m05-inventory`
