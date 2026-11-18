# M07 — Transactional Outbox & Idempotent Consumers

| | |
|---|---|
| **Release** | R2 Reliable Eventing |
| **Duration** | 1 week (~23 h) |
| **Depends on** | M06 |
| **Unlocks** | M08 and every event-driven milestone after it |
| **Tag** | `m07-outbox-inbox` |

---

## 1. Why this milestone exists

M06 proved that "save then publish" loses events and "publish then save" invents them. This milestone fixes both **permanently, in the chassis**, so no service can make the mistake again. On the receiving side, at-least-once delivery means duplicates *will* arrive, so every consumer needs to be idempotent. Together, outbox + inbox give you **effectively-once processing** without distributed transactions. This is the most important pattern pair in event-driven microservices.

---

## 2. Learning objectives

- Implement the transactional outbox and explain exactly why it is atomic.
- Implement a polling relay that is safe with multiple instances (`FOR UPDATE SKIP LOCKED`), and explain its ordering trade-off.
- Implement the idempotent-consumer (inbox) pattern and explain why the inbox write must share the business transaction.
- List every source of duplicates in the pipeline.
- Explain when you need leader election and when `SKIP LOCKED` removes the need.

---

## 3. Concepts

### 3.1 Transactional outbox
In **one local DB transaction**: update the business tables *and* insert the event into an `outbox` table. A separate **relay** reads unsent outbox rows, publishes them to Kafka, then marks them sent.
**Why it works:** the event exists **if and only if** the business change committed. Atomicity comes from the local DB transaction, which you already trust.
**Cost:** publication latency (polling interval), an extra table and a relay process, and duplicates (below).

### 3.2 Why the relay produces duplicates
Relay publishes → Kafka acks → relay crashes **before** marking the row sent → after restart it publishes again. You cannot close this gap. **So consumers must be idempotent.**

