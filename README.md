# payment-approvals

A multi-tenant system where a company's members request supplier payments, approvers sign off according to company policy, and the approved payment is executed through a saga. Phase 1 alone (milestones M1 to M3) is a complete, showable project.

## Product rules

Each rule becomes a test.

1. A user only sees data from their own tenant. A request from another tenant returns 404, never 403.
2. Requesters create requests; approvers approve or reject; admins manage the approval policy.
3. The number of approvals required depends on payment method and amount, per the tenant's policy, snapshotted when the request is submitted.
4. Nobody approves their own request. Nobody approves the same request twice.
5. Every state change is recorded in an append-only audit trail with actor and time.

**States:** `PENDING_APPROVAL` → `APPROVED` or `REJECTED` (Phase 1). Phase 2 adds `PAID` and `PAYMENT_FAILED`.

## Stack

Java 25, Spring Boot 4.1 and Maven; Spring Security as an OAuth2 resource server with Keycloak; PostgreSQL with Flyway; JUnit 5, Testcontainers, WireMock and Playwright; Angular 22 first, React 19 later. Phase 2 adds Kafka (KRaft) with an outbox, Stripe test mode, OpenTelemetry and the Grafana LGTM stack. Deployment goes from Docker Compose to k3d/k3s, then to EKS in bursts via Terraform.

## Folder layout

```
payment-approvals/
├── approval-service/   Spring Boot API: requests, policy, approvals, audit
├── payment-service/    Ledger and payment saga (Phase 2)
├── contracts/          OpenAPI spec and Kafka event schemas; clients are generated from here
├── web-angular/        Angular client (Phase 1, M3)
├── web-react/          React client (Phase 3)
├── infra/              Compose, Keycloak realm, Helm, Terraform
└── docs/               ADRs, domain model, backlog
```

## Roadmap

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

The full backlog is in [docs/backlog.md](docs/backlog.md).

## License

[Apache 2.0](LICENSE)
