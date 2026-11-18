# M21 — Release v1.0 & Consolidation

| | |
|---|---|
| **Release** | R6 Intelligence & Launch |
| **Duration** | 1 week (~20 h) |
| **Depends on** | M20 (or whichever milestone you reached after applying the cut order) |
| **Unlocks** | **Gate G6: v1.0 released** |
| **Tag** | `v1.0.0`, `g6-release` |

---

## 1. Why this milestone exists

Two goals: **ship** (the project must be complete, presentable and reproducible) and **consolidate the learning** (turn 20 milestones of experience into knowledge you can explain clearly under interview pressure). The most valuable output of the whole project is the *retrospective*: an honest evaluation of your own architecture, including whether microservices were the right call.

---

## 2. Learning objectives

- Communicate a distributed architecture clearly at three levels: a 30-second pitch, a 5-minute demo, and a 45-minute deep dive.
- Critically evaluate your boundaries and patterns with evidence (incidents, drills, metrics).
- Answer system-design questions using NexusFlow as the reference answer.

---

## 3. Concepts

### 3.1 Architecture retrospective
Look at each boundary and pattern: did it pay for itself? Evidence: how often changes crossed service boundaries (git history), which drills found real bugs, where complexity hurt. A thoughtful "a modular monolith would have served the *product* better; microservices served the *learning* goal" is a senior-level conclusion.

### 3.2 Communicating architecture
Layered explanation: problem → users → flagship scenario → C4 container view → one deep dive (outbox+saga) → evidence (drills, traces, measured numbers) → trade-offs and what you'd change.

---

## 4. Scope

**In:** README, final diagrams, ADR index, docs consistency pass, demo video + failure-recovery clip, resume bullets (only what you built and measured), self-check re-run, a mock system-design interview, the retrospective, a release with notes.
**Out:** new features.

---

## 5. Specification: required artefacts

| Artefact | Content |
|---|---|
| `README.md` | Problem, target user, flagship scenario, architecture diagram, tech stack, quick start (`task demo`, `task k8s:up`), demo credentials strategy, **limitations**, links to docs |
| `docs/architecture/` | C4 L1/L2 (final), event-flow diagram, saga diagrams, context map |
| `docs/adr/README.md` | Index of all ADRs with status |
| `docs/performance.md` | Measured numbers + methodology (from M17) |
| `docs/ai/`, `docs/ml/` | Evaluation reports |
| `docs/runbooks/` | All runbooks, each tested once |
| `docs/retrospective.md` | §6 T06 |
| Demo video (3–5 min) | Blueprint narrative: normal state → 5-day delay → event propagation → risk + evidence → compare/approve → audit → observability (trace, lag) |
| Failure clip (1–2 min) | Kill the Risk consumer under load → lag rises → restart → recovery; or DLT → fix → replay |
| Release notes | Features, architecture highlights, known limitations |

---

## 6. Tasks

| ID | Task | Est | P |
|---|---|---|---|
| M21-T01 | Docs consistency pass (contracts ↔ code, ADR statuses, service READMEs) | 3h | P0 |
| M21-T02 | README + final diagrams + ADR index | 3h | P0 |
| M21-T03 | Demo video + failure clip | 4h | P0 |
| M21-T04 | Resume bullets + a 30-second pitch + a 5-minute talk outline | 1.5h | P0 |
| M21-T05 | Re-answer **every** self-check question (M00–M20) without notes; re-study failures | 4h | P0 |
| M21-T06 | Retrospective: per boundary (right/wrong + evidence), per pattern (worth it?), "modular monolith?", what you'd change, what you'd learn next | 2.5h | P0 |
| M21-T07 | Mock system-design interview: "Design a supply-chain disruption platform" (record yourself, 45 min) | 1.5h | P0 |
| M21-T08 | Fresh-clone verification on a clean machine/VM; tag `v1.0.0`; GitHub release | 1h | P0 |

---

## 7. Break tests

| # | Check | Expected |
|---|---|---|
| B1 | A fresh clone on a clean machine follows only the README | Demo runs without help |
| B2 | Every resume bullet | Backed by code + a drill/metric you can show |
| B3 | Self-check re-run | ≥ 90% answered confidently; the rest re-studied |
| B4 | The mock interview recording | You can explain the outbox, sagas, idempotency, CQRS and tracing with NexusFlow examples |

> ✅ **Gate G6:** `v1.0.0` released, the demo video published, the docs complete.

---

## 8. Pitfalls

- Claiming capabilities you didn't build, or numbers you didn't measure.
- Polishing visuals instead of explanations.
- Skipping the retrospective. It is the highest-value learning artefact.

## 9. Deliverables

All artefacts in §5, tags `v1.0.0`, `g6-release`.

## 10. Self-check (the final exam)

1. Explain NexusFlow in 30 seconds.
2. Walk through the flagship flow, naming every pattern that keeps it correct.
3. What are the three most important distributed-systems lessons you learned, each with a drill that taught it?
4. Which boundary would you redraw and why?
5. When would you *not* use microservices?

## 11. References

Your own journal, ADRs and drills. Plus *System Design Interview* style practice and *Building Microservices* 2e ch. 1 (revisit with fresh eyes).

## 12. Exit checklist

- [ ] All §5 artefacts committed
- [ ] B1–B4 done
- [ ] `v1.0.0` released · 🎉 project complete
