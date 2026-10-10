# ADR-003: Multi-Tenancy and Tenant Isolation

* **Status:** Proposed
* **Date:** 2026-10-10
* **Decision owners:** Enterprise Notification Platform project
* **Related documents:**

  * `docs/requirements/requirements-baseline.md`
  * `docs/architecture/architecture-baseline.md`
  * `docs/architecture/domain-model.md`
  * `docs/architecture/application-architecture.md`
  * `docs/architecture/persistence-architecture.md`
  * `docs/architecture/security-architecture.md`
  * `docs/adr/ADR-001-initial-application-architecture.md`
  * `docs/adr/ADR-002-asynchronous-processing-strategy.md`

## 1. Context

The Enterprise Notification Platform will serve individual developers and enterprise customers from the beginning. Multiple customers will use the same application and underlying persistence infrastructure, while their notification data, credentials, delivery records, and other tenant-owned resources must remain isolated.

A tenant represents a customer boundary within the platform. An individual developer may own a tenant, and an enterprise may have a tenant associated with its organization. The exact account, membership, and organization-management model can evolve without changing the fundamental isolation requirement.

Tenant identity must be established through authentication and authorization, not trusted directly from arbitrary client-supplied input. Every operation involving tenant-owned resources must enforce the applicable tenant boundary.

The initial architecture uses a modular monolith and a relational database. The isolation approach must provide strong security guarantees without introducing unnecessary infrastructure or coupling domain rules to a particular database feature.

## 2. Decision Drivers

The decision is guided by the following considerations:

* Prevent unauthorized cross-tenant access to data and operations.
* Establish tenant context from authenticated identity and credentials.
* Enforce tenant boundaries consistently across API requests and background processing.
* Support tenant-owned notifications, idempotency records, delivery attempts, and related resources.
* Preserve Clean Architecture and Ports & Adapters dependency rules.
* Make tenant isolation testable and observable.
* Minimize the risk of accidentally omitting tenant filters in data-access operations.
* Preserve the option to introduce additional database-level isolation controls where justified.

## 3. Decision

The initial release will use **application-enforced tenant isolation with tenant-scoped persistence operations and database-level integrity constraints**.

Tenant isolation will be treated as a security invariant that applies to every application entry point, including REST requests and background processing.

The implementation will follow these principles:

1. **Authenticated tenant context:** Resolve the tenant associated with the authenticated API credential or other authenticated identity. Client-supplied tenant identifiers must not independently establish or override the authorized tenant context.

2. **Explicit authorization:** Application use cases must operate within an established tenant context and verify that requested resources belong to that tenant before permitting access or modification.

3. **Tenant ownership in persistence:** Tenant-owned records must carry an appropriate `tenant_id` or an equivalent explicit ownership relationship. Database constraints and relationships must preserve ownership consistency.

4. **Tenant-scoped data access:** Persistence interfaces and implementations must support tenant-scoped operations. Queries for tenant-owned resources must apply the appropriate tenant boundary rather than relying on callers to remember an informal convention.

5. **Consistent enforcement across entry points:** API handlers, background workers, scheduled tasks, and internal processing paths must preserve the correct tenant context or explicitly use an authorized platform-level operation with a defined scope.

6. **No implicit privilege escalation:** Missing, invalid, or ambiguous tenant context must not cause an operation to run with unrestricted access. Platform-level operations must be explicitly authorized and narrowly scoped.

7. **Defense in depth:** Application authorization, persistence query constraints, database integrity constraints, and automated security tests must work together to reduce the likelihood and impact of isolation failures.

8. **Independent domain rules:** Domain logic must not depend on HTTP headers, API-key formats, framework-specific security objects, or database-specific isolation mechanisms.

9. **Additional database isolation controls:** Database row-level security may be evaluated as an additional defense after the database choice and connection-management approach are confirmed. It is not mandated for the initial release by this decision.

## 4. Alternatives Considered

### 4.1 Application-Enforced Tenant Isolation

**Description:** Establish tenant context through authentication, pass it through application use cases, and enforce tenant ownership in persistence operations and authorization checks.

**Advantages:**

* Fits the modular monolith and Clean Architecture approach.
* Keeps tenant authorization explicit and testable.
* Avoids requiring database-specific security features for the initial implementation.
* Supports multiple authentication mechanisms that resolve to the same tenant context.
* Keeps domain rules independent of the persistence technology.

**Disadvantages:**

* An incorrectly implemented query or authorization path can expose cross-tenant data.
* Developers must consistently preserve tenant context across application boundaries.
* Automated tests and code review are necessary to identify missing isolation checks.

