# ADR-002: Asynchronous Notification Processing Strategy

* **Status:** Proposed
* **Date:** 2026-10-10
* **Decision owners:** Enterprise Notification Platform project
* **Related documents:**

    * `docs/requirements/requirements-baseline.md`
    * `docs/architecture/architecture-baseline.md`
    * `docs/architecture/container-architecture.md`
    * `docs/architecture/domain-model.md`
    * `docs/architecture/application-architecture.md`
    * `docs/architecture/persistence-architecture.md`
    * `docs/architecture/asynchronous-processing.md`
    * `docs/architecture/reliability-and-observability.md`
    * `docs/adr/ADR-001-initial-application-architecture.md`

## 1. Context

The Enterprise Notification Platform must accept notification requests through a REST API and process accepted notifications asynchronously.

The API must not report successful acceptance until the notification has been durably recorded and is eligible for subsequent processing.

The platform must recover from application restarts, transient infrastructure failures, and SMS provider failures without silently losing accepted notifications. It must also distinguish retryable failures from permanent failures and uncertain outcomes in which a provider may have accepted a request even though the platform did not receive a definitive response.

The initial architecture uses a modular monolith, with API handling and background processing running within the same deployable application. The processing mechanism must support that deployment model without preventing future evolution.

The main alternatives are database-backed work discovery, a transactional outbox with message-broker delivery, and direct publication to a message broker.

## 2. Decision Drivers

The decision is guided by the following considerations:

* Durable acceptance of notification requests.
* Recovery of work after process crashes and restarts.
* Safe coordination of concurrent processing attempts.
* Reliable persistence of notification state and processing outcomes.
* Explicit handling of retries and uncertain external outcomes.
* Compatibility with the modular monolith architecture.
* Manageable initial deployment and operational complexity.
* Testability of concurrency, failure, and recovery behavior.
* A migration path toward a broker if justified by measured requirements.

## 3. Decision

The initial release will use **database-backed work discovery** for asynchronous notification processing.

The relational database will be the durable source of truth for notification state and processing eligibility. Background processing will discover and claim eligible notifications from persisted records rather than relying on an in-memory queue as the sole record of pending work.

The processing mechanism will follow these principles:

1. **Durable acceptance:** Persist the notification and its initial processing state before returning HTTP `202 Accepted`.
2. **Recoverable work:** Pending work must remain discoverable after application restarts.
3. **Concurrency control:** Workers must claim work through a concurrency-safe mechanism so that multiple processing attempts do not unintentionally claim the same item.
4. **Bounded processing claims:** A claim must have a defined lifetime or recovery mechanism so abandoned work can become eligible for recovery.
5. **State consistency:** Persist notification state transitions and relevant attempt information using explicit transaction boundaries.
6. **Controlled retries:** Retry eligible failures according to a bounded policy, with appropriate delay and terminal failure handling.
7. **External outcome uncertainty:** Do not assume that a timeout means the SMS provider did not accept the request. Record and handle uncertain outcomes explicitly.
8. **Separation of responsibilities:** Keep work discovery, processing orchestration, and provider communication behind appropriate application and infrastructure boundaries.

The database-backed mechanism will run within the initial deployable application. A dedicated message broker is not required for the first release.

The implementation must not claim exactly-once SMS delivery. Database transactions can protect platform state, but they cannot atomically commit a transaction together with an external SMS provider's actions.

## 4. Alternatives Considered

### 4.1 Database-Backed Work Discovery

**Description:** Persist notifications and their processing state in the relational database. Background workers discover eligible records and claim them using a concurrency-safe mechanism.

**Advantages:**

* Uses the database as the durable source of truth.
* Avoids introducing a separate broker during the initial release.
* Supports transactional persistence of notification state and processing eligibility.
* Simplifies local development, deployment, and initial operations.
* Makes pending, claimed, completed, and failed work inspectable through persisted state.

**Disadvantages:**

* Polling introduces a trade-off between processing latency and database load.
* Work discovery and claiming require careful concurrency control.
* Large backlogs or high polling frequency can create database contention.
* Worker recovery, claim expiry, and retry scheduling must be implemented correctly.

**Assessment:** Selected for the initial release because it fits the current deployment model and avoids additional infrastructure while preserving durable processing.

### 4.2 Transactional Outbox with a Message Broker

**Description:** Persist notification state and an outbox record in the same database transaction. A separate publisher delivers outbox records to a message broker, from which consumers process the work.

**Advantages:**

