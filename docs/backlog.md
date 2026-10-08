# Payment Approvals Backlog

Oct 7, 2026 · @Thiago Veloso Pereira

## Overview

A multi-tenant system where a company's members request supplier payments, approvers sign off according to company policy, and the approved payment is executed through a saga. Phase 1 alone (milestones M1 to M3) is a complete, showable project.

**The rules the product enforces** (each one becomes a test):

1. A user only sees data from their own tenant. A request from another tenant returns 404, never 403, so its existence is not leaked.
2. Requesters create requests; approvers approve or reject; admins manage the approval policy.
3. The number of approvals required depends on payment method and amount, per the tenant's policy, snapshotted when the request is submitted.
4. Nobody approves their own request. Nobody approves the same request twice.
5. Every state change is recorded in an append-only audit trail with actor and time.

**States:** `PENDING_APPROVAL` → `APPROVED` or `REJECTED` (Phase 1). Phase 2 adds `PAID` and `PAYMENT_FAILED`.

### Stack and decisions

| Area | Choice |
| --- | --- |
| Language and framework | Java 25 (LTS), Spring Boot 4.1, Maven |
| Security | Spring Security OAuth2 resource server, JWT, method security |
| Identity provider | Keycloak in Docker, realm imported from a committed JSON |
| Database | PostgreSQL, Flyway, tenant\_id column per table |
| Tests | JUnit 5, Testcontainers (Postgres, Keycloak), WireMock, Playwright for E2E |
| Frontends | Angular 22 first, React 19 later, same API, Authorization Code + PKCE |
| Messaging (Phase 2) | Kafka in KRaft mode in a container, Outbox |
| Payments (Phase 2) | Stripe test mode, WireMock for failure injection |
| Observability | Actuator, Micrometer, OpenTelemetry, Grafana LGTM stack |
| Deploy | Docker Compose, then k3d or k3s, then EKS in bursts via Terraform |
| CI and review | GitHub Actions, GHCR, CodeRabbit, Apache 2.0 license |

**Repository layout (monorepo):** `approval-service/`, `payment-service/` (Phase 2), `web-angular/`, `web-react/` (Phase 3), `infra/` (Compose, Keycloak realm, Helm, Terraform), `docs/` (ADRs, domain).

## GitHub setup

Same flow as bcb-mcp: create labels and milestones, then the epics as issues, then each ticket with **Create sub-issue** inside its epic. Turn on auto-add so every issue lands on the board.

**Repository:** `payment-approvals` (rename freely). Public, Apache 2.0, CodeRabbit installed.

**Labels:** `epic`, `type: feature`, `type: test`, `type: chore`, `type: docs`, `area: backend`, `area: angular`, `area: react`, `area: infra`

**Milestones** (named by content, not by weekend, so they never go stale):

| Milestone | Phase | Done when |
| --- | --- | --- |
| M1 Foundation | 1 | Authenticated call to the API works end to end on Compose |
| M2 Requests and authorization | 1 | All five product rules enforced and proven by tests |
| M3 Angular client | 1 | A requester and an approver complete the flow in the browser |
| M4 Observability | 2 | Metrics, logs and traces in Grafana, one SLO with an alert |
| M5 Payments saga | 2 | Approved request is paid or compensated, with failures injected |
| M6 Kubernetes | 2 | Everything runs on k3d or k3s from a Helm chart via Argo CD |
| M7 React client | 3 | Same screens as Angular, same API |
| M8 Social login and BFF | 3 | Google login brokered by Keycloak; tokens kept out of the browser |
| M9 EKS | 3 | Created, demonstrated and destroyed with Terraform |

**Epics** (create first, label `epic`):

