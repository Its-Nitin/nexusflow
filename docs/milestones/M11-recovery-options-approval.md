# M11 — Recovery Service I: Options, State Machine & Approval

| | |
|---|---|
| **Release** | R3 Disruption Flow |
| **Duration** | 1 week (~22 h) |
| **Depends on** | M10 |
| **Unlocks** | M12 |
| **Tag** | `m11-recovery-approval` |

---

## 1. Why this milestone exists

Detecting a risk is half the product. The other half is **deciding and acting**, and acting means changing state in *several* services (Inventory, Shipment, Procurement), with a human approval step in the middle. Before writing the saga (M12) you need: a service that **owns the workflow**, a precise **state machine**, deterministic options, and a safe, audited approval. That design work is where most saga bugs are prevented.

---

## 2. Learning objectives

- Explain the **process manager** pattern and why a dedicated workflow owner beats spreading the logic across services.
- Design a workflow state machine with explicit transitions, guards and terminal states.
- Decide deliberately where to use sync queries (fresh data at decision time) inside an event-driven flow.
- Make concurrent human decisions safe (optimistic locking + idempotent commands).
- Design a saga on paper: steps, commands, replies, compensations, pivot, timeouts.

---

## 3. Concepts

### 3.1 Process manager / orchestrator
A component that holds the state of a multi-step business process and decides the next step from events and replies. **Why a separate Recovery service:** workflow state is its own domain (a plan has a lifecycle, approvals and an audit). Embedding it in Risk or Intelligence mixes responsibilities. (This fixes a gap in the blueprint.)

### 3.2 Human-in-the-loop
Some steps wait for a person for minutes or days. The workflow must be **durable** (persisted state, not an in-memory thread), have **expiry** (a proposal goes stale when the data changes), and record an **audit** (who, when, evidence, reason).

### 3.3 Staleness of proposals
Options are computed at time T. By approval at T+2h, stock may have moved. Mitigations: an `expiresAt` on the proposal, **re-validation at approval** (a fresh sync query), and verification after execution (M12). This is the honest handling of eventual consistency in decisions.

### 3.4 Sync query at decision time (a deliberate exception)
Recovery queries Inventory's `GET /inventory` for **current** alternative stock when proposing and again when approving, with timeout and circuit breaker (M09). If Inventory is down, the proposal is marked `NEEDS_REVALIDATION` and approval is blocked: fail safe, not fail silent.

### 3.5 Saga design vocabulary (used in M12)
- **Compensatable step:** can be undone semantically (release a reallocation).
- **Pivot step:** the go/no-go point. After it, the saga must complete (e.g. an emergency PO sent to the supplier).
- **Retriable step:** after the pivot; it must eventually succeed (retry until it does).
- **Semantic lock:** a flag marking data as "in a saga" (e.g. reservation `PENDING_REALLOCATION`).

---

## 4. Scope

**In:** Recovery service, consuming `RiskAssessmentCompleted`, option generation, the plan state machine, approve/reject with audit, re-validation, expiry, the `recovery.events` outbox events, the **saga design document**.
**Out:** saga execution (M12).

---

## 5. Technical specification

### 5.1 Option generation (deterministic)
Inputs: the assessment (shortfall, affected commitments), fresh alternative stock (sync), supplier catalogue (a projection of `SupplierProductListed`), the shipment (projection).

| Option | Feasible when | Effect | Cost model (configurable) |
|---|---|---|---|
| `REALLOCATE_STOCK` | Another warehouse W2 has `available − safetyStock ≥ needed` | Transfer qty W2→W1 before `requiredByDate` (transfer lead time from config) | qty × transferCostPerUnit |
| `EXPEDITE_SHIPMENT` | Shipment not arrived; mode upgrade possible (SEA→AIR, ROAD→EXPRESS) | `revisedEta − expediteDaysSaved` | fixed fee + per-unit premium |
| `ALTERNATE_SUPPLIER_PO` | A supplier lists the product with `leadTimeDays` ≤ days until `requiredByDate` | Emergency PO for the shortfall | qty × unitPrice + rush fee |
| `ACCEPT_RISK` | Always | Do nothing; notify | 0 (but commitments missed) |

Each option has `coverage` (the share of shortfall covered), `cost: Money`, `feasibleUntil`, `steps[]`. Rank by: full coverage first, then cost, then fewer steps. Combined options (partial reallocation + emergency PO) are a P1 stretch.

### 5.2 State machine
```
PROPOSED ──approve(MANAGER)──▶ APPROVED ──(M12)──▶ EXECUTING ──▶ COMPLETED
   │  │                                               └──▶ COMPENSATING ──▶ COMPENSATED | FAILED
   │  └──reject──▶ REJECTED
   └──expire / newer assessment──▶ EXPIRED
```
Guards: approve only if PROPOSED, not expired, and re-validation passes; one decision per plan (optimistic `version`).

