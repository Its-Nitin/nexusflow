# M00 — Engineering Standards & Codebase Guidelines

| | |
|---|---|
| **Release** | R0 Foundation |
| **Duration** | 1 week (~22 h) |
| **Depends on** | — (first milestone) |
| **Unlocks** | M01 onwards. Every later line of code must follow what is decided here |
| **Tag** | `m00-standards` |

---

## 1. Why this milestone exists

In a monolith, inconsistency hurts one codebase. In microservices it hurts **N codebases**. Without agreed rules, eight services end up with eight error formats, eight logging styles, eight package layouts and eight ways of publishing events. Debugging a request that crosses five of them becomes guesswork.

Companies that run microservices well fix this with a **paved road** (also called a "golden path"): written standards, plus tooling that *enforces* them automatically. This milestone builds NexusFlow's paved road **before** any feature code exists, so every service is consistent from its first commit.

A rule that isn't enforced by tooling is only a suggestion. So for every rule you will also decide **how it is enforced** (formatter, compiler check, ArchUnit test, CI gate, PR checklist), and you will **prove the enforcement works by deliberately violating it**.

---

## 2. Learning objectives

By the end you can:
- Explain why consistency across services matters more than consistency inside a single service.
- Explain the **microservice chassis** pattern and how a shared build plugin and a `platform` library act as one.
- Explain **architecture fitness functions** and write ArchUnit rules that enforce hexagonal layering.
- Explain **shift-left quality**: catching problems at commit and PR time instead of in production.
- Set up a Maven multi-module monorepo with **parent POMs** (inheritance) and a root **aggregator POM**, so every service gets the same versions and quality gates.
- Defend each standard with a *reason*, not just "best practice."

---

## 3. Concepts

### 3.1 Microservice chassis
**What:** a reusable foundation for cross-cutting concerns (config, logging, health, tracing, error handling, messaging) that every service builds on.
**Why we use it:** without it, every service re-implements the same infrastructure code slightly differently. The chassis keeps behaviour uniform, so a correlation ID works the same way everywhere.
**Trade-off:** a shared library *couples* services. If it contains domain logic, services can no longer change independently. **Rule: the chassis contains only technical concerns, never domain types.**
**In NexusFlow:** the `nexusflow-service-parent` POM (build-time chassis) plus the `platform/*` modules (runtime chassis).

### 3.2 Paved road / golden path
**What:** the easiest way to do something is also the correct way.
**Why:** people (including future you) follow the path of least resistance. If a compliant service POM is just "parent + artifactId + dependencies without versions", nobody will create a non-compliant one.

### 3.2a Maven: inheritance vs aggregation (just enough to start)
- **Aggregation** (root `pom.xml`, `<packaging>pom</packaging>`, `<modules>`): *which* modules build together, and in what order (the reactor).
- **Inheritance** (`<parent>`): *how* a module is built, i.e. the versions, plugins and rules it inherits.
- A module has **one** parent. So the chassis is a chain: `nexusflow-java-parent` (Java conventions, for libraries) ← `nexusflow-service-parent` (adds Spring Boot packaging + ArchUnit, for services). This is the Maven answer to "composable conventions"; record the single-parent limitation in ADR-0002.
- `<dependencyManagement>` / `<pluginManagement>` **declare** versions and config without adding anything. Children then use dependencies and plugins *without* versions. That is the "versions live in one place" rule.

### 3.3 Architecture fitness functions
**What:** automated tests that check *architectural* properties, not behaviour. Examples: "the domain layer must not import Spring" and "controllers must not call repositories directly."
**Why:** architecture erodes quietly, one shortcut at a time. Fitness functions turn architecture into a failing build.
**Tool:** ArchUnit (JUnit-based).

### 3.4 Hexagonal architecture (ports & adapters) per service
**What:** the domain and application core sit in the middle and know nothing about HTTP, Kafka or SQL. Adapters on the outside translate.
**Why it matters especially in distributed systems:** you will *swap transports* during this project. The Risk service starts with REST calls (M09), moves to Kafka projections, and the forecast source changes in M19. With ports and adapters these are adapter swaps, not rewrites. It also makes the domain unit-testable without containers.
**Trade-off:** more packages and mapping code. Acceptable here because the learning value and the swaps are real.