| Title | Description |
| --- | --- |
| `Epic: Foundation` | Project skeleton, Keycloak, database, CI and the security baseline. |
| `Epic: Requests and authorization` | Payment requests, approval policy, approvals and audit trail, with tenant isolation. |
| `Epic: Angular client` | Angular 22 client with PKCE login and the requester, approver and admin screens. |
| `Epic: Quality and testing` | Test strategy, test tooling and non-functional tests: smoke, mutation, load and performance. |
| `Epic: Observability` | Metrics, structured logs, distributed tracing and one SLO. |
| `Epic: Payments saga` | payment-service with ledger, orchestrated saga, Stripe and compensation. |
| `Epic: Kubernetes` | Helm chart, probes, config and GitOps on a local or VPS cluster. |
| `Epic: React client` | React 19 client with the same screens and login. |
| `Epic: Social login and BFF` | Google login through Keycloak and a Backend For Frontend. |
| `Epic: EKS deployment` | Terraform for an ephemeral EKS environment. |

GitHub numbers issues in creation order, so the epics take the first numbers. Use GitHub's numbers in commits (`Closes #12`), not the order in this doc.

## Epic: Foundation (M1)

Nine tickets. Done when an authenticated request reaches the API on Compose and the wrong token is refused for the right reason.

**Title:** `Spike: domain model and first ADRs` · **Labels:** `type: docs` · **Milestone:** M1 Foundation

```markdown
Write down the domain and the decisions before any code.

## Acceptance criteria
- [ ] docs/domain.md: entities (Tenant, PaymentRequest, ApprovalPolicy, PolicyRule, Approval, AuditEvent), states and allowed transitions
- [ ] docs/domain.md: the five product rules, numbered, so tests can reference them
- [ ] ADR-001 Keycloak as identity provider (alternatives: Spring Authorization Server, Auth0)
- [ ] ADR-002 tenant isolation: tenant_id column, compared with schema per tenant and Postgres row level security
- [ ] ADR-003 how the tenant reaches the token (user attribute mapper vs Keycloak Organizations)
```

**Title:** `Project setup` · **Labels:** `type: chore`, `area: backend` · **Milestone:** M1 Foundation

```markdown
Create the monorepo and the approval-service skeleton.

## Acceptance criteria
- [ ] Folders: approval-service/, infra/, docs/
- [ ] Java 25, Spring Boot 4.1, Maven wrapper
- [ ] Dependencies: web, security, oauth2-resource-server, data-jpa, validation, actuator, flyway, postgresql
- [ ] Virtual threads enabled
- [ ] GitHub Actions runs ./mvnw verify on every push and PR
- [ ] Apache 2.0 LICENSE, README stub, CodeRabbit config
```

**Title:** `Local infrastructure with Docker Compose` · **Labels:** `type: chore`, `area: infra` · **Milestone:** M1 Foundation

```markdown
Postgres and Keycloak running locally with one command.

## Acceptance criteria
- [ ] infra/compose.yaml with Postgres and Keycloak, both with healthchecks
- [ ] Keycloak realm "approvals" imported at startup from infra/keycloak/realm.json
- [ ] Clients: approvals-api (bearer only) and approvals-web (public, Authorization Code + PKCE)
- [ ] Realm roles: requester, approver, admin
- [ ] tenant_id claim present in the access token (mapper per ADR-003)
- [ ] Seed users: two tenants (acme, beta), each with a requester, two approvers and an admin
- [ ] README section: how to start, the seed users and how to get a token with curl
```

**Title:** `Implement persistence baseline` · **Labels:** `type: feature`, `area: backend` · **Milestone:** M1 Foundation

```markdown
Schema and JPA mapping for the Phase 1 entities.

## Acceptance criteria
- [ ] Flyway V1 migration with all Phase 1 tables, every business table carries tenant_id NOT NULL
- [ ] Money stored as NUMERIC(19,2), mapped to BigDecimal, never double
- [ ] Optimistic locking (@Version) on PaymentRequest
- [ ] Audit table is insert only: no update or delete path in the code
- [ ] Indexes on (tenant_id, status) and (tenant_id, created_at)
- [ ] spring.jpa.open-in-view=false
```

