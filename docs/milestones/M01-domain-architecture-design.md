# M01 — Domain & Architecture Design

| | |
|---|---|
| **Release** | R0 Foundation |
| **Duration** | 0.5 week (~12 h) |
| **Depends on** | M00 |
| **Unlocks** | M02 and every service milestone |
| **Tag** | `m01-design` |

---

## 1. Why this milestone exists

The most expensive microservices mistake is **wrong boundaries**. Split one business capability across two services and every change needs two deployments plus a chatty sync call, which is a *distributed monolith*. Boundaries must come from the **domain** (who owns which decisions and data), not from database tables or UI screens.

This milestone is short and practical: you design *NexusFlow's* boundaries and contracts, then start coding in M02. The deeper theory (consistency models, CAP) comes later, when it bites you.

---

## 2. Learning objectives

- Run an EventStorming session solo and turn it into bounded contexts.
- Justify every service boundary with a business capability and data ownership.
- Draw a context map showing upstream/downstream relationships.
- Decide, per interaction, **sync (REST) vs async (events/commands)**, and say why.
- Produce C4 Level 1 and 2 diagrams and an event catalogue that later milestones implement.

---

## 3. Concepts

### 3.1 Bounded context
**What:** a boundary inside which a model and its language are consistent. "Reservation" in Inventory means *stock set aside for a commitment*; in a travel system it means something else.
**Why we use it:** it is the natural unit for a microservice. One team (or one service) owns the model, its data and its rules.
**In NexusFlow:** Procurement, Inventory, Shipment, Risk, Recovery, Notification. Query is a *read-side* context.

### 3.2 Aggregate
**What:** a cluster of objects changed together in one transaction, with one root (e.g. `PurchaseOrder` + its lines).
**Why:** an aggregate is the **unit of consistency** (strongly consistent inside, eventually consistent between aggregates) and the natural **Kafka partition key**.
**Rule:** one transaction modifies one aggregate. Anything wider needs events or a saga.

