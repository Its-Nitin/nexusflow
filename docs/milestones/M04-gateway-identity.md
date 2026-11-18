# M04 — API Gateway & Identity

| | |
|---|---|
| **Release** | R0 Foundation |
| **Duration** | 1 week (~22 h) |
| **Depends on** | M03 |
| **Unlocks** | All services become secured by default. **Gate G0** |
| **Tag** | `m04-edge-security`, `g0-foundation` |

---

## 1. Why this milestone exists

In a monolith there is one front door. In microservices there are many doors, plus internal traffic between services. Two questions must be answered once and applied everywhere: **who is calling?** (authentication) and **may they do this, for this tenant?** (authorisation). Doing this late means retrofitting every service, so we do it now while there is one.

---

## 2. Learning objectives

- Explain the OAuth 2.0 / OIDC flows you use: Authorization Code + PKCE for the SPA, Client Credentials for service to service.
- Explain how a resource server validates a JWT *offline* using JWKS, and what that means when the identity provider goes down.
- Explain **edge-only auth vs zero trust**, and why every service validates tokens.
- Implement tenant isolation that a forgotten `WHERE` clause can't break.
- Explain the gateway's responsibilities, and what must *not* go in the gateway (business logic).
- Implement token-bucket rate limiting backed by Redis.

---

## 3. Concepts

### 3.1 API gateway
**What:** a single entry point that routes requests to services and handles cross-cutting edge concerns: authentication, rate limiting, CORS, request IDs, TLS termination.
**Why we use it:** clients don't need to know service locations; edge policies live in one place; internal services aren't exposed.
**Trade-offs:** an extra hop; a potential bottleneck or single point of failure (scale it horizontally); and the temptation to put business logic or aggregation in it (don't, because that's how gateways become monoliths).

### 3.2 OAuth 2.0 / OIDC in one paragraph
Keycloak (the identity provider) authenticates the user and issues a signed **JWT access token** with claims (`sub`, `roles`, `tenant_id`, `exp`, `aud`). Services verify the signature with Keycloak's public keys (JWKS), fetched once and cached. **No call to Keycloak per request** — that's what makes it scale, and it's why revocation isn't instant (tokens stay valid until `exp`). Keep access tokens short-lived (5–15 min).

### 3.3 Zero trust (defence in depth)
**Edge-only:** the gateway validates; internal services trust anything on the network. If one internal service is compromised, or a port is accidentally exposed, everything is open.
**Zero trust:** every service validates the JWT and enforces its own authorisation. **We use zero trust.** The cost is a small CPU cost per request, plus config in every service (which the chassis provides).

### 3.4 Service-to-service identity
When Risk calls Inventory (M09), whose identity is used?
- **User token relay:** keeps the user's permissions (used for the AI assistant in M20).
- **Client credentials:** the service's own identity, for background/event-driven work where no user exists.
NexusFlow uses both, deliberately.

### 3.5 Multi-tenancy isolation
Relying on developers to add `WHERE tenant_id = ?` to every query fails eventually. Use **Hibernate's `@TenantId`** (discriminator-based multi-tenancy), with a `CurrentTenantIdentifierResolver` reading the tenant from the security context. Every query is filtered automatically. Cross-tenant reads return **404, not 403**: 403 would leak that the resource exists.

### 3.6 Rate limiting
**Token bucket:** each client has a bucket of N tokens refilled at rate R; each request consumes one; an empty bucket means 429. It's stored in **Redis**, so all gateway instances share the counts (a distributed-state problem in miniature).

### 3.7 SPA token handling: two designs
| | SPA holds tokens (PKCE, Bearer) | BFF / gateway holds tokens (session cookie + TokenRelay) |
|---|---|---|
| Simplicity | Simpler | More moving parts |
| Security | Tokens in browser JS (XSS risk) | Tokens never reach the browser |
**Choice:** SPA + PKCE for v1 (simpler), recorded in an ADR with the BFF upgrade path.

---

## 4. Scope

**In:** Keycloak realm (committed export), gateway routes + JWT validation + correlation ID + CORS + rate limiting, Procurement as a resource server with roles and tenant isolation, a client-credentials client, negative security tests, a `platform-security` module so later services get all this by default.
**Out:** UI login (M15), token relay for the AI assistant (M20).

---

## 5. Technical specification

### 5.1 Keycloak realm `nexusflow`
- Clients: `nexusflow-ui` (public, PKCE), `nexusflow-gateway` (bearer-only/resource), `risk-service`, `recovery-service`, `intelligence-service` (confidential, client credentials).
- Realm roles: `PLANNER`, `MANAGER`, `ANALYST`, `SERVICE`.
- Users: `planner.a`, `manager.a` (tenant A), `analyst.b`, `manager.b` (tenant B).
- Protocol mapper: user attribute `tenant_id` → token claim `tenant_id`. Audience mapper for `nexusflow-api`.
- Token lifetime: 10 min. Export to `infra/keycloak/realm-nexusflow.json`, imported at startup.