**Title:** `Test persistence baseline` · **Labels:** `type: test`, `area: backend` · **Milestone:** M1 Foundation

```markdown
## Acceptance criteria
- [ ] Testcontainers Postgres, Flyway runs from scratch on every test run
- [ ] Repository tests for save and find by tenant
- [ ] Optimistic lock test: two stale updates, the second fails with the expected exception
- [ ] Reused container across the test suite, so the suite stays fast
```

**Title:** `Implement security baseline` · **Labels:** `type: feature`, `area: backend` · **Milestone:** M1 Foundation

```markdown
approval-service as an OAuth2 resource server validating Keycloak JWTs.

## Acceptance criteria
- [ ] JWT validated through issuer-uri, audience checked
- [ ] Keycloak realm roles mapped to Spring authorities (ROLE_REQUESTER, ROLE_APPROVER, ROLE_ADMIN)
- [ ] tenant_id read from the token into a typed principal; requests without it are rejected
- [ ] Stateless session, CSRF disabled for the API, CORS limited to the web origin
- [ ] Method security enabled; actuator health public, everything else authenticated
- [ ] GET /me returns the user id, tenant and roles (useful for both frontends)
```

**Title:** `Test security baseline` · **Labels:** `type: test`, `area: backend` · **Milestone:** M1 Foundation

```markdown
## Acceptance criteria
- [ ] Slice tests with the jwt() request post processor for role and claim combinations
- [ ] Integration test with a real Keycloak (Testcontainers) obtaining real tokens for seed users
- [ ] No token: 401. Expired token: 401. Wrong issuer: 401
- [ ] Valid token without the needed role: 403
- [ ] Token without tenant_id: rejected
```

**Title:** `Implement error model` · **Labels:** `type: feature`, `area: backend` · **Milestone:** M1 Foundation

```markdown
One error format for the whole API, used by both frontends.

## Acceptance criteria
- [ ] ProblemDetail (RFC 9457) for every error, from one @RestControllerAdvice
- [ ] Validation errors list each field and message
- [ ] 401 and 403 from the security filter chain use the same format
- [ ] No stack traces or internal messages in responses
- [ ] Domain exceptions map to status codes in one place
```

**Title:** `Test error model` · **Labels:** `type: test`, `area: backend` · **Milestone:** M1 Foundation

```markdown
## Acceptance criteria
- [ ] Snapshot of the ProblemDetail JSON for validation, 401, 403, 404 and 409
- [ ] Unexpected exception returns 500 with a generic message and logs the stack trace
- [ ] Test asserts the response body never contains exception class names
```

## Epic: Requests and authorization (M2)

Eleven tickets. This is the heart of the project: every product rule from the overview gets an implementation and a test that names it.

**Title:** `Implement approval policy management` · **Labels:** `type: feature`, `area: backend` · **Milestone:** M2 Requests and authorization

```markdown
Admins define how many approvals each kind of payment needs.

## Acceptance criteria
- [ ] A policy has rules: payment method (PIX, BOLETO, TED) + amount range → required approvals (1 to 3)
- [ ] GET, PUT /policy, admin only
- [ ] Ranges for the same method cannot overlap or leave gaps (validated, 422 with a clear message)
- [ ] Every tenant gets a default policy when created (1 approval for everything)
- [ ] Changing the policy does not affect requests already submitted (rule 3)
```

**Title:** `Test approval policy management` · **Labels:** `type: test`, `area: backend` · **Milestone:** M2 Requests and authorization

```markdown
## Acceptance criteria
- [ ] Unit tests for range validation: overlap, gap, boundary values (exactly 10000.00)
- [ ] Unit tests for rule resolution given method and amount
- [ ] Requester and approver get 403 on PUT /policy
```

**Title:** `Implement create payment request` · **Labels:** `type: feature`, `area: backend` · **Milestone:** M2 Requests and authorization

