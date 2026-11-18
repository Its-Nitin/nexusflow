# M16 — Observability

| | |
|---|---|
| **Release** | R5 Production Ready |
| **Duration** | 1 week (~22 h) |
| **Depends on** | M15 |
| **Unlocks** | M17, M18 |
| **Tag** | `m16-observability` |

---

## 1. Why this milestone exists

A single user action in NexusFlow now crosses the gateway, 6 services, 3 Kafka hops and an outbox relay. When it is slow or wrong, logs from one service tell you almost nothing. **Observability** means being able to answer *new* questions about the system's behaviour from its outputs: traces show the path, metrics show the trends, and logs show the details. The hard part in event-driven systems is keeping the trace connected **across asynchronous boundaries**, including the outbox's delay.

---

## 2. Learning objectives

- Explain traces, spans and context propagation (W3C `traceparent`), including over Kafka headers.
- Carry trace context through the transactional outbox (stored and restored).
- Build RED dashboards and explain why averages lie (percentiles, histograms).
- Monitor consumer lag, outbox backlog, DLT size and saga states as the vital signs of an event-driven system.
- Define SLIs/SLOs and alert on symptoms (burn rate), not causes.
- Debug an injected fault using only telemetry.

---

## 3. Concepts

### 3.1 The three signals
| Signal | Answers | Tool |
|---|---|---|
| Traces | Where did the time go across services? Which path did this request take? | OpenTelemetry → Tempo or Jaeger |
| Metrics | Is it getting worse? How often? How many? | Micrometer → Prometheus → Grafana |
| Logs | What exactly happened in this step? | JSON stdout → Loki → Grafana |
**Correlation is the multiplier:** logs carry `traceId`; Grafana links metrics (exemplars) → traces → logs.

### 3.2 Context propagation over async boundaries
HTTP: the `traceparent` header. Kafka: the same, in **record headers** (Spring Kafka + Micrometer Tracing / the OTel agent handle it when enabled). **The outbox breaks the chain** unless you **store `traceparent` in the outbox row** at write time and set it as a header when the relay publishes. Then the consumer span becomes a child (or a *link*) of the original request.

### 3.3 RED and USE
- **RED** (request-driven services): Rate, Errors, Duration (p50/p95/p99 from histograms).
- **USE** (resources): Utilisation, Saturation, Errors (DB pool, threads, CPU).
- **Averages hide tail latency.** p99 is what unlucky users feel, and in fan-out flows the tail compounds.

### 3.4 Event-driven vital signs
- **Consumer lag** (messages behind, per group/partition): the best single health signal.
- **Outbox pending count/age**: is publishing stuck?
- **DLT message count**: are events being parked?
- **Saga status counts / age of RUNNING sagas**: are workflows stuck?
- **Projection staleness**: how stale are read models?

### 3.5 SLIs, SLOs, error budgets
- **SLI:** a measured ratio, e.g. "share of `ShipmentDelayed` events whose `RiskAssessmentCompleted` is published within 5 s."
- **SLO:** the target, e.g. 95% over 7 days.
- **Error budget:** the 5% you may "spend." Alert on **burn rate** (spending the budget too fast), not on every blip.

### 3.6 Cardinality
Metric labels with unbounded values (userId, eventId) explode Prometheus. Put IDs in **traces and logs**, not metric labels.

---

## 4. Scope

**In:** OTel tracing in all services + gateway; Kafka + outbox propagation; the `obs` Compose profile (Prometheus, Grafana, Tempo/Jaeger, Loki, kafka-exporter or the Kafka JMX exporter); dashboards; 3 SLOs + alert rules; a debugging drill.
**Out:** production alert routing (PagerDuty etc.); log retention policies beyond local.

---

## 5. Technical specification

### 5.1 Instrumentation
Either the **OpenTelemetry Java agent** (zero-code; attach via `JAVA_TOOL_OPTIONS`) or **Micrometer Tracing + OTel bridge** (Spring-native). Pick one, write an ADR, and apply it in `platform-observability`. Export OTLP to the collector or directly to Tempo/Jaeger.

### 5.2 Outbox propagation
`outbox.headers.traceparent` captured from the current span at write time; the relay sets it as a Kafka header. The relay's own publish span links to it.