**Assessment:** Selected as the initial enforcement model, with explicit tenant-scoped persistence operations, database integrity constraints, and defense-in-depth testing.

### 4.2 Database Row-Level Security

**Description:** Use database-enforced policies to restrict which tenant-owned records a database session or transaction can access.

**Advantages:**

* Can provide an additional protection layer if application queries omit tenant filters.
* Centralizes certain access restrictions within the database.
* Can help enforce isolation across multiple application paths.

**Disadvantages:**

* Introduces database-specific policies and operational considerations.
* Requires careful management of tenant context within database connections and transactions.
* Connection pooling, background processing, privileged roles, and administrative operations require deliberate configuration.
* Misconfigured policies or overly privileged database roles can undermine the intended protection.

**Assessment:** Retained as an option for additional defense in depth. Adoption will be evaluated against the selected database and actual implementation requirements rather than assumed in advance.

### 4.3 Separate Database or Schema per Tenant

**Description:** Isolate each tenant in a separate database or schema.

**Advantages:**

* Can offer stronger physical or logical separation for particular enterprise, compliance, or contractual requirements.
* May allow tenant-specific backup, restoration, or operational policies.

**Disadvantages:**

* Increases provisioning, migration, connection-management, and operational complexity.
* Makes a shared-service model more difficult to operate efficiently.
* Is not justified by the current requirements for the initial release.

**Assessment:** Not selected for the initial release. It may be reconsidered if future customer, regulatory, or contractual requirements justify the additional complexity.

## 5. Consequences

### Positive Consequences

* Tenant isolation becomes an explicit architectural invariant.
* Authentication, application use cases, and persistence operations have clear responsibilities.
* Tenant-owned data can be consistently scoped and audited.
* Automated tests can verify cross-tenant access restrictions.
* The architecture retains flexibility to add database-level isolation controls later.

### Trade-offs and Constraints

* Tenant context must be propagated correctly through application and background-processing boundaries.
* Persistence interfaces and queries require deliberate tenant-scoping design.
* Cross-tenant administrative or operational access requires explicit authorization and auditability.
* Application-enforced isolation depends on implementation discipline and comprehensive security testing.
* Any future row-level security implementation will require careful validation of connection, transaction, and database-role behavior.

## 6. Implementation Guidance

The implementation should:

1. Define a framework-independent tenant context representation suitable for use by application use cases.
2. Associate API credentials with an authorized tenant and resolve tenant context only after successful authentication.
3. Define how tenant membership and authorization will work if human or organization-based access is introduced.
4. Include tenant ownership in the data model for tenant-owned resources.
5. Design persistence interfaces so tenant-scoped access is explicit and difficult to bypass accidentally.
6. Ensure notification retrieval, status checks, idempotency handling, delivery attempts, and other tenant-owned operations enforce ownership consistently.
7. Ensure background processing derives the correct tenant association from persisted work rather than trusting unverified external input.
8. Define explicit, narrowly scoped authorization for platform-level operations that legitimately span tenants.
9. Avoid exposing the existence or details of another tenant's resources through API responses or error messages.
10. Add automated tests that attempt cross-tenant reads, updates, status checks, and other unauthorized operations.
11. Test behavior when tenant context is missing, invalid, or inconsistent with resource ownership.
12. Ensure logs, metrics, and traces support investigation without exposing credentials or unnecessary sensitive customer data.
13. Evaluate database row-level security after the database and connection-management design have been established.

The detailed tenant membership model, database-specific controls, and administrative access model may be refined as the authentication and persistence decisions are completed. Such refinements must preserve the isolation guarantees established by this ADR.

## 7. Conditions for Reconsideration

This decision may be revisited if:

* Enterprise contracts or regulatory obligations require stronger tenant-specific isolation.
* Security reviews identify risks that application-level enforcement alone does not adequately mitigate.
* The selected database provides an appropriate additional isolation mechanism whose operational costs are acceptable.
* Tenant-specific backup, restoration, residency, or encryption requirements justify separate data stores.
* The account and organization model evolves in ways that materially change authorization boundaries.

Any change must document its security guarantees, operational consequences, migration requirements, and effect on existing tenants.

## 8. References

* `docs/requirements/requirements-baseline.md`
* `docs/architecture/architecture-baseline.md`
* `docs/architecture/domain-model.md`
* `docs/architecture/application-architecture.md`
* `docs/architecture/persistence-architecture.md`
* `docs/architecture/security-architecture.md`
* `docs/adr/ADR-001-initial-application-architecture.md`
* `docs/adr/ADR-002-asynchronous-processing-strategy.md`

## 9. Status

**Accepted:** The decision is documented and has been formally accepted.
