# M17 — Contract Testing, Chaos Engineering & Load

| | |
|---|---|
| **Release** | R5 Production Ready |
| **Duration** | 1 week (~22 h) |
| **Depends on** | M16 |
| **Unlocks** | M18 |
| **Tag** | `m17-contracts-chaos-load` |

---

## 1. Why this milestone exists

Three questions every microservice system must answer *before* production:
1. **Can I deploy one service without breaking another?** → consumer-driven contract tests.
2. **What happens when the network or infrastructure misbehaves?** → chaos experiments with a hypothesis.
3. **How far does it scale, and where is the limit?** → a load test with a method, plus a scaling experiment on Kafka consumers.

So far you've tested failures one at a time. Now they become a **repeatable suite**, with numbers you measured yourself.

---

## 2. Learning objectives

- Explain consumer-driven contracts and how they enable independent deployability.
- Write contract tests for both REST APIs and asynchronous messages.
- Run chaos experiments: steady-state hypothesis, blast radius, observe, learn.
- Run a load test methodically (warm-up, repeated runs, recorded environment).
- Explain why consumer parallelism is bounded by partitions, and demonstrate it.

---

## 3. Concepts

### 3.1 Consumer-driven contract testing (CDC testing)
Each **consumer** writes down exactly what it uses from a provider (fields, status codes, message shapes). The **provider** verifies it still satisfies every consumer's contract in its own CI. A provider can't break a consumer without its build going red. **Why not just E2E tests?** E2E is slow, flaky and needs everything deployed. Contracts test each pair in isolation, fast.
Tools: **Pact** (consumer-driven, a broker to share pacts; supports messages) or **Spring Cloud Contract** (provider-defined, generates stubs). Pick one; write an ADR.
(Schema-registry compatibility (M08) checks *shape*; contract tests check *what consumers rely on*. They complement each other.)

### 3.2 Chaos engineering
1. Define the **steady state** (e.g. "the flagship flow completes; detection SLO met").
2. Hypothesise: "if Risk loses Kafka for 60 s, assessments are delayed but none are lost."
3. Inject the fault with a small **blast radius**.
4. Observe via dashboards (M16).
5. Learn: confirm the hypothesis, or fix and add a regression drill.

### 3.3 Load testing methodology
Define the scenario and the measured paths → warm up → run ≥ 3 times → report median and spread → record hardware, versions, config, data volume. **Never publish numbers you didn't measure** (the blueprint rule).

### 3.4 Scaling consumers
Throughput scales with consumers **only up to the partition count**. Beyond that, extra consumers idle. Repartitioning changes key→partition mapping (ordering risk during the transition). Also note the processing cost per message (DB writes per event) usually dominates. Scaling consumers can just move the bottleneck to Postgres.

---

## 4. Scope

**In:** contract tests for 2 REST pairs + 3 event pairs; a CI contract-verification stage; the E2E flagship in CI (exists from M15; make it part of the gates); a Toxiproxy chaos suite; k6 load tests; a partition/consumer scaling experiment; `docs/performance.md`; runbooks.
**Out:** production-scale benchmarks.

---

## 5. Technical specification

### 5.1 Contract pairs
| Consumer | Provider | Type |
|---|---|---|
| Recovery | Inventory `GET /inventory` | REST |
| Risk (sync adapter) | Procurement `GET /purchase-orders/{id}` | REST |
| Risk | Shipment → `ShipmentDelayed` | Message |
| Recovery | Risk → `RiskAssessmentCompleted` | Message |
| Query | Recovery → `RecoveryPlanApproved` | Message |

### 5.2 Chaos suite (`scripts/drills/chaos/`)
| Experiment | Fault | Hypothesis |
|---|---|---|
| C1 | Risk ↔ Kafka partition, 60 s | Delayed assessments, zero loss, lag recovers < 2 min |
| C2 | Postgres latency +500 ms for Inventory | Reservation p95 rises; the circuit breaker protects Recovery; no saga corruption |
| C3 | Kill a random service container during the flagship flow | The flow completes or compensates; no stuck sagas > deadline |
| C4 | Keycloak down 5 min | Existing sessions keep working until token expiry; new logins fail clearly |
| C5 | Kafka broker restart | Producers/consumers reconnect; outbox backlog drains |

### 5.3 Load tests (k6)
- L1: `POST /shipments/{id}/events` (delay reports) at a ramping rate → measure the end-to-end detection latency (SLO metric) and lag.
- L2: dashboard reads (`/control-tower/*`) → p95.
- L3: concurrent reservations on hot SKUs → conflicts and p95.

### 5.4 Scaling experiment
`shipment.events` with 3 partitions: Risk consumers 1 → 2 → 3 → 4, then 6 partitions with 3 → 6 consumers. Plot throughput and lag. Explain the plateau.

---

## 6. Tasks

| ID | Task | Est | Concept | P |
|---|---|---|---|---|
| M17-T01 | Choose Pact or SCC (ADR); set up | 2h | CDC testing | P0 |
| M17-T02 | 2 REST contracts + provider verification | 3h | — | P0 |
| M17-T03 | 3 message contracts + provider verification | 3h | Async contracts | P0 |
| M17-T04 | CI: contract-verification stage blocks the merge | 1h | Independent deployability | P0 |
| M17-T05 | Chaos suite C1–C5 scripted with hypotheses + results | 5h | Chaos engineering | P0 |
| M17-T06 | k6 scenarios L1–L3; 3 runs each | 3h | Load methodology | P0 |
| M17-T07 | Scaling experiment + chart | 2h | Consumer parallelism | P0 |
| M17-T08 | `docs/performance.md` + runbooks: consumer down, projection rebuild, service restart, Kafka restart | 2h | Operability | P0 |

---

## 7. Break tests

| # | Drill | Expected |
|---|---|---|
| B1 | Rename a field Risk relies on in `ShipmentDelayed` | Shipment's provider verification fails in CI before merge |
| B2 | Remove a field no consumer uses | Verification passes (contracts allow safe evolution) |
| B3 | C1–C5 | Every hypothesis confirmed, or a bug found + fixed + a regression drill added |
| B4 | 4 consumers on 3 partitions | One idle consumer visible in the lag/assignment metrics |

---

## 8. Pitfalls

- Contracts that assert everything the provider returns (over-specified, so they break on harmless changes).
- Chaos without a hypothesis is just breaking things.
- Load testing from the same laptop that runs the system: note the interference in the methodology.
- Reporting a single run.

## 9. Deliverables

Contract tests + CI stage, chaos suite + results, k6 scripts, scaling chart, `docs/performance.md`, runbooks, ADR, tag.

## 10. Self-check

1. How do consumer-driven contracts enable independent deployment?
2. Schema-registry compatibility vs contract tests: what does each catch?
3. Describe the steps of a chaos experiment with C1 as the example.
4. Why does throughput plateau when consumers exceed partitions?
5. What must accompany any performance number you publish?

## 11. References (when stuck)

Pact docs (incl. message pacts) · Spring Cloud Contract docs · principlesofchaos.org · k6 docs · *Building Microservices* 2e ch. 9 (testing) · *Kafka: The Definitive Guide* (consumer scaling)

## 12. Exit checklist

- [ ] B1–B4 pass; contracts gate merges
- [ ] Chaos results + performance doc committed (measured numbers only)
- [ ] ADR written · tag `m17-contracts-chaos-load`
