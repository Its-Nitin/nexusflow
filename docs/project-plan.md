# NexusFlow — Project Plan (Milestone Roadmap)

> **What this is:** the master plan for building NexusFlow from an empty folder to a released v1.0.
> **Governing rule:** this project exists to **learn microservices and distributed-systems architecture by building them.** Every milestone produces working, production-shaped code *and* teaches named distributed-systems concepts through deliberate failure drills. Finishing the project is mandatory.
> **Capacity:** 3–5 h/day, about 22–25 h/week.
> **Duration:** 22 milestones (M00–M21), about **21 weeks + 2 buffer weeks = 23 weeks**.
> **Source:** `NexusFlow_Project_Blueprint.pdf` v1.0, with the corrections in §7.
> **Detailed milestone docs:** [`docs/milestones/`](milestones/). Each milestone has its own file with concepts, specifications, tasks, break tests and an exit checklist.

---

## 1. How Each Milestone Doc Is Structured

Every `docs/milestones/Mxx-*.md` file follows the same template:

| Section | Purpose |
|---|---|
| **Header** | Release, duration, dependencies, what it unlocks |
| **1. Why this milestone exists** | The real problem it solves, in this project |
| **2. Learning objectives** | What you will be able to *explain and do* afterwards |
| **3. Concepts** | Each concept: what it is, **why we use it**, the trade-offs, and where it shows up in NexusFlow |
| **4. Scope** | In scope / out of scope |
| **5. Technical specification** | Data model, APIs, events, configuration, algorithms |
| **6. Tasks** | Numbered `Mxx-Tnn` tasks with estimates, run through the build→break→learn→fix loop |
| **7. Break tests** | Failure drills. **These are the acceptance criteria** |
| **8. Pitfalls** | Common mistakes and how to avoid them |
| **9. Deliverables** | Files, ADRs, diagrams, git tag |
| **10. Self-check** | Interview-style questions to answer out loud without notes |
| **11. References** | What to read *when stuck* (not up front) |
| **12. Exit checklist** | Tick every box before you start the next milestone |

---

## 2. The Working Loop (learn while building)

Every task runs through this loop. **There are no study-only days.**

1. **Build:** write the straightforward version.
2. **Break:** run the break test. *Write a one-line prediction first.*
3. **Learn:** read only what explains the result (30–60 min, targeted).
4. **Fix:** apply the pattern in the real code, then re-run the break test until it passes.
5. **Record:** journal entry (*problem → cause → pattern → trade-off*); an ADR if a decision was made.

Time split: about 80% hands-on, about 20% targeted reading and writing, mixed together through each day.

**Weekly rhythm:** Mon–Thu tasks · Fri break tests + milestone demo · Sat ADRs, journal, self-check, plan next · Sun off.

---

## 3. Definition of Done (enforced from M00 onwards)

**Task done:** code merged with tests · break test automated (or scripted in `scripts/drills/`) · all quality gates green · journal entry written.

**Milestone done:** every box in the milestone's exit checklist ticked · demo runs from a clean clone · ADRs written · self-check answered aloud · git tag pushed.

**Release done:** all milestones in it are done, and the release gate demo runs end to end.

---

## 4. Roadmap

