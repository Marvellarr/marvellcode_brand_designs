# MarvelCode System Architecture

**Status:** Pre-development architecture  
**Purpose:** Structure the system before implementation begins  
**Scope:** Product domains, application boundaries, data flow, permissions, APIs, and delivery sequencing

## 1. Architectural Intent

This document defines the structure of the MarvelCode system before coding begins. It is intentionally independent of visual references, mockups, colors, typography, and frontend presentation. The implementation team should use this document to decide what to build, where each responsibility belongs, how modules communicate, and how the product can evolve without a large rewrite.

MarvelCode should be designed as a multi-surface platform with a public web presence, an identity system, and an authenticated workspace. These surfaces may share one repository and deployment pipeline, but they must remain separated by responsibility and authorization boundary.

The core architectural principles are:

1. **Separate public content from authenticated operations.** Marketing pages must not depend on private workspace data.
2. **Organize code by business capability, not by screen.** A deployment feature should own its API, validation, types, and UI rather than scattering them across generic folders.
3. **Keep domain logic independent from infrastructure.** Database, authentication provider, queue provider, and external services must be replaceable behind interfaces.
4. **Enforce authorization on the server.** Client-side visibility controls are only a presentation concern.
5. **Start modular, not distributed.** Use a modular monolith until scale or ownership boundaries justify separate services.
6. **Design asynchronous operations explicitly.** Deployments, analytics processing, notifications, and integrations should have durable job states rather than blocking requests.
7. **Make observability part of the system.** Every important action should be traceable through structured logs, metrics, and audit events.

## 2. System Context

```text
                              +----------------------+
                              |  Public Web Client   |
                              |  content, docs, SEO  |
                              +----------+-----------+
                                         |
                                         v
+-------------+       +------------------+------------------+       +----------------+
| Admin /     |------>|       Web Application              |------>| Identity       |
| Operator    |       | public routes + authenticated UI  |       | Provider       |
+-------------+       +------------------+------------------+       +----------------+
                                         |
                                         v
                              +----------+-----------+
                              |   Application API       |
                              | auth, domains, policy   |
                              +--+-----+-----+-----+----+
                                 |     |     |     |
                                 v     v     v     v
                              +----+ +----+ +----+ +------+
                              | DB | |Jobs| |Blob| |Audit |
                              +----+ +----+ +----+ +------+
                                         |
                                         v
                              +----------+-----------+
                              | External Integrations |
                              | cloud, deploy, model, |
                              | notification providers |
                              +------------------------+
```

### Primary actors

| Actor | Responsibilities |
|---|---|
| Visitor | Reads public content and submits low-risk inquiries. |
| User | Authenticates and works within one or more workspaces. |
| Workspace administrator | Manages members, permissions, environments, and integrations. |
| Operator | Monitors system health, jobs, deployments, and incidents. |
| Platform administrator | Manages global configuration, policies, and support operations. |
| Automated worker | Executes asynchronous jobs and reports state transitions. |
| External provider | Supplies identity, infrastructure, model, notification, or deployment capabilities. |

## 3. Product Surfaces

### 3.1 Public web surface

The public surface contains company information, product capabilities, documentation, security information, service status, and contact flows. It may use static generation or server rendering. It must not query private workspace tables directly.

### 3.2 Identity surface

The identity surface handles registration, sign-in, email verification, password recovery, session management, and account-level security settings. Identity credentials should be delegated to a proven identity provider or isolated authentication module rather than implemented ad hoc inside feature code.

### 3.3 Workspace surface

The workspace surface is the authenticated product. It provides operational views and actions for deployments, services, models, analytics, environments, integrations, and settings. Every request is scoped to a workspace and checked against the authenticated user's membership and permission set.

### 3.4 Administration surface

Administration should be a separate capability from ordinary workspace settings. It may share the application shell, but it requires elevated permissions and must be isolated through explicit routes, policies, audit events, and feature flags.

## 4. Recommended Application Boundary

Begin with a **modular monolith**:

```text
Browser / API clients
        |
        v
Web application and API
        |
        +-- Identity module
        +-- User and workspace module
        +-- Service catalog module
        +-- Environment module
        +-- Deployment module
        +-- Model registry module
        +-- Analytics module
        +-- Notification module
        +-- Integration module
        +-- Documentation/content module
        +-- Audit and observability module
        |
        +-- Relational database
        +-- Object storage
        +-- Durable job queue
```