### 5.2 Gateway (Spring Cloud Gateway)
- Routes: `/api/v1/suppliers/**`, `/api/v1/purchase-orders/**` → procurement; others are added as services appear.
- Filters: JWT validation (resource server), `CorrelationId` (generate if missing), `RequestRateLimiter` (Redis, key = JWT `sub`; replenish 10/s, burst 20), request size limit, CORS for the UI origin.
- The gateway **does not** strip the Authorization header. Services validate it again.

### 5.3 `platform-security` module (auto-configured)
- JWT resource server config: issuer URI, audience check, role mapping from `realm_access.roles` → `ROLE_*`.
- A `TenantContext` populated from the `tenant_id` claim; MDC `tenantId`.
- Hibernate `CurrentTenantIdentifierResolver` bound to `TenantContext`.
- Method security enabled; deny-by-default for unannotated endpoints, except actuator health.

### 5.4 Procurement authorisation matrix
| Endpoint | PLANNER | MANAGER | ANALYST |
|---|---|---|---|
| GET suppliers/POs | ✔ | ✔ | ✔ |
| Create supplier / PO | ✔ | ✔ | ✖ |
| Confirm / cancel PO | ✖ | ✔ | ✖ |

---

## 6. Tasks

| ID | Task | Est | Concept | P |
|---|---|---|---|---|
| M04-T01 | Keycloak in Compose (`auth` profile); configure the realm via the UI; export it to JSON; auto-import | 3h | OIDC provider | P0 |
| M04-T02 | Gateway module: routes, correlation filter, CORS | 3h | Edge responsibilities | P0 |
| M04-T03 | Gateway JWT validation | 1.5h | Token validation via JWKS | P0 |
| M04-T04 | `platform-security`: resource-server auto-config, role mapping | 3h | Zero trust via the chassis | P0 |
| M04-T05 | Tenant isolation with Hibernate `@TenantId` + resolver; migrate Procurement tables (`tenant_id`) | 3h | Multi-tenancy | P0 |
| M04-T06 | `@PreAuthorize` per the authorisation matrix | 1h | Least privilege | P0 |
| M04-T07 | Redis rate limiter at the gateway | 2h | Distributed rate limiting | P0 |
| M04-T08 | Client-credentials client + a test obtaining a service token | 1h | Service identity | P1 |
| M04-T09 | Negative security test suite (Testcontainers Keycloak, or signed test JWTs) | 3h | Security as tests | P0 |
| M04-T10 | ADRs: *Auth at edge + every service*, *SPA token handling*, *No service registry (DNS discovery)* | 1.5h | — | P0 |

---

## 7. Break tests

| # | Drill | Expected |
|---|---|---|
| B1 | Call `procurement:8081` directly, no token | 401 |
| B2 | Valid token, ANALYST confirms a PO | 403 |
| B3 | Tenant-A token reads a tenant-B PO by ID | **404** |
| B4 | Tenant-A token lists POs | Only tenant-A rows (seed both tenants) |
| B5 | Expired token (wait, or mint a short-lived one) | 401 with `WWW-Authenticate` |
| B6 | Token for the wrong audience | 401 |
| B7 | 100 req/s from one user | 429s after the burst; Redis keys visible; another user unaffected |
| B8 | Stop Keycloak | Existing tokens keep working until expiry; new logins fail. Explain via JWKS caching |
| B9 | Kill Redis | Decide: rate limiter fails open or closed? Observe the default, and document the choice |

---

## 8. Pitfalls

- Mapping roles wrongly (Keycloak nests roles in `realm_access.roles`), so every check fails with 403.
- Issuer mismatch: tokens issued for `localhost:8180` vs services validating against `keycloak:8180`. Use one hostname consistently (Keycloak `KC_HOSTNAME`), or configure the JWK set URI separately from the issuer.
- Putting tenant checks in controllers by hand. Rely on the Hibernate tenant filter plus tests.
- Rate limiting by IP behind a proxy, so all users share one bucket.

## 9. Deliverables

Gateway, Keycloak realm export, `platform-security`, secured Procurement, security tests, 3 ADRs, tags `m04-edge-security` + `g0-foundation`.

> ✅ **Gate G0:** (1) a PR violating M00 standards is blocked; (2) `planner.a` logs in (token via Keycloak), creates a PO through the gateway, and gets 404 for a tenant-B PO.

## 10. Self-check

1. Why can services validate JWTs without calling Keycloak? What is the consequence for revocation?
2. Edge-only vs zero trust: what attack does zero trust stop?
3. Why 404 instead of 403 for cross-tenant access?
4. Client credentials vs token relay: when do you use each in NexusFlow?
5. Should a rate limiter fail open or closed when Redis is down? Argue both sides.

## 11. References (when stuck)

Spring Security *OAuth2 Resource Server* docs · Spring Cloud Gateway docs (RequestRateLimiter) · Keycloak server admin guide (mappers, realm export) · Hibernate ORM user guide *Multitenancy / @TenantId* · *Building Microservices* 2e ch. 11 · oauth.net/2/pkce

## 12. Exit checklist

- [ ] B1–B9 pass and are automated where possible
- [ ] `platform-security` auto-configures (new services secure by default)
- [ ] Realm export committed; `task up` imports it
- [ ] ADRs written · Gate G0 demo recorded in the journal · tags pushed
