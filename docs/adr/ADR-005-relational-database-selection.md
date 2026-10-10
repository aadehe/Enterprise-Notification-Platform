# ADR-005: Relational Database Selection

* **Status:** Proposed
* **Date:** 2026-10-10
* **Decision owners:** Enterprise Notification Platform project
* **Related documents:**

    * `docs/requirements/requirements-baseline.md`
    * `docs/architecture/architecture-baseline.md`
    * `docs/architecture/persistence-architecture.md`
    * `docs/architecture/asynchronous-processing.md`
    * `docs/adr/ADR-002-asynchronous-processing-strategy.md`
    * `docs/adr/ADR-003-multi-tenancy-and-tenant-isolation.md`

## 1. Context

The Enterprise Notification Platform requires durable persistence for tenants, credentials, notifications, idempotency records, delivery attempts, and other operational data.

Persistence must support reliable notification acceptance, transaction-safe state changes, tenant isolation, concurrent background processing, recovery after application restarts, and efficient retrieval of notification status.

The initial architecture uses a modular monolith with explicit persistence boundaries. The selected database should support these requirements while keeping infrastructure and operational complexity proportionate to the initial product.

## 2. Decision Drivers

The decision is guided by the following considerations:

* Transactional consistency for notification acceptance and related records.
* Durable storage and recovery after application or infrastructure failures.
* Concurrency control for idempotency and background work claiming.
* Relational integrity between tenants and their owned resources.
* Efficient querying and indexing for notification status and operational workloads.
* Support for schema migrations and automated integration testing.
* Compatibility with Kotlin and the planned application architecture.
* Operational maturity, documentation, and availability of managed hosting options.
* Flexibility to introduce additional infrastructure only when justified.

## 3. Decision

**PostgreSQL is the recommended primary relational database for the initial release.** This ADR remains proposed until the decision is formally reviewed and accepted.

PostgreSQL will serve as the authoritative persistence store for core platform records, including notification lifecycle state and the records needed to recover asynchronous work.

The implementation will follow these principles:

1. **Transactional consistency:** Use database transactions to protect related changes that must succeed or fail together.

2. **Durable acceptance:** Persist an accepted notification and the required processing state before returning `202 Accepted` to the API client.

3. **Concurrency safety:** Use appropriate database constraints, transactions, and concurrency-control mechanisms to protect idempotency and work claiming.

4. **Relational integrity:** Define primary keys, foreign keys, uniqueness constraints, and other appropriate constraints to preserve valid relationships between tenants and their resources.

5. **Tenant isolation:** Include explicit tenant ownership and tenant-scoped access patterns for tenant-owned records. Database integrity constraints must support the ownership model.

6. **Explicit persistence boundaries:** Access the database through infrastructure adapters and application-facing abstractions consistent with Clean Architecture and Ports & Adapters.

7. **Managed schema evolution:** Use versioned database migrations and validate them through automated tests.

8. **Recoverable processing:** Store sufficient state to identify eligible work and recover records left in an incomplete processing state after failures.

9. **Bounded operational complexity:** Start with one primary relational database. Additional data stores must have a documented justification.

10. **Database portability at appropriate boundaries:** Keep database-specific behavior inside persistence adapters where practical. Do not pursue complete database independence at the expense of correctness or useful database capabilities.

11. **Security:** Use authenticated database connections, least-privilege database roles, protected credentials, encrypted connections where applicable, and appropriate backup and restoration controls.

12. **Measured optimization:** Introduce specialized indexing, partitioning, replicas, or other scaling mechanisms when workload measurements and operational requirements justify them.

## 4. Alternatives Considered

### 4.1 PostgreSQL

**Description:** Use PostgreSQL as the primary relational database for core platform persistence.

**Advantages:**

* Provides transactions, relational integrity constraints, and mature concurrency-control capabilities.
* Supports uniqueness constraints and transaction patterns useful for idempotency.
* Offers mechanisms suitable for coordinating concurrent background work.
* Supports indexes, structured queries, and schema migrations.
* Provides options for database-level security controls, including row-level security.
* Has a mature ecosystem and managed deployment options.

**Disadvantages:**

* Requires database operations, backup, restoration, monitoring, and migration discipline.
* Database-specific features may introduce coupling when used outside persistence boundaries.
* Poorly designed transactions, queries, or indexes can create contention and performance problems.