```markdown
POST /requests, requester role.

## Acceptance criteria
- [ ] Body: supplier name, supplier document, amount, payment method, due date, description
- [ ] Amount > 0 with 2 decimal places; due date not in the past
- [ ] tenant_id and requester id come from the token, never from the body
- [ ] Required approvals resolved from the policy and stored on the request (snapshot)
- [ ] Status PENDING_APPROVAL; returns 201 with Location header
- [ ] Audit event REQUEST_SUBMITTED recorded in the same transaction
```

**Title:** `Test create payment request` · **Labels:** `type: test`, `area: backend` · **Milestone:** M2 Requests and authorization

```markdown
## Acceptance criteria
- [ ] Validation cases: zero, negative, three decimals, past due date, missing fields
- [ ] A tenant_id sent in the body is ignored
- [ ] Required approvals match the policy at submit time
- [ ] Approver without requester role gets 403
```

**Title:** `Implement list and get requests` · **Labels:** `type: feature`, `area: backend` · **Milestone:** M2 Requests and authorization

```markdown
GET /requests (paged, filter by status) and GET /requests/{id}.

## Acceptance criteria
- [ ] Requesters see only their own requests; approvers and admins see all requests of their tenant
- [ ] Every query filtered by tenant_id from the token, in one place (not repeated per query by hand)
- [ ] A request from another tenant returns 404, never 403 (rule 1)
- [ ] Response shows approvals so far and approvals required
- [ ] Page size capped; default sort newest first
```

**Title:** `Test list and get requests` · **Labels:** `type: test`, `area: backend` · **Milestone:** M2 Requests and authorization

```markdown
## Acceptance criteria
- [ ] Requester A does not see requester B's requests in the same tenant
- [ ] Approver sees all requests of the tenant
- [ ] Status filter and pagination work, page size cap enforced
```

**Title:** `Implement approve and reject` · **Labels:** `type: feature`, `area: backend` · **Milestone:** M2 Requests and authorization

```markdown
POST /requests/{id}/approve and POST /requests/{id}/reject (with reason), approver role.

## Acceptance criteria
- [ ] Authorization rule in a dedicated component used by @PreAuthorize, not inside the controller
- [ ] Cannot approve or reject your own request, even with the approver role (rule 4)
- [ ] Cannot approve the same request twice (rule 4), enforced by a unique constraint too
- [ ] Only PENDING_APPROVAL requests accept decisions; otherwise 409
- [ ] Reaching the required count moves to APPROVED; one rejection moves to REJECTED
- [ ] Concurrent decisions handled with optimistic locking; the loser gets 409, never a lost approval
- [ ] Audit events APPROVAL_GIVEN, REQUEST_APPROVED, REQUEST_REJECTED
```

**Title:** `Test approve and reject` · **Labels:** `type: test`, `area: backend` · **Milestone:** M2 Requests and authorization

```markdown
## Acceptance criteria
- [ ] Self approval refused for a user who has both requester and approver roles
- [ ] Second approval by the same user refused
- [ ] Two required approvals: first keeps PENDING_APPROVAL, second moves to APPROVED
- [ ] Decision on APPROVED or REJECTED request returns 409
- [ ] Concurrency test: two approvers approve at the same instant on a request needing 2; final state APPROVED with exactly 2 approvals
```

**Title:** `Implement audit trail` · **Labels:** `type: feature`, `area: backend` · **Milestone:** M2 Requests and authorization

```markdown
GET /requests/{id}/history.

## Acceptance criteria
- [ ] Every state change recorded with actor id, actor name, event type, timestamp and details (rule 5)
- [ ] Events written in the same transaction as the change they describe
- [ ] History ordered oldest first; visible to whoever can see the request
- [ ] No endpoint or repository method updates or deletes audit events
```

**Title:** `Test audit trail` · **Labels:** `type: test`, `area: backend` · **Milestone:** M2 Requests and authorization

```markdown
## Acceptance criteria
- [ ] Full flow (submit, approve, approve) produces the expected event sequence
- [ ] A failed decision (409) writes no event
- [ ] Rollback test: if the state change fails, no orphan audit event remains
```