### 3.5 Shift-left quality
**What:** move checks as early as possible: IDE → pre-commit → build → PR → CI.
**Why:** a formatting error caught in CI costs minutes. The same class of error caught in production during an incident costs hours.

### 3.6 Conventional Commits and trunk-based development
**What:** structured commit messages (`feat(inventory): add atomic reservation`) and short-lived branches merged into `main` often.
**Why:** a readable history *is* documentation of how the architecture evolved, which is valuable for your portfolio and your retrospective. Short branches avoid painful merges across a monorepo.

### 3.7 Architecture Decision Records (ADRs)
**What:** short documents recording a decision, its context, the alternatives and the consequences.
**Why:** in distributed systems most decisions are trade-offs (consistency vs availability, sync vs async). Six weeks later you will not remember *why*. Interviewers ask "why", too.

---

## 4. Scope

**In scope:** written standards (`docs/standards/`), repo initialisation, Maven Wrapper + root aggregator POM + parent POMs (`build-parent/`), centralised dependency management (BOMs), formatting, static analysis, ArchUnit rules, test conventions, git workflow, PR template, secret scanning, dependency update policy, documentation templates, CI skeleton running the gates.

**Out of scope:** Docker Compose infrastructure and `platform` runtime code (M02), and real business services (M03+). You will use a tiny **`sample-service`** module only to prove that the rules fire. Delete it at the end of M02.

---

## 5. Technical specification: the standards to write

Create `docs/standards/` with the files below. Each file is short (1–3 pages) and has three parts: **Rule → Reason → Enforcement**.

### 5.1 `01-tech-stack-and-versions.md`
- Verify current versions on day 1 and **pin** them in `build-parent/java-parent/pom.xml` (`<properties>` + `<dependencyManagement>`): JDK (Java 25 LTS, or 21 if a library forces it), Maven (3.9.x, or 4.x only if it is GA and every plugin you use supports it), Spring Boot 4.0.x, the matching Spring Cloud release train, Testcontainers, Kafka client 4.x, Postgres 17/18, Keycloak, Node LTS, Angular.
- **Maven** (via the Maven Wrapper `./mvnw`, version pinned in `.mvn/wrapper/maven-wrapper.properties`) is the build tool. Record the decision, and Gradle as the rejected alternative, in ADR-0002 (*Build tool & monorepo layout*).
- Spring Boot and Spring Cloud versions come from their **BOMs**: `java-parent` inherits `spring-boot-starter-parent` and imports `spring-cloud-dependencies` (`<scope>import</scope>`). Every other library version is a property in `java-parent`.
- Enforcement: `maven.compiler.release=25`; **Maven Enforcer** in `java-parent` with `requireJavaVersion`, `requireMavenVersion`, `dependencyConvergence`, `requirePluginVersions` and `bannedDependencies`; parent POMs are the *only* place versions appear (a CI script rejects any `<version>` in `services/*/pom.xml` and `platform/*/pom.xml` other than the `<parent>` one).

### 5.2 `02-code-style.md`
| Rule | Reason | Enforcement |
|---|---|---|
| Formatter: **palantir-java-format** (or google-java-format), applied by `spotless-maven-plugin` | No style debates; diffs show only real changes | `spotless:check` bound to `verify`; `spotless:apply` in pre-commit |
| `.editorconfig` for all file types (UTF-8, LF, final newline, 2-space for YAML/TS/JSON) | Consistent across IDEs | Spotless misc + EditorConfig |
| Naming: classes `PascalCase`, methods `camelCase`, constants `UPPER_SNAKE`, packages lowercase; no abbreviations except `Id`, `Dto`, `Api` | Readability | Review + Checkstyle naming rules (optional) |
| Prefer immutable types: Java `record`s for DTOs, commands, events, value objects | Thread safety; safe to pass through async boundaries | Review; ArchUnit: classes in `..event..` must be records |
| No `null` returns from public methods; use `Optional` for "maybe absent" query results | Avoid NPEs crossing layers | Error Prone + NullAway (`@NullMarked` packages) |
| No field injection (`@Autowired` on fields); constructor injection only | Testability, immutability | ArchUnit rule |
| Time: `Instant` for timestamps, `LocalDate` for business dates; **never** `LocalDateTime` for event times; inject `Clock` | Distributed nodes ⇒ UTC and testable time | ArchUnit: no `LocalDateTime.now()`, no `Instant.now()` outside `Clock` usage |
| Money: `BigDecimal` + ISO-4217 currency in a `Money` value object; never `double` | Rounding errors | ArchUnit/Checkstyle: no `double`/`float` fields in `..domain..` |
| Lombok: **not used** (records and the IDE cover it) | Less magic; explicit code is better for learning | Enforcer `bannedDependencies` in `java-parent` |