**Assessment:** Recommended for the initial release because it satisfies the platform's transactional and relational requirements without requiring additional database technologies.

### 4.2 MySQL

**Description:** Use MySQL as the primary relational database.

**Advantages:**

* Mature relational database technology with transactional storage engines.
* Broad ecosystem and managed hosting availability.
* Supports relational integrity, indexing, and transactional application workloads.

**Disadvantages:**

* Provides a different set of database-specific features and operational trade-offs.
* Selecting it would require validating the planned concurrency, query, and integrity patterns against its behavior.

**Assessment:** Viable, but PostgreSQL is the preferred candidate for this project. Introducing support for multiple primary databases is not justified by the current requirements.

### 4.3 NoSQL Database as the Primary Store

**Description:** Use a document or key-value database as the primary persistence technology.

**Advantages:**

* Can be appropriate for particular access patterns and large-scale distributed workloads.
* Some products provide flexible data models or specialized scaling capabilities.

**Disadvantages:**

* The platform has important relationships and consistency requirements across tenants, notifications, idempotency records, and delivery attempts.
* The selected data model and transaction capabilities would need careful validation against these requirements.
* A NoSQL-first approach does not provide a clear initial advantage sufficient to justify departing from the relational model.

**Assessment:** Not selected for the initial release. Specialized storage may be evaluated later if concrete workload requirements justify it.

## 5. Consequences

### Positive Consequences

* Core records have a single authoritative persistence system.
* Transactional operations can protect important state changes.
* Relational constraints can help enforce ownership and data integrity.
* Database-backed asynchronous work discovery can be implemented without requiring a separate message broker.
* The project can establish a consistent approach to migrations, integration tests, backups, and recovery.

### Trade-offs and Constraints

* Database availability and performance are important to API acceptance and background processing.
* Transaction boundaries and concurrent operations require careful design.
* Database migrations must preserve existing data and support safe deployment procedures.
* Backup and restoration procedures must be validated rather than assumed to work.
* Database-specific features should remain within documented infrastructure boundaries unless their use elsewhere is explicitly justified.

## 6. Implementation Guidance

The implementation should:

1. Configure PostgreSQL through externalized application configuration and protected credentials.
2. Use a migration tool to manage versioned schema changes.
3. Define relational tables and constraints for tenant ownership, notification state, idempotency, and delivery attempts.
4. Establish transaction boundaries around operations that require atomic database changes.
5. Implement concurrency-safe idempotency enforcement using database constraints and appropriate transactional logic.
6. Implement background work claiming and recovery without holding database transactions open during external provider calls.
7. Design indexes from expected access patterns and validate them using query plans and measurements.
8. Ensure persistence queries consistently enforce tenant-scoped access.
9. Use separate least-privilege database roles where appropriate for application operations and administrative tasks.
10. Add integration tests against a real PostgreSQL instance for constraints, transactions, migrations, concurrency, and recovery behavior.
11. Define backup, restoration, connection-management, and database monitoring procedures before production deployment.
12. Avoid adding a separate cache, search engine, or secondary database until a specific requirement demonstrates its value.
13. Document database-specific concurrency and query behavior where application correctness depends on it.

The exact schema, isolation levels, indexing strategy, migration tool, and deployment model will be finalized during implementation. These details must preserve the architectural guarantees established by this decision.

## 7. Conditions for Reconsideration

This decision may be revisited if:

* Measured workload characteristics demonstrate that PostgreSQL cannot satisfy established performance or availability objectives.
* A specific product requirement is materially better served by another persistence technology.
* Regulatory, contractual, or operational requirements demand a different data-storage model.
* Experience from implementation and testing reveals a substantial mismatch between the proposed data model and the selected database.

Any proposed replacement must document its effect on consistency, concurrency, tenant isolation, operational complexity, migration, and recovery.

## 8. References

* `docs/requirements/requirements-baseline.md`
* `docs/architecture/architecture-baseline.md`
* `docs/architecture/persistence-architecture.md`
* `docs/architecture/asynchronous-processing.md`
* `docs/adr/ADR-002-asynchronous-processing-strategy.md`
* `docs/adr/ADR-003-multi-tenancy-and-tenant-isolation.md`

## 9. Status

**Proposed:** PostgreSQL is recommended for the initial release. Formal acceptance remains pending architecture review.
