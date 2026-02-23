# Helix

**Helix** is a concept-first, production-inspired microservices platform built to demonstrate modern distributed system architecture patterns.
The goal is learning and architectural clarity, not business functionality.

Helix focuses on showcasing real-world backend concepts such as:

* API Gateway pattern
* Event-driven architecture
* CQRS
* Choreography-based SAGA
* Outbox pattern
* Async-first communication
* Gateway-centric security
* Observability and trace propagation

---

## Architecture Overview

Helix follows a modern cloud-native architecture:

```
Client
   ↓
helix-gateway (public)
   ↓
Internal Network (private)
   ↓
helix-auth
helix-user
helix-events (Kafka)
helix-config
```

### Core Principles

* Only the gateway is publicly exposed.
* Internal services are private and trusted via network boundaries.
* Business logic lives inside services, never in the gateway.
* Services own their data and domain logic.
* Async communication is preferred over sync.

---

## Core Services

| Service         | Responsibility                                    |
| --------------- | ------------------------------------------------- |
| `helix-gateway` | Routing, auth enforcement, trace propagation      |
| `helix-auth`    | Authentication, JWT issuance, roles & permissions |
| `helix-user`    | User lifecycle and profile management             |
| `helix-events`  | Kafka event platform                              |
| `helix-config`  | Centralized configuration                         |
| `helix-commons` | Shared infrastructure utilities                   |

---

## Security Model

### Gateway-Only Authorization

* JWT validation occurs only at the gateway.
* Internal services do not perform authorization checks.
* Gateway injects identity headers:

```
X-User-Id
X-User-Roles
X-Trace-Id
X-Request-Id
```

### JWT Claims

Minimal JWT payload:

```
sub
roles
iat
exp
jti
iss
```

Permissions are resolved at gateway level, not embedded in the token.

### Token Lifecycle

* Short-lived access token (~10 minutes).
* Refresh token (~14 days).
* Credentials used only when refresh expires.

---

## Communication Model

Priority order:

```
Async (Kafka) > gRPC > REST
```

### Async (Kafka)

Used for:

* Domain events
* Batch and long-running processes
* Cross-service state propagation
* Saga workflows

### Sync Calls

Used for:

* Immediate validation
* Small read operations
* Real-time responses

Rules:

* Avoid chained sync calls.
* No circular dependencies between services.

---

## Event Architecture

### Naming Convention

```
<Domain><Action>Event
```

Examples:

```
UserCreatedEvent
RoleAssignedEvent
```

### Kafka Topics

One topic per event type:

```
user-created-events
role-assigned-events
```

### Producers & Consumers

```
UserCreatedEventProducer
UserCreatedEventConsumer
```

### Event Ownership

* Producer service owns the schema.
* Consumers adapt to producer contracts.

---

## Outbox Pattern

Each service maintains its own outbox table:

```
<service>_outbox
```

Example:

```
user_service_outbox
```

### Transaction Flow

1. Business data saved.
2. Outbox event written in same transaction.
3. Background publisher sends to Kafka.
4. Outbox status updated.

This guarantees consistency between database state and published events.

---

## CQRS

Each service separates read and write paths:

```
WriteRepository
ReadRepository
```

Initially both use the same PostgreSQL instance.
Future scaling allows read replicas without service refactoring.

---

## Internal Service Structure

Helix uses feature-first organization with a familiar layered pattern.

```
feature/
  controller/
  service/
  repository/
    read/
    write/
  event/
    producer/
    consumer/
  outbox/
  dto/
  model/
```

Pattern:

```
Controller → Service → Repository
```

---

## Database Rules

* PostgreSQL is the primary datastore.
* Each service owns its database/schema.
* No cross-service database access.
* No shared domain tables.

---

## Monorepo Strategy

Helix is structured as a monorepo for learning and architectural clarity.

```
helix/
  services/
  platform/
  libs/
```

CI/CD should use path-based triggers so only changed services are built.

---

## helix-commons Guidelines

Contains only reusable infrastructure components.

Allowed:

* Base event abstractions
* Kafka interfaces
* Tracing utilities
* Common error models

Not allowed:

* Domain models
* DTOs
* Business entities
* Service-specific events

---

## Observability

* Trace ID propagated across requests and events.
* Structured logging.
* Prometheus metrics.
* Grafana dashboards.

---

## Deployment

Current target:

* Docker Compose (local development)

Architecture target:

* Kubernetes-ready
* Cloud agnostic

---

## Design Documentation

All architectural decisions are recorded using ADRs:

```
docs/adr/
```

Example:

```
ADR-001 Gateway-only authorization
ADR-002 Event-per-topic strategy
ADR-003 Outbox pattern
```

---

## Learning Goals

Helix is intentionally explicit in implementation:

* Patterns are visible in code.
* Minimal hidden abstractions.
* Concept clarity over framework magic.

---

## Future Evolution

Planned improvements include:

* Internal service identity (mTLS or service tokens)
* Schema registry for events
* Advanced observability
* Domain-specific services
* Kubernetes production deployment

---

## Status

Architecture finalized.
Implementation phase begins next.