### 5.3 `03-service-architecture.md` (hexagonal package layout)
Every service uses exactly this layout:
```
com.nexusflow.<service>
 ├─ domain/                 # entities, value objects, domain events, domain services. NO Spring, NO JPA annotations*
 ├─ application/            # use cases (one class per command/query), ports
 │   ├─ port/in/            # use-case interfaces (driving ports)
 │   └─ port/out/           # repository, event-publisher, external-service interfaces (driven ports)
 ├─ adapter/
 │   ├─ in/web/             # REST controllers, request/response DTOs, mappers
 │   ├─ in/messaging/       # Kafka listeners
 │   ├─ out/persistence/    # JPA entities, Spring Data repositories, port implementations
 │   ├─ out/messaging/      # outbox writer, command senders
 │   └─ out/client/         # HTTP clients to other services
 └─ config/                 # Spring configuration and wiring
```
*Decision to record in an ADR: separate JPA entities from domain entities (purer, with more mapping) or annotate domain entities with JPA (pragmatic). **Recommendation: separate them** in learning mode. Pure domain makes the M09 adapter swap and the M10 determinism tests trivial.

**ArchUnit rules (in `platform-test`, applied to every service):**
1. `domain` depends on nothing outside `domain` and `java.*` (no Spring, Jakarta, Jackson).
2. `application` may depend on `domain` only (plus `jakarta.transaction`/Spring `@Transactional` if you allow it; decide and document).
3. `adapter.in` must not access `adapter.out` directly. It goes through `application.port.in`.
4. Controllers live only in `adapter.in.web`; `@KafkaListener` only in `adapter.in.messaging`.
5. No cycles between packages (`slices().should().beFreeOfCycles()`).
6. No class from another service's package (`com.nexusflow.<other>`) is ever imported. **This is the code-level guard on service boundaries.**
7. Classes named `*Controller`, `*Listener`, `*Repository`, `*UseCase` sit in their correct packages.

### 5.4 `04-api-guidelines.md`
- URLs: `/api/v1/<plural-kebab-resource>` (e.g. `/api/v1/purchase-orders/{id}`). Actions that aren't CRUD use sub-resources: `POST /recommendations/{id}/approve`.
- JSON: `camelCase`; dates ISO-8601 (`2026-11-16` for dates, `2026-11-16T10:00:00Z` for instants, always UTC with `Z`); money as `{ "amount": "1250.00", "currency": "EUR" }` (string amount, so no float rounding).
- IDs: UUID strings. Never expose database sequence numbers.
- Errors: **RFC 9457 Problem Details** with `type`, `title`, `status`, `detail`, `instance`, plus extensions `code` (machine-readable, e.g. `PO_INVALID_TRANSITION`), `correlationId` and `errors[]` for field validation.
- Status codes: 200 / 201 (+`Location`) / 202 (accepted, async) / 204 / 400 validation / 401 / 403 / 404 (also for other-tenant resources) / 409 conflict or state / 412 precondition (ETag) / 422 semantic / 429 / 503 with `Retry-After`.
- Pagination: `?page=0&size=20&sort=createdAt,desc`; response `{ items, page, size, totalItems }`. Max `size` is 100.
- Concurrency: `ETag`/`If-Match` on updates of mutable resources (optimistic concurrency at the API level).
- Idempotency: every non-idempotent `POST` that a client may retry accepts an `Idempotency-Key` header (implemented in M05).
- Headers: `X-Correlation-Id` accepted and echoed; `traceparent` propagated.
- Contract-first: the OpenAPI spec in `contracts/openapi/<service>.yaml` is the source of truth; interfaces are generated from it.
- Versioning: a breaking change means a new `/v2` path. Additive changes never break clients. Clients must ignore unknown fields (tolerant reader).
- Enforcement: Spectral lint of OpenAPI files in CI (with a small `.spectral.yaml` ruleset for these rules); generated interfaces mean the code can't drift from the spec.