**Title:** `Test tenant isolation across all endpoints` · **Labels:** `type: test`, `area: backend` · **Milestone:** M2 Requests and authorization

```markdown
One suite that proves rule 1 for the whole API, so a new endpoint cannot forget it.

## Acceptance criteria
- [ ] Data created in tenant acme; every endpoint called with tokens from tenant beta (each role)
- [ ] GET, approve, reject and history on acme ids return 404
- [ ] List endpoints return zero acme items
- [ ] The suite iterates over all mapped endpoints, and fails if a new endpoint is not covered
```

## Epic: Angular client (M3)

Twelve tickets. Done when a requester from acme submits a request and two acme approvers take it to APPROVED, entirely in the browser. Modern Angular only: standalone components, signals, new control flow, `inject()`.

**Title:** `Angular project setup` · **Labels:** `type: chore`, `area: angular` · **Milestone:** M3 Angular client

```markdown
## Acceptance criteria
- [ ] web-angular/ with Angular 22, standalone components, strict TypeScript
- [ ] ESLint and Prettier; lint and unit tests run in GitHub Actions
- [ ] API base URL from environment config; dev proxy to approval-service
- [ ] Typed API models matching the backend DTOs (generated from OpenAPI or written by hand, decision noted in README)
```

**Title:** `Implement login and route guards` · **Labels:** `type: feature`, `area: angular` · **Milestone:** M3 Angular client

```markdown
Login against Keycloak with Authorization Code + PKCE.

## Acceptance criteria
- [ ] OIDC library chosen and noted in an ADR (angular-auth-oidc-client or keycloak-angular)
- [ ] Functional HTTP interceptor adds the access token only to API calls
- [ ] Silent token refresh; expired session redirects to login
- [ ] Current user loaded from GET /me into a signal (id, tenant, roles)
- [ ] Route guards by role: /policy admin only, /queue approver only
- [ ] Logout ends the Keycloak session
```

**Title:** `Test login and route guards` · **Labels:** `type: test`, `area: angular` · **Milestone:** M3 Angular client

```markdown
## Acceptance criteria
- [ ] Guard tests: allowed role passes, missing role redirects
- [ ] Interceptor test: token added to API URLs, never to third party URLs
- [ ] 401 from the API triggers re-login
```

**Title:** `Implement requests list and detail` · **Labels:** `type: feature`, `area: angular` · **Milestone:** M3 Angular client

```markdown
## Acceptance criteria
- [ ] List with status filter and pagination; state held in signals
- [ ] Detail shows approvals so far vs required and the audit history
- [ ] Loading, empty and error states on every screen
- [ ] Errors shown from the ProblemDetail response, not a generic message
- [ ] @if / @for (with track) control flow; no NgModules
```

**Title:** `Test requests list and detail` · **Labels:** `type: test`, `area: angular` · **Milestone:** M3 Angular client

```markdown
## Acceptance criteria
- [ ] Component tests with HttpTestingController: success, empty, error
- [ ] Changing the filter cancels the previous request (switchMap behavior)
- [ ] 404 on detail shows a not found page
```

**Title:** `Implement create request form` · **Labels:** `type: feature`, `area: angular` · **Milestone:** M3 Angular client

```markdown
## Acceptance criteria
- [ ] Reactive form with the same validation rules as the backend
- [ ] Amount input handles BRL format (1.234,56) and sends a plain decimal
- [ ] Server validation errors mapped to the right fields
- [ ] Submit button disabled while sending; no double submit
- [ ] After success, navigate to the new request's detail
```

**Title:** `Test create request form` · **Labels:** `type: test`, `area: angular` · **Milestone:** M3 Angular client

```markdown
## Acceptance criteria
- [ ] Each validator tested, including amount formats
- [ ] Field level server error displayed on the right input
- [ ] Double click sends exactly one request
```

**Title:** `Implement approver queue and decisions` · **Labels:** `type: feature`, `area: angular` · **Milestone:** M3 Angular client