| # | Milestone | Weeks | Release | Core distributed-systems lessons | Doc |
|---|---|---|---|---|---|
| **M00** | **Engineering Standards & Codebase Guidelines** | 1 | R0 | Microservice chassis, golden path, fitness functions, shift-left quality | [M00](milestones/M00-engineering-standards.md) |
| M01 | Domain & Architecture Design | 0.5 | R0 | Bounded contexts, context mapping, sync vs async | [M01](milestones/M01-domain-architecture-design.md) |
| M02 | Platform Skeleton & Local Infrastructure | 0.5 | R0 | Database per service, 12-factor, backing services | [M02](milestones/M02-platform-skeleton.md) |
| M03 | Procurement Service (reference service) | 1 | R0 | Service contracts, state machines, health vs readiness | [M03](milestones/M03-procurement-service.md) |
| M04 | API Gateway & Identity | 1 | R0 | Edge, OIDC/JWT, zero trust, tenant isolation, rate limiting | [M04](milestones/M04-gateway-identity.md) |
| M05 | Inventory Service & Concurrency | 1 | R1 | Lost updates, locking strategies, idempotent APIs | [M05](milestones/M05-inventory-concurrency.md) |
| M06 | Shipment Service, Kafka & the Dual-Write Problem | 1 | R1 | Partitions, ordering, consumer groups, delivery semantics, dual write | [M06](milestones/M06-shipment-kafka-dual-write.md) |
| M07 | Transactional Outbox & Idempotent Consumers | 1 | R2 | Outbox, inbox, at-least-once, SKIP LOCKED | [M07](milestones/M07-outbox-idempotent-consumers.md) |
| M08 | Failure Handling, Ordering & Schema Evolution | 1 | R2 | Retries, DLT, replay, stale events, schema registry, CDC | [M08](milestones/M08-failure-handling-schemas.md) |
| M09 | Risk Service I: Data You Don't Own | 1 | R3 | Temporal coupling, resilience patterns, event-carried state | [M09](milestones/M09-risk-data-ownership.md) |
| M10 | Risk Service II: Deterministic Risk Engine | 1 | R3 | Determinism, replayability, explainability | [M10](milestones/M10-risk-engine.md) |
| M11 | Recovery Service I: Options & Approval | 1 | R3 | Process managers, human-in-the-loop, concurrent decisions | [M11](milestones/M11-recovery-options-approval.md) |
| M12 | Recovery Service II: Saga Execution | 1 | R3 | Saga orchestration, compensation, timeouts, isolation | [M12](milestones/M12-recovery-saga.md) |
| M13 | Notification Service & Real-Time Fan-Out | 1 | R3 | Choreography, dedupe to humans, multi-instance fan-out | [M13](milestones/M13-notification-fanout.md) |
| M14 | Control-Tower Query Service (CQRS) | 1 | R4 | CQRS read side, API composition, staleness | [M14](milestones/M14-query-service-cqrs.md) |
| M15 | Control-Tower UI & MVP | 1 | R4 | Eventual consistency in UX, E2E across services | [M15](milestones/M15-control-tower-ui-mvp.md) |
| M16 | Observability | 1 | R5 | Distributed tracing over async, RED, SLOs, lag | [M16](milestones/M16-observability.md) |
| M17 | Contract Testing, Chaos & Load | 1 | R5 | Consumer-driven contracts, chaos engineering, scaling limits | [M17](milestones/M17-contracts-chaos-load.md) |
| M18 | Kubernetes Deployment | 1 | R5 | Service discovery, probes, rolling deploys, autoscaling | [M18](milestones/M18-kubernetes.md) |
| M19 | Forecasting Service (Polyglot) | 1 | R6 | Polyglot contracts, graceful degradation | [M19](milestones/M19-forecasting-service.md) |
| M20 | AI Assistant (Safe Tool Use) | 1 | R6 | Identity propagation, trust boundaries | [M20](milestones/M20-ai-assistant.md) |
| M21 | Release v1.0 & Consolidation | 1 | R6 | Architecture retrospective, communication | [M21](milestones/M21-release.md) |
| — | Buffer | 2 | — | Absorbs overruns | — |

### 4.1 Releases and gates

| Release | Milestones | Gate demo |
|---|---|---|
| **R0 Foundation** | M00–M04 | **G0:** quality gates block a bad PR; a planner logs in via the gateway, creates a PO, and cannot read another tenant's PO |
| **R1 Core Domain** | M05–M06 | **G1:** no overselling under 100 concurrent requests; the lost-event (dual-write) bug is demonstrated live |
| **R2 Reliable Eventing** | M07–M08 | **G2:** `scripts/drills/reliability` kills Kafka and relays and injects duplicates, poison and stale events, and the final state is always correct |
| **R3 Disruption Flow** | M09–M13 | **G3:** a 5-day delay → risk → plan → approval → saga (and compensation on failure) → alerts, all via API |
| **R4 Control Tower MVP** | M14–M15 | **G4 — MVP COMPLETE:** the full flagship scenario in the browser from a clean clone |
| **R5 Production Ready** | M16–M18 | **G5:** on Kubernetes, with one trace across all services, and a chaos suite that passes |
| **R6 Intelligence & Launch** | M19–M21 | **G6:** v1.0 tagged, demo video, docs complete |

