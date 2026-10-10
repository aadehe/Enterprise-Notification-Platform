# ADR-004: Authentication and Credential Management

* **Status:** Proposed
* **Date:** 2026-10-10
* **Decision owners:** Enterprise Notification Platform project
* **Related documents:**

    * `docs/requirements/requirements-baseline.md`
    * `docs/architecture/architecture-baseline.md`
    * `docs/architecture/application-architecture.md`
    * `docs/architecture/security-architecture.md`
    * `docs/adr/ADR-001-initial-application-architecture.md`
    * `docs/adr/ADR-003-multi-tenancy-and-tenant-isolation.md`

## 1. Context

The Enterprise Notification Platform will expose a REST API for individual developers and enterprise customers. Clients must authenticate before accessing protected platform capabilities, and successful authentication must establish an identity and the tenant context used for authorization.

The initial product prioritizes programmatic integration. Human-facing dashboard and administrative capabilities will follow after the API foundation is stable.

The platform therefore needs a secure machine-to-machine authentication mechanism that supports credential issuance, verification, rotation, revocation, and tenant association. It must also distinguish authentication from authorization and avoid coupling domain logic to a particular authentication framework or identity provider.

Human authentication will become relevant when user-facing account management, organizational membership, and dashboard capabilities are introduced.

## 2. Decision Drivers

The decision is guided by the following considerations:

* Secure authentication for individual developers and enterprise integrations.
* Reliable association between credentials and authorized tenant identities.
* Secure credential storage, transmission, rotation, and revocation.
* Compatibility with tenant isolation and least-privilege authorization.
* A clear distinction between authentication and authorization.
* Support for credential lifecycle auditing and operational investigation.
* Compatibility with Clean Architecture and Ports & Adapters.
* Avoidance of unnecessary identity infrastructure during the initial release.
* A migration path toward standards-based human authentication when required.

## 3. Decision

The initial release will use **tenant-associated API keys for machine-to-machine authentication**.

API keys will be treated as secrets. The platform will verify credentials without storing their reusable plaintext values, and successful authentication will establish a trusted identity and associated tenant context for subsequent authorization.

The implementation will follow these principles:

1. **Credential-to-tenant association:** Every API key must be associated with a specific tenant and an identifiable credential record. A credential must not grant access to arbitrary tenants.

2. **Secure key generation:** API keys must be generated using a cryptographically secure random source. Their format should support distinguishing platform-issued credentials from unrelated values without exposing sensitive information.

3. **Protected storage:** Store a one-way cryptographic verifier or hash of each API key rather than the reusable plaintext secret. Select the verification approach based on the key's entropy and the security properties of the chosen algorithm. Never log or persist the plaintext key after issuance.

4. **One-time secret disclosure:** Reveal the complete API key only when it is created. Subsequent administrative views must not expose the original secret.

5. **Credential lifecycle:** Support credential creation, revocation, and replacement. The design should accommodate expiration and rotation policies without requiring changes to domain rules.

6. **Revocation enforcement:** Revoked credentials must no longer authorize new requests. Credential verification and revocation behavior must be defined consistently across application instances.

7. **Authentication before authorization:** Authenticate the presented credential before establishing trusted tenant context. Application use cases must then enforce the permissions and resource ownership applicable to that context.

8. **Least privilege:** Design credential permissions so that more restrictive access can be introduced where justified. The initial permission model must not grant platform-administration privileges to ordinary tenant credentials.

9. **Transport security:** Require HTTPS for API access in deployed environments. Credentials must be transmitted through an appropriate authentication header and must not appear in URLs, query parameters, or logs.

10. **Framework independence:** Keep credential verification and identity resolution behind application-facing abstractions where appropriate. Ktor-specific authentication mechanisms must not become dependencies of domain rules.

11. **Human authentication:** Defer implementation of human sign-in and session management until those capabilities are required. The architecture should permit integration with an OAuth 2.0/OpenID Connect identity provider without replacing the machine-to-machine authentication model.

12. **Auditable administration:** Record relevant credential lifecycle events, such as creation, revocation, and replacement, without recording the secret itself.

## 4. Alternatives Considered

### 4.1 Tenant-Associated API Keys

**Description:** Issue high-entropy secrets to developers or enterprise integrations. Associate each credential with a tenant and verify presented keys against protected credential records.

**Advantages:**

* Well suited to REST API integrations and automated clients.
* Straightforward to use from different programming languages and environments.
* Supports explicit tenant association and credential revocation.
* Avoids requiring a separate identity provider for initial machine access.
* Can be implemented within the existing application architecture.

**Disadvantages:**

* Secrets must be protected carefully in client environments.
* Rotation and revocation require explicit lifecycle support.
* Long-lived credentials can be exposed through compromised clients or logs.
* Fine-grained permissions require an explicit authorization model.