### 5.5 `05-event-and-messaging-guidelines.md`
- Event names: **past tense, PascalCase**, business language: `ShipmentDelayed`, not `ShipmentUpdatedEvent` or `UpdateShipment`.
- Command names: **imperative**: `ReallocateReservation`.
- Topics: `<bounded-context>.events`, `<bounded-context>.commands`, `<service>.replies`; DLT `<topic>.<consumer-group>.DLT`. Topics are created by infrastructure scripts; `auto.create.topics.enable=false`.
- Partition key = `aggregateId` (ordering per aggregate).
- The envelope is mandatory: `eventId (UUID)`, `eventType`, `eventVersion (int)`, `aggregateType`, `aggregateId`, `aggregateVersion (long)`, `occurredAt (Instant)`, `tenantId`, `correlationId`, `causationId`, `payload`.
- Events are **facts**: immutable, never deleted or edited. Corrections are new events.
- Payloads carry what consumers need to avoid calling back (event-carried state), but **no PII beyond what's needed**, and never secrets.
- Schema changes: additive only within a version; breaking means a new `eventVersion` plus a migration plan. The compatibility mode is BACKWARD (enforced by the schema registry from M08).
- Delivery contract: **at-least-once**. Every consumer **must** be idempotent. Never assume ordering across different aggregates.
- Publishing: **only via the transactional outbox** (from M07). Calling `KafkaTemplate.send` directly from a use case is forbidden (ArchUnit rule added in M07).
- Every event type is documented in `contracts/events/` with an example.

### 5.6 `06-database-guidelines.md`
- One database **and one DB user** per service; a service's user has no grants on other databases.
- Naming: `snake_case` tables (plural) and columns; PK `id UUID`; FK `<entity>_id` *only within the same service*.
- Standard columns: `created_at timestamptz`, `updated_at timestamptz`, `version bigint` (optimistic locking) on mutable aggregates, `tenant_id uuid not null` on tenant data.
- Migrations: Flyway, `V<yyyyMMddHHmm>__<description>.sql`. **Never edit an applied migration.** Schema changes follow **expand → migrate → contract** so old and new versions of a service can run side by side during a rolling deploy (matters in M18).
- No cross-service foreign keys or joins. References to other services' entities are plain UUID columns.
- Money `numeric(19,4)` + `currency char(3)`; timestamps `timestamptz`; enums stored as `text` with a `CHECK` constraint.
- Indexes for every foreign key and every query predicate you write. Review query plans for the hot paths (inventory reservation, outbox polling).
- `spring.jpa.hibernate.ddl-auto=validate` (never `update`); `open-in-view=false`.
- Enforcement: Flyway validate on startup; Testcontainers tests run migrations; CI fails if `ddl-auto` is anything other than `validate`/`none`.

### 5.7 `07-testing-standards.md`
| Level | Scope | Tools | Rules |
|---|---|---|---|
| Unit | domain + application | JUnit 5, AssertJ, (Mockito only for ports) | No Spring context; fast (<100 ms each) |
| Integration | one adapter against real infrastructure | Testcontainers (Postgres, Kafka) | **No H2**: a different database means different behaviour |
| Architecture | package rules | ArchUnit | Runs in every service |
| Contract | API/event compatibility | Spectral (M00), Pact or Spring Cloud Contract (M17) | |
| Component/E2E | the service or the system | REST-assured, Compose-based | |
| Drill | failure behaviour | Testcontainers + Toxiproxy, `scripts/drills/` | Every break test from a milestone becomes a drill |