```markdown
## Acceptance criteria
- [ ] Queue of PENDING_APPROVAL requests of the tenant
- [ ] Approve and reject (reason required) with confirmation
- [ ] Buttons hidden on the user's own requests and on requests already decided by them, with an explanation
- [ ] 409 from a concurrent decision shows a clear message and refreshes the request
```

**Title:** `Test approver queue and decisions` · **Labels:** `type: test`, `area: angular` · **Milestone:** M3 Angular client

```markdown
## Acceptance criteria
- [ ] Own request shows no decision buttons
- [ ] Reject without reason blocked
- [ ] 409 handling refreshes the data
```

**Title:** `Implement policy admin screen` · **Labels:** `type: feature`, `area: angular` · **Milestone:** M3 Angular client

```markdown
## Acceptance criteria
- [ ] Edit rules per payment method: amount ranges and required approvals
- [ ] Dynamic rows (FormArray): add and remove a rule
- [ ] Overlap and gap errors from the backend shown on the affected rows
```

**Title:** `Test policy admin screen` · **Labels:** `type: test`, `area: angular` · **Milestone:** M3 Angular client

```markdown
## Acceptance criteria
- [ ] Add and remove rows; form value matches the PUT body
- [ ] Backend 422 errors mapped to rows
```

**Title:** `End to end test with Playwright` · **Labels:** `type: test`, `area: angular` · **Milestone:** M3 Angular client

```markdown
The whole Phase 1 flow against the real stack on Compose.

## Acceptance criteria
- [ ] Requester submits R$ 15.000 by boleto; policy requires 2 approvals
- [ ] Requester sees no approve button on their own request
- [ ] Approver 1 approves: still pending. Approver 2 approves: APPROVED
- [ ] A beta user cannot open the acme request URL (not found)
- [ ] Runs in GitHub Actions with Compose, on PRs to main
```

## Epic: Quality and testing (cross-cutting)

Ten tickets. Unit, integration and E2E tests already live in each feature's test ticket; this epic adds the strategy, the tooling and the test types that cover the whole system. The epic has no single milestone; each ticket carries its own, so it lands when its prerequisites exist.

| Test type | Where it lives | When it runs |
| --- | --- | --- |
| Unit | each feature's test ticket | every push |
| Integration (Testcontainers) | each feature's test ticket | every push |
| Architecture (ArchUnit) | this epic, M1 | every push |
| Static analysis (SonarQube Cloud) | this epic, M1 | every PR to main, blocks merge |
| Smoke | this epic, M1 (reused in M6 and M9) | after every deploy and in CI on Compose |
| End to end (Playwright) | Angular epic, M3 | PRs to main |
| Mutation (PIT) | this epic, M2 | nightly and on demand |
| Load (Gatling) | this epic, M4 | on demand, never on PRs |
| Performance baseline | this epic, M4 | on demand, before and after relevant changes |
| Saga resilience under load | this epic, M5 | on demand |

**Title:** `Write test strategy` · **Labels:** `type: docs` · **Milestone:** M1 Foundation

```markdown
docs/testing.md: what each test type proves, where it runs and how it is named.

## Acceptance criteria
- [ ] The test pyramid for this project, with one sentence per test type on what it proves
- [ ] Naming: unit tests *Test, integration tests *IT
- [ ] Which tests run on push, on PR, nightly, after deploy and on demand
- [ ] Tools per type, with the reason for each choice
- [ ] Coverage policy: what is measured, the threshold, and what is excluded
```

**Title:** `Separate unit and integration test runs` · **Labels:** `type: chore`, `area: backend` · **Milestone:** M1 Foundation

```markdown
Fast feedback without Docker; full verification in CI.

## Acceptance criteria
- [ ] Surefire runs *Test, Failsafe runs *IT
- [ ] ./mvnw test runs only unit tests and needs no Docker
- [ ] ./mvnw verify runs both
- [ ] JaCoCo report merges unit and integration coverage
- [ ] Coverage threshold on the domain and authorization packages fails the build (value from docs/testing.md)
- [ ] CI uploads the test and coverage reports as artifacts
```