**Assessment:** Selected for the initial release because it directly supports the product's API-first requirements.

### 4.2 OAuth 2.0 Access Tokens for Machine Clients

**Description:** Authenticate machine clients through an authorization server and issue access tokens with defined scopes and lifetimes.

**Advantages:**

* Supports standardized token issuance and scope-based authorization.
* Can centralize client registration, token issuance, and policy management.
* May be appropriate for larger enterprise integrations or delegated access.

**Disadvantages:**

* Requires additional authorization-server capabilities or an external identity provider.
* Introduces token issuance, validation, expiry, and key-management considerations.
* Adds operational complexity that is not yet justified by the initial requirements.

**Assessment:** Retained as a future option if enterprise integration requirements justify the additional infrastructure.

### 4.3 Human Authentication Through OAuth 2.0/OpenID Connect

**Description:** Delegate human authentication to an identity provider using established protocols, with application-level authorization determining what authenticated users may do.

**Advantages:**

* Avoids building and maintaining a custom password and identity-management system.
* Supports established identity federation and sign-in flows.
* Can support future dashboard access and organizational membership.

**Disadvantages:**

* Requires careful handling of identity-provider configuration, token validation, account linking, and authorization.
* Introduces integration and operational responsibilities.
* Does not replace tenant authorization or machine-to-machine credential management.

**Assessment:** Preferred direction for future human-facing capabilities when required. It is not part of the initial machine-to-machine authentication implementation.

## 5. Consequences

### Positive Consequences

* Initial API clients have a clear authentication mechanism.
* Credentials have explicit tenant associations and lifecycle responsibilities.
* Plaintext API keys need not be retained by the platform.
* Authentication and tenant authorization remain separate concerns.
* Future human authentication can be introduced without coupling domain rules to a specific identity provider.

### Trade-offs and Constraints

* API keys are sensitive credentials and must be protected by both the platform and its clients.
* Secure issuance, storage, rotation, and revocation require deliberate implementation.
* The platform must ensure that revoked keys cannot continue authorizing requests through stale application state.
* The authorization model must define the permissions available to each credential.
* Human identity, organizational membership, and dashboard authorization remain future design work.

## 6. Implementation Guidance

The implementation should:

1. Define a credential record containing a stable identifier, tenant association, protected verifier, lifecycle status, creation metadata, and relevant lifecycle timestamps.
2. Generate API keys using a cryptographically secure random generator with sufficient entropy.
3. Use a documented key format and, where useful, a non-secret identifier or prefix to support efficient credential lookup.
4. Ensure the complete secret is displayed only at issuance and is never included in application logs, error messages, or ordinary database records.
5. Define how API keys are verified and how credential records are looked up efficiently.
6. Define revocation behavior and ensure credential status is checked consistently across application instances.
7. Establish a credential replacement workflow that permits clients to transition safely between keys.
8. Resolve the authenticated identity and tenant context before invoking protected application use cases.
9. Keep authentication failures generic enough to avoid unnecessarily disclosing whether a particular credential exists.
10. Apply appropriate request-rate controls and monitoring to detect repeated authentication failures and suspicious usage.
11. Ensure tenant authorization remains mandatory after authentication; possession of a valid key must not grant access outside its authorized tenant.
12. Test valid, invalid, expired, and revoked credentials, as well as attempts to access resources belonging to another tenant.
13. Audit credential lifecycle events without recording secret values.
14. Define the authorization and permission model before exposing sensitive administrative operations.
15. Evaluate OAuth 2.0/OpenID Connect integration when human-facing capabilities or enterprise identity requirements make it necessary.

The exact API-key format, permission granularity, expiration policy, and credential-verification implementation will be finalized during implementation and security review. These details must preserve the security principles established by this ADR.

## 7. Conditions for Reconsideration

This decision may be revisited if:

* Enterprise customers require federated machine authentication or standardized token-based integration.
* Credential lifecycle or authorization requirements exceed what the initial API-key model can reasonably support.
* Human-facing dashboard and administrative capabilities require sign-in, organizational membership, or delegated access.
* Security reviews identify weaknesses that justify a different credential-verification or authentication approach.
* Operational evidence supports introducing a centralized authorization service.

Any change must document its security properties, operational consequences, migration requirements, and effect on existing integrations.

## 8. References

* `docs/requirements/requirements-baseline.md`
* `docs/architecture/architecture-baseline.md`
* `docs/architecture/application-architecture.md`
* `docs/architecture/security-architecture.md`
* `docs/adr/ADR-001-initial-application-architecture.md`
* `docs/adr/ADR-003-multi-tenancy-and-tenant-isolation.md`

## 9. Status

**Proposed:** The decision is documented for review and has not yet been formally accepted.