### 5.3 Model
```
recovery_plan(id, tenant_id, assessment_id, status, selected_option jsonb, options jsonb, expires_at, version, created_at)
approval(id, plan_id, decision[APPROVED|REJECTED], actor, actor_roles, reason, evidence_snapshot jsonb, decided_at)
audit_log(id, plan_id, action, actor, details jsonb, at)
```

### 5.4 API
`GET /api/v1/recovery-plans?status=` · `GET /api/v1/recovery-plans/{id}` · `POST /api/v1/recovery-plans/{id}/approve` `{optionId, reason}` (MANAGER, Idempotency-Key) · `POST …/reject` `{reason}`

### 5.5 Events
`RecoveryPlanProposed`, `RecoveryPlanApproved`, `RecoveryPlanRejected`, `RecoveryPlanExpired` on `recovery.events` (outbox).

### 5.6 Saga design document (`docs/architecture/recovery-saga.md`)
For each option type, a table: step # · participant · command · success reply · failure reply · compensation · compensatable/pivot/retriable · timeout. Plus Mermaid sequence diagrams for the happy path and one failure path.

Example: **REALLOCATE + ALTERNATE_SUPPLIER_PO (combined)**
| # | Participant | Command | Compensation | Type |
|---|---|---|---|---|
| 1 | Inventory | `ReallocateReservation` (semantic lock, partial transfer) | `ReleaseReallocation` | Compensatable |
| 2 | Procurement | `CreateEmergencyPurchaseOrder` (DRAFT, internal) | `CancelPurchaseOrder` | Compensatable |
| 3 | Procurement | `ConfirmEmergencyPurchaseOrder` (sent to supplier) | — | **Pivot** |
| 4 | Inventory | `FinaliseReallocation` (clear lock) | — | Retriable |

---

## 6. Tasks

| ID | Task | Est | Concept | P |
|---|---|---|---|---|
| M11-T01 | Recovery service from the template; consume `RiskAssessmentCompleted` (idempotent) | 2h | — | P0 |
| M11-T02 | Supplier-catalogue + shipment projections | 2h | Event-carried state (reuse) | P0 |
| M11-T03 | Inventory sync client with resilience (fresh availability) | 1.5h | Deliberate sync | P0 |
| M11-T04 | Pure option generator + ranking; scenario tests | 5h | Deterministic decisions | P0 |
| M11-T05 | Plan state machine (domain) + tests for every transition and guard | 3h | Process manager state | P0 |
| M11-T06 | Approve/reject: re-validation, optimistic lock, Idempotency-Key, audit in the same transaction | 3h | Concurrent human decisions | P0 |
| M11-T07 | Expiry sweeper (SKIP LOCKED) + superseding on a newer assessment | 1.5h | Durable timers | P0 |
| M11-T08 | Outbox events | 1h | — | P0 |
| M11-T09 | Saga design document + diagrams | 3h | **Saga design** | P0 |

---

## 7. Break tests

| # | Drill | Expected |
|---|---|---|
| B1 | Flagship: the assessment arrives | A plan with ranked options; the cheapest full-coverage option first |
| B2 | Two managers approve concurrently | One 200, one 409 |
| B3 | Approve with the same Idempotency-Key twice | One approval, same response |
| B4 | A PLANNER approves | 403 |
| B5 | Drain W2 stock between proposal and approval | Approval rejected: `PLAN_STALE`; re-proposal triggered |
| B6 | Inventory down at approval | Approval blocked (`NEEDS_REVALIDATION`), not approved blindly |
| B7 | A newer assessment for the same risk | The old plan is EXPIRED |
| B8 | Recovery restarts with PROPOSED plans | Plans intact; expiry still works |

---

## 8. Pitfalls

- Options computed only from stale projections (approving against old stock).
- In-memory workflow state.
- An audit written outside the decision's transaction.
- Over-engineering the optimisation. Simple ranking is fine; correctness is the goal.

## 9. Deliverables

Recovery service (options + approval), saga design doc, ADRs *Dedicated process manager for recovery* and *Sync re-validation at decision time*, tag.

## 10. Self-check

1. Why a dedicated Recovery service rather than putting the workflow in Risk?
2. How does NexusFlow handle a proposal going stale before approval?
3. Pivot vs compensatable vs retriable steps: define each with a NexusFlow example.
4. Why is the sync call to Inventory acceptable here when you removed it from Risk?
5. How do you guarantee a single decision per plan?

## 11. References (when stuck)

*Microservices Patterns* ch. 4 (sagas; pivot/compensatable/retriable) · *Enterprise Integration Patterns*, *Process Manager* · microservices.io *Saga* · Caitie McCaffrey, *Applying the Saga Pattern* (talk)

## 12. Exit checklist

- [ ] B1–B8 pass
- [ ] Saga design doc walked through on paper against the M12 failure scenarios
- [ ] ADRs written · tag `m11-recovery-approval`