### 5.3 Custom metrics (from earlier milestones, now standardised)
`nexusflow_outbox_pending_events`, `nexusflow_outbox_oldest_pending_age_seconds`, `nexusflow_dlt_messages_total{topic}`, `nexusflow_projection_lag_seconds{projection}`, `nexusflow_saga_instances{status}`, `nexusflow_saga_oldest_running_age_seconds`, `nexusflow_risk_assessment_latency_seconds` (delay event `occurredAt` → assessment published; histogram).

### 5.4 Dashboards (provisioned as JSON in `infra/grafana/`)
1. **System overview:** RED per service, error rate, p95.
2. **Event pipeline:** consumer lag per group, outbox backlog/age, DLT counts, projection staleness.
3. **Business flow:** delays → assessments → plans → sagas (counts, latencies, states).
4. **JVM/DB:** heap, GC, Hikari pool usage, threads.

### 5.5 SLOs
| SLO | SLI | Target |
|---|---|---|
| Disruption detection latency | delay→assessment ≤ 5 s | 95% / 7 d |
| API availability (gateway) | non-5xx responses | 99% / 7 d |
| Saga completion | sagas finished (COMPLETED/COMPENSATED) within 2 min | 99% |
Alert rules: multi-window burn rate (e.g. 1 h and 5 min windows).

---

## 6. Tasks

| ID | Task | Est | Concept | P |
|---|---|---|---|---|
| M16-T01 | `obs` profile: Prometheus, Grafana (provisioned data sources), Tempo/Jaeger, Loki, Kafka lag exporter | 3h | Observability stack | P0 |
| M16-T02 | Tracing in `platform-observability`; ADR on agent vs Micrometer | 3h | Distributed tracing | P0 |
| M16-T03 | Kafka header propagation verified; **outbox traceparent** capture + restore | 3h | Async context propagation | P0 |
| M16-T04 | Logs → Loki with traceId; Grafana log ↔ trace linking | 2h | Signal correlation | P0 |
| M16-T05 | Custom metrics standardised (§5.3) | 2h | Event-driven vital signs | P0 |
| M16-T06 | 4 dashboards as code | 4h | RED/USE | P0 |
| M16-T07 | SLO recording + burn-rate alert rules | 2.5h | SLOs, error budgets | P0 |
| M16-T08 | Debugging drill (§7 B3) | 2h | Debugging distributed systems | P0 |
| M16-T09 | Runbooks updated with dashboard links | 0.5h | — | P0 |

---

## 7. Break tests

| # | Drill | Expected |
|---|---|---|
| B1 | Flagship flow | **One trace** shows gateway → shipment → (outbox) → Kafka → risk → Kafka → recovery → participants → notification |
| B2 | Stop the Risk consumer for 2 min under load | Lag dashboard rises; the detection SLO burn-rate alert fires; it recovers after restart |
| B3 | Ask someone, or a script with a random seed, to inject a hidden 2 s sleep for one SKU in one service | You find the service, span and SKU **using only Grafana** (write down the steps) |
| B4 | Poison message | DLT count panel increases; the alert fires |
| B5 | Add a userId label to a metric (then remove it) | Observe the series explosion in Prometheus; journal the lesson |

---

## 8. Pitfalls

- Trace continuity broken at the outbox (the most common gap).
- Sampling everything at 100% in prod (fine locally; note the trade-off).
- Dashboards nobody can read: every panel needs a title that states a question.
- Alerting on CPU instead of user-facing symptoms.

## 9. Deliverables

`obs` profile, tracing in all services, outbox propagation, dashboards as code, SLOs + alerts, debugging drill write-up, ADR, tag.

## 10. Self-check

1. How does trace context survive the outbox and Kafka?
2. Why is consumer lag the key health signal for event-driven services?
3. Why p95/p99 rather than averages? Why does the tail compound in fan-out?
4. Define SLI, SLO and error budget with a NexusFlow example.
5. What is metric cardinality, and where do IDs belong instead?

## 11. References (when stuck)

OpenTelemetry docs (Java, context propagation, messaging semantic conventions) · Micrometer Tracing docs · Google SRE Book ch. 4 (SLOs) + SRE Workbook ch. on alerting on SLOs · Grafana Tempo/Loki docs · Brendan Gregg, *USE method*

## 12. Exit checklist

- [ ] B1–B5 done; the B1 trace screenshot in docs
- [ ] Dashboards + alerts provisioned from code
- [ ] ADR written · tag `m16-observability`
