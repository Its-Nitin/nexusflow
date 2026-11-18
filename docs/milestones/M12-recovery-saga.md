# M12 — Recovery Service II: Saga Execution & Compensation

| | |
|---|---|
| **Release** | R3 Disruption Flow |
| **Duration** | 1 week (~25 h). The hardest milestone; use buffer if needed, never cut corners |
| **Depends on** | M11 |
| **Unlocks** | M13, M14 |
| **Tag** | `m12-recovery-saga` |

---

## 1. Why this milestone exists

An approved plan must change state in up to three services. There is **no distributed transaction**: each service commits locally, any step can fail, and messages can be duplicated, delayed or lost-then-redelivered. A **saga** keeps the business consistent anyway: it runs local transactions in sequence and, when something fails, runs **compensating transactions** to semantically undo the completed steps. This is the defining pattern of microservice data management.

---

## 2. Learning objectives

- Implement an orchestrated saga with durable state, async commands and replies.
- Make every participant handler idempotent and every reply correlation safe.
- Implement compensation in reverse order, and explain what to do when a compensation fails.
- Implement step timeouts with durable timers.
- Explain the saga's **lack of isolation (ACD, not ACID)** and apply countermeasures (semantic locks, re-read/verify).
- Compare orchestration with choreography and with workflow engines (Temporal).

---

## 3. Concepts