**Title:** `Architecture tests with ArchUnit` · **Labels:** `type: test`, `area: backend` · **Milestone:** M1 Foundation

```markdown
Rules about the code's structure, enforced by the build.

## Acceptance criteria
- [ ] Controllers never access repositories directly
- [ ] Domain package has no dependency on Spring web or persistence annotations
- [ ] No class outside the audit package writes audit events
- [ ] Every controller method has an authorization annotation or is explicitly public
- [ ] Rules documented in docs/testing.md
```

**Title:** `Smoke test suite` · **Labels:** `type: test`, `area: infra` · **Milestone:** M1 Foundation

```markdown
A few fast checks that prove a deployed environment is alive and wired correctly. Reused after deploys in M6 (Kubernetes) and M9 (EKS).

## Acceptance criteria
- [ ] Runs against a base URL passed as a parameter, not a hardcoded host
- [ ] Checks: health endpoint UP, token obtained from Keycloak for a seed user, GET /me returns the right tenant
- [ ] Creates one request and reads it back, then cleans up or uses a dedicated smoke tenant
- [ ] Finishes in under 30 seconds and exits non zero on any failure
- [ ] Runs in CI against the Compose stack after the build
```

**Title:** `Static analysis with SonarQube Cloud` · **Labels:** `type: chore`, `area: infra` · **Milestone:** M1 Foundation

```markdown
Quality gate on every pull request to main.

## Acceptance criteria
- [ ] SonarQube Cloud organization bound to the GitHub account, Free plan
- [ ] One project per module: approval-service (Java) and web-angular (TypeScript), added when that module exists
- [ ] CI-based analysis from GitHub Actions (not automatic analysis), so coverage is imported
- [ ] JaCoCo XML report and the Angular lcov report passed to the analysis
- [ ] SONAR_TOKEN stored as a repository secret
- [ ] Quality gate "Sonar way" on new code; result reported as a status check on the PR
- [ ] Badge for quality gate status in the README
```

**Title:** `Branch protection on main` · **Labels:** `type: chore`, `area: infra` · **Milestone:** M1 Foundation

```markdown
Nothing reaches main without passing the gates.

## Acceptance criteria
- [ ] Ruleset on main: changes only through pull requests, no force push, no deletion
- [ ] Required status checks: CI build and tests, SonarQube Cloud quality gate
- [ ] Conversation resolution required before merging, so every CodeRabbit comment must be addressed
- [ ] No required approving review (a solo author cannot approve their own PR); documented in CONTRIBUTING.md
- [ ] Verified with a test PR that fails the quality gate and cannot be merged
```

**Title:** `Mutation testing with PIT` · **Labels:** `type: test`, `area: backend` · **Milestone:** M2 Requests and authorization

```markdown
Prove the tests actually catch broken rules, not just execute lines.

## Acceptance criteria
- [ ] pitest-maven-plugin with the JUnit 5 plugin, scoped to the domain and authorization packages
- [ ] Mutation score threshold set in docs/testing.md; build fails below it
- [ ] Runs nightly and on demand in GitHub Actions, not on every push
- [ ] HTML report uploaded as an artifact
- [ ] Surviving mutants in the five product rules reviewed and fixed with new tests; findings noted in docs/testing.md
```

**Title:** `Load test with Gatling` · **Labels:** `type: test`, `area: backend` · **Milestone:** M4 Observability

```markdown
Realistic traffic against the Compose stack, written in Gatling's Java DSL.

## Acceptance criteria
- [ ] Tokens for seed users obtained from Keycloak before the scenario starts
- [ ] Scenario 1: requesters submit requests at a ramping rate
- [ ] Scenario 2: approvers decide on the same requests concurrently
- [ ] Assertions on p95 latency and error rate, with targets taken from the M4 SLO
- [ ] After the run, a check that no request has more approvals than required and none was lost
- [ ] Runs on demand from a workflow_dispatch, never on PRs; HTML report uploaded
```

