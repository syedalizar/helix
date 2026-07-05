# Helix Implementation Plan

## Purpose

Helix is an architecture laboratory for learning and demonstrating production-grade distributed-system concepts. It uses a deliberately small identity and account-lifecycle domain so architecture, failure behavior, operational concerns, and trade-offs remain the focus.

Reference workflow:

```text
Register identity -> create profile -> assign role -> authenticate
-> update account -> suspend/reactivate -> delete account
```

Identity and credentials belong to the authentication domain. Profiles and account lifecycle belong to the user domain. A stable identity ID connects them; their databases are never shared.

## Delivery principles

- Start with the simplest architecture that demonstrates the current problem.
- Record consequential decisions before or alongside implementation.
- Treat testing, security, observability, and operations as delivery work.
- Demonstrate failure and recovery, not only successful requests.
- Choose synchronous or asynchronous communication by semantics, not ranking.
- Preserve comparisons so learners can understand why the architecture evolves.
- Complete implementation, tests, documentation, and operational evidence together.

## Technology baseline

- Java 21 and Spring Boot
- Gradle with Kotlin DSL
- PostgreSQL and Flyway
- Spring Security and Spring Cloud Gateway
- Apache Kafka in KRaft mode
- Docker Compose; Kubernetes-ready packaging later
- Testcontainers, JUnit 5, AssertJ, and architecture tests
- OpenTelemetry, Prometheus, and Grafana

Exact versions will be selected during Milestone 0 using current support and compatibility matrices.

## Documentation structure

```text
docs/
  product/        scope, outcomes, and domain vocabulary
  architecture/   C4 views, data flows, and state models
  adr/            architecture decision records
  api/            HTTP contracts and conventions
  events/         event schemas, catalog, and compatibility
  security/       threat models and security controls
  operations/     runbooks, SLOs, alerts, and recovery
  testing/        test strategy and evidence
  experiments/    reproducible architectural demonstrations
  delivery/       risks, milestone reviews, and releases
```

ADRs use `Proposed`, `Accepted`, `Superseded`, or `Deprecated`. Accepted ADRs are not rewritten; a changed decision supersedes the original.

---

## Milestone 0 — Inception and architecture baseline

### Goal

Define what Helix teaches, establish domain and repository boundaries, validate the technical baseline, and create the decision process before application code.

### Work

- Establish monorepo layout and module conventions.
- Select compatible framework and infrastructure versions.
- Define formatting, analysis, testing, dependency, and commit conventions.
- Create templates for ADRs, runbooks, experiments, and milestone reviews.
- Define local-development prerequisites and supported environments.

### Decisions

- Architecture laboratory scope and explicit non-goals.
- Authentication and user bounded contexts.
- Monorepo ownership and allowed module dependencies.
- Version selection and upgrade policy.
- Strict scope of `helix-commons`.

### Required documents

- `docs/product/project-charter.md` — mission, scope, audience, non-goals, success criteria.
- `docs/product/learning-outcomes.md` — concepts and objective evidence for each.
- `docs/product/domain-glossary.md` — identity, credential, user, profile, account, and role.
- `docs/architecture/system-context.md` — actors, system boundary, and C4 context view.
- `docs/architecture/architecture-principles.md` — decision guardrails.
- `docs/architecture/repository-structure.md` — modules, ownership, dependency rules.
- `docs/adr/ADR-001-monorepo.md`.
- `docs/adr/ADR-002-technology-baseline.md`.
- `docs/adr/ADR-003-domain-boundaries.md`.
- `docs/adr/ADR-004-shared-library-policy.md`.
- `docs/delivery/risk-register.md`.
- `docs/delivery/definition-of-done.md`.

### Exit criteria

- Baseline decisions are accepted.
- Every learning outcome maps to a planned implementation or experiment.
- The compatibility matrix is verified.
- No unresolved decision blocks the first application.

---

## Milestone 1 — Modular-monolith baseline

### Goal

Implement the workflow as a modular monolith, establishing correct domain boundaries, transactions, APIs, security, and tests before introducing distributed-system costs.

### Work

- Build one Spring Boot application with isolated auth and user modules.
- Enforce boundaries with build rules and architecture tests.
- Implement registration, login, profile operations, role assignment, account status, and deletion.
- Use module-owned schemas in one PostgreSQL instance.
- Add Flyway, validation, consistent errors, health endpoints, and structured logs.
- Add unit, repository, API integration, and end-to-end tests.
- Run the baseline with Docker Compose.

### Decisions

