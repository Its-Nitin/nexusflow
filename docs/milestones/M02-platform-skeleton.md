# M02 — Platform Skeleton & Local Infrastructure

| | |
|---|---|
| **Release** | R0 Foundation |
| **Duration** | 0.5 week (~12 h) |
| **Depends on** | M00, M01 |
| **Unlocks** | M03 (first real service) |
| **Tag** | `m02-platform` |

---

## 1. Why this milestone exists

A distributed system needs **backing services** (database, broker, cache, identity) that start reliably, in the right order, with isolated data per service. If `docker compose up` is flaky, you'll lose hours every week. This milestone builds the local runtime and the **runtime chassis** (`platform` modules) that every service plugs into.

---

## 2. Learning objectives

- Explain the three ways to implement *database per service* (schema per service, database per service, server per service) and why we pick database + separate user.
- Explain the twelve-factor principles that matter most for microservices: config, backing services, disposability, logs.
- Run multi-container infrastructure with health-checked startup ordering and resource limits.
- Explain container DNS-based discovery under Compose.

---

## 3. Concepts

### 3.1 Database per service: implementation options
| Option | Isolation | Cost | Our use |
|---|---|---|---|
| Schema per service (one DB) | Weak (easy to cross-join by accident) | Lowest | No |
| **Database + user per service, one server** | Strong logical isolation; grants prevent cross-access | Low | **Yes (local)** |
| Server per service | Full isolation incl. resources | High | Production idea; K8s option in M18 |

**Why database per service at all:** services can change schemas independently, and no service can couple itself to another's tables. **The cost:** no joins across services, no cross-service transactions. That cost is *why* you'll need events, projections and sagas later.

### 3.2 Twelve-factor app (the relevant factors)
- **III Config:** config lives in environment variables, never in code. The same image runs everywhere.
- **IV Backing services:** Postgres and Kafka are attached resources addressed by URL.
- **IX Disposability:** fast startup, graceful shutdown. Containers get killed; that's normal.
- **XI Logs:** write to stdout; the platform collects them.

### 3.3 Startup ordering and health
`depends_on` alone only waits for a container to *start*, not for it to be *ready*. Use `healthcheck` + `depends_on: condition: service_healthy`. Even so, **services must tolerate dependencies being unavailable** (retry on startup, readiness DOWN), because in production nothing guarantees ordering.

### 3.4 Discovery under Compose
Containers resolve each other by service name via Docker's embedded DNS (`postgres:5432`, `kafka:9092`). This is *server-side discovery via DNS*, the same idea Kubernetes Services use (M18).

### 3.5 Kafka listeners (a classic trap)
Kafka advertises addresses to clients. Inside Docker, clients need `kafka:9092`; from your host, `localhost:29092`. Configure **two listeners** (`INTERNAL` and `EXTERNAL`). Most "can't connect to Kafka" bugs are wrong advertised listeners.

---

## 4. Scope

**In:** Compose with profiles, Postgres with per-service DB/users, Kafka (KRaft) + Kafka UI, Redis, a Mailpit placeholder, the `platform` modules skeleton, a task runner, the CI pipeline building images, the seed/reset script structure.
**Out:** Keycloak (M04), schema registry (M08), observability stack (M16).

---

## 5. Technical specification

### 5.1 Repository layout
```
nexusflow/
  pom.xml                       root aggregator (M00)
  mvnw  mvnw.cmd  .mvn/         Maven Wrapper (M00)
  build-parent/                 java-parent, service-parent POMs (M00)
  platform/
    platform-test/              (M00: ArchUnit rules, Testcontainers base)
    platform-web/               correlation-id filter, Problem Details handler, pagination DTOs
    platform-observability/     JSON logging config, MDC helpers
    platform-messaging/         event envelope (M06), outbox (M07), inbox (M07)
    platform-idempotency/       Idempotency-Key support (M05)
  services/                     (M03+)
  contracts/openapi/  contracts/events/
  infra/compose/docker-compose.yml  infra/compose/postgres/init/  infra/kafka/topics.sh
  scripts/  scripts/drills/
  Taskfile.yml                  cross-platform task runner (works on Windows)
```

### 5.2 Compose profiles
| Profile | Contents |
|---|---|
| `core` | postgres, kafka, kafka-ui, redis |
| `auth` | keycloak (M04) |
| `apps` | all services (built images) |
| `obs` | prometheus, grafana, jaeger/tempo, loki (M16) |
| `chaos` | toxiproxy (M09+) |
| `ai` | forecasting, intelligence (M19+) |

Resource limits: `mem_limit` on every container; JVM services `JAVA_TOOL_OPTIONS=-Xmx320m -XX:+UseSerialGC` locally.

