# M06 — Shipment Service, Kafka Hands-On & the Dual-Write Problem

| | |
|---|---|
| **Release** | R1 Core Domain |
| **Duration** | 1 week (~24 h) |
| **Depends on** | M03 (template), M05 |
| **Unlocks** | M07. **Gate G1** |
| **Tag** | `m06-shipment-kafka`, `g1-core-domain` |

---

## 1. Why this milestone exists

This is where NexusFlow becomes *event-driven*. You'll publish your first real events and learn Kafka's semantics **on your own topic**: partitions, keys, ordering, consumer groups, rebalancing, offsets, delivery guarantees. Then you'll deliberately build event publishing the **wrong** way and prove that it loses data. That bug is the dual-write problem, and seeing it is what makes M07's outbox pattern make sense.

---

## 2. Learning objectives

- Explain topics, partitions, offsets, consumer groups and how they give both ordering and parallelism.
- Choose a partition key and explain the ordering guarantee it provides (and what it doesn't).
- Explain at-most-once, at-least-once and exactly-once, and where each comes from (commit timing).
- Explain rebalancing and the `max.poll.interval.ms` trap.
- Explain the **dual-write problem** and demonstrate both failure modes.
- Explain why Kafka's exactly-once (transactions) does not cover your Postgres database.

---

## 3. Concepts

### 3.1 Kafka's model in five facts
1. A **topic** is split into **partitions**; each partition is an append-only, ordered log.
2. A producer picks the partition by **key hash** (same key → same partition → ordered). No key → spread across partitions (no ordering).
3. A **consumer group** shares partitions: each partition is read by exactly one consumer in the group. Max useful parallelism = partition count.
4. Consumers track progress by committing **offsets**. Where you commit relative to processing decides your delivery semantics.
5. Messages are retained by time/size, not deleted on consumption, so they can be replayed (used heavily in M09/M14).

### 3.2 Delivery semantics come from commit timing
| Commit… | Crash between process & commit → | Semantics |
|---|---|---|
| before processing | message skipped | at-most-once (loss) |
| after processing | message reprocessed | **at-least-once (duplicates)**, which is what we use |
| atomically with output (Kafka transactions, Kafka→Kafka only) | neither | exactly-once *within Kafka* |
**Takeaway:** at-least-once + **idempotent consumers** (M07) is the practical standard.

### 3.3 Rebalancing
When consumers join, leave or are considered dead, partitions are reassigned. With eager rebalancing every consumer stops; with **cooperative-sticky** only the moved partitions pause. A consumer is considered dead if it doesn't call `poll()` within `max.poll.interval.ms`. Slow processing → kicked out → rebalance → it rejoins → it's slow again: a **rebalance storm**.

### 3.4 Event envelope
Defined in M00 §5.5. Implemented now in `platform-messaging` as a generic `EventEnvelope<T>` record. JSON on the wire for now (schema registry in M08).

### 3.5 The dual-write problem
A use case must (1) update the DB and (2) publish an event. These are two different systems with **no shared transaction**:
- `save()` then `send()`: crash or Kafka down after the commit → **event lost**. The DB says "delayed," and nobody downstream ever knows.
- `send()` then `save()`: DB failure after the send → **phantom event** about a change that never happened.
- Wrapping both in `@Transactional`: it doesn't help. Kafka doesn't participate in the DB transaction.
**Solutions (M07):** transactional outbox, or CDC.

### 3.6 Why not 2PC/XA?
Kafka doesn't support XA. Even where available, 2PC blocks on coordinator failure and couples availability. This is the first time you meet "distributed transactions are avoided," and it comes back in M12.

---

## 4. Scope

**In:** Shipment service (model, API, carrier-event ingestion with dedupe), `EventEnvelope`, naive publishing, a Risk-service *stub* consumer, Kafka drills, dual-write drills, a Kafka-transactions mini-drill.
**Out:** outbox/inbox (M07), retries/DLT (M08).

---

## 5. Technical specification

### 5.1 Model
```
Shipment(id, tenantId, purchaseOrderId, carrier, mode[SEA|AIR|ROAD], status[PLANNED|IN_TRANSIT|DELAYED|ARRIVED|CANCELLED],
         plannedEta: LocalDate, currentEta: LocalDate, version)
ShipmentEvent(id, shipmentId, providerEventId UNIQUE NULLABLE, type[DEPARTED|DELAY_REPORTED|ETA_UPDATED|ARRIVED], occurredAt, reasonCode, newEta, source[USER|CARRIER])
```
Rule: a `DELAY_REPORTED` or `ETA_UPDATED` that moves `currentEta` later emits **`ShipmentDelayed`** with `previousEta`, `revisedEta`, `delayDays`, `reasonCode`.

### 5.2 API
| Method | Path | Notes |
|---|---|---|
| POST | `/api/v1/shipments` | register (links to the PO by ID) |
| POST | `/api/v1/shipments/{id}/events` | carrier/user event; dedupe on `providerEventId` (duplicate → 200 with the existing event) |
| GET | `/api/v1/shipments/{id}` / `?status=&etaFrom=` | queries |

### 5.3 Topic
`shipment.events`, 3 partitions, key = `shipmentId`, retention 7 days.

### 5.4 Stub consumer
In `risk-service` (create the module now): `@KafkaListener(topics="shipment.events", groupId="risk")` that logs `partition`, `offset`, `key`, `eventType`, `aggregateVersion`, and sleeps a configurable time (for drills).

---

## 6. Tasks

| ID | Task | Est | Concept | P |
|---|---|---|---|---|
| M06-T01 | Shipment service from the template: model, migrations, API, provider-event dedupe (unique constraint) | 4h | Idempotent ingestion of external events | P0 |
| M06-T02 | `platform-messaging`: `EventEnvelope<T>`, JSON serde config, `aggregateVersion` from the entity version | 2h | Integration event contract | P0 |
| M06-T03 | **Naive publishing:** `repository.save()` then `kafkaTemplate.send()` | 1h | (sets up the break) | P0 |
| M06-T04 | Risk-service stub consumer + `kafka-ui` inspection | 1h | Consumer groups | P0 |
| M06-T05 | **Drill A, keys:** publish 30 events across 3 shipments with no key, then with key = shipmentId; compare per-shipment ordering in the consumer log | 1.5h | Partitioning & ordering | P0 |
| M06-T06 | **Drill B, rebalance:** run 3 stub instances; kill one; observe the reassignment; switch to `cooperative-sticky` and compare pauses | 1.5h | Rebalancing | P0 |
| M06-T07 | **Drill C, commit timing:** auto-commit + crash mid-batch → find the lost messages; manual `AckMode.RECORD` after processing + crash → duplicates | 2h | Delivery semantics | P0 |
| M06-T08 | **Drill D, slow consumer:** processing time > `max.poll.interval.ms` → rebalance storm; fix with smaller `max.poll.records` / a correct interval | 1.5h | Liveness of consumers | P0 |
| M06-T09 | **Drill E, over-provisioning:** 4 consumers, 3 partitions | 0.5h | Parallelism limit | P0 |
| M06-T10 | **Dual-write drill 1:** stop Kafka, report a delay, start Kafka | 1h | Dual write: loss | P0 |
| M06-T11 | **Dual-write drill 2:** send first, then force a DB failure (e.g. a constraint violation after the send) | 1h | Dual write: phantom | P0 |
| M06-T12 | **Dual-write drill 3:** wrap both in `@Transactional` and repeat drill 1, proving it doesn't help | 0.5h | Transaction boundaries | P0 |
| M06-T13 | Kafka transactions mini-drill: read `shipment.events` → write to `lab.output` transactionally; consumer with `read_committed` vs `read_uncommitted` | 2h | EOS scope | P1 |
| M06-T14 | Write `docs/architecture/dual-write-problem.md` (problem statement, both failure sequences as diagrams, candidate solutions) | 1.5h | — | P0 |
| M06-T15 | Seed: flagship shipment SH-77 for PO-1001 | 0.5h | — | P0 |

---

## 7. Break tests

All drills (A–E, dual-write 1–3) are executed with a **written prediction vs actual** in the journal. Acceptance:
- [ ] You can show the per-key ordering difference from the logs.
- [ ] You reproduced *both* a lost and a duplicated message by commit timing alone.
- [ ] You reproduced a lost event *and* a phantom event from dual writes.
- [ ] The provider-event dedupe returns the same result for a duplicate carrier event.

> ✅ **Gate G1:** (1) Inventory concurrency tests prove no overselling; (2) live demo: Kafka down → delay recorded → event lost forever.

---

## 8. Pitfalls

- Using `kafkaTemplate.send()` without handling the returned future: failures vanish silently. (Even so, the naive version is broken. That's the point.)
- Assuming ordering across *different* shipments. Only per-key ordering exists.
- Changing the partition count later: key→partition mapping changes and ordering breaks during the transition.
- Spring Kafka's default error handler retrying forever by default in some configurations. You'll configure it in M08; for now, know it's there.

## 9. Deliverables

Shipment service, `EventEnvelope`, risk stub, `docs/architecture/dual-write-problem.md`, drill notes, ADR *Topic design & partition keys*, tags `m06-shipment-kafka`, `g1-core-domain`.

## 10. Self-check

1. How do partition count, ordering and consumer parallelism relate?
2. Where do duplicates come from with "commit after processing"?
3. What causes a rebalance storm and how do you prevent it?
4. Explain the dual-write problem with both failure orders.
5. Why can't Kafka transactions make Postgres + Kafka atomic?
6. Why not use 2PC?

## 11. References (when stuck)

*Kafka: The Definitive Guide* 2e ch. 3–4 (producers/consumers), ch. 8 (exactly-once) · DDIA ch. 11 · Spring for Apache Kafka reference (listener containers, AckMode) · Confluent *Kafka 101* · microservices.io *Transactional outbox* (the problem section)

## 12. Exit checklist

- [ ] All drills done with predictions; dual-write doc written
- [ ] Shipment API + dedupe tests green
- [ ] ADR written · Gate G1 demo · tags pushed
