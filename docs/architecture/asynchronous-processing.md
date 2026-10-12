# Asynchronous Processing Architecture

## 1. Purpose

This document defines how the Enterprise Notification Platform discovers, claims, processes, and recovers accepted notifications.

It establishes the reliability requirements for asynchronous execution, including processing coordination, retries, failure recovery, and interaction with the external SMS provider.

The design must preserve durable acceptance without requiring a dedicated message broker unless an explicit architectural decision justifies one.

## 2. Design Drivers

The asynchronous-processing design must:

* Preserve the platform's durable-acceptance guarantee.
* Recover accepted work after application restarts.
* Prevent concurrent processing executions from corrupting notification state.
* Track individual delivery attempts.
* Support bounded retries for eligible failures.
* Distinguish definite failures from uncertain provider outcomes.
* Minimize avoidable duplicate SMS submissions.
* Support observability and operational investigation.
* Remain compatible with Clean Architecture and the initial single-deployable-application approach.

The design must not promise exactly-once external delivery where the provider protocol cannot guarantee it.

## 3. Recommended V1 Approach

The preferred initial approach is database-backed work discovery and processing coordination.

The application will persist accepted notifications and use a background processing loop to discover eligible work. The processing loop will claim work using database-supported concurrency controls, execute the relevant application use case, and persist the resulting state.

The REST API and background processing loop will initially run within the same deployable application. Their responsibilities must remain logically separate.

This approach reduces the number of infrastructure components while using the database as the durable source of truth.

The final choice must be recorded in an Architecture Decision Record after evaluating database-backed polling, a transactional outbox, and a dedicated message broker against the platform's reliability and operational requirements.

## 4. Logical Processing Flow

The intended processing flow is:

1. A client submits a notification through the REST API.
2. The application authenticates the request, establishes tenant context, and validates the submission.
3. The application coordinates idempotency and durable notification acceptance.
4. The database commits the accepted notification and its associated idempotency record.
5. The background processing loop discovers eligible work.
6. A processing execution claims the work using concurrency-safe coordination.
7. The application records the processing attempt and invokes the provider integration.
8. The provider outcome is interpreted and persisted.
9. Where applicable, subsequent delivery-status information updates the notification lifecycle.
10. Eligible failures are scheduled for retry; terminal failures remain recorded for investigation.

The notification record remains authoritative. The in-memory state of a running application must never be the only record that a notification requires processing.

## 5. Work Discovery

The processing mechanism must reliably identify notifications that are eligible for work.

The initial design should evaluate polling the database for records in an eligible lifecycle state, using appropriate indexes and bounded batches.

The polling mechanism must:

* Identify eligible notifications efficiently.
* Avoid repeatedly selecting work that is already being processed.
* Respect retry scheduling and any applicable processing limits.
* Support multiple processing executions without conflicting ownership.
* Recover unfinished work after a process restart.
* Avoid requiring a database scan across unrelated tenants when a narrower access pattern is practical.

The final query strategy, polling interval, batch size, and database-specific implementation will be established during implementation design and validated through testing.

## 6. Work Claiming and Concurrency

Discovering eligible work does not establish exclusive ownership of it.

Before processing begins, an execution must acquire a durable claim or perform an equivalent atomic state transition that prevents another execution from processing the same work concurrently.

The implementation must address:

* Concurrent attempts to claim the same notification.
* Application instances running simultaneously.
* Process termination after claiming work.
* Expiration or recovery of stale claims.
* Prevention of conflicting lifecycle transitions.
* Protection against a stale execution overwriting a newer result.

Database-supported locking, conditional updates, or another concurrency-safe mechanism may be used, subject to the persistence design.

Claims should include enough information to identify the owning execution or attempt and determine when recovery is appropriate.

The implementation must not rely on a process-local mutex as the sole coordination mechanism when multiple application instances or concurrent executions are possible.

## 7. Processing Attempts

Each provider submission attempt must be distinguishable from the overall notification lifecycle.

Before an external submission, the platform should persist sufficient attempt information to support recovery and investigation.

The attempt model must record the relevant identifiers, timestamps, provider correlation information when available, and meaningful outcomes.

The design must distinguish at least the following situations:

* An attempt was prepared but no provider request was made.
* A provider request was rejected with a known outcome.
* The provider accepted the submission.
* A provider request failed before submission could be established.
* The provider outcome is uncertain.
* Delivery status was subsequently reported by the provider.

A notification's processing claim and its delivery attempt are related but distinct concepts. The implementation must not equate acquiring a claim with completing a provider submission.

## 8. External Provider Interaction

Provider calls must occur outside long-running database transactions unless a documented technical requirement justifies otherwise.

The application must persist appropriate attempt information, invoke the provider adapter, and record the interpreted outcome.

The design must account for the possibility that the provider accepts an SMS while the platform fails to record the response.

In that situation, the database alone cannot determine whether the SMS was submitted. The platform must use available provider capabilities—such as idempotency, lookup, or reconciliation—to reduce uncertainty where possible.

If the provider cannot establish the outcome, the platform must preserve that uncertainty and follow an explicit recovery policy rather than automatically assuming that another submission is safe.

## 9. Retry Policy

Retries must be deliberate, bounded, and based on failure classification.

The retry policy must define:

* Which outcomes are eligible for retry.
* Maximum attempts or another explicit retry limit.
* Delay and backoff behavior.
* How retry eligibility is persisted.
* How a scheduled retry becomes eligible for processing.
* The final state when retry limits are exhausted.
* How operators can investigate repeated failures.

