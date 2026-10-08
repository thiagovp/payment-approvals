# approval-service

The approval-service owns the payment request lifecycle. Requesters submit supplier payment requests, approvers approve or reject them, and admins manage each tenant's approval policy. The service snapshots the number of required approvals from that policy at submit time, enforces tenant isolation and separation of duties (nobody approves their own request or approves the same request twice), and records every state change in an append-only audit trail. It is a Spring Boot OAuth2 resource server that validates Keycloak JWTs and stores its data in PostgreSQL.