- Test split: unit tests are `*Test` (Surefire, run by `./mvnw test`); integration tests are `*IT` (Failsafe, run by `./mvnw verify`). The fast loop never needs Docker.
- Test names: `methodUnderTest_condition_expectedResult` or `@DisplayName("reserving more than available is rejected")`.
- Structure: Given / When / Then comments.
- Deterministic: inject `Clock`, fixed seeds, no `Thread.sleep` (use Awaitility for async assertions).
- Coverage: JaCoCo reports; the gate is **≥ 80% line coverage on `domain` and `application`**. Adapters are covered by integration tests, not by a percentage target.
- Shared Testcontainers via a `platform-test` base class with reusable containers.

### 5.8 `08-logging-and-observability.md`
- JSON logs to stdout (12-factor); no file appenders.
- Mandatory MDC keys: `traceId`, `spanId`, `correlationId`, `tenantId`, `service`; plus `eventId` and `messageKey` in consumers.
- Levels: `ERROR` = needs human action; `WARN` = degraded but handled (retry, fallback); `INFO` = business state changes (one line per use case); `DEBUG` = off in shared environments.
- **Never log:** tokens, passwords, full request bodies containing PII, or LLM prompts without redaction.
- Metric naming: `nexusflow_<domain>_<measure>_<unit>` (e.g. `nexusflow_outbox_pending_events`). Low-cardinality labels only (never userId or eventId as a label).
- Every service exposes `/actuator/health/liveness`, `/actuator/health/readiness` and `/actuator/prometheus`.

### 5.9 `09-security-guidelines.md`
- No secrets in git, ever. `.env` is git-ignored; `.env.example` holds non-secret defaults. **gitleaks** runs in pre-commit and CI.
- Every service validates JWTs itself (zero trust, M04); deny by default; `@PreAuthorize` on use cases.
- Tenant scoping is mandatory on every query (enforced through a Hibernate tenant filter in M04, plus a cross-tenant test per endpoint).
- Validate all input at adapter boundaries (Bean Validation); never trust event payloads blindly either.
- Dependencies: Dependabot or Renovate weekly; OWASP dependency-check or Trivy in CI; fail on HIGH/CRITICAL with a known fix.
- Containers: non-root user, distroless or slim base images, no `latest` tags.

### 5.10 `10-git-workflow.md`
- Trunk-based: `main` is always releasable; branches live less than 2 days: `<type>/<milestone>-<task>-<slug>`, e.g. `feat/M05-T05-atomic-reservation`.
- **Conventional Commits:** `feat|fix|refactor|test|docs|build|ci|chore(<scope>): <summary>`, where scope = service or `platform`. Reference the task: `Refs: M05-T05`.
- PRs, even solo: a PR template with a checklist (below); self-review the diff on GitHub before merging; squash merge.
- Branch protection on `main`: CI must pass; no direct pushes.
- PR template checklist:
  - [ ] Follows `docs/standards` (layout, API, events, DB)
  - [ ] Tests added; break test automated
  - [ ] No secrets, no PII in logs
  - [ ] OpenAPI / event contracts updated
  - [ ] Migrations follow expand/contract
  - [ ] ADR written if a decision was made
  - [ ] Journal entry written

### 5.11 `11-documentation-standards.md`
- **ADR template (MADR style):** Title · Status · Context · Decision drivers · Options considered · Decision · Consequences (positive / negative) · Related milestones. File: `docs/adr/NNNN-kebab-title.md`.
- **Runbook template:** Symptom · Impact · Detection (dashboard/alert) · Diagnosis steps · Remediation · Verification · Prevention.
- **Journal entry template:** Date · Task ID · Problem I hit · Why it happened · What I changed · Trade-off / what it costs · Open questions.
- **Service README template** (one per service): purpose · owned data · APIs · events produced/consumed · config · how to run/test.
- Diagrams as code: Mermaid in markdown, or Structurizr DSL for C4.