### 3.3 `SELECT … FOR UPDATE SKIP LOCKED`
Multiple relay instances each grab a batch of rows; rows locked by one instance are *skipped* by the others instead of blocking. It gives work distribution without a leader.
**Ordering trap:** two relays can publish events of the **same aggregate** out of order (relay A gets v1, relay B gets v2, B is faster). Options:
1. One relay instance (simple; fine for NexusFlow's volume). Use a leader lock (ShedLock / `pg_try_advisory_lock`) so only one runs.
2. Partition work by aggregate hash (each relay owns hash buckets).
3. Accept reordering and rely on consumer-side version checks (M08).
**Choose and document.** Recommended: option 1 + option 3 as defence in depth.

### 3.4 Idempotent consumer (inbox)
Consumer transaction: `INSERT INTO processed_events(event_id, consumer_group) …` → if it violates the unique key, it's a duplicate, so skip; otherwise apply the side effects → commit → then commit the Kafka offset.
**Why the same transaction:** if the side effects commit but the inbox row doesn't (or vice versa), a crash leads to either a double application or a lost one.
**Alternative:** naturally idempotent operations (upserts, "set status = X if version < N"). Use them where possible; the inbox is the general solution.

### 3.5 Sources of duplicates (memorise)
1. Producer retries (mitigated by the idempotent producer, `enable.idempotence=true`).
2. Outbox relay crash after publish, before mark-sent.
3. Consumer crash after processing, before offset commit.
4. Rebalance during processing.
5. Manual replays (DLT replay in M08, projection rebuilds in M09).
6. Upstream clients retrying commands (handled by idempotency keys, M05).

### 3.6 Leader election / distributed locks
Needed when exactly one instance must do something (a singleton scheduler). Options: DB advisory locks, ShedLock (DB table), and later Kubernetes Lease objects. Every lock needs a **lease/timeout**; otherwise a crashed leader holds it forever. Also note **fencing tokens**: a paused old leader may wake up and act. That's why idempotent, versioned writes still matter.

### 3.7 Alternative: CDC
The relay can instead read the Postgres WAL (Debezium). You'll compare it in M08.

---

## 4. Scope

**In:** `platform-messaging` outbox (writer, relay, cleanup, metrics), inbox consumer support, converting Shipment, Procurement and Inventory to emit their catalogue events via the outbox, an ArchUnit rule banning direct `KafkaTemplate` use in application code, drills.
**Out:** retry/DLT policy, schemas, ordering checks (M08).

---

## 5. Technical specification

### 5.1 Tables (per service)
```sql
CREATE TABLE outbox (
  id              uuid PRIMARY KEY,          -- = eventId
  aggregate_type  text NOT NULL,
  aggregate_id    uuid NOT NULL,              -- = Kafka key
  aggregate_version bigint NOT NULL,
  event_type      text NOT NULL,
  topic           text NOT NULL,
  payload         jsonb NOT NULL,             -- full envelope
  headers         jsonb NOT NULL DEFAULT '{}',-- correlationId, traceparent (M16)
  created_at      timestamptz NOT NULL,
  published_at    timestamptz NULL,
  attempts        int NOT NULL DEFAULT 0,
  last_error      text NULL
);
CREATE INDEX outbox_unpublished ON outbox (created_at) WHERE published_at IS NULL;

CREATE TABLE processed_events (
  event_id       uuid NOT NULL,
  consumer_group text NOT NULL,
  processed_at   timestamptz NOT NULL,
  PRIMARY KEY (event_id, consumer_group)
);
```

### 5.2 Outbox relay algorithm
```
every 200 ms (configurable), if leader:
  BEGIN
    rows = SELECT * FROM outbox WHERE published_at IS NULL
           ORDER BY created_at, aggregate_version LIMIT 100 FOR UPDATE SKIP LOCKED
    for row in rows:                                   -- in order
      send(topic, key=aggregate_id, value=payload, headers).get(timeout)   -- sync ack
      UPDATE outbox SET published_at = now() WHERE id = row.id
      (on failure: attempts++, last_error, stop batch to preserve order)
  COMMIT
cleanup job: delete published rows older than 7 days
metrics: nexusflow_outbox_pending_events, nexusflow_outbox_publish_latency_seconds, nexusflow_outbox_failures_total
```

### 5.3 API for application code (port)
`DomainEventPublisher.publish(aggregate, event)` → the outbox adapter writes the row **in the caller's transaction** (`Propagation.MANDATORY`, so calling it outside a transaction fails fast).

### 5.4 Consumer support
`IdempotentEventHandler<T>` wrapper: `@Transactional` → insert into `processed_events` (ON CONFLICT DO NOTHING; if 0 rows, skip and log at DEBUG) → delegate → commit. Listener ack mode `RECORD` after the handler returns.

### 5.5 Events to convert now
- Procurement: `PurchaseOrderConfirmed/Updated/Cancelled/Received`, `SupplierProductListed`
- Inventory: `StockReceived`, `StockLevelChanged`, `InventoryBelowThreshold`, `ReservationCreated/Released`
- Shipment: `ShipmentRegistered/Departed/Delayed/Arrived`

---

## 6. Tasks

| ID | Task | Est | Concept | P |
|---|---|---|---|---|
| M07-T01 | Outbox table migration template + `OutboxEventPublisher` (MANDATORY propagation) | 3h | Transactional outbox | P0 |
| M07-T02 | Relay with SKIP LOCKED + sync send + mark-sent + stop-on-failure | 4h | Relay mechanics | P0 |
| M07-T03 | Leader lock for the relay (ShedLock or advisory lock) + cleanup job | 2h | Leader election, leases | P0 |
| M07-T04 | Convert Shipment; re-run the M06 dual-write drills | 1.5h | — | P0 |
| M07-T05 | Convert Procurement + Inventory events (§5.5) | 4h | State changes as integration events | P0 |
| M07-T06 | Inbox: `processed_events` + `IdempotentEventHandler`; apply it to the risk stub | 3h | Idempotent consumer | P0 |
| M07-T07 | ArchUnit rule: `KafkaTemplate` only in `platform-messaging` | 0.5h | Fitness function | P0 |
| M07-T08 | Outbox metrics | 1h | Observability of async pipelines | P1 |
| M07-T09 | Drills + automation (Testcontainers + Toxiproxy/`docker pause`) | 3h | — | P0 |
| M07-T10 | ADRs: *Outbox via polling relay + single leader*, *Idempotent consumer strategy* | 1h | — | P0 |

---

## 7. Break tests

| # | Drill | Expected |
|---|---|---|
| B1 | M06 dual-write drill 1 (Kafka down during a delay report) | The delay is committed; the event is published after Kafka returns. **No loss** |
| B2 | M06 drill 2 (DB failure) | No phantom event (the outbox row rolled back with the business change) |
| B3 | Kill the relay after send, before mark-sent (inject a failure point) | The event is published twice; the consumer applies it **once** |
| B4 | Inject the same event 3× directly into the topic | One side effect; 2 DEBUG "duplicate skipped" |
| B5 | Kill the consumer after the side-effect commit, before the offset commit | Redelivered → skipped by the inbox |
| B6 | Run 2 relay instances | Only the leader publishes; kill the leader → the other takes over after lease expiry |
| B7 | Disable the leader lock, 2 relays, 1 aggregate updated rapidly 20× | Observe possible out-of-order versions in the consumer log (motivates the M08 version check) |
| B8 | Call `publish()` outside a transaction | Fails fast (IllegalTransactionStateException) |

---

## 8. Pitfalls

- Marking rows sent *before* the Kafka ack: back to losing events.
- Async `send()` without waiting for the ack inside the relay batch.
- Inbox insert in a separate transaction (`REQUIRES_NEW`): breaks the guarantee.
- An unbounded outbox table: add cleanup and an index on unpublished rows.
- Holding the `FOR UPDATE` lock while doing slow work: keep relay batches small and quick.

## 9. Deliverables

Outbox + inbox in `platform-messaging`, 3 services converted, drills automated, 2 ADRs, tag `m07-outbox-inbox`.

## 10. Self-check

1. Why exactly is the outbox atomic? Which system provides the atomicity?
2. Name 5 sources of duplicate messages.
3. Why must the inbox row and the side effect be in one transaction?
4. What does SKIP LOCKED buy you, and what ordering risk does it introduce?
5. Why do leases need timeouts, and what is a fencing token?
6. "Effectively-once" vs "exactly-once": what's the honest wording, and why?

## 11. References (when stuck)

*Microservices Patterns* ch. 3 (transactional messaging, idempotent consumers) · microservices.io *Transactional outbox*, *Idempotent consumer* · PostgreSQL docs `SELECT … FOR UPDATE SKIP LOCKED` · ShedLock README · Martin Kleppmann, *How to do distributed locking* (fencing tokens)

## 12. Exit checklist

- [ ] B1–B8 pass; B1–B5 automated
- [ ] No `KafkaTemplate` outside the platform (ArchUnit)
- [ ] ADRs written · tag `m07-outbox-inbox`
