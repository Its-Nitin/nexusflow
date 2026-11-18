# M19 — Forecasting Service (Polyglot Microservice)

| | |
|---|---|
| **Release** | R6 Intelligence & Launch |
| **Duration** | 1 week (~22 h) |
| **Depends on** | M10 (`DemandSource` port), M18 |
| **Unlocks** | M20 |
| **Tag** | `m19-forecasting` |

---

## 1. Why this milestone exists

Microservices promise that each service can use the **best technology for its job**. Forecasting fits Python's ecosystem better than Java's. This milestone tests that promise: can a Python service join the platform through **contracts alone** (OpenAPI, event schemas, JWT, tracing, metrics), with no shared code? It also teaches **graceful degradation**: Risk must keep working when Forecasting is down, which is why `DemandSource` was designed as a swappable port back in M10.

---

## 2. Learning objectives

- Integrate a non-JVM service purely via contracts and platform conventions.
- Swap an implementation behind a port without touching domain logic.
- Implement graceful degradation (circuit breaker + fallback), with the fallback made **visible** in outputs.
- Evaluate a forecast honestly: time-based split, naive baselines, MAE/WAPE, no leakage.
- Version models and expose model metadata.

---

## 3. Concepts

### 3.1 Polyglot microservices
**Gain:** the right tool per job. **Cost:** every cross-cutting concern the Java chassis gave you for free (auth, logging format, tracing, metrics, health, error format) must be re-implemented in Python. This is the real price of polyglot, and the reason many companies limit language choice. You'll feel it, measure it (hours spent), and write about it.

### 3.2 Contract-only integration
The Python service publishes an OpenAPI spec under `contracts/openapi/forecasting.yaml` (FastAPI generates one; commit it and lint it with Spectral), validates JWTs from Keycloak, emits `ForecastPublished` events following the same envelope + schema registry, and exports OTel traces and Prometheus metrics.

### 3.3 Graceful degradation
When a dependency fails, give a **reduced but correct** result rather than an error, and **say so**. Risk records `demandSource = FORECAST | FALLBACK_AVERAGE | DEFAULT`, so a degraded assessment is never mistaken for a full one.

### 3.4 Honest model evaluation
- **Time-based split:** train on the past, validate and test on later periods. Never shuffle.
- **Baselines first:** naive (yesterday), seasonal-naive (same weekday last week), moving average. A model that doesn't beat seasonal-naive isn't worth deploying.
- **Metrics:** MAE (units), WAPE = Σ|error| / Σ actual (robust to zero days, unlike MAPE).
- **Leakage:** features (lags, rolling means) may only use data available at forecast time.

---

## 4. Scope

**In:** a synthetic demand generator, an evaluation notebook/script + report, a FastAPI service (forecast endpoint, model metadata, JWT, OTel, metrics, health), a `ForecastPublished` event (optional precomputation), a Risk `ForecastDemandSource` adapter with circuit breaker + fallback, Compose `ai` profile + Helm chart.
**Out:** model retraining pipelines, drift monitoring (future).

---

## 5. Technical specification

### 5.1 Synthetic data
Per product/warehouse, 365 days: base level + weekly seasonality + trend + noise + random zero days + a few promotions (spikes). Seeded and deterministic. Also seed Inventory ISSUE movements consistently with it, so `DemandSource` v1 and the forecast see the same history.

### 5.2 Models
1. Naive · 2. Seasonal-naive (7 d) · 3. Moving average (28 d) · 4. One of: exponential smoothing (statsmodels) **or** gradient boosting with lag/rolling features (scikit-learn). Select per the validation WAPE; report on the test set.

### 5.3 API
`GET /api/v1/forecasts?productId=&warehouseId=&horizonDays=14` → `{ modelVersion, generatedAt, dailyForecast:[{date, qty}], avgDailyDemand, metrics:{wape, mae} }` · `GET /api/v1/models/current` · `/health/live`, `/health/ready`, `/metrics`.

### 5.4 Risk integration
`ForecastDemandSource` (adapter) → HTTP with client-credentials token, timeout 800 ms, circuit breaker; fallback → the `DeterministicDemandSource` from M10. The assessment stores `demandSource` and `modelVersion`. **Bump `calculationVersion`.**

---

## 6. Tasks

| ID | Task | Est | Concept | P |
|---|---|---|---|---|
| M19-T01 | Synthetic generator (Python) + consistent seed into Inventory movements | 2.5h | — | P0 |
| M19-T02 | Evaluation script: split, baselines, model, MAE/WAPE, report `docs/ml/forecast-evaluation.md` | 5h | Honest evaluation | P0 |
| M19-T03 | FastAPI service: endpoints, model loading + version metadata | 3h | — | P0 |
| M19-T04 | Python chassis work: JWT validation (Keycloak JWKS), JSON logs with traceId, OTel, Prometheus, health; **track the hours spent** | 4h | Polyglot cost | P0 |
| M19-T05 | Commit OpenAPI; Spectral lint; generated Java client for Risk | 1.5h | Contract-only integration | P0 |
| M19-T06 | Risk `ForecastDemandSource` + breaker + fallback + `demandSource` recorded | 3h | Graceful degradation, port swap | P0 |
| M19-T07 | Compose `ai` profile + Helm values | 1.5h | — | P0 |
| M19-T08 | ADRs: *Polyglot forecasting service*, *Demand source degradation policy* | 1h | — | P0 |

---

## 7. Break tests

| # | Drill | Expected |
|---|---|---|
| B1 | Stop Forecasting | Assessments continue with `demandSource=FALLBACK_AVERAGE`; the breaker opens; the dashboard shows it |
| B2 | Forecasting slow (Toxiproxy +2 s) | Timeout → fallback; no consumer stall |
| B3 | Call Forecasting without a token / with a tenant-B token for tenant-A data | 401 / 404 |
| B4 | The trace for the flagship flow | Includes the Python service span |
| B5 | Evaluation | The chosen model beats seasonal-naive on test WAPE, **or** you keep the baseline and document why |

---

## 8. Pitfalls

- Random train/test split (leakage → fake accuracy).
- Using MAPE with zero-demand days (division by zero).
- Silent fallback (indistinguishable outputs).
- Re-implementing the Java chassis badly in Python. Use libraries (python-jose/PyJWT, the OTel Python distro, prometheus-client).

## 9. Deliverables

Forecasting service, evaluation report, Risk adapter, ADRs, a note on the polyglot cost (hours), tag.

## 10. Self-check

1. What did polyglot cost you, concretely, and when is it worth it?
2. How did the M10 `DemandSource` port make this swap safe?
3. Why must degraded results be labelled?
4. Why a time-based split? Why WAPE over MAPE here?

## 11. References (when stuck)

*Forecasting: Principles and Practice* (Hyndman & Athanasopoulos, free online) ch. on benchmarks and evaluation · FastAPI docs · OpenTelemetry Python docs · scikit-learn `TimeSeriesSplit` · *Release It!* (graceful degradation)

## 12. Exit checklist

- [ ] B1–B5 pass
- [ ] Evaluation report committed
- [ ] ADRs written · tag `m19-forecasting`