- Aggregate and transaction boundaries.
- API resources, errors, and versioning.
- Password hashing and initial token/session model.
- Module interaction rules.
- Account deletion and personal-data semantics.

### Required documents

- `docs/architecture/modular-monolith.md`.
- `docs/architecture/account-lifecycle.md`.
- `docs/architecture/data-model-baseline.md`.
- `docs/api/api-guidelines.md`.
- `docs/api/openapi.yaml`.
- `docs/security/baseline-threat-model.md`.
- `docs/security/credential-policy.md`.
- `docs/testing/test-strategy.md`.
- `docs/operations/local-development.md`.
- `docs/adr/ADR-005-modular-monolith-first.md`.
- `docs/adr/ADR-006-api-and-error-contract.md`.
- `docs/adr/ADR-007-database-migrations.md`.
- `docs/delivery/milestone-1-review.md`.

### Verification and exit criteria

- Complete lifecycle works through the API.
- Tests prevent cross-module persistence access.
- Migrations work for new and upgraded databases.
- Security tests cover invalid credentials, tokens, leakage, and privilege violations.
- Baseline latency, failure behavior, and complexity are recorded for comparison.

---

## Milestone 2 — Service extraction and data ownership

### Goal

Extract `helix-auth-service` and `helix-user-service`, making network and database ownership explicit while initially preserving synchronous behavior.

### Work

- Create independently buildable and deployable services.
- Give each service its own database and credentials.
- Replace in-process calls with documented HTTP APIs where required.
- Add timeouts, safe bounded retries, request IDs, and error translation.
- Add consumer-driven contract and service integration tests.
- Enforce the absence of cross-service database access.

### Decisions

- Owner of the public registration request.
- Operations requiring immediate consistency.
- Timeout, retry, and idempotency policy.
- API ownership and compatibility policy.
- Local service discovery and addressing.

### Required documents

- `docs/architecture/service-decomposition.md`.
- `docs/architecture/container-diagram.md`.
- `docs/architecture/synchronous-registration-sequence.md`.
- `docs/architecture/data-ownership.md`.
- `docs/api/auth-service-openapi.yaml`.
- `docs/api/user-service-openapi.yaml`.
- `docs/api/compatibility-policy.md`.
- `docs/operations/service-failure-runbook.md`.
- `docs/adr/ADR-008-service-extraction.md`.
- `docs/adr/ADR-009-database-per-service.md`.
- `docs/adr/ADR-010-synchronous-communication.md`.
- `docs/experiments/monolith-vs-services.md`.
- `docs/delivery/milestone-2-review.md`.

### Verification and exit criteria

- Each service builds, migrates, starts, and tests independently.
- Database permissions prevent cross-service access.
- Contract tests detect breaking changes.
- Slow, unavailable, and malformed downstream responses are reproducible.
- Added complexity and changed failure behavior are documented.

---

## Milestone 3 — Gateway and security boundary

### Goal

Introduce `helix-gateway-service` as the sole public entry point and demonstrate routing, authentication, authorization, and boundary protection.

### Work

- Build the gateway with Spring Cloud Gateway.
- Issue signed access and rotating refresh tokens from auth service.
- Validate access tokens and route policies at the gateway.
- Remove client identity headers and inject trusted user, role, trace, and request headers.
- Add rate and payload limits, CORS, and normalized external errors.
- Place services on a non-public network.
- Add refresh-token reuse detection and signing-key rotation readiness.

### Decisions

- External and internal trust boundaries.
- Signing algorithm, key publication, rotation, and revocation.
- Gateway authorization versus service-owned invariants.
- Identity propagation and limitations of unsigned internal headers.
- Refresh-token storage and session semantics.

### Required documents

- `docs/architecture/gateway-request-flow.md`.
- `docs/security/gateway-threat-model.md`.
- `docs/security/trust-boundaries.md`.
- `docs/security/authentication-flows.md`.
- `docs/security/authorization-matrix.md`.
- `docs/security/token-specification.md`.
- `docs/operations/key-rotation-runbook.md`.
- `docs/operations/security-incident-runbook.md`.
- `docs/adr/ADR-011-spring-cloud-gateway.md`.
- `docs/adr/ADR-012-token-and-session-model.md`.
- `docs/adr/ADR-013-authorization-boundaries.md`.
- `docs/experiments/gateway-bypass.md`.
- `docs/delivery/milestone-3-review.md`.

### Verification and exit criteria