### 5.3 Postgres init (`infra/compose/postgres/init/01-databases.sql`)
For each service `<svc>` in procurement, inventory, shipment, risk, recovery, notification, query:
```sql
CREATE USER <svc>_user WITH PASSWORD '<from env>';
CREATE DATABASE <svc>_db OWNER <svc>_user;
REVOKE ALL ON DATABASE <svc>_db FROM PUBLIC;
```
(Generate this from a list via a small script; passwords come from `.env`.)

### 5.4 Kafka
- Single broker, KRaft mode (combined controller+broker), `auto.create.topics.enable=false`.
- `infra/kafka/topics.sh` creates topics explicitly with partitions (3 for event topics) and retention. It is idempotent (`--if-not-exists`).

### 5.5 Taskfile targets
`task up` (core), `task up:all`, `task down`, `task reset` (wipe volumes + reseed), `task build`, `task test`, `task drills`, `task demo` (later).

### 5.6 Platform runtime modules (skeleton now, filled later)
- `platform-web`: `CorrelationIdFilter` (reads/creates `X-Correlation-Id`, puts it in MDC, echoes it in the response); `GlobalProblemDetailsHandler`; `PageResponse<T>`.
- `platform-observability`: logback JSON encoder config (e.g. logstash-logback-encoder, or Spring Boot's structured logging support), applied by default.
- Each is auto-configured (Spring Boot auto-configuration), so services get it just by depending on it. **That's the chassis.**

### 5.7 CI additions
Build each service's container image (Spring Boot buildpacks or a Dockerfile with a non-root, slim JRE); cache `~/.m2`; run Testcontainers tests (`*IT`, Failsafe) (GitHub runners have Docker).

---

## 6. Tasks

| ID | Task | Est | Concept | P |
|---|---|---|---|---|
| M02-T01 | Compose `core` profile: postgres (healthcheck `pg_isready`), kafka KRaft with internal/external listeners (healthcheck), kafka-ui, redis | 3h | Backing services, health-gated startup | P0 |
| M02-T02 | Postgres init script: DB + user per service, revoke public | 1h | Database per service | P0 |
| M02-T03 | `topics.sh`; disable auto-create | 1h | Topics as infrastructure | P0 |
| M02-T04 | `platform-web` + `platform-observability` skeletons with auto-configuration | 3h | Runtime chassis | P0 |
| M02-T05 | `Taskfile.yml` targets; README quick start | 1h | Developer experience | P0 |
| M02-T06 | CI: image build + Testcontainers job | 2h | Independent artefacts | P0 |
| M02-T07 | WSL2 `.wslconfig` memory; document the laptop budget in the README | 0.5h | Resource planning | P0 |
| M02-T08 | Break tests | 1.5h | — | P0 |

---

## 7. Break tests

| # | Drill | Expected |
|---|---|---|
| B1 | From the `procurement_user` connection, `\c inventory_db` | **Permission denied** |
| B2 | `docker compose up` from a fresh clone (`task reset`) | All healthy, < 2 min |
| B3 | Connect to Kafka from the host with the internal listener address | Fails. Explain why, then use the external one |
| B4 | Produce to a non-existent topic | Error (auto-create disabled) |
| B5 | `docker kill` Postgres and restart it | Data persists (volume); `task reset` wipes it |
| B6 | Remove the healthcheck dependency, start apps before Kafka is ready | Observe the failure; restore. Note it for M03's startup resilience |

---

## 8. Pitfalls

- Advertised listener misconfiguration (§3.5).
- Using the `latest` image tag. Pin versions (M00 rule).
- Putting passwords in `docker-compose.yml`. Use `${VAR}` from `.env`.
- Docker Desktop memory set too low → Kafka OOM-killed silently. Check `docker stats`.

## 9. Deliverables

Compose + init + topics scripts, platform skeleton modules, Taskfile, CI image build, ADR *Local runtime: Compose profiles & database-per-service implementation*, tag `m02-platform`.

## 10. Self-check

1. What are the three ways to implement database per service? Why did you choose yours?
2. Why isn't `depends_on` enough, and why must services tolerate missing dependencies anyway?
3. Explain Kafka advertised listeners in two sentences.
4. Which twelve-factor principles does your Compose setup demonstrate?

## 11. References (when stuck)

12factor.net · microservices.io *Database per service* · Docker Compose docs (healthcheck, profiles) · Confluent blog *Kafka Listeners – Explained* · Spring Boot docs *Creating your own auto-configuration*

## 12. Exit checklist

- [ ] `task reset && task up` healthy from a clean clone
- [ ] B1–B6 done with predictions
- [ ] Platform modules auto-configure in `sample-service` (then delete `sample-service`)
- [ ] ADR written · tag `m02-platform`
