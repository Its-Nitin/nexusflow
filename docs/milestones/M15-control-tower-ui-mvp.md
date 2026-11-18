# M15 — Control-Tower UI & MVP

| | |
|---|---|
| **Release** | R4 Control Tower MVP |
| **Duration** | 1 week. **UI work hard-capped at 20 h** |
| **Depends on** | M04, M13, M14 |
| **Unlocks** | **Gate G4: MVP complete.** Everything after this is production-hardening and extension |
| **Tag** | `m15-ui`, `g4-mvp` |

---

## 1. Why this milestone exists

The flagship scenario must be demonstrable end to end in a browser. That's what makes the project *complete*. The UI is not the learning goal, so it is **time-boxed**. Its distributed-systems lessons are specific: **honest eventual consistency in the UX**, SPA security (PKCE), and **end-to-end verification across all services**.

---

## 2. Learning objectives

- Implement OIDC Authorization Code + PKCE in an SPA with token refresh.
- Design UX for eventual consistency: `as of` badges, pending states, live updates.
- Strictly separate commands (to owners via the gateway) from reads (from Query).
- Write an E2E test that drives the full distributed flow.

---

## 3. Concepts

### 3.1 Eventual consistency in UX
Users tolerate delay; they don't tolerate **contradiction or silence**. Patterns: show `as of HH:mm:ss`; after a command, show a "Processing…" state from the command response; update live via SSE; never show a success that hasn't happened ("Approved, executing" ≠ "Completed").

### 3.2 Command/query separation in the client
- Reads: `query-service` only (one place, fast, available).
- Commands: the owning service via the gateway (`POST /recovery-plans/{id}/approve` → Recovery).
- Command responses return the resulting state/version → the UI waits for a view with `sourceVersion ≥ version` or an SSE event.

### 3.3 SPA security
PKCE prevents authorization-code interception. Store tokens in memory (not localStorage); silent refresh; send them as Bearer tokens via an HTTP interceptor; handle 401 → re-login, 403 → message. (The BFF alternative is recorded in an M04 ADR.)

### 3.4 E2E testing across services
An E2E test proves *integration*, which no unit test can. Keep it **few and valuable**: one API-driven flagship test (fast, stable) and one Playwright UI smoke test.

---

## 4. Scope

**In:** Angular app (login, summary, shipments, risk board + detail, recovery plan compare/approve, timeline/audit, notifications toasts), the flagship seed, `task demo`, E2E tests, demo script.
**Out:** admin screens, CSV import, polish beyond usability.

---

## 5. Technical specification

### 5.1 Pages
| Page | Data source | Actions |
|---|---|---|
| Login | Keycloak (PKCE) | — |
| Control tower summary | `/control-tower/summary` | — |
| Shipments | `/control-tower/shipments` | Report a delay (→ Shipment service) |
| Risk board / detail | `/control-tower/risks`, `/risks/{id}` | Run a simulation (→ Risk) |
| Recovery plan | Query + `/recovery-plans/{id}` | Approve/Reject (→ Recovery, Idempotency-Key) |
| Timeline / audit | `/recovery-plans/{id}/timeline`, `/audit` | — |
| Notifications | SSE stream + list | Mark read |

### 5.2 Tech
Angular (latest stable) + TypeScript + RxJS; an OIDC client library (e.g. angular-auth-oidc-client); a component library (Angular Material) to avoid custom CSS; generated API clients from the OpenAPI contracts (contract-first pays off again).

### 5.3 Demo
`task demo` = reset + seed the flagship (suppliers, products, W1/W2 stock, reservations, PO-1001, SH-77) + print the login credentials and steps.

---

## 6. Tasks

| ID | Task | Est | Concept | P |
|---|---|---|---|---|
| M15-T01 | Angular scaffold, Material, routing, OIDC PKCE login, interceptor | 3h | SPA security | P0 |
| M15-T02 | Generated API clients from OpenAPI | 1h | Contract-first | P0 |
| M15-T03 | Summary + shipments pages (+ report-delay form) | 3h | Command vs query | P0 |
| M15-T04 | Risk board + detail with evidence, calc version, `as of` | 3h | Explainability, staleness | P0 |
| M15-T05 | Plan compare + approve/reject + pending state | 4h | Eventual consistency UX | P0 |
| M15-T06 | Timeline/audit page | 2h | — | P0 |
| M15-T07 | SSE toasts + notification list | 2h | Live updates | P0 |
| M15-T08 | `task demo` + seed | 1.5h | Reproducible demo | P0 |
| M15-T09 | API-driven E2E flagship test in CI (Compose job) | 3h | E2E across services | P0 |
| M15-T10 | Playwright smoke test (login → approve) | 2h | — | P1 |

---

## 7. Break tests

| # | Drill | Expected |
|---|---|---|
| B1 | Approve, then stop the Query service | The UI shows "approved, executing", no false "completed"; recovers when Query returns |
| B2 | Stop Risk during browsing | Dashboard still loads (from Query) with `as of`; the simulation button shows a clear error |
| B3 | Double-click Approve | One approval (Idempotency-Key + disabled button) |
| B4 | Token expiry while idle | Silent refresh, or a clean re-login; no broken state |
| B5 | Fresh clone → `task demo` → full flagship in the browser | Works without manual steps |

> ✅ **Gate G4 — MVP COMPLETE:** from a clean clone, `task demo` → browser: PO + stock + ETA → introduce a 5-day delay → risk with evidence → compare options → approve → saga completes → audit timeline → alert toast. **Record a rough demo video now** (the polished one comes in M21).

---

## 8. Pitfalls

- UI scope creep. When 20 h are used, stop and ship what works.
- The UI calling owner services for reads "because it's quicker."
- Tokens in localStorage.
- A flaky UI E2E test in CI. Keep the API-level test as the gate; Playwright is a smoke test.

## 9. Deliverables

The Angular app, demo command, E2E tests, rough demo video, tags `m15-ui`, `g4-mvp`.

## 10. Self-check

1. How does the UI avoid lying about state under eventual consistency?
2. Why read only from Query while sending commands to the owners?
3. What does PKCE protect against?
4. Why is an API-level E2E test a better CI gate than a UI test?

## 11. References (when stuck)

Angular docs · angular-auth-oidc-client docs · oauth.net *OAuth 2.0 for Browser-Based Apps* · Playwright docs

## 12. Exit checklist

- [ ] B1–B5 pass
- [ ] Gate G4 demo from a clean clone ✔ (**project complete: core**)
- [ ] Tags pushed
