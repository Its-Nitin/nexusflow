# M13 — Notification Service: Choreography & Real-Time Fan-Out

| | |
|---|---|
| **Release** | R3 Disruption Flow |
| **Duration** | 1 week (~20 h) |
| **Depends on** | M10, M12 |
| **Unlocks** | M15 (live alerts). **Gate G3** |
| **Tag** | `m13-notification`, `g3-disruption-flow` |

---

## 1. Why this milestone exists

The last mile of a distributed workflow is a **human**. Humans are the least idempotent consumer there is: two identical emails or toasts erode trust. Real-time push also has a distributed-systems problem of its own: the user's browser is connected to instance A, but the event is consumed by instance B. This milestone teaches **choreography** (reacting to events with no orchestrator), **dedupe at the human boundary**, and **multi-instance fan-out**.

---

## 2. Learning objectives

- Implement a purely choreographed service and contrast it with the M12 orchestrator.
- Derive logical notification identity from source events so retries never duplicate.
- Handle an unreliable external channel (SMTP) with retries and delivery tracking.
- Explain SSE vs WebSocket vs polling.
- Solve the multi-instance fan-out problem with a pub/sub backplane.

---

## 3. Concepts

### 3.1 Choreography
Notification subscribes to `risk.events` and `recovery.events`. **Nobody tells it what to do**; adding it required zero changes to other services (the M01 design-review B1 promise, now proven).
**Risk of choreography at scale:** the overall flow becomes hard to see, which is one reason tracing (M16) matters.

### 3.2 Logical identity & dedupe
`notificationId = UUIDv5(sourceEventId + recipient + channel)`. However many times the event is delivered or the send is retried, there is one logical notification with possibly many `DeliveryAttempt`s.

### 3.3 Unreliable external channels
SMTP can be down or slow. Separate **"notification created"** (in a transaction, from the event) from **"delivery attempted"** (async worker with backoff). This is the outbox idea again, applied to email: the event commits the intent; a worker delivers.

### 3.4 Push technologies
| | SSE | WebSocket | Polling |
|---|---|---|---|
| Direction | Server → client | Both | Client pulls |
| Through proxies/gateway | Plain HTTP; easy | Upgrade needed | Easy |
| Reconnect | Built in (`Last-Event-ID`) | Manual | N/A |
For one-way alerts, **SSE** is simplest and correct.

### 3.5 Multi-instance fan-out
Notification runs 2+ instances in one consumer group: each event goes to **one** instance, but the user's SSE connection may be on the **other**. Options:
1. **Pub/sub backplane (Redis):** the consuming instance publishes to Redis channel `user:{id}`; every instance subscribed for its connected users forwards. ✔ chosen
2. A consumer group per instance (every instance reads every event; it only works for push, not email, and needs a different group ID per pod).
3. Sticky routing by user (fragile under rebalancing).

---

## 4. Scope

**In:** the Notification service, subscriptions/rules (which events → which roles), in-app notifications API, email via Mailpit with a retry worker, an SSE endpoint through the gateway, a Redis backplane, 2-instance drills.
**Out:** user preference UI (later), mobile push.

---

## 5. Technical specification

### 5.1 Rules (config)
| Event | Recipients | Channels |
|---|---|---|
| `RiskAssessmentCompleted` severity ≥ HIGH | PLANNER, MANAGER of the tenant | in-app + email |
| `RecoveryPlanProposed` | MANAGER | in-app + email |
| `RecoveryPlanCompleted/Compensated` | PLANNER, MANAGER | in-app |
| `RecoveryPlanFailed` | MANAGER | in-app + email (urgent) |
Recipients are resolved from a user directory. For v1: a small projection of Keycloak users via the admin API at startup, or a static config per tenant. Document the shortcut.

### 5.2 Model
```
notification(id (deterministic), tenant_id, recipient, channel[IN_APP|EMAIL], type, title, body, source_event_id,
             status[PENDING|SENT|FAILED|READ], created_at)
delivery_attempt(id, notification_id, attempt_no, outcome, error, attempted_at)
```

### 5.3 API
`GET /api/v1/notifications?unread=true` · `POST /api/v1/notifications/{id}/read` · `GET /api/v1/notifications/stream` (SSE; `Last-Event-ID` resume from the stored notifications)

### 5.4 Delivery worker
Polls `PENDING` email notifications (SKIP LOCKED), sends via SMTP (Mailpit), backoff 1 s → 30 s → 5 min, max 5 attempts → FAILED (alertable metric).

---

## 6. Tasks

| ID | Task | Est | Concept | P |
|---|---|---|---|---|
| M13-T01 | Service from the template; consumers for risk + recovery events (inbox) | 2h | Choreography | P0 |
| M13-T02 | Rules engine (config) + recipient resolution | 2.5h | — | P0 |
| M13-T03 | Deterministic notification IDs; create in-app + email records in one transaction | 2h | Dedupe at the human boundary | P0 |
| M13-T04 | Mailpit in Compose; delivery worker with backoff + attempts | 3h | Unreliable channels | P0 |
| M13-T05 | Notifications REST API | 1.5h | — | P0 |
| M13-T06 | SSE endpoint + gateway route (disable response buffering/timeouts for the stream) | 3h | Long-lived connections | P0 |
| M13-T07 | Redis pub/sub backplane; run 2 instances | 3h | **Multi-instance fan-out** | P0 |
| M13-T08 | Reflection doc: which flows are orchestrated vs choreographed, and why | 1h | — | P0 |
| M13-T09 | Gate G3 script `scripts/demo/g3-disruption-flow` | 2h | — | P0 |

---

## 7. Break tests

| # | Drill | Expected |
|---|---|---|
| B1 | The same `RiskAssessmentCompleted` delivered 3× | One notification per recipient/channel |
| B2 | Mailpit stopped for 2 min | Attempts recorded; delivered once after recovery; no duplicate emails |
| B3 | 2 instances; a user connected to instance A; the event consumed by instance B | The toast arrives exactly once |
| B4 | Kill the instance holding the SSE connection | The client reconnects (`Last-Event-ID`) and receives missed notifications, with no duplicates |
| B5 | Redis down | Document the degradation (in-app still stored; live push degraded; client falls back to polling) |

> ✅ **Gate G3:** a scripted flow via the API: record a 5-day delay → risk assessment → plan proposed → manager approves → saga completes (and, in a second run with injected failure, compensates) → notifications delivered.

---

## 8. Pitfalls

- Generating random notification IDs (duplicates on redelivery).
- Sending email inside the Kafka consumer transaction (slow SMTP → `max.poll.interval` breach).
- The gateway buffering SSE responses, or timing them out.
- Holding too many SSE connections on dev JVMs (thread usage). Note it for M17.

## 9. Deliverables

Notification service, backplane, G3 script, reflection doc, ADR *Real-time push: SSE + Redis backplane*, tags.

## 10. Self-check

1. How does choreography prove the M01 claim that consumers can be added without touching producers?
2. Why separate "notification created" from "delivery attempted"?
3. Explain the fan-out problem and three solutions.
4. SSE vs WebSocket for this feature: why SSE?

## 11. References (when stuck)

*Microservices Patterns* ch. 4 (choreography) · MDN *Server-sent events* · Spring docs `SseEmitter` · Redis docs *Pub/Sub* · Spring Cloud Gateway docs on streaming responses

## 12. Exit checklist

- [ ] B1–B5 done; B1–B3 automated
- [ ] Gate G3 script passes (happy + compensation runs)
- [ ] ADR written · tags pushed
