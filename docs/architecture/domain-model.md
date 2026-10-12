# Domain Model and Notification Lifecycle

## 1. Purpose

This document defines the core domain concepts, business invariants, and notification lifecycle of the Enterprise Notification Platform.

It establishes a provider-independent model of notification processing so that business rules remain consistent regardless of the REST framework, persistence technology, asynchronous processing mechanism, or external SMS provider.

## 2. Domain Scope

The initial domain focuses on accepting, tracking, and processing SMS notifications for individual developers and enterprise customers.

The domain must support:

* Tenant-scoped notification ownership.
* Durable and idempotent notification acceptance.
* Explicit notification lifecycle transitions.
* Asynchronous processing and delivery-attempt tracking.
* Distinction between provider submission and final delivery outcomes.
* Recoverable processing failures and controlled retries.
* Consistent notification status exposed through the REST API.

The model should accommodate future notification channels without introducing unnecessary abstractions before they are justified.

## 3. Core Domain Concepts

### 3.1 Tenant

A Tenant represents an individual developer's or enterprise customer's isolated account within the platform.

A tenant owns its notifications and other tenant-scoped resources.

**Responsibilities and invariants:**

* Every notification belongs to exactly one tenant.
* Tenant identity is established through authenticated request context.
* A client-supplied tenant identifier must not independently determine resource ownership.
* Tenant-owned resources must not be accessible across tenant boundaries without explicit authorization.
* Credentials are associated with the appropriate tenant and must be managed securely.

A tenant is a domain ownership boundary, not merely a field used for filtering database queries.

### 3.2 Notification

A Notification represents the platform's accepted intent to send a message to a recipient through a selected notification channel.

For the initial version, the supported channel is SMS.

A notification should conceptually contain:

* A unique notification identifier.
* The owning tenant identifier.
* The recipient address or phone number.
* The message content or a protected reference to it.
* The selected notification channel.
* Its current lifecycle status.
* Creation and update timestamps.
* Relevant processing and delivery metadata.
* An optional client-provided idempotency key reference or association.

The exact persistence representation and field types will be defined by the application and persistence designs.

**Responsibilities and invariants:**

* A notification has exactly one owning tenant.
* Its identifier remains stable throughout its lifecycle.
* Its lifecycle status changes only through permitted transitions.
* The platform must distinguish acceptance, provider submission, and confirmed delivery.
* Provider-specific response formats must not define the domain's status model.
* Notification status must not imply more certainty than the available evidence supports.

### 3.3 Idempotency Record

An Idempotency Record represents the platform's record of a client submission associated with an idempotency key.

Its purpose is to allow a client to retry a request safely when the client cannot determine whether the original request succeeded.

The record must be scoped to the appropriate tenant and request operation. Its design must associate the key with sufficient request information to detect incompatible reuse.

**Responsibilities and invariants:**

* Concurrent requests using the same applicable key must not create multiple accepted notifications.
* Repeating an equivalent request with the same key must not create a second notification.
* Reusing a key with a materially different request must be handled according to a documented API policy.
* The idempotency record and accepted notification must be persisted consistently.
* Idempotency retention and expiration rules must be explicitly defined.
* Idempotency must remain effective across application restarts and concurrent requests.

The exact key scope, request fingerprinting rules, retention period, and replay response behavior are API and persistence design decisions.

Idempotency prevents duplicate acceptance within the defined scope. It does not, by itself, guarantee exactly-once delivery through an external provider.

### 3.4 Delivery Attempt

A Delivery Attempt represents an individual attempt to submit a notification through the provider integration.

A notification may have multiple delivery attempts if processing is retried.

A delivery attempt should conceptually capture:

* Its identifier and associated notification.
* The attempt sequence or number.
* The provider involved.
* The attempt's processing outcome.
* Relevant timestamps.
* Provider correlation information, when available.
* Failure classification and retry-related information, where applicable.

A delivery attempt is distinct from the notification itself. One notification may remain the same business request while multiple technical attempts are recorded.

The detailed attempt model must support investigation of failures and reconciliation of uncertain provider outcomes.

### 3.5 SMS Provider

The SMS Provider represents an external service through which the platform submits SMS messages and obtains submission or delivery-status information.

The domain must depend on provider-independent outcomes rather than a provider's SDK types or response codes.

