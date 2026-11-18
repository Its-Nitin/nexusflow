# M08 — Failure Handling, Ordering & Schema Evolution

| | |
|---|---|
| **Release** | R2 Reliable Eventing |
| **Duration** | 1 week (~24 h) |
| **Depends on** | M07 |
| **Unlocks** | M09+. **Gate G2** |
| **Tag** | `m08-failure-schemas`, `g2-reliable-eventing` |

---

## 1. Why this milestone exists

Outbox + inbox stop loss and double-processing. Three things can still take an event-driven system down:
1. **A poison message**: one malformed event that fails forever and blocks its partition.
2. **Stale or out-of-order events**: an old update overwriting a newer state.
3. **Schema drift**: a producer changes a field and silently breaks consumers.

This milestone makes the pipeline robust to all three, and adds the **operational tooling** (DLT replay) every real event-driven team needs.

---

## 2. Learning objectives

- Design a retry policy: transient vs permanent errors, backoff with jitter, bounded attempts.
- Explain blocking vs non-blocking retries and their effect on ordering.
- Operate a dead-letter topic: inspect, fix, replay.
- Implement stale-event protection with aggregate versions.
- Explain schema compatibility modes (BACKWARD, FORWARD, FULL) and evolve an event safely.
- Compare a polling outbox with log-based CDC (Debezium).

---

## 3. Concepts

### 3.1 Transient vs permanent failures
- **Transient** (DB timeout, network blip): retry with **exponential backoff + jitter** (jitter avoids synchronised retry storms).
- **Permanent** (deserialisation error, validation failure, a bug): retrying is pointless, so send it to the **DLT** immediately.
Classify exceptions explicitly. Default to permanent for unknown `IllegalArgumentException`-style errors.

### 3.2 Blocking vs non-blocking retries
| | Blocking (retry in place) | Non-blocking (retry topics) |
|---|---|---|
| Ordering per key | **Preserved** | Broken: later events overtake the retried one |
| Partition throughput | Blocked during retries | Continues |
**NexusFlow choice:** blocking retries with a short bounded backoff (e.g. 3 attempts: 1 s, 2 s, 4 s), then the DLT. Ordering matters for projections. Record the reasoning in an ADR.

### 3.3 Dead-letter topic (DLT)
A parking place for messages that couldn't be processed, with headers carrying the original topic, partition, offset, exception and stack trace. A DLT without a **replay procedure** is just a slow way to lose data. You need: alerting (M16), inspection, a fix, then replay to the original topic, and idempotency makes the replay safe.

### 3.4 Stale / out-of-order events
Even with per-key ordering, reordering can happen (multiple relays as in M07-B7, replays, DLT replays after newer events). Consumers store `last_seen_version` per aggregate and **ignore events with `aggregateVersion ≤ last_seen`**. For projections this is the standard guard.

### 3.5 Schema evolution & compatibility
| Mode | Meaning | Allows |
|---|---|---|
| BACKWARD | New consumers can read old data | Add optional fields, delete fields |
| FORWARD | Old consumers can read new data | Add fields, delete optional fields |
| FULL | Both | Add/delete optional fields only |
**NexusFlow:** BACKWARD (the registry default), plus the **tolerant reader** rule (consumers ignore unknown fields). Breaking changes mean a new `eventVersion` and, if needed, a new topic, with a migration period where both are produced.
**Registry choice:** Apicurio Registry (Apache 2.0) or Confluent Schema Registry, with **JSON Schema** so payloads stay readable JSON (Avro is the more compact, more common alternative; mention it in the ADR).

### 3.6 CDC vs polling outbox
| | Polling relay (M07) | Debezium CDC |
|---|---|---|
| Latency | Poll interval | Near real time |
| DB load | Repeated queries | Reads the WAL (logical replication slot) |
| Moving parts | In-app | Kafka Connect + connector + replication slot |
| Failure mode | Backlog in the table | Unconsumed replication slot → WAL growth (disk fills) |

---

## 4. Scope

**In:** error handler + retry policy + exception classification, DLT per consumer group, a replay tool + runbook, version guard in the inbox handler, schema registry + JSON Schemas for all current events + a CI compatibility check, one evolution exercise, a Debezium comparison lab.
**Out:** alerting on the DLT (M16).

---

## 5. Technical specification

### 5.1 Error handling (platform default)
- `DefaultErrorHandler` + `DeadLetterPublishingRecoverer` → `<topic>.<group>.DLT` (same partition number).
- `ExponentialBackOffWithMaxRetries(3)`, initial 1 s, multiplier 2, plus jitter.
- Not retryable: `DeserializationException`, `MethodArgumentNotValidException`, `SchemaValidationException`, `NonRetryableEventException` (a custom marker).
- `ErrorHandlingDeserializer` wraps the real deserializer, so a bad payload goes to the DLT instead of crash-looping.

