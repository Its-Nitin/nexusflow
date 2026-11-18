# M18 — Kubernetes Deployment

| | |
|---|---|
| **Release** | R5 Production Ready |
| **Duration** | 1 week (~24 h) |
| **Depends on** | M17 |
| **Unlocks** | **Gate G5** |
| **Tag** | `m18-kubernetes`, `g5-production-ready` |

---

## 1. Why this milestone exists

Docker Compose runs containers. Kubernetes **operates** them: it restarts failed ones, routes only to ready ones, rolls out new versions gradually, scales them, and gives every service a stable DNS name. Several distributed-systems ideas you've used informally become explicit here: **service discovery, health-based routing, graceful termination during rollouts, horizontal scaling, and config/secret separation.** This is also where M00's expand/contract migrations and M03's probes and graceful shutdown get tested for real.

---

## 2. Learning objectives

- Deploy all services with Helm (or Kustomize): Deployments, Services, ConfigMaps, Secrets, Ingress.
- Explain Kubernetes Services as server-side discovery + load balancing.
- Tune startup/readiness/liveness probes and explain the cascading-restart failure mode.
- Achieve zero-downtime rolling updates (preStop, graceful shutdown, Kafka consumer handover).
- Explain why CPU-based HPA is a poor signal for consumers, and autoscale on lag (KEDA).
- Run stateful infrastructure via operators (Strimzi, CloudNativePG), or justify keeping it outside.

---

## 3. Concepts

### 3.1 Service discovery in Kubernetes
A **Service** gives a stable virtual IP and DNS name (`inventory-service.nexusflow.svc`) that load-balances across the **ready** pods. Pods come and go; the name doesn't. This is why NexusFlow never needed Eureka (M04 ADR).

### 3.2 Probes, precisely
- **startupProbe:** gives slow-starting JVMs time; liveness/readiness wait until it passes.
- **readinessProbe:** failing → removed from Service endpoints (no traffic), not restarted.
- **livenessProbe:** failing → restarted. Must not depend on external systems (M03), or a DB blip restarts everything.

### 3.3 Rolling updates without dropped requests
Sequence on pod termination: removed from endpoints **asynchronously** while `SIGTERM` is sent. Without a short `preStop` sleep, the pod may receive traffic after it started shutting down. Then graceful shutdown finishes in-flight requests; Kafka consumers **commit and leave the group** (fast cooperative rebalance). `maxUnavailable: 0`, `maxSurge: 1`. And **schema migrations must be backward compatible** (expand/contract), because old and new pods run together.

### 3.4 Scaling consumers
HPA on CPU reacts to symptoms; for consumers the true demand signal is **lag**. **KEDA** scales Deployments on Kafka lag, capped at the partition count (M17).

### 3.5 Stateful infrastructure
Databases and brokers need stable identity and storage (StatefulSets) plus operational knowledge (failover, upgrades). **Operators** encode that knowledge: Strimzi (Kafka), CloudNativePG (Postgres). Running them teaches a lot. In real companies, managed services are often the better call. Document it.

### 3.6 Config & secrets
ConfigMaps for non-secret config; Secrets for credentials. Secrets in git are **encrypted** (SOPS or sealed-secrets), never plain base64.

---

## 4. Scope

**In:** a local cluster (k3d or kind), the Helm umbrella chart + per-service charts (or a library chart), Ingress → gateway, probes + preStop + resources, the operators or external infrastructure, encrypted secrets, Keycloak on the cluster, `task k8s:up`, rolling-update and pod-kill drills, KEDA (P1).
**Out:** cloud deployment, service mesh, GitOps (mention as future).

---

## 5. Technical specification

### 5.1 Chart structure
`infra/k8s/charts/nexusflow-service` (a library/base chart: Deployment, Service, ConfigMap, probes, resources, preStop, PodDisruptionBudget, ServiceMonitor) + `infra/k8s/charts/nexusflow` (umbrella: one release per service via values). Values per service: image, port, env, resources (`requests` 256Mi/100m, `limits` 512Mi), replicas.

### 5.2 Pod template essentials
```yaml
startupProbe:   httpGet /actuator/health/liveness  failureThreshold 30 periodSeconds 2
readinessProbe: httpGet /actuator/health/readiness periodSeconds 5
livenessProbe:  httpGet /actuator/health/liveness  periodSeconds 10
lifecycle.preStop.exec: ["sh","-c","sleep 5"]
terminationGracePeriodSeconds: 30
strategy: RollingUpdate maxUnavailable 0, maxSurge 1
```