**Title:** `Performance baseline and analysis` · **Labels:** `type: test`, `area: backend` · **Milestone:** M4 Observability

```markdown
Numbers to compare against, and the reasons behind them.

## Acceptance criteria
- [ ] Seed script with 100k requests across both tenants
- [ ] p50, p95, p99 and throughput recorded for list, get, create and approve
- [ ] EXPLAIN ANALYZE for the list query with filters; index decisions justified
- [ ] Same load with virtual threads on and off; results compared
- [ ] Findings in docs/performance.md with the Grafana screenshots
```

**Title:** `Saga resilience under load` · **Labels:** `type: test`, `area: backend` · **Milestone:** M5 Payments saga

```markdown
The payment flow keeps its invariants when things fail at volume.

## Acceptance criteria
- [ ] Load run with injected failures: a percentage of Stripe timeouts and 500s, duplicate Kafka events, duplicate webhooks
- [ ] One payment-service instance restarted mid run
- [ ] After the run: no request charged twice, ledger balance equals the sum of its entries, every APPROVED request ends PAID or PAYMENT_FAILED
- [ ] Results and any bug found recorded in docs/performance.md
```

## Later epics (Phases 2 and 3)

Create these epics now so the roadmap is visible, but write their tickets only when you reach them: by then Phase 1 will have changed what they need. Each candidate below becomes a dev ticket plus a test ticket, as in Phase 1.

### Epic: Observability (M4)

- Structured JSON logs with trace id and tenant id in every line
- OpenTelemetry tracing; Grafana LGTM container in Compose
- Custom Micrometer metrics: requests submitted, approvals, decision latency
- Grafana dashboard committed as JSON
- One SLO (for example, 99% of decisions under 2 s) with a burn rate alert

### Epic: Payments saga (M5)

- Outbox in approval-service: REQUEST\_APPROVED published to Kafka (KRaft container)
- payment-service skeleton, same security baseline
- Ledger module: entries table, cached balance, reserve, confirm and release, idempotent by request id
- Saga orchestrator as a persisted state machine; resumes after a crash
- Stripe test mode client with idempotency keys and webhook handling
- Compensation: release the reservation and move the request to PAYMENT\_FAILED
- Failure injection with WireMock: timeout, 500, duplicate webhook, duplicate Kafka event
- Trace propagation across Kafka, visible as one trace in Grafana
- ADR: orchestration vs choreography; ADR: why two services, and when a modular monolith is the better call

### Epic: Kubernetes (M6)

- Images with spring-boot:build-image, pushed to GHCR from Actions
- Helm chart: Deployments, Services, ConfigMaps, Secrets, resource limits
- Liveness and readiness probes from Actuator health groups
- HPA on approval-service
- Kafka with the Strimzi operator; Keycloak and Postgres in the cluster or on the VPS
- Argo CD syncing the chart from the repo (GitOps)

### Epic: React client (M7)

- Vite + React 19 + TypeScript setup, same lint and CI as Angular
- OIDC login with PKCE, protected routes by role
- TanStack Query for server state; React Hook Form + Zod for forms
- Same four screens as Angular, same Playwright scenario
- docs/angular-vs-react.md: what was easier or harder in each, written from experience

### Epic: Social login and BFF (M8)

- Google as identity provider brokered by Keycloak, mapped to an existing tenant by invitation
- BFF with Spring Cloud Gateway: session cookie in the browser, tokens kept server side (token relay)
- ADR: SPA with tokens vs BFF, with the security reasons

### Epic: EKS deployment (M9)

- Terraform: VPC, EKS, node group, ECR or GHCR access, RDS or in-cluster Postgres (decision in ADR)
- One command up, one command down; cost estimate in the README
- Billing alert configured before the first apply
- Recorded demo, then terraform destroy