Provider-specific details belong in infrastructure adapters. Mapping provider responses to domain outcomes must be explicit and testable.

## 4. Notification Lifecycle

### 4.1 Lifecycle States

The initial lifecycle uses the following states:

| State             | Meaning                                                                                                                 |
| ----------------- | ----------------------------------------------------------------------------------------------------------------------- |
| `ACCEPTED`        | The platform has accepted the request and durably recorded it for asynchronous processing.                              |
| `QUEUED`          | The notification is eligible and available for processing.                                                              |
| `PROCESSING`      | A processing attempt is actively handling the notification.                                                             |
| `SUBMITTED`       | The external provider has accepted the SMS submission. This does not establish final delivery.                          |
| `DELIVERED`       | The platform has received sufficient provider evidence to record successful delivery.                                   |
| `REJECTED`        | The platform did not accept the notification request.                                                                   |
| `FAILED`          | Processing or submission reached a failure outcome that requires no immediate further attempt, subject to retry policy. |
| `RETRY_SCHEDULED` | A retry has been scheduled following an eligible failure.                                                               |
| `DELIVERY_FAILED` | The provider has reported a delivery failure.                                                                           |

These states describe the notification's externally meaningful lifecycle. Technical details of individual attempts should be recorded separately where necessary.

### 4.2 Permitted Transitions

The initial permitted transitions are:

| Current state      | Next state        | Meaning                                                                    |
| ------------------ | ----------------- | -------------------------------------------------------------------------- |
| Request validation | `REJECTED`        | The request is invalid or not eligible for acceptance.                     |
| Request accepted   | `ACCEPTED`        | The platform has durably accepted the notification.                        |
| `ACCEPTED`         | `QUEUED`          | The notification becomes eligible for processing.                          |
| `QUEUED`           | `PROCESSING`      | A processing attempt begins.                                               |
| `PROCESSING`       | `SUBMITTED`       | The provider confirms acceptance of the SMS submission.                    |
| `SUBMITTED`        | `DELIVERED`       | Provider evidence confirms delivery.                                       |
| `SUBMITTED`        | `DELIVERY_FAILED` | Provider evidence confirms delivery failure.                               |
| `QUEUED`           | `FAILED`          | Processing cannot proceed and the notification reaches a failure outcome.  |
| `PROCESSING`       | `FAILED`          | Processing or submission fails according to the applicable failure policy. |
| `FAILED`           | `RETRY_SCHEDULED` | The failure is eligible for retry and a retry is scheduled.                |
| `RETRY_SCHEDULED`  | `QUEUED`          | The notification becomes eligible for another processing attempt.          |

This table defines the baseline, not an exhaustive implementation of every failure scenario. Additional transitions must be justified by explicit domain rules.

### 4.3 Lifecycle Rules

The lifecycle must enforce the following rules:

1. `DELIVERED` must not be assigned solely because a provider accepted a submission.
2. `REJECTED` represents failure to accept a request; it is distinct from failure during asynchronous processing.
3. `FAILED` and `DELIVERY_FAILED` have different meanings and must not be treated as interchangeable.
4. A notification must not transition to an arbitrary state merely because an API handler or provider adapter requests it.
5. State changes must be validated against the permitted transition rules.
6. Concurrent processing must not cause conflicting or invalid lifecycle transitions.
7. A retry must be associated with a defined retry policy and an identifiable processing attempt.
8. Terminal outcomes must not be silently overwritten by stale or out-of-order provider updates.
9. Where provider evidence is insufficient to determine the outcome, the platform must preserve that uncertainty rather than fabricate a definitive result.

The final API representation of lifecycle states must remain consistent with these domain semantics.

## 5. Domain Invariants

The following invariants apply throughout the platform.

### 5.1 Ownership

Every notification is owned by one tenant. Tenant ownership is established at acceptance and must not be reassigned through ordinary client operations.

### 5.2 Identity

Notification identifiers must uniquely identify notification records within the platform's defined identifier scheme.

### 5.3 Idempotent Acceptance

Within the defined tenant, operation, key, and retention scope, a repeated equivalent submission must not create a second accepted notification.

### 5.4 Lifecycle Integrity

Every persisted lifecycle transition must be permitted by the domain rules and must be applied in a concurrency-safe manner.

### 5.5 Durable Acceptance

An accepted notification must have durable state and a recoverable processing path. Acceptance must not depend solely on transient in-memory state.