### 5.12 `12-definition-of-done.md`
Copy the task, milestone and release DoD from `docs/project-plan.md` §3 and make it the PR checklist source.

### 5.13 `13-ai-assisted-development.md` (learning protection)
- **You write the first version yourself.** AI may review, explain, suggest tests or debug after you have tried.
- Never paste generated code you can't explain line by line. If you can't explain it, it goes in the journal as an open question.
- Break tests and predictions are always yours.
- AI-generated code follows the same standards and gates; nothing is exempt.

---

## 6. Tasks

| ID | Task | Est | Concept | P |
|---|---|---|---|---|
| M00-T01 | Create the GitHub repo `nexusflow`; `git init`; commit the blueprint and `docs/`; enable branch protection on `main` | 1h | Trunk-based workflow | P0 |
| M00-T02 | Verify current versions (start.spring.io, Kafka, Keycloak); write `01-tech-stack-and-versions.md`; ADR *Build tool & monorepo layout* | 2h | Pinning, reproducibility | P0 |
| M00-T03 | Maven Wrapper (`mvn wrapper:wrapper`); root aggregator `pom.xml`; `build-parent/java-parent` (inherits `spring-boot-starter-parent`; imports the Spring Cloud BOM; version properties; Enforcer, Spotless, Error Prone + NullAway via `maven-compiler-plugin`, JaCoCo with `check`, Surefire/Failsafe) and `build-parent/service-parent` (inherits java-parent; adds `spring-boot-maven-plugin` + ArchUnit/`platform-test` test dependency) | 5h | Build-time chassis, golden path, inheritance vs aggregation | P0 |
| M00-T04 | `sample-service` module whose `pom.xml` has only `<parent>`, `<artifactId>` and version-less `<dependencies>` | 0.5h | Golden path | P0 |
| M00-T05 | Write standards files 02–13 (above) | 6h | Every concept in §3 | P0 |
| M00-T06 | `platform/platform-test` module: ArchUnit rules 1–7 as a reusable `NexusflowArchitectureRules` class; `sample-service` runs them | 3h | Fitness functions | P0 |
| M00-T07 | `.editorconfig`, `.gitignore`, `.env.example`, `.gitattributes` (LF) | 0.5h | Consistency on Windows + Linux | P0 |
| M00-T08 | Pre-commit hooks (the `pre-commit` framework): `./mvnw spotless:apply`, gitleaks, commit-message lint | 1.5h | Shift-left | P0 |
| M00-T09 | `.github/pull_request_template.md`, issue templates (task, bug), `CODEOWNERS` (yourself) | 0.5h | Process as code | P0 |
| M00-T10 | CI skeleton `.github/workflows/ci.yml` (cache `~/.m2`): `./mvnw -B verify` (Enforcer → `spotless:check` → compile with Error Prone → unit + ArchUnit tests → integration tests → JaCoCo gate) → no-version-literal script → gitleaks → dependency scan → Spectral (for when specs exist) | 2h | Quality gates | P0 |
| M00-T11 | Dependabot/Renovate config | 0.5h | Supply-chain hygiene | P1 |
| M00-T12 | Break tests (§7), with predictions in the journal | 2h | Prove enforcement | P0 |

---

## 7. Break tests (acceptance criteria)

For each one, write a prediction, make the violation in a branch, open a PR, and confirm it is **blocked**:

| # | Violation | Must be caught by |
|---|---|---|
| B1 | Badly formatted Java file | pre-commit (auto-fixed) and CI `spotless:check` |
| B2 | `import org.springframework…` inside `domain` | ArchUnit rule 1 |
| B3 | Controller calls a repository directly | ArchUnit rule 3 |
| B4 | Field `@Autowired` | ArchUnit rule |
| B5 | `double price` in a domain class | ArchUnit/Checkstyle rule |
| B6 | `Instant.now()` called directly in application code | ArchUnit rule |
| B7 | Fake AWS key committed in a test file | gitleaks (pre-commit + CI) |
| B8 | Commit message `fixed stuff` | commit-message lint |
| B9 | A `<version>` on a dependency in `sample-service/pom.xml` | CI no-version-literal script |
| B9a | Add Lombok as a dependency | Enforcer `bannedDependencies` |
| B10 | Domain coverage below 80% | JaCoCo gate |
| B11 | Method returning `null` from a `@NullMarked` package | NullAway |

