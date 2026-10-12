# Persistence Architecture

## 1. Purpose

This document defines the persistence responsibilities and data consistency requirements of the Enterprise Notification Platform.

It establishes how the platform will preserve notification state, tenant ownership, idempotency, and recoverable asynchronous processing while keeping persistence technology independent of the domain model.

## 2. Design Drivers

The persistence architecture must support:

* Durable notification acceptance.
* Concurrency-safe idempotency.
* Tenant-scoped data access.
* Valid notification lifecycle transitions.
* Recovery of unfinished processing after application restarts.
* Consistent recording of notifications and processing-dispatch state.
* Traceable processing attempts and failure outcomes.
* Efficient status retrieval through the REST API.
* A maintainable initial deployment without unnecessary infrastructure.

Correctness and recoverability take priority over premature optimization for scale.

## 3. Recommended Persistence Approach

The initial design should use a relational database, with PostgreSQL as the preferred candidate.

A relational database is a strong fit because the platform requires transactional consistency, uniqueness constraints, concurrent-write coordination, and relationships between tenants, notifications, idempotency records, and processing attempts.

PostgreSQL is a recommendation to validate and formally record through an Architecture Decision Record (ADR), rather than an already-finalized technology commitment.

The initial deployment should use one authoritative primary database for transactional application state. Separate databases for different logical responsibilities are not justified for V1.

A dedicated message broker is not required merely to achieve asynchronous processing. A database-backed processing mechanism should be evaluated first against the reliability and operational requirements.

## 4. Logical Data Responsibilities

The persistence design must support the following logical records. These describe responsibilities rather than a finalized relational schema.

### 4.1 Tenant Records

Tenant records represent platform customer accounts and support tenant lifecycle and ownership rules.

Tenant-owned resources must be associated with an explicit tenant identifier.

Authentication credentials and authorization-related records must be designed alongside the security architecture.

### 4.2 Notification Records

Notification records represent accepted notification requests and their current lifecycle state.

The persisted information must support:

* Unique notification identification.
* Tenant ownership.
* Recipient and message information, subject to data-protection requirements.
* Channel identification.
* Lifecycle state.
* Creation and update timestamps.
* Processing eligibility and recovery.
* Correlation with delivery attempts and provider outcomes.

The persisted state must remain the authoritative record of the platform's notification lifecycle.

### 4.3 Idempotency Records

Idempotency records associate a tenant-scoped submission key with the accepted request and its outcome.

The design must support:

* Uniqueness within the defined idempotency scope.
* Detection of incompatible reuse of a key.
* Consistent association with the accepted notification.
* Concurrency-safe handling of simultaneous submissions.
* Defined retention and expiration behavior.

Database-enforced uniqueness must protect against races that application-level checks alone cannot prevent.

### 4.4 Delivery Attempt Records

Delivery attempt records capture individual attempts to submit notifications to an external provider.

They should support investigation of:

* Attempt sequence and timestamps.
* Processing outcomes.
* Provider correlation identifiers, when available.
* Retry eligibility and failure classification.
* Uncertain submission outcomes.
* Provider-reported submission and delivery information.

The design must distinguish the notification's overall lifecycle from the history of its individual attempts.

### 4.5 Processing Coordination Records

The persistence design must provide enough durable information to identify eligible work and recover unfinished processing.

Depending on the selected approach, this may use notification state, explicit processing records, a transactional outbox, or a combination of these.

The final design must avoid relying exclusively on an in-memory queue or scheduler as the authoritative record of pending work.

## 5. Transaction Boundaries

### 5.1 Notification Acceptance

Notification acceptance must be atomic with respect to the records required to establish durable acceptance and idempotency.

The persistence operation must ensure that the platform does not commit an accepted notification while losing the only durable record needed to make it available for processing.

Where a separate outbox or processing-dispatch record is required, the notification, idempotency record, and dispatch record must be committed in the same database transaction.

The API must return its successful asynchronous-acceptance response only after the transaction commits and the platform has established a recoverable processing path.

If the transaction fails, the API must not report successful acceptance.

### 5.2 Processing State Changes

Changes to processing state must be applied through transactions that preserve valid lifecycle transitions and prevent conflicting concurrent updates.

The design must establish how a processing execution claims eligible work and how stale claims are recovered after process failure.

A worker must not assume that it exclusively owns a notification merely because it previously read the record.

### 5.3 Provider Interaction

An external SMS-provider request cannot ordinarily participate in the database transaction.

The platform must therefore avoid holding a database transaction open across a potentially slow provider call without a demonstrated need.

The processing flow must record enough information before and after the external call to support recovery, investigation, and reconciliation.

A failure between provider acceptance and the platform's recording of that outcome must be treated as a potentially uncertain external result.

Database transactions alone cannot guarantee exactly-once delivery to an external recipient.

## 6. Idempotency and Concurrency

Idempotency must be enforced durably rather than through an in-memory cache.

The implementation must use database-supported uniqueness and transaction semantics to coordinate simultaneous requests using the same applicable key.

The design must account for these cases:

1. A first submission is accepted successfully.
2. A client retries after the original response is lost.
3. Two equivalent requests arrive concurrently with the same key.
4. The same key is reused with materially different request content.
5. A transaction fails before acceptance is committed.
6. The application restarts after acceptance has committed.

Equivalent retries must resolve to the original accepted operation within the defined idempotency policy. They must not create a second notification.

The API contract must define how the platform responds to incompatible key reuse and how long idempotency records remain effective.

## 7. Asynchronous Processing and Recovery

The database-backed processing design must support reliable discovery and claiming of pending work.

At a minimum, the implementation must define:

* How eligible notifications are discovered.
* How concurrent workers claim work without conflicting ownership.
* How processing attempts are recorded.
* How interrupted processing is detected.
* How stale claims are recovered.
* How retry eligibility and scheduling are persisted.
* How exhausted retries become a defined terminal outcome.
* How uncertain provider outcomes are reconciled where possible.

The design should favor recoverable, explicit state transitions over reliance on transient process memory.

A transactional outbox should be used if the selected design requires notification acceptance and dispatch intent to be recorded together while dispatch occurs separately. If the notification table itself can safely serve as the durable work source, a separate outbox may not be necessary for V1.

The final choice must be documented through the asynchronous-processing design and a corresponding ADR.

## 8. Tenant Isolation

Every tenant-owned notification and associated record must be linked to its owning tenant directly or through an unambiguous, enforceable relationship.

Application queries must scope resource access to the authenticated tenant context.

The persistence design should reinforce this boundary through appropriate constraints, indexes, and query patterns.

Where practical, relationships between tenant-owned records should prevent inconsistent cross-tenant associations.

Knowing or guessing another tenant's notification identifier must not grant access to its data.

Database-level row security may be evaluated, but it should not be introduced automatically without assessing its operational complexity and how it complements application-level authorization.

## 9. Indexing and Query Responsibilities

The initial schema should support the principal access patterns:

* Retrieve a notification by identifier within the authorized tenant context.
* Find pending notifications eligible for processing.
* Enforce idempotency uniqueness.
* Retrieve delivery attempts associated with a notification.
* Find records requiring retry or recovery.
* Support operational investigation and retention processes.

Indexes must be based on actual query and concurrency requirements rather than added indiscriminately.

Notification status retrieval must not require scanning unrelated tenants' records.

Pagination for API collections must remain consistent with the requirements baseline, including cursor-based pagination where applicable.

## 10. Data Protection and Retention

Persisted notification content, recipient information, credentials, and provider metadata must be handled according to their sensitivity.

The design must establish:

* Which fields contain sensitive information.
* Which fields require encryption at rest or additional application-level protection.
* Which fields must never appear in logs.
* How database credentials and connection secrets are managed.
* How retention and deletion requirements apply to notifications and delivery attempts.
* How idempotency expiration interacts with duplicate-submission prevention.
* How backups and recovery procedures protect data confidentiality and availability.

Retention periods and detailed deletion rules remain subject to explicit product, security, and operational decisions.

## 11. Schema Evolution and Migrations

Database schema changes must be version-controlled and applied through a repeatable migration process.

Migrations must be reviewed alongside application changes for their effect on data integrity, deployment safety, and compatibility with existing records.

Destructive schema changes must not be assumed safe merely because the application compiles.

The implementation process must provide a documented approach for applying migrations in local development, automated tests, and deployed environments.

## 12. Testing Requirements

The persistence implementation must be tested for correctness, not only successful CRUD operations.

Tests should cover:

* Atomic notification acceptance.
* Idempotency uniqueness and concurrent submissions.
* Rollback when acceptance fails.
* Valid and invalid lifecycle transitions.
* Tenant-scoped access.
* Concurrent processing claims.
* Recovery of unfinished work.
* Persistence of retry and attempt history.
* Schema migration behavior.
* Failure handling at transaction boundaries.

Tests involving database-specific constraints and concurrency must use a realistic database environment rather than relying exclusively on mocks.

## 13. Open Decisions

The following decisions require explicit resolution:

1. Confirm PostgreSQL as the initial persistence technology.
2. Select the database-backed processing approach and determine whether a transactional outbox is necessary.
3. Define the transaction boundary for notification acceptance and idempotency.
4. Define safe work-claiming and stale-claim recovery behavior.
5. Finalize the initial logical data model and relational constraints.
6. Establish sensitive-data handling and retention requirements.
7. Select and document the database migration approach.
8. Define backup and recovery expectations appropriate to the initial deployment.

Each material technology or pattern choice must be supported by its requirements, alternatives, trade-offs, and operational implications.

## 14. Relationship to Other Architecture Artifacts

This document is derived from:

* `docs/requirements/requirements-baseline.md`
* `docs/architecture/architecture-drivers-and-principles.md`
* `docs/architecture/container-architecture.md`
* `docs/architecture/domain-model.md`
* `docs/architecture/application-architecture.md`

It informs the asynchronous-processing, security, reliability, and implementation designs, as well as the relevant Architecture Decision Records.

## 15. Status

**Status:** Proposed for architecture review.

The persistence responsibilities and consistency requirements are defined. PostgreSQL is the preferred candidate, while the final technology selection, processing coordination pattern, schema, and operational controls remain subject to documented decisions.