### 5.3 Infrastructure options (choose and ADR)
| Option | Learning | Effort |
|---|---|---|
| Strimzi + CloudNativePG + Keycloak in-cluster | Highest | High |
| Infrastructure stays in Compose, cluster connects to the host | Low | Low |
Recommendation: Strimzi + CNPG if time allows; else external, with a written note.

### 5.4 Flyway in K8s
Run migrations on app startup (simple), or as a pre-upgrade Helm hook Job (cleaner). Either way, expand/contract.

---

## 6. Tasks

| ID | Task | Est | Concept | P |
|---|---|---|---|---|
| M18-T01 | k3d/kind cluster + ingress controller + `task k8s:up` | 2h | Cluster basics | P0 |
| M18-T02 | Base chart + one service deployed and reachable | 4h | Declarative deployment | P0 |
| M18-T03 | All services via the umbrella chart; config via ConfigMaps | 3h | Config separation | P0 |
| M18-T04 | Infrastructure: Strimzi + CNPG + Keycloak (or external) | 5h | Operators / stateful sets | P0 |
| M18-T05 | Secrets with SOPS/sealed-secrets | 2h | Secret management | P0 |
| M18-T06 | Probes, preStop, PDBs, resources tuned | 2h | Safe rollouts | P0 |
| M18-T07 | Observability stack on the cluster (or point the cluster at the Compose `obs`) | 2h | — | P0 |
| M18-T08 | Drills (§7) | 2.5h | — | P0 |
| M18-T09 | KEDA ScaledObject for Risk on lag | 2h | Lag-based autoscaling | P1 |
| M18-T10 | Deployment guide + ADRs (*K8s infra choice*, *Migration strategy in K8s*) | 1.5h | — | P0 |

---

## 7. Break tests

| # | Drill | Expected |
|---|---|---|
| B1 | `kubectl delete pod` Risk mid-processing (under load) | No loss, no duplicate effects (outbox/inbox); lag recovers |
| B2 | Rolling update of Inventory under k6 load | 0 failed requests at the gateway |
| B3 | Remove preStop and repeat B2 | Some failed requests: observe and explain |
| B4 | Break the readiness endpoint (config flag) on one pod | The pod receives no traffic; it is not restarted |
| B5 | Make liveness depend on the DB (wrong on purpose), then kill the DB | All pods restart (cascading): explain, then revert |
| B6 | A rolling update containing an expand-only migration | Old and new pods work simultaneously |
| B7 | (P1) Burst of delay events | KEDA scales Risk up to ≤ the partition count, then back down |

> ✅ **Gate G5:** the flagship demo runs on Kubernetes, one trace spans all services, and the M17 chaos suite passes against the cluster.

---

## 8. Pitfalls

- Too-low memory limits → OOMKilled JVMs (set `-XX:MaxRAMPercentage=75`, not a fixed `-Xmx` larger than the limit).
- Liveness = readiness (the same endpoint) → cascading restarts.
- `latest` image tags with `imagePullPolicy: IfNotPresent` (pods run stale images).
- Keycloak issuer URL mismatch between the in-cluster and browser hostnames (again, as in M04).

## 9. Deliverables

Charts, cluster scripts, encrypted secrets, deployment guide, drills, ADRs, tags.

## 10. Self-check

1. How does a Kubernetes Service implement discovery and load balancing?
2. Startup vs readiness vs liveness: what does each failing probe cause?
3. Why is a preStop delay needed for zero-downtime rollouts?
4. Why must migrations be backward compatible in a rolling deployment?
5. Why scale consumers on lag rather than CPU, and what caps the scale?

## 11. References (when stuck)

kubernetes.io docs (Services, probes, rolling updates, pod lifecycle) · Helm docs · Strimzi docs · CloudNativePG docs · KEDA Kafka scaler docs · Spring Boot docs *Deploying to Kubernetes*

## 12. Exit checklist

- [ ] B1–B6 pass (B7 if done)
- [ ] `task k8s:up` from a clean clone works
- [ ] Gate G5 · ADRs · tags pushed