Record any violation that was **not** caught as a gap, and fix the tooling.

---

## 8. Pitfalls

- **Writing standards you won't enforce.** Every rule needs an "Enforcement" line, or remove it.
- **Over-engineering the gates.** If the build takes more than 2 minutes for `sample-service`, you'll start skipping it. Keep the fast checks local and the slow ones in CI.
- **Putting domain code in `platform`.** This creates a distributed monolith. Platform is technical only.
- **Error Prone/NullAway friction.** Start with NullAway on `domain` and `application` packages only and widen later.
- **Error Prone on JDK 16+** needs `--add-exports`/`--add-opens` flags in `.mvn/jvm.config`, otherwise the compiler fails with cryptic access errors. Follow the Error Prone Maven install docs exactly.
- **Slow full reactor builds.** Build one service with its dependencies: `./mvnw -pl services/<svc> -am verify`. Consider `mvnd` (Maven Daemon) later if builds feel slow.
- **Using `<dependencies>` in a parent where you meant `<dependencyManagement>`.** The first forces the dependency on every child; the second only declares the version.
- **Windows line endings** breaking scripts and Spotless: `.gitattributes` with `* text=auto eol=lf`.
- **Spending more than a week here.** Standards evolve. Version them, and amend through ADRs when a later milestone teaches you something.

---

## 9. Deliverables

- `docs/standards/01…13-*.md`
- `mvnw`, `mvnw.cmd`, `.mvn/` (wrapper, `jvm.config`), root `pom.xml`, `build-parent/java-parent/pom.xml`, `build-parent/service-parent/pom.xml`, `scripts/ci/check-no-version-literals.sh`
- `platform/platform-test` with `NexusflowArchitectureRules`
- `.editorconfig`, `.gitattributes`, `.gitignore`, `.env.example`, pre-commit config
- `.github/` PR template, issue templates, `ci.yml`, Dependabot/Renovate
- ADRs: *Record architecture decisions*, *Build tool & monorepo layout*, *Hexagonal architecture per service*, *Separate persistence model from domain model*
- Journal entries for B1–B11
- Tag `m00-standards`

---

## 10. Self-check

1. Why does consistency matter more across 8 services than within 1?
2. What is a microservice chassis? What must never go into it, and why?
3. What is an architecture fitness function? Give three from your codebase.
4. Why hexagonal architecture *specifically* for a project where transports will change?
5. Why `Instant` + an injected `Clock` instead of `LocalDateTime.now()`, in a distributed system?
6. Why no H2 in integration tests?
7. Why must migrations follow expand/contract once you do rolling deployments?
8. What does trunk-based development buy a solo developer?

---

## 11. References (when stuck)

- microservices.io → *Microservice chassis*, *Service template*
- *Building Evolutionary Architectures* (Ford, Parsons, Kua), ch. on fitness functions
- ArchUnit user guide · Maven docs: *Introduction to the POM* (inheritance vs aggregation), *Introduction to the Dependency Mechanism* (dependency management, BOM import), Maven Enforcer rules, Maven Wrapper · Spring Boot Maven plugin reference · Error Prone *Installation → Maven*
- Alistair Cockburn, *Hexagonal Architecture* (original article)
- conventionalcommits.org · RFC 9457 · Zalando RESTful API Guidelines (an excellent reference for §5.4)
- MADR template (adr.github.io)

---

## 12. Exit checklist

- [ ] All 13 standards files written, each rule with Reason + Enforcement
- [ ] `sample-service` builds with a POM that has only parent, artifactId and version-less dependencies
- [ ] CI runs all gates on every PR; `main` protected
- [ ] B1–B11 (incl. B9a) violations all blocked (evidence in the journal)
- [ ] ADRs written
- [ ] Self-check answered aloud
- [ ] Tag `m00-standards` pushed