### 5.2 Replay tool
Admin endpoint in each service (MANAGER/SERVICE role) or a CLI in `scripts/`:
`POST /admin/dlt/{group}/replay?fromOffset=&toOffset=&dryRun=true` → reads the DLT, re-publishes to the original topic with header `x-replayed-from-dlt=true`, and logs a summary. Runbook: `docs/runbooks/dlt-replay.md`.

### 5.3 Version guard
Extend `IdempotentEventHandler`: table `aggregate_versions(aggregate_type, aggregate_id, consumer_group, last_version)`; inside the same transaction, if `event.aggregateVersion <= last_version` → skip (log "stale"); else process and upsert.

### 5.4 Schemas
`contracts/events/schemas/<eventType>.v<N>.json`. Registered at build or startup. CI step: check compatibility of changed schemas against the registry (or with a local compatibility checker), and fail the PR if incompatible. Producer serializer validates against the schema.

---

## 6. Tasks

| ID | Task | Est | Concept | P |
|---|---|---|---|---|
| M08-T01 | Platform error handler, backoff + jitter, classification, `ErrorHandlingDeserializer` | 3h | Retries, poison pills | P0 |
| M08-T02 | DLT topics (`topics.sh`) + recoverer | 1h | Dead letters | P0 |
| M08-T03 | Replay endpoint/CLI + runbook | 3h | Operability | P0 |
| M08-T04 | Version guard in the inbox handler | 2h | Stale events | P0 |
| M08-T05 | Schema registry in Compose; JSON Schemas for all events; serializer validation | 4h | Schema contracts | P0 |
| M08-T06 | CI compatibility check | 1.5h | Contract enforcement | P0 |
| M08-T07 | Evolution exercise: `ShipmentDelayed` adds optional `carrierCode` (compatible); then attempt to rename `delayDays` (incompatible) | 1.5h | Schema evolution | P0 |
| M08-T08 | **Lab:** Debezium Postgres connector + outbox event router for Shipment (Kafka Connect in the `chaos`/lab profile); compare with polling; then revert or keep it as an option | 4h | CDC | P1 |
| M08-T09 | `scripts/drills/reliability` combining M07 + M08 drills (Gate G2) | 2h | — | P0 |
| M08-T10 | ADRs: *Retry & DLT policy*, *Event schema & compatibility*, *Polling vs CDC* | 1.5h | — | P0 |

---

## 7. Break tests

| # | Drill | Expected |
|---|---|---|
| B1 | Publish malformed JSON to `shipment.events` | DLT after 0 retries (non-retryable); following events still processed |
| B2 | Make the risk stub throw a transient error for 2 attempts | Succeeds on attempt 3; no DLT |
| B3 | Make it throw a "bug" exception for one event type | Event in the DLT; fix the code; replay → processed exactly once |
| B4 | Deliver v3 then v2 of an aggregate (manual produce) | v2 skipped as stale |
| B5 | Register an incompatible schema | Rejected by the registry and by the CI check |
| B6 | An old consumer reads an event with a new optional field | Works (tolerant reader) |
| B7 | (Lab) stop Kafka Connect for 10 min while updating shipments | Observe replication-slot WAL growth; events delivered on resume |

> ✅ **Gate G2:** `scripts/drills/reliability` kills Kafka, kills relays, injects duplicates, poison and stale events, and the final state is correct every time.

---

## 8. Pitfalls

- Infinite retries (an old default in some setups) block a partition forever.
- Non-blocking retry topics chosen by default, silently breaking ordering for projections.
- DLT messages missing the original headers, so you can't tell where they came from.
- A replay tool that bypasses idempotency (it must go through the normal consumer path).
- An abandoned Debezium replication slot filling the Postgres disk.

## 9. Deliverables

Platform error handling, DLTs, replay tool + runbook, version guard, schema registry + schemas + CI check, CDC lab notes, the reliability drill script, 3 ADRs, tags.

## 10. Self-check

1. Why do blocking retries preserve ordering and non-blocking ones don't?
2. What makes a DLT safe to replay?
3. How can events arrive out of order even with per-key partitioning?
4. BACKWARD vs FORWARD compatibility: which protects consumers when producers deploy first?
5. One operational risk of CDC that the polling relay doesn't have?

## 11. References (when stuck)

Spring for Apache Kafka reference: *Handling exceptions*, *Dead-letter publishing*, *Non-blocking retries* · Confluent docs *Schema evolution and compatibility* · Apicurio Registry docs · Debezium docs *Outbox event router*, *PostgreSQL connector* · *Release It!* (on retries and backoff)

## 12. Exit checklist

- [ ] B1–B6 automated; B7 documented
- [ ] Runbook `dlt-replay.md` tested once end to end
- [ ] ADRs written · Gate G2 passed · tags pushed
