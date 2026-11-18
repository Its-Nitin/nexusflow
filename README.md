# NexusFlow: Intelligent Supply Chain Control Tower

A control tower that detects supply-chain disruptions (for example, a delayed shipment), assesses their risk to customer commitments, and orchestrates approved recovery actions across services.

## Why this repository exists

NexusFlow is a **learning project**: its purpose is to learn microservices and distributed-systems architecture by building a complete system. That means the transactional outbox, idempotent consumers, sagas, CQRS, observability and failure drills. Each milestone builds a feature, breaks it on purpose, and fixes it with the right pattern.

## Status

**M00: engineering standards** (in progress)

## Documentation

- [Project plan](docs/project-plan.md): roadmap, releases and gates, service map
- [Milestones](docs/milestones/): one detailed document per milestone (M00–M21)
- [Learning journal](docs/learning/journal.md)

## Tech stack

Java 25 · Spring Boot 4 · Maven · Apache Kafka · PostgreSQL · Redis · Keycloak · Angular · Docker Compose → Kubernetes