### 3.3 Domain events vs integration events vs commands
- **Domain event:** something that happened inside the service (may stay internal).
- **Integration event:** a published fact for other services (`ShipmentDelayed`). It's a contract, so it is versioned.
- **Command:** a request for a specific service to do something (`ReallocateReservation`). It has one recipient and may be rejected.
**Why the distinction:** events decouple (the producer doesn't know its consumers); commands couple (the sender knows the receiver). Use each on purpose.

### 3.4 Context map
**What:** a diagram of the relationships between contexts: upstream/downstream, customer/supplier, conformist, anti-corruption layer (ACL), published language.
**Why:** it shows the direction of coupling. NexusFlow's event envelope and schemas are its **published language**; Risk is downstream of Procurement, Inventory and Shipment.

### 3.5 Sync vs async communication
| | Sync (REST) | Async (events via Kafka) |
|---|---|---|
| Coupling | Temporal: both must be up | Decoupled in time |
| Consistency | Immediate answer | Eventual |
| Use when | The user needs an answer now; queries; commands needing validation feedback | Notifying others of facts; long workflows; fan-out |
| Failure mode | Cascading failures | Lag, duplicates, ordering issues |

**NexusFlow rules (an ADR):**
1. UI → services: sync (via the gateway).
2. Service → service *state propagation*: async events.
3. Saga steps: async commands + replies.
4. Service → service *sync queries*: allowed only when fresh data is required at decision time (e.g. Recovery checking current stock), always with timeout + circuit breaker.

### 3.6 Microservices vs modular monolith (be honest)
For one developer, a modular monolith is usually the better *engineering* choice. You choose microservices here **to learn their problems**. Writing that honestly in ADR-0001 shows senior judgement. Interviewers love the answer "I know when *not* to use them."

### 3.7 Conway's law
Systems mirror the communication structure of the organisation that builds them. As a solo developer, you simulate "teams" by enforcing boundaries with tooling (ArchUnit rule 6, separate DB users).

---

## 4. Scope

**In:** EventStorming, bounded contexts, context map, C4 L1/L2, the event and command catalogue, service ownership table, 3 user journeys, the ADRs.
**Out:** detailed API specs (each service milestone), code.

---

## 5. Technical specification (the design you'll produce)

### 5.1 Flagship scenario to storm
> PO-1001 from supplier "Acme Metals" for 500 units of P-104, shipment SH-77 planned ETA 10 Nov. Warehouse W1 has 120 available units of P-104; reservations exist for customer orders due 12 Nov and 14 Nov. The carrier reports a 5-day delay (new ETA 15 Nov).

### 5.2 Expected event catalogue (validate it in your session; adjust if you disagree, and record why)

| Context | Integration events | Commands handled (from M12) |
|---|---|---|
| Procurement | `SupplierRegistered`, `SupplierProductListed`, `PurchaseOrderConfirmed`, `PurchaseOrderUpdated`, `PurchaseOrderCancelled`, `PurchaseOrderReceived` | `CreateEmergencyPurchaseOrder`, `CancelPurchaseOrder` |
| Inventory | `StockReceived`, `StockLevelChanged`, `InventoryBelowThreshold`, `ReservationCreated`, `ReservationReleased`, `ReservationReallocated` | `ReallocateReservation`, `ReleaseReallocation` |
| Shipment | `ShipmentRegistered`, `ShipmentDeparted`, `ShipmentDelayed`, `ShipmentArrived`, `ShipmentExpedited` | `ExpediteShipment`, `CancelExpedite` |
| Risk | `RiskAssessmentCompleted`, `RiskResolved` | — |
| Recovery | `RecoveryPlanProposed`, `RecoveryPlanApproved`, `RecoveryPlanRejected`, `RecoveryPlanCompleted`, `RecoveryPlanCompensated`, `RecoveryPlanFailed` | — |
| Notification | — (consumer only) | — |

For each event, document: producer, consumers, partition key, key payload fields, and which business rule emits it.

### 5.3 Ownership table
| Data | Owner | Who else needs it | How they get it |
|---|---|---|---|
| Supplier, supplier products, lead times | Procurement | Risk, Recovery | Events → local projection |
| PO + lines, expected dates | Procurement | Risk, Shipment, Query | Events |
| Stock balances, reservations (commitments) | Inventory | Risk, Recovery, Query | Events; Recovery also queries sync at decision time |
| Shipment, ETAs | Shipment | Risk, Query, Notification | Events |
| Risk assessments | Risk | Recovery, Notification, Query | Events |
| Recovery plans, approvals, audit | Recovery | Notification, Query | Events |

### 5.4 User journeys (write each as numbered steps with the services touched)
1. **Planner detects risk:** delay reported → sees the risk on the dashboard → opens the evidence.
2. **Manager resolves risk:** compares options → approves → watches execution → sees the audit.
3. **Analyst investigates:** filters past disruptions → reads the timeline of events and decisions.

### 5.5 C4 diagrams
- **L1 Context:** users (Planner, Manager, Analyst), NexusFlow, Keycloak, Email (Mailpit), future carrier/ERP systems.
- **L2 Container:** UI, Gateway, the 7 services (+2 later), Postgres DBs, Kafka, Schema Registry, Redis, observability stack. Annotate arrows with *sync/async* and the topic names.

---

## 6. Tasks

| ID | Task | Est | Concept | P |
|---|---|---|---|---|
| M01-T01 | EventStorming (Miro/Excalidraw/paper): orange events on a timeline → blue commands → yellow aggregates → purple policies ("when X then Y") → green read models → pink external systems | 3h | Events, aggregates, policies | P0 |
| M01-T02 | Group into bounded contexts; write the ownership table (§5.3) | 1.5h | Bounded contexts, data ownership | P0 |
| M01-T03 | Event and command catalogue in `contracts/events/README.md` (§5.2) | 2h | Integration events vs commands | P0 |
| M01-T04 | Context map (Mermaid) | 1h | Upstream/downstream, published language | P0 |
| M01-T05 | C4 L1 and L2 (Structurizr DSL or Mermaid) in `docs/architecture/` | 2h | Container view of a distributed system | P0 |
| M01-T06 | User journeys (§5.4) | 1h | — | P0 |
| M01-T07 | ADRs: *Why microservices (learning goal; monolith trade-off)*, *Service boundaries*, *Sync vs async communication rules* | 1.5h | Trade-off reasoning | P0 |

---

## 7. Break tests (design reviews)

Design can't be "run," so it is stress-tested with scenarios. For each, walk the design and write down the result:

| # | Scenario | Expected outcome |
|---|---|---|
| B1 | "Add a new consumer of shipment delays (e.g. a billing service)." | No change to Shipment. Only a new consumer group. |
| B2 | "Inventory is down for 10 minutes." | Risk still assesses (from projections); only inventory *commands* fail. |
| B3 | "Change how PO lead time is calculated." | Only Procurement deploys. |
| B4 | "Which service decides if a commitment is at risk?" | Exactly one answer: Risk. |
| B5 | "Two services both want to write `reservation.status`." | Must be impossible. Only Inventory writes it. |

If any scenario needs changes in 2+ services, or has two owners, **fix the boundary now**.

---

## 8. Pitfalls

- **Entity services** ("SupplierService", "ProductService") instead of capability services: this leads to chatty CRUD and a distributed monolith.
- **Shared "common" data models** for events: each consumer maps the published schema into its own model.
- **Notification or Risk "reaching into" other databases.** Never.
- **Spending more than 2 days here.** You will refine boundaries later via ADRs; that's normal.

---

## 9. Deliverables

`docs/architecture/eventstorming.png|md`, `context-map.md`, `c4-context`, `c4-container`; `contracts/events/README.md`; `docs/architecture/user-journeys.md`; 3 ADRs; tag `m01-design`.

## 10. Self-check

1. Why is an aggregate the right Kafka partition key?
2. When is a sync call between services acceptable in NexusFlow? Give the example.
3. Event vs command: which one couples the sender to the receiver, and why?
4. Give two signs of a distributed monolith.
5. Why would a modular monolith be a better *engineering* choice here, and why are you not using one?

## 11. References (when stuck)

*Learning Domain-Driven Design* (Khononov) ch. 1–4, 11 · *Building Microservices* 2e ch. 1–3 · Alberto Brandolini, *Introducing EventStorming* · c4model.com · Martin Fowler, *What do you mean by "Event-Driven"?*

## 12. Exit checklist

- [ ] Every entity and event has exactly one owner
- [ ] B1–B5 design reviews pass
- [ ] Event catalogue complete with keys
- [ ] C4 L1/L2 + context map committed
- [ ] 3 ADRs written · tag `m01-design`