A modular monolith provides one deployable unit and one transaction boundary while preserving internal ownership boundaries. Extract a service only when there is a clear reason, such as independent scaling, separate deployment ownership, strict network isolation, or a provider-specific workload.

Do not start with microservices, a service mesh, or event-driven infrastructure for every operation. Those choices add operational complexity before the product's domain boundaries are proven.

## 5. Domain Modules

### 5.1 Identity and access

Owns users, sessions, authentication events, recovery flows, MFA configuration, and identity-provider integration. It exposes identity facts to the rest of the system but does not own workspace business rules.

### 5.2 Workspace and membership

Owns workspaces, memberships, roles, invitations, workspace preferences, environments, and membership lifecycle. This module is the root scope for most product data.

### 5.3 Authorization policy

Owns permission evaluation and policy definitions. A policy decision should receive the actor, workspace, resource, action, and context, then return allow or deny with a reason suitable for logging.

Example actions:

```text
workspace.read
workspace.manage
member.invite
member.remove
service.read
service.manage
deployment.read
deployment.create
deployment.cancel
model.read
model.manage
integration.read
integration.manage
audit.read
```

### 5.4 Service catalog

Owns registered services, repositories, runtime metadata, deployment targets, health status, and service ownership. It provides the stable identity used by deployments, metrics, incidents, and documentation.

### 5.5 Environment management

Owns environments such as development, staging, and production. It defines environment-level policy, available integrations, deployment restrictions, approval requirements, and configuration references.

### 5.6 Deployment orchestration

Owns deployment requests, deployment plans, execution state, approvals, cancellation, rollback references, and provider adapters. The API should create a deployment job quickly and return a deployment identifier. A worker performs the long-running execution.

Deployment state should be explicit:

```text
requested -> queued -> running -> succeeded
                         |-> failed
                         |-> cancelled
                         |-> requires_approval
```

State transitions must be validated by the domain module rather than updated freely by controllers or UI clients.

### 5.7 Model registry

Owns model definitions, versions, providers, capabilities, lifecycle state, and workspace availability. It should not assume that every model is hosted by the same provider.

### 5.8 Analytics and metrics

Owns metric definitions, time-series ingestion, aggregation jobs, report queries, and retention policy. Operational dashboards should read from query-optimized projections rather than repeatedly scanning transactional tables.

### 5.9 Notifications

Owns notification preferences, delivery attempts, templates, channels, and retry state. Email, in-app, and webhook delivery should be adapters behind one notification interface.

### 5.10 Integrations and secrets

Owns provider connections, capability discovery, credential references, and integration health. Store secret material in a dedicated secret manager or encrypted secret store. The application database should retain only a provider reference and non-sensitive metadata where possible.

### 5.11 Content and documentation

Owns public pages, documentation articles, legal documents, release notes, and publication state. Public content should be deployable without requiring access to workspace data.

### 5.12 Audit and observability

Owns immutable audit events for security-sensitive and operational actions. Application logs and audit records are different: logs support debugging, while audit records explain who performed which action, on what resource, and when.

## 6. Repository Structure Before Coding

Use capability-based ownership from the beginning:

```text
src/
  app/                         # route composition and request adapters
  modules/
    identity/
      domain/
      application/
      infrastructure/
      http/
      ui/
    workspaces/
    authorization/
    services/
    environments/
    deployments/
    models/
    analytics/
    notifications/
    integrations/
    content/
    audit/
  platform/
    database/
    queue/
    storage/
    observability/
    configuration/
  shared/
    errors/
    validation/
    types/
    dates/
    ids/
  jobs/
    deployment-worker/
    analytics-worker/
    notification-worker/
  contracts/
    api/
    events/

  tests/
    unit/
    integration/
    contract/
    end-to-end/

content/
  public/
  documentation/
  legal/

infra/
  environments/
  migrations/
  deployment/
```

The `domain` layer contains business rules. The `application` layer coordinates use cases. The `infrastructure` layer talks to databases and providers. The `http` layer translates requests and responses. The `ui` layer presents the capability. This separation prevents controllers and components from accumulating business logic.

## 7. Core Data Model

The initial relational model should include the following entities:

| Entity | Main responsibility |
|---|---|
| `users` | Account identity and profile metadata. |
| `sessions` | Active authenticated sessions or provider references. |
| `workspaces` | Tenant boundary for product data. |
| `memberships` | User-to-workspace relationship and role. |
| `roles` / `permissions` | Authorization vocabulary and assignments. |
| `invitations` | Pending workspace membership invitations. |
| `environments` | Workspace deployment contexts. |
| `services` | Registered deployable services. |
| `deployments` | Requested and executed deployment operations. |
| `deployment_events` | Append-only execution history. |
| `models` / `model_versions` | Model registry and lifecycle metadata. |
| `integrations` | External provider connections and capabilities. |
| `metric_definitions` | Meaning and unit of measurable signals. |
| `metric_points` or warehouse projection | Queryable operational measurements. |
| `notifications` | User-facing notification records. |
| `audit_events` | Immutable security and operations history. |

Every workspace-owned table should include `workspace_id`, unless it is purely global. Add indexes for workspace scope, status, timestamps, and the most common list filters. Use soft deletion only when retention or audit requirements demand it; otherwise prefer explicit lifecycle states.

## 8. Request and Data Flow

### Synchronous read

```text
Client -> route/controller -> authentication -> authorization
       -> application use case -> repository/query service -> database
       <- response DTO <- controller <- client
```

### Synchronous command creating a job

```text
Client -> authenticated command endpoint
       -> validate input and policy
       -> create domain record in pending state
       -> enqueue durable job
       <- command id and current state
```

### Asynchronous worker

```text
Worker -> claim job -> load domain record
       -> call provider adapter
       -> record progress and provider references
       -> transition domain state
       -> write audit event
       -> publish notification/event
```

### Event usage

Use domain events for decoupling within the modular monolith, not as a substitute for every function call. Events are appropriate for `DeploymentSucceeded`, `DeploymentFailed`, `MemberInvited`, `IntegrationChanged`, and `IncidentOpened`. Event handlers must be idempotent and retryable.

## 9. API Design

Use a versioned API boundary such as `/api/v1`. Keep transport models separate from database models. Each endpoint should define authentication requirements, authorization action, request schema, response schema, error codes, pagination rules, and idempotency behavior.

Suggested resource groups:

```text
POST   /api/v1/auth/session
GET    /api/v1/me
GET    /api/v1/workspaces
POST   /api/v1/workspaces
GET    /api/v1/workspaces/:workspaceId/members
POST   /api/v1/workspaces/:workspaceId/invitations
GET    /api/v1/workspaces/:workspaceId/services
POST   /api/v1/workspaces/:workspaceId/services
GET    /api/v1/workspaces/:workspaceId/deployments
POST   /api/v1/workspaces/:workspaceId/deployments
POST   /api/v1/workspaces/:workspaceId/deployments/:id/cancel
GET    /api/v1/workspaces/:workspaceId/models
GET    /api/v1/workspaces/:workspaceId/metrics
GET    /api/v1/workspaces/:workspaceId/audit-events
```

Use cursor pagination for event-like resources, stable sorting for all lists, structured error responses, request correlation IDs, and idempotency keys for commands that can be retried.

## 10. Authorization Model

Use workspace-scoped role-based access control as the baseline, with resource checks where needed.

| Role | Default scope |
|---|---|
| Viewer | Read workspace resources and dashboards. |
| Developer | Read resources and create or manage development deployments. |
| Operator | Manage operational actions and monitor production resources. |
| Administrator | Manage members, integrations, environments, and policies. |
| Owner | Full workspace control, including ownership and billing-related settings if added later. |

Authorization must be applied in this order:

1. Authenticate the request.
2. Resolve the target workspace.
3. Verify membership and account status.
4. Evaluate the required action against the resource and environment.
5. Execute the use case.
6. Write an audit event for sensitive actions.

Never treat a hidden button, route guard, or client-side role check as authorization.

## 11. Reliability and Operational Requirements

The system should define service-level expectations before implementation:

| Area | Initial target |
|---|---|
| Public page availability | 99.9% monthly target after production launch |
| API read latency | p95 under 500 ms for ordinary workspace reads |
| Command response | Return accepted job state under 1 second when dependencies are available |
| Job execution | Durable retries with visible failure state |
| Audit durability | No silent loss for security-sensitive actions |
| Recovery | Database backup and tested restoration procedure |
| Observability | Correlation ID across request, job, provider call, and audit event |