- Services are unreachable through the public network.
- Spoofed identity headers are discarded.
- Role/resource combinations have positive and negative tests.
- Refresh rotation, reuse detection, revocation, and key rotation are tested.
- Gateway-only authorization risk is explicitly accepted or mitigated.

---

## Milestone 4 — Kafka and transactional outbox

### Goal

Introduce event-driven state propagation and prove reliable publication without an unsafe database/Kafka dual write.

### Work

- Add Kafka in KRaft mode.
- Define a versioned event envelope and domain-oriented topics.
- Implement service-owned outbox tables and publishers.
- Add idempotent consumers and processed-event tracking where appropriate.
- Configure partitions, keys, retention, retry, and dead-letter handling.
- Test PostgreSQL and Kafka behavior with Testcontainers.

### Decisions

- Domain-topic versus event-per-type strategy.
- Event ownership, naming, versioning, and compatibility.
- Delivery guarantees and idempotency mechanisms.
- Ordering guarantees and partition keys.
- Retryable versus terminal failures.

### Required documents

- `docs/architecture/event-driven-architecture.md`.
- `docs/architecture/outbox-flow.md`.
- `docs/events/event-envelope.md`.
- `docs/events/event-catalog.md`.
- `docs/events/schema-compatibility.md`.
- `docs/events/topic-catalog.md`.
- `docs/operations/kafka-runbook.md`.
- `docs/operations/outbox-runbook.md`.
- `docs/adr/ADR-014-kafka-kraft.md`.
- `docs/adr/ADR-015-topic-strategy.md`.
- `docs/adr/ADR-016-transactional-outbox.md`.
- `docs/adr/ADR-017-delivery-and-idempotency.md`.
- `docs/experiments/dual-write-failure.md`.
- `docs/experiments/duplicate-and-out-of-order-events.md`.
- `docs/delivery/milestone-4-review.md`.

### Verification and exit criteria

- Committed changes publish after Kafka recovery; rolled-back changes never publish.
- Duplicate delivery does not duplicate domain effects.
- Compatibility checks reject breaking schemas.
- Outbox backlog and consumer lag are measurable.
- Guarantees are described as at-least-once without unsupported exactly-once claims.

---

## Milestone 5 — Asynchronous workflow and Saga comparison

### Goal

Evolve registration to eventual consistency and compare choreography with orchestration through explicit state, failure, timeout, and compensation behavior.

### Work

- Create profiles asynchronously from identity events.
- Persist observable registration status and transitions.
- Handle duplicate, delayed, missing, and reordered events.
- Implement choreography first.
- Implement a small orchestration reference for direct comparison.
- Provide status-query behavior for accepted asynchronous requests.

### Decisions

- Saga owner and persisted state model.
- Compensation semantics for credentials and profiles.
- Final choreography or orchestration selection.
- Timeout, retry, terminal-state, and manual-recovery rules.
- Client contract for incomplete operations.

### Required documents

- `docs/architecture/registration-saga.md`.
- `docs/architecture/choreography-sequence.md`.
- `docs/architecture/orchestration-sequence.md`.
- `docs/api/async-operation-contract.md`.
- `docs/operations/saga-recovery-runbook.md`.
- `docs/adr/ADR-018-eventual-consistency.md`.
- `docs/adr/ADR-019-saga-style.md`.
- `docs/adr/ADR-020-compensation-policy.md`.
- `docs/experiments/saga-failure-matrix.md`.
- `docs/experiments/sync-vs-async-registration.md`.
- `docs/delivery/milestone-5-review.md`.

### Verification and exit criteria

- Every transition and terminal state is tested.
- Failure at every step converges to a documented state.
- Replayed events do not corrupt workflow state.
- Operators can locate and recover incomplete Sagas.
- The selected style is justified by evidence.

---

## Milestone 6 — CQRS and projection-based reads

### Goal

Demonstrate meaningful CQRS with an account-summary projection derived from multiple domains, rather than merely naming separate repository interfaces.

### Work

- Create `helix-account-query-service` or an independently deployable query component.
- Consume identity, profile, role, and account-status events.
- Build an optimized read model and read-only API.
- Support projection replay, rebuilding, and version migration.
- Expose projection lag and result freshness.

### Decisions

- Whether the query model warrants a separate service.
- Projection ownership and acceptable staleness.
- Rebuild, replay, migration, and cutover procedure.
- Whether stale projections ever fall back to source services.

### Required documents