### 5.6 Provider Independence

Domain rules must not depend on a particular SMS provider's SDK, response codes, or transport mechanism.

### 5.7 Outcome Accuracy

The platform must distinguish what it knows from what it cannot yet establish about submission and delivery outcomes.

### 5.8 Auditability

The platform must retain sufficient lifecycle and attempt information to investigate processing outcomes, subject to defined retention and data-protection policies.

## 6. Failure Classification and Retry Semantics

Failures must be classified according to their meaning and whether another attempt is safe and appropriate.

### 6.1 Non-Retryable Failures

Examples include invalid notification data and other conditions that cannot be corrected by repeating the same operation without a change in input or policy.

These failures should not trigger automatic retries unless a documented rule establishes otherwise.

### 6.2 Retryable Failures

Examples may include temporary provider unavailability, rate limiting, or transient infrastructure failures.

Retry behavior must be bounded by a documented policy covering eligibility, attempt limits, delay or backoff, and the final outcome when retries are exhausted.

### 6.3 Uncertain External Outcomes

A timeout or connection failure can occur after a provider has received or accepted a request but before the platform receives confirmation.

Such an outcome must not automatically be treated as proof that the provider did not accept the submission.

The architecture must define how uncertain attempts are reconciled and whether the provider supports idempotency, lookup, or another safe recovery mechanism. When certainty cannot be achieved, the platform must avoid presenting an unsupported definitive outcome.

### 6.4 Delivery Failures

A provider-reported delivery failure is distinct from a failure to submit a request.

Whether a delivery failure can trigger another submission depends on the provider's semantics and the platform's delivery policy. It must not be assumed that every delivery failure is safely retryable.

## 7. Domain Events and State Changes

Lifecycle transitions may produce domain events or equivalent recorded facts where these are useful for application coordination, auditability, or future integration.

Potential events include:

* `NotificationAccepted`
* `NotificationQueued`
* `NotificationProcessingStarted`
* `NotificationSubmitted`
* `NotificationDelivered`
* `NotificationFailed`
* `NotificationDeliveryFailed`
* `NotificationRetryScheduled`

These names are proposed domain concepts, not a commitment to a particular event-bus or messaging technology.

The implementation must distinguish domain events from external messages and provider callbacks. Reliable publication or processing of events that cross transaction boundaries must be addressed by the persistence and asynchronous-processing designs.

## 8. Separation of Domain and Infrastructure

The domain model must not depend on:

* Ktor request or response types.
* Database entities or ORM-specific types.
* A particular database product.
* Message broker APIs.
* SMS provider SDKs or response models.
* Framework-specific dependency injection mechanisms.

The application layer coordinates use cases and defines the required interfaces. Infrastructure adapters translate between external representations and the platform's domain concepts.

This separation allows business rules to be tested independently and infrastructure choices to evolve without rewriting the core domain model.

## 9. Open Questions

The following items require further analysis before implementation:

1. What exact data constitutes a materially different request when an idempotency key is reused?
2. What is the idempotency retention period, and what response is returned for a replay?
3. Which provider capabilities are available for reconciling uncertain submission outcomes?
4. What retry limits, backoff rules, and terminal-failure policies should V1 adopt?
5. How should duplicate, delayed, or out-of-order provider delivery-status updates be handled?
6. Which lifecycle and attempt data must be retained, and for how long?
7. Which domain events, if any, are necessary for V1 beyond persisted lifecycle state?

These decisions must be made in conjunction with the API, persistence, asynchronous-processing, and provider-integration designs.

## 10. Relationship to Other Architecture Artifacts

This document is derived from:

* `docs/requirements/requirements-baseline.md`
* `docs/architecture/architecture-drivers-and-principles.md`
* `docs/architecture/system-context.md`
* `docs/architecture/container-architecture.md`

It provides the domain foundation for:

* Application architecture and use cases.
* Persistence and concurrency rules.
* Asynchronous processing and retry design.
* API contracts and status representation.
* Security and tenant isolation.
* Reliability and operational investigation.

## 11. Status

**Status:** Proposed for architecture review.

The principal domain concepts, invariants, and baseline lifecycle transitions are defined. Detailed retry policies, idempotency semantics, provider reconciliation, and persistence mechanics remain subject to subsequent architecture decisions.