### 3.1 Why not 2PC
Two-phase commit needs every participant to support it (Kafka doesn't), holds locks across services while waiting, and **blocks** if the coordinator dies. It trades availability for atomicity, which is the wrong trade for independent services.

### 3.2 Orchestration vs choreography
| | Orchestration (Recovery tells participants what to do) | Choreography (participants react to each other's events) |
|---|---|---|
| Visibility | One place shows the whole flow | Flow is implicit, spread out |
| Coupling | Orchestrator knows participants | Participants know each other's events |
| Good for | Multi-step, conditional, compensating workflows | Simple fan-out reactions |
NexusFlow uses **orchestration** for recovery (here) and **choreography** for notifications (M13), so you have both styles in one system.

### 3.3 Sagas are ACD, not ACID
No isolation: other transactions can see intermediate states (stock reallocated but the PO not yet created). Anomalies: **lost updates, dirty reads, fuzzy reads**. Countermeasures:
- **Semantic lock:** mark records `PENDING_*` so others know; release on completion or compensation.
- **Commutative updates:** design operations so order doesn't matter (increment/decrement).
- **Re-read value / verification:** before the final step, re-read state and verify assumptions.
- **Pessimistic view:** order steps to minimise the business risk of dirty reads (compensatable first, pivot last).

### 3.4 Compensation is semantic, not a rollback
You can't "un-send" an email or "un-confirm" a supplier order. Compensation is a **new business action** (cancel the PO, release the reallocation). Steps after the pivot must be **retriable** until they succeed.

### 3.5 When compensation fails
Retry with backoff (compensations must be idempotent and retriable). If it still fails, set the saga to `FAILED`, raise an alert, and require manual intervention through a runbook. **Never silently swallow it.**

### 3.6 Durable orchestration
The orchestrator persists saga state **before** sending each command (outbox), and updates it on each reply (inbox + version guard). Crash at any point → on restart, state + outbox + redelivered replies resume the saga. In-memory sagas don't survive deploys.

### 3.7 Workflow engines
Temporal, Camunda and others provide durable execution, timers and retries as a platform. A stretch lab: re-implement one saga path in Temporal and compare code size and failure semantics. Knowing *what they solve* is what matters in interviews.

---

## 4. Scope

**In:** the command/reply protocol, participant handlers (Inventory, Shipment, Procurement), the orchestrator engine (persisted saga instance + step state), compensation, step deadlines, verification, semantic locks, plan-status events, failure drills, ADRs.
**Out:** UI (M15), notifications (M13).

---

## 5. Technical specification

### 5.1 Command & reply envelopes
```
Command: { commandId, sagaId, stepId, commandType, tenantId, correlationId, replyTo:"recovery.replies", issuedAt, payload }
Reply:   { commandId, sagaId, stepId, outcome: SUCCESS|FAILURE, reasonCode, payload, occurredAt }
Topics:  inventory.commands (key = reservationId/productId), shipment.commands (key = shipmentId),
         procurement.commands (key = poId or sagaId), recovery.replies (key = sagaId)
```
Participants process commands through the **inbox** (dedupe by `commandId`) and send the reply through **their outbox**, in the same transaction as the state change.

### 5.2 Participant commands
| Service | Command | Effect | Compensation |
|---|---|---|---|
| Inventory | `ReallocateReservation{reservationId, fromWarehouse, toWarehouse, qty}` | Transfer + reservation `PENDING_REALLOCATION` (semantic lock) | `ReleaseReallocation` (reverse transfer, restore reservation) |
| Inventory | `FinaliseReallocation{reservationId}` | Clear the lock | — (retriable) |
| Shipment | `ExpediteShipment{shipmentId, newMode}` | Mode/ETA/cost updated; `ShipmentExpedited` | `CancelExpedite` |
| Procurement | `CreateEmergencyPurchaseOrder{supplierId, productId, qty, neededBy}` | DRAFT PO flagged emergency | `CancelPurchaseOrder` |
| Procurement | `ConfirmEmergencyPurchaseOrder{poId}` | CONFIRMED (**pivot**) | — |

Business-rule rejections (e.g. insufficient stock at W2 now) → `FAILURE` reply with a `reasonCode` (not an exception → DLT).

### 5.3 Orchestrator model
```
saga_instance(id, plan_id, tenant_id, saga_type, status[RUNNING|COMPENSATING|COMPLETED|COMPENSATED|FAILED],
              current_step, version, started_at, updated_at)
saga_step(saga_id, step_no, command_type, command_id, status[PENDING|SENT|SUCCEEDED|FAILED|COMPENSATING|COMPENSATED|SKIPPED],
          deadline_at, attempts, compensation_command_id, last_error)
```

### 5.4 Engine algorithm
```
on RecoveryPlanApproved (inbox):
  create saga_instance + steps from the plan; mark step1 SENT; outbox command1; plan → EXECUTING   [one tx]
on Reply (inbox + match commandId to step; ignore unknown/stale):
  SUCCESS → step SUCCEEDED; if next step: send it; else → VERIFY
  FAILURE → if before pivot: status COMPENSATING; send compensation for the last SUCCEEDED step (reverse order)
            if after pivot: retry the step (it is retriable) with backoff
on compensation reply SUCCESS → next compensation … → all done → COMPENSATED; plan → COMPENSATED
on compensation FAILURE → retry N times → FAILED + alert + runbook
deadline sweeper (every 5 s, SKIP LOCKED): SENT step past deadline_at → resend the same command (same commandId, so idempotent)
     up to N attempts → treat as FAILURE
VERIFY: query participants (sync, resilient) to confirm the expected end state → COMPLETED; plan → COMPLETED (event)
```

### 5.5 Events
`RecoveryPlanExecutionStarted`, `RecoveryPlanCompleted`, `RecoveryPlanCompensated`, `RecoveryPlanFailed` (with step details) on `recovery.events`.

---

## 6. Tasks

| ID | Task | Est | Concept | P |
|---|---|---|---|---|
| M12-T01 | Command/reply envelopes in `platform-messaging`; topics | 2h | Async request/reply | P0 |
| M12-T02 | Inventory handlers (reallocate/release/finalise) with inbox + outbox reply + semantic lock | 4h | Participant idempotency, semantic lock | P0 |
| M12-T03 | Shipment handlers (expedite/cancel) | 2h | — | P0 |
| M12-T04 | Procurement handlers (create/confirm/cancel emergency PO) | 2.5h | Pivot step | P0 |
| M12-T05 | Orchestrator: saga tables, start on approval, step progression | 4h | Durable orchestration | P0 |
| M12-T06 | Compensation path (reverse order) + compensation-failure handling | 3h | Compensating transactions | P0 |
| M12-T07 | Deadline sweeper + resend with the same commandId | 2h | Durable timers | P0 |
| M12-T08 | Verification step | 1.5h | Re-read countermeasure | P0 |
| M12-T09 | Failure-injection hooks (config flag per participant: fail command X / no reply) for drills | 1.5h | Testability of failure | P0 |
| M12-T10 | Saga integration tests (all participants via Testcontainers + Compose) | 3h | — | P0 |
| M12-T11 | Runbook `docs/runbooks/saga-failed.md`; ADRs *Orchestration for recovery*, *Saga isolation countermeasures* | 1.5h | — | P0 |
| M12-T12 | **Stretch lab:** one saga path in Temporal; compare | 4h | Workflow engines | P1 |

---

## 7. Break tests

| # | Drill | Expected |
|---|---|---|
| B1 | Happy path (flagship) | COMPLETED; stock transferred; semantic lock cleared; events emitted |
| B2 | Procurement fails `CreateEmergencyPurchaseOrder` | Reallocation compensated; plan COMPENSATED; stock back to its original state |
| B3 | Kill Recovery after step 1 is sent, before its reply | On restart, the reply is processed and the saga continues |
| B4 | A participant processes a command but its reply is lost (kill it before the outbox publish) | Deadline → resend the same commandId → inbox dedupes → the reply is sent once, no double effect |
| B5 | Duplicate reply | Ignored (inbox/step state) |
| B6 | Compensation fails 3× | Saga FAILED; alert log; runbook followed manually to resolve |
| B7 | During a saga, another user reserves the reservation under the semantic lock | Rejected / told it is pending (document the behaviour) |
| B8 | Failure *after* the pivot (FinaliseReallocation fails transiently) | Retried until success; no compensation |

---

## 8. Pitfalls

- Sending a command before persisting saga state (crash = an orphan command).
- New commandIds on resend (breaks participant idempotency).
- Treating business rejections as exceptions (they go to the DLT instead of driving compensation).
- Compensation that assumes the forward step fully succeeded (it may have partially applied; make compensations tolerant).
- A participant reading the orchestrator's database "to check". Never.

## 9. Deliverables

The saga engine, participant handlers, drills automated, runbook, 2 ADRs, optional Temporal lab notes, tag.

## 10. Self-check

1. Why not 2PC? Give two reasons.
2. Orchestration vs choreography: why each where you used them?
3. Which is your pivot step, and why is it placed last among the business steps?
4. What anomalies can a saga cause, and which countermeasures did you implement?
5. Walk through B4 step by step: why is there no double effect?
6. What happens when a compensation fails?

## 11. References (when stuck)

*Microservices Patterns* ch. 4 (sagas, countermeasures) · Garcia-Molina & Salem, *Sagas* (1987 paper, short) · Caitie McCaffrey, *Distributed Sagas* (talk) · Temporal docs *What is a durable execution?* · microservices.io *Saga*

## 12. Exit checklist

- [ ] B1–B8 pass; B1–B5 and B8 automated
- [ ] Every compensation path has a test
- [ ] Runbook + ADRs written · tag `m12-recovery-saga`