- `docs/architecture/cqrs-model.md`.
- `docs/architecture/account-summary-projection.md`.
- `docs/api/account-query-openapi.yaml`.
- `docs/operations/projection-rebuild-runbook.md`.
- `docs/adr/ADR-021-cqrs-boundary.md`.
- `docs/adr/ADR-022-projection-consistency.md`.
- `docs/experiments/projection-rebuild.md`.
- `docs/delivery/milestone-6-review.md`.

### Verification and exit criteria

- Projection state converges with source-of-truth state.
- Full replay equals incremental processing.
- Duplicate and reordered events cannot corrupt the projection.
- API consumers can understand data freshness.
- CQRS shows a measurable benefit and explicit operational cost.

---

## Milestone 7 — Observability and operational readiness

### Goal

Make HTTP requests, database work, outbox publication, Kafka consumption, Sagas, and projections understandable in operation.

### Work

- Propagate trace/request context through HTTP and events.
- Add OpenTelemetry and a collector.
- Expose Prometheus metrics and Grafana dashboards.
- Standardize structured logs and sensitive-data redaction.
- Instrument latency, errors, saturation, outbox age, consumer lag, Saga duration, and projection freshness.
- Define actionable alerts and dependency-aware readiness.
- Add security audit events.

### Decisions

- Telemetry naming and cardinality limits.
- Learning-environment SLIs and SLOs.
- Trace sampling and retention assumptions.
- Audit versus operational logging boundaries.
- Alert ownership and response expectations.

### Required documents

- `docs/architecture/observability-architecture.md`.
- `docs/operations/sli-slo.md`.
- `docs/operations/alert-catalog.md`.
- `docs/operations/dashboard-guide.md`.
- `docs/operations/incident-response.md`.
- `docs/security/logging-and-redaction.md`.
- `docs/adr/ADR-023-observability-stack.md`.
- `docs/adr/ADR-024-trace-propagation.md`.
- `docs/experiments/end-to-end-trace.md`.
- `docs/delivery/milestone-7-review.md`.

### Verification and exit criteria

- One registration is traceable from gateway to final projection.
- Injected latency, Kafka lag, and outbox backlog appear in dashboards.
- Alerts link to tested runbooks.
- Automated checks confirm tokens, credentials, and personal data are not logged.
- Common failures can be diagnosed without source-code inspection.

---

## Milestone 8 — Resilience and security hardening

### Goal

Validate realistic failure behavior and evolve internal trust beyond network location and unsigned identity headers.

### Work

- Add dependency-specific timeouts, retries, circuit breakers, and bulkheads.
- Implement graceful degradation where meaningful.
- Add service tokens or mTLS for internal authentication.
- Add dependency, container, static, and secret scanning.
- Add graceful shutdown, resource limits, and disruption tests.
- Automate database, Kafka, consumer, and network fault injection.
- Exercise backup, restore, and reconciliation.

### Decisions

- Resilience policy per dependency.
- Service-token versus mTLS identity.
- Retry budgets and ownership across layers.
- Demonstration RPO and RTO.
- Vulnerability severity and remediation policy.

### Required documents

- `docs/architecture/resilience-model.md`.
- `docs/security/internal-service-identity.md`.
- `docs/security/security-controls.md`.
- `docs/security/vulnerability-management.md`.
- `docs/operations/backup-restore-runbook.md`.
- `docs/operations/disaster-recovery.md`.
- `docs/adr/ADR-025-resilience-policies.md`.
- `docs/adr/ADR-026-internal-service-identity.md`.
- `docs/experiments/failure-injection-catalog.md`.
- `docs/experiments/recovery-exercise.md`.
- `docs/delivery/milestone-8-review.md`.

### Verification and exit criteria

- Retries cannot cause uncontrolled amplification or duplicate effects.
- Unauthorized clients/services cannot forge internal identity.
- Fault experiments reach documented degraded or recovered states.
- Backup restoration reconstructs a validated system.
- Critical security findings block completion.

---

## Milestone 9 — CI/CD and deployment evolution

### Goal

Demonstrate repeatable delivery, artifact integrity, safe data evolution, and the transition from Compose to Kubernetes-ready deployment.

### Work

- Add path-aware CI for affected Gradle modules.
- Run formatting, analysis, unit, integration, architecture, contract, and security checks.
- Build immutable images and software bills of materials.
- Validate database migrations and compatibility before release.
- Add Kubernetes manifests through Helm or Kustomize.
- Demonstrate rolling deployment, rollback, and mixed-version compatibility.
- Add release versioning and release notes.

### Decisions