Temporary infrastructure failures or provider rate limits may be retryable when the provider's semantics support a safe recovery strategy.

Invalid requests and other non-retryable failures must not trigger automatic retries without an explicit rule.

Provider-reported delivery failures must be evaluated separately from submission failures. A delivery failure does not automatically mean that another submission is appropriate.

The precise retry values and policy are implementation decisions and must be documented before production use.

## 10. Stale Work and Recovery

The processing mechanism must recover work left unfinished by application termination, process crashes, or other interruptions.

Recovery must distinguish between:

* Work claimed but not yet submitted.
* Work submitted with a known provider response.
* Work whose provider outcome is uncertain.
* Work awaiting a scheduled retry.
* Work that has reached a terminal failure state.

A stale claim may be released or reclaimed only according to a defined recovery rule. Recovery must not blindly repeat an external side effect whose outcome remains uncertain.

The implementation must ensure that stale or delayed executions cannot overwrite newer state or incorrectly return terminal notifications to a processing state.

Recovery behavior must be tested at meaningful failure boundaries, including process termination immediately before and after an external provider request.

## 11. Delivery-Status Updates

Delivery-status information may arrive separately from the original provider submission response.

The selected provider's supported mechanism—such as an authenticated callback or polling—will be documented once the provider is chosen.

Incoming status information must be validated and correlated with the appropriate notification or delivery attempt.

The implementation must account for duplicate, delayed, and out-of-order status updates.

A provider callback must not be trusted solely because it contains a notification identifier. The integration must establish that the update is authentic and associated with a valid provider submission.

Lifecycle transitions must follow the domain rules. A stale status update must not silently overwrite a newer or terminal outcome.

## 12. Transactional Outbox Evaluation

A transactional outbox is an option when the architecture needs to persist business state and a separate dispatch instruction atomically, then publish or process that instruction independently.

For the initial database-backed approach, the notification table may itself provide the durable work source if eligible work can be discovered and claimed reliably without a separate dispatch record.

A separate outbox should be introduced when it solves a concrete consistency or integration problem, not simply because the platform is described as event-driven.

If an outbox is adopted, its design must define:

* Which transaction writes the outbox record.
* How pending records are discovered and claimed.
* How records are marked as dispatched or processed.
* How failed dispatch is retried.
* How duplicate dispatch is handled.
* How outbox records are retained and cleaned up.

The choice between direct database-backed work discovery and an outbox must be documented through an ADR.

## 13. Dedicated Message Broker Evaluation

A dedicated message broker may become appropriate if measured throughput, workload isolation, operational requirements, or independent scaling needs justify it.

It must not be introduced solely to make the architecture appear more sophisticated or because the platform uses asynchronous processing.

Before adding a broker, the architecture must evaluate its effect on:

* Operational complexity and failure modes.
* Message durability and delivery semantics.
* Consistency between database state and published messages.
* Duplicate processing and idempotency.
* Monitoring and operational recovery.
* Deployment and infrastructure costs.

Any future broker integration must preserve the domain and application dependency rules. Business logic must not depend directly on a specific broker's client API.

## 14. Observability

Asynchronous processing must produce enough operational information to investigate failures and processing delays.

The implementation should support:

* Correlation of a notification with its delivery attempts.
* Visibility into processing state and retry history.
* Identification of stale claims and overdue work.
* Metrics for queue or backlog depth, processing latency, retry counts, and failure rates.
* Logging of relevant provider interactions without exposing credentials or sensitive message content.
* Correlation identifiers that connect API acceptance to background processing.

Specific metrics, alert thresholds, and operational procedures will be defined in the reliability and observability architecture.

## 15. Testing Requirements

Testing must validate recovery and correctness under failure, not only the successful processing path.

Tests should cover:

* Recovery of accepted work after application restart.
* Concurrent work claiming.
* Duplicate submission requests and idempotency.
* Provider rejection and retryable failures.
* Provider timeouts and uncertain outcomes.
* Retry scheduling and exhaustion.
* Stale claim recovery.
* Duplicate and out-of-order delivery-status updates.
* Failure between provider acceptance and persistence of the response.
* Isolation of invalid or terminal lifecycle transitions.

Tests involving concurrency and persistence must use realistic integration conditions where mocks would conceal the behavior being verified.

## 16. Open Decisions

The following items require explicit resolution:

1. Confirm database-backed polling as the initial processing mechanism.
2. Determine whether a separate transactional outbox is necessary.
3. Define safe work claiming and stale-claim recovery.
4. Define the initial retry policy and attempt limits.
5. Select the initial SMS provider and its submission and delivery-status mechanisms.
6. Define the recovery policy for uncertain provider outcomes.
7. Define operational metrics and thresholds appropriate to V1.

Each decision must be evaluated against the requirements baseline, persistence guarantees, implementation complexity, reliability, and expected evolution of the platform.

## 17. Relationship to Other Architecture Artifacts

This document is derived from:

* `docs/requirements/requirements-baseline.md`
* `docs/architecture/architecture-drivers-and-principles.md`
* `docs/architecture/container-architecture.md`
* `docs/architecture/domain-model.md`
* `docs/architecture/application-architecture.md`
* `docs/architecture/persistence-architecture.md`

It informs the security, reliability, provider-integration, and implementation designs, together with the relevant Architecture Decision Records.

## 18. Status

**Status:** Proposed for architecture review.

The processing responsibilities, failure categories, and recovery requirements are defined. The exact coordination mechanism, retry policy, provider integration, and associated operational controls remain subject to documented decisions.