### 4.2 Dependency graph

```
M00 ─▶ M01 ─▶ M02 ─▶ M03 ─┬─▶ M04
                          ├─▶ M05 ─┐
                          └─▶ M06 ─┴─▶ M07 ─▶ M08 ─▶ M09 ─▶ M10 ─▶ M11 ─▶ M12 ─┬─▶ M13
                                                                              └─▶ M14 ─▶ M15 (MVP)
M15 ─▶ M16 ─▶ M17 ─▶ M18 ─▶ M19 ─▶ M20 ─▶ M21
```

**Critical path to MVP:** M00 → M01 → M02 → M03 → M05 → M06 → M07 → M08 → M09 → M10 → M11 → M12 → M14 → M15.
M04 and M13 are required for the MVP but can be done in parallel with or after neighbouring milestones if needed.

---

## 5. Target Architecture (Baseline)

```
 Angular UI ──▶ API Gateway (JWT, rate limit, correlation-id) ──▶ services
   Procurement · Inventory · Shipment · Risk · Recovery · Notification · Query · (Intelligence) · (Forecasting/Python)
   each with its own Postgres database + outbox ──▶ Kafka (+ Schema Registry) ──▶ consumers
 Keycloak (OIDC) · Redis · OpenTelemetry → Tempo/Jaeger · Prometheus · Loki · Grafana
```

| Service | Port | Owns |
|---|---|---|
| gateway | 8080 | — |
| procurement-service | 8081 | Supplier, SupplierProduct, PurchaseOrder, POLine |
| inventory-service | 8082 | Warehouse, Product, InventoryBalance, StockMovement, Reservation |
| shipment-service | 8083 | Shipment, ShipmentEvent |
| risk-service | 8084 | Projections, RiskAssessment, SimulationRun |
| recovery-service | 8085 | RecoveryPlan, Approval, SagaInstance |
| notification-service | 8086 | Notification, DeliveryAttempt |
| query-service | 8087 | Read models |
| intelligence-service | 8088 | Assistant sessions, tool-call audit |
| forecasting-service (Python) | 8090 | ForecastRun, model metadata |

**Infrastructure:** Keycloak 8180 · Postgres 5432 · Kafka 9092 · Schema Registry 8181 · Redis 6379 · Mailpit 8025 · Prometheus 9090 · Grafana 3000 · Jaeger 16686 · Loki 3100 · Toxiproxy 8474.

| Topic | Producer | Consumers |
|---|---|---|
| `procurement.events` | Procurement | Risk, Recovery, Query |
| `inventory.events` | Inventory | Risk, Query |
| `shipment.events` | Shipment | Risk, Notification, Query |
| `risk.events` | Risk | Recovery, Notification, Query |
| `recovery.events` | Recovery | Notification, Query |
| `inventory.commands`, `shipment.commands`, `procurement.commands` | Recovery | owning service |
| `recovery.replies` | saga participants | Recovery |
| `<topic>.<group>.DLT` | error handlers | replay tool |

---

## 6. Concept → Milestone Traceability

| Concept | Built | Proven (break test) |
|---|---|---|
| Coding standards, architecture fitness functions | M00 | M00 (violations are rejected) |
| Bounded contexts, service boundaries | M01 | M21 retrospective |
| Database per service, 12-factor | M02 | M02 |
| API contracts, Problem Details, state machines | M03 | M03 |
| OIDC/JWT, zero trust, tenant isolation, rate limiting | M04 | M04 |
| Optimistic / pessimistic / atomic updates | M05 | M05 |
| Idempotent HTTP commands | M05 | M05 |
| Kafka partitions, keys, ordering, rebalancing, offsets | M06 | M06 |
| Exactly-once semantics and its limits | M06 | M06 |
| Dual-write problem | M06 | M06 |
| Transactional outbox, SKIP LOCKED | M07 | M07 |
| Idempotent consumer (inbox) | M07 | M07, M12 |
| Retries, backoff, DLT, replay | M08 | M08 |
| Out-of-order / stale events | M08 | M08 |
| Schema registry and evolution | M08 | M08, M17 |
| CDC (Debezium) | M08 | M08 |
| Temporal coupling, cascading failure | M09 | M09 |
| Timeout, retry, circuit breaker, bulkhead | M09 | M09, M17 |
| Event-carried state transfer, projections, replay | M09 | M09, M14 |
| Deterministic, explainable processing | M10 | M10 |
| Process manager, human-in-the-loop | M11 | M11 |
| Saga orchestration, compensation, timeouts, semantic locks | M12 | M12 |
| Choreography | M13 | M13 |
| Multi-instance fan-out | M13 | M13 |
| CQRS read models, staleness | M14 | M14 |
| Eventual consistency in UX | M15 | M15 |
| Distributed tracing across async boundaries, SLOs | M16 | M16 |
| Consumer-driven contract testing | M17 | M17 |
| Chaos engineering, scaling limits | M17 | M17 |
| Service discovery, probes, rolling deploys, autoscaling | M18 | M18 |
| Polyglot contracts, graceful degradation | M19 | M19 |
| Identity propagation, trust boundaries | M20 | M20 |