- Artifact and service versioning.
- Branch, review, and release strategy.
- Kubernetes packaging approach.
- Deployment order and compatibility window.
- Rollback versus roll-forward for databases and events.

### Required documents

- `docs/architecture/deployment-architecture.md`.
- `docs/delivery/ci-cd-design.md`.
- `docs/delivery/release-strategy.md`.
- `docs/delivery/change-management.md`.
- `docs/operations/deployment-runbook.md`.
- `docs/operations/database-release-runbook.md`.
- `docs/adr/ADR-027-ci-path-filtering.md`.
- `docs/adr/ADR-028-deployment-packaging.md`.
- `docs/adr/ADR-029-release-compatibility.md`.
- `docs/experiments/rolling-upgrade.md`.
- `docs/delivery/milestone-9-review.md`.

### Verification and exit criteria

- A clean checkout builds, tests, packages, and starts reproducibly.
- Affected-module detection is correct.
- Mixed versions retain API/event compatibility.
- A failed release can safely roll back or roll forward.
- Artifacts are immutable, traceable, and scanned.

---

## Milestone 10 — Learning showcase and final assessment

### Goal

Turn the implementation into a coherent, reproducible demonstration of architectural reasoning rather than a collection of technologies.

### Work

- Script walkthroughs for core workflows and failures.
- Produce final architecture diagrams and evidence links.
- Measure costs introduced at each evolutionary stage.
- Review ADRs, accepted risks, and known limitations.
- Create quick-start and guided-learning paths.
- Automatically verify documentation links and commands where practical.

### Decisions

- Select the final reference architecture while preserving comparisons.
- Identify patterns that are demonstrations rather than recommendations.
- Separate intentional limitations from future work.

### Required documents

- `docs/learning-path/README.md`.
- `docs/learning-path/concept-map.md`.
- `docs/architecture/final-architecture.md`.
- `docs/architecture/evolution-narrative.md`.
- `docs/architecture/trade-off-summary.md`.
- `docs/experiments/demo-catalog.md`.
- `docs/delivery/known-limitations.md`.
- `docs/delivery/final-assessment.md`.
- `docs/operations/quick-start.md`.

### Verification and exit criteria

- A new learner can complete the guided path from a clean checkout.
- Every architectural claim links to code, tests, telemetry, or an experiment.
- Documentation distinguishes contextual decisions from universal advice.
- Every Milestone 0 learning outcome has objective evidence.

## Cross-milestone quality gates

Every milestone must satisfy:

### Architecture

- New boundaries and dependencies are documented.
- Services never access another service's database.
- Shared libraries contain no service-specific domain types.
- Accepted ADRs match implementation.

### Security

- Threat models change with trust boundaries.
- Secrets and credentials are neither committed nor logged.
- Authorization includes negative tests.
- New reachable surfaces receive explicit review.

### Testing

- Happy, validation, authorization, and relevant failure paths are automated.
- Real infrastructure is used through Testcontainers where practical.
- API and event compatibility are checked.
- Flaky tests are treated as defects.

### Operations

- Health and readiness reflect actual dependency semantics.
- New failure modes have telemetry and runbooks.
- Migrations and recovery procedures are reproducible.
- Local startup and cleanup remain current.

### Documentation

- Diagrams, contracts, ADRs, and runbooks change with behavior.
- Examples and commands are verified.
- The milestone review records deviations and accepted debt.

## Decision workflow

For a consequential architectural choice:

1. Describe the problem, constraints, and decision owner.
2. Record viable options and their operational, security, consistency, coupling, complexity, and learning trade-offs.
3. Validate uncertain assumptions with a time-boxed experiment.
4. Create a `Proposed` ADR for difficult-to-reverse decisions.
5. Accept it before multiple components depend on it.
6. Supersede it if later evidence changes the decision.

Small, reversible implementation details do not require ADRs.

## Milestone review format

Each milestone review records:

- Planned versus delivered scope
- Demonstration commands and evidence
- Test and quality results
- Security and operational findings
- Performance or reliability observations
- ADRs accepted or superseded
- Known limitations and accepted debt
- Risks added, retired, or changed
- Go/no-go decision for the next milestone

## Execution order

```text
Inception
  -> modular monolith
  -> service extraction
  -> gateway/security
  -> Kafka/outbox
  -> Saga comparison
  -> CQRS projection
  -> observability
  -> resilience/security
  -> delivery/deployment
  -> learning showcase
```

Implementation begins with Milestone 0. Later scaffolding should not be generated early unless the current milestone requires it; doing so would imply decisions and capabilities that have not yet been validated.