Add timeouts, retry policies, circuit breakers, and provider-specific error mapping at integration boundaries. Do not retry non-idempotent commands without an idempotency key.

## 12. Security Baseline

Before production, implement:

- Secure session management with HTTP-only, secure, same-site cookies or an established identity provider.
- MFA support for privileged accounts.
- Server-side workspace and resource authorization.
- Rate limits for authentication, invitations, recovery, and public forms.
- CSRF protection for cookie-authenticated mutations.
- Strict input validation and output encoding.
- Encrypted secrets and no credentials in source control.
- Content Security Policy and strict transport security.
- Redacted logs and sensitive-field filtering.
- Immutable audit records for access, deployment, integration, role, and credential actions.
- Dependency scanning, secret scanning, and protected production environments.

## 13. Development Sequence

### Stage 1 — Architecture foundation

Decide the framework, identity provider, database, queue, object storage, deployment target, and observability provider. Write ADRs for each decision. Create the repository structure, configuration validation, error model, ID strategy, logging conventions, and migration process.

### Stage 2 — Identity and workspace foundation

Implement users, sessions, workspaces, memberships, invitations, roles, permission checks, and audit events. Build contract tests for authentication and authorization before adding operational features.

### Stage 3 — Service and environment catalog

Implement services, environments, integrations, and provider capability discovery. At this stage the system should be able to represent what exists, even if it cannot deploy anything yet.

### Stage 4 — Deployment workflow

Implement deployment creation, validation, approval policy, queueing, worker execution, provider adapters, status transitions, cancellation, failure handling, and audit history. Use a fake provider in automated tests.

### Stage 5 — Models, analytics, and notifications

Add model registry, metrics ingestion/projections, dashboard queries, notification preferences, and delivery adapters. Keep analytics read models separate from command-side transactional tables.

### Stage 6 — Public content and documentation

Add public content, documentation, legal pages, status information, and contact workflows as a separate surface. These pages should be deployable and testable without authenticated workspace data.

### Stage 7 — Hardening

Add end-to-end tests, contract tests against providers, backup restoration tests, load testing, accessibility checks, security review, incident procedures, and production deployment gates.

## 14. Testing Strategy

Use four test levels:

1. **Unit tests:** domain rules, state transitions, permission decisions, validators, and retry logic.
2. **Integration tests:** repositories, migrations, queue behavior, identity callbacks, and provider adapters using test services or fakes.
3. **Contract tests:** API request/response schemas and event payloads between modules and external providers.
4. **End-to-end tests:** sign-in, workspace creation, invitation, deployment request, approval, failure, and recovery journeys.

Every domain module should have tests for its happy path, authorization failures, invalid transitions, retries, duplicate requests, and dependency failures.

## 15. Decisions to Make Before Coding

The following decisions materially affect the implementation and should be recorded as short architecture decision records:

- Web framework and rendering strategy.
- Identity provider and MFA approach.
- Relational database and migration tooling.
- Queue and worker runtime.
- Object storage and secret manager.
- Deployment provider and environment topology.
- Metrics storage strategy: relational projection, time-series store, or warehouse.
- Notification channels and provider choices.
- Workspace tenancy model and data isolation requirements.
- Retention, deletion, and backup policies.
- Production observability and incident response ownership.

If a decision is not required to begin the foundation, defer it rather than introducing a speculative dependency.

## 16. Definition of Architecture-Ready

The system is ready to enter implementation when the team can answer, in writing:

- What belongs to each domain module?
- Which requests are synchronous and which become jobs?
- How is every workspace-owned resource authorized?
- Which data is transactional, analytical, temporary, or immutable?
- How are external providers isolated behind adapters?
- How are failures retried, surfaced, and audited?
- How are deployments, environments, secrets, and backups separated?
- What is the first vertical slice that proves the architecture?

The recommended first vertical slice is: **authenticate a user, create or select a workspace, register a service, create a development environment, request a deployment through a fake provider, process it asynchronously, and display the resulting audit trail**. This slice validates the most important architectural boundaries before large amounts of frontend or provider-specific code are written.