---

## 7. Corrections Applied to the Blueprint

| # | Blueprint gap | Applied in |
|---|---|---|
| 1 | AI in weeks 7–8 crowds out distributed-systems work | Moved to M19–M20 |
| 2 | No codebase standards | **New M00** |
| 3 | No owner for "commitments" | `Reservation(commitmentRef, customerRef, qty, requiredByDate)` in Inventory (M05) |
| 4 | No owner for the recovery saga | New Recovery service (M11–M12) |
| 5 | No owner for dashboard reads | New Query service (M14) |
| 6 | Gateway not designed | M04 |
| 7 | Unclear how Risk gets its data | Sync version built and measured, then replaced by projections (M09) |
| 8 | No schema registry; only one outbox approach | M08 |
| 9 | Observability limited to metrics | Traces + logs across Kafka (M16) |
| 10 | Kubernetes deferred | Mandatory M18 |
| 11 | Old versions (Spring Boot 3.5) | Java 25 LTS (or 21), Spring Boot 4.0.x, Kafka 4.x (KRaft). Verify and pin in M00 |
| 12 | Risk depends on forecasting that comes later | Deterministic `DemandSource` first; forecast swapped in during M19 |
| 13 | Laptop memory | Compose profiles, JVM limits, one Postgres server (M02) |

---

## 8. Schedule Control

**Slip rule:** if a milestone overruns by more than 50%, stop and choose: move its *P1* tasks to the backlog, or spend buffer. Record the decision in the journal.

**Cut order** (if the schedule must shrink):
1. M20 AI assistant
2. M19 forecasting (keep the deterministic DemandSource)
3. M18 autoscaling, then the rest of M18
4. M13 Redis backplane (single-instance SSE)
5. M08 Debezium comparison
6. M15 UI polish (keep the functional pages)

**Never cut:** M00 standards, outbox, inbox/idempotency, concurrency tests, saga compensation, DLT + replay, basic tracing, or **any break test**.

| Risk | Likelihood | Impact | Mitigation |
|---|---|---|---|
| Laptop RAM exhausted | High | High | Compose profiles, `-Xmx`, one Postgres server, stop `obs` while coding |
| Saga milestones overrun | High | Med | Split into M11 and M12; buffer reserved |
| Keycloak/Spring Security time sink | Med | Med | Committed realm export; 1-day timebox before asking for help |
| Version incompatibilities | Med | Med | BOMs and parent-POM version properties pinned in M00; upgrades deliberate |
| UI eats the schedule | Med | Med | Hard 20 h cap in M15 |
| Motivation dip | Med | High | A demo gate every 2–5 milestones; tags show visible progress |

---

## 9. Tracking

- GitHub Project board: *Backlog → Ready → In progress → Break-tested → Done*.
- One GitHub milestone per `Mxx`; one issue per task `Mxx-Tnn`; labels `concept:*`, `service:*`, `type:build|break|doc`.
- Tags: `m03-procurement` at each milestone; `g4-mvp` at each gate; `v1.0.0` at release.
- Saturday 30-minute review: hours vs estimate, cut decisions, unclear concepts.

## 10. Start Here

Open [M00 — Engineering Standards & Codebase Guidelines](milestones/M00-engineering-standards.md).