* Avoids the database-commit/message-publication dual-write gap when implemented correctly.
* Supports broker-based work distribution and consumer coordination.
* Provides a clearer path toward independently scalable processing components.
* Can be suitable when throughput, latency, or workload isolation requirements justify a broker.

**Disadvantages:**

* Introduces additional infrastructure and operational responsibilities.
* Requires outbox publication, duplicate-publication handling, and consumer idempotency.
* Broker delivery does not eliminate uncertainty around external provider outcomes.
* Adds deployment and failure-recovery scenarios that are not yet necessary for the initial release.

**Assessment:** A credible future option if the platform needs broker-based distribution. It is not required for the initial implementation.

### 4.3 Direct Message-Broker Publication

**Description:** The API writes the notification to the database and separately publishes a message to a broker for processing.

**Advantages:**

* Enables asynchronous distribution through established messaging infrastructure.
* Can support high-throughput consumers and independent processing capacity.

**Disadvantages:**

* A database commit can succeed while message publication fails, or publication can succeed while the database transaction fails.
* Requires additional coordination or recovery mechanisms to prevent accepted work from becoming undiscoverable or inconsistently represented.
* Introduces broker operations and failure modes immediately.

**Assessment:** Not selected as the initial approach because direct publication alone does not adequately address the consistency gap between database persistence and message publication.

## 5. Consequences

### Positive Consequences

* Accepted notifications remain represented in durable storage.
* Application restarts do not inherently lose pending work.
* The initial deployment avoids requiring a separate message broker.
* Notification state and processing progress can be inspected and tested.
* The design retains a path toward broker-based processing if future evidence supports it.

### Trade-offs and Constraints

* Polling intervals and query efficiency will influence processing latency and database load.
* Concurrent work claiming requires deliberate transaction and locking design.
* Expired claims and abandoned processing attempts require recovery rules.
* Database-backed work discovery may become unsuitable as throughput or isolation requirements grow.
* Uncertain provider outcomes remain a separate reliability problem; changing the queueing mechanism does not eliminate them.
* The application must protect API responsiveness from background-processing resource contention.

## 6. Implementation Guidance

The implementation should:

1. Persist a notification and its initial processing state before acknowledging acceptance.
2. Define the database transaction boundary for acceptance and idempotency handling.
3. Use a concurrency-safe claim mechanism appropriate to the selected relational database.
4. Define how claimed work is represented, how claim expiry is detected, and how abandoned work is recovered.
5. Keep provider calls outside long-running database transactions.
6. Persist processing outcomes and delivery-attempt information with clear state-transition rules.
7. Distinguish retryable failures, permanent failures, and uncertain provider outcomes.
8. Apply bounded retries with appropriate delays and an explicit terminal failure policy.
9. Make processing safe against repeated execution wherever possible, while recognizing that external delivery may still be duplicated.
10. Test process crashes, concurrent claims, database failures, provider timeouts, and recovery scenarios.
11. Measure polling latency, backlog size, processing duration, claim expiry, retry rates, and database load.
12. Keep the work-discovery mechanism behind an appropriate application or infrastructure boundary so that future changes do not unnecessarily affect domain rules.

The precise SQL strategy, indexes, polling interval, claim duration, and retry parameters will be defined during implementation and validated through testing.

## 7. Conditions for Reconsideration

This decision may be revisited when observed evidence demonstrates that database-backed work discovery no longer meets the platform's requirements, including:

* Sustained database load from polling materially affects API or persistence performance.
* Required processing latency cannot be met with an acceptable polling and database-load trade-off.
* Backlog size or throughput requires a different work-distribution model.
* Background processing requires independent deployment or stronger resource isolation.
* Operational requirements justify broker-based buffering and consumer coordination.

Any reconsideration must compare viable alternatives, including a transactional outbox with a message broker, and document migration risks, consistency guarantees, operational costs, and recovery behavior.

Anticipated future scale alone is not sufficient justification.

## 8. References

* `docs/requirements/requirements-baseline.md`
* `docs/architecture/architecture-baseline.md`
* `docs/architecture/container-architecture.md`
* `docs/architecture/domain-model.md`
* `docs/architecture/application-architecture.md`
* `docs/architecture/persistence-architecture.md`
* `docs/architecture/asynchronous-processing.md`
* `docs/architecture/reliability-and-observability.md`
* `docs/adr/ADR-001-initial-application-architecture.md`

## 9. Status

**Proposed:** The decision is documented for review and has not yet been formally accepted.
