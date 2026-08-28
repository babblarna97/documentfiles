PE-SPEC-11: Integration Authentication & Authorization
1. DOCUMENT CONTROL
| Attribute | Value |
|---|---|
| Document Title | 11 Integration Authentication & Authorization.md |
| Document ID | PE-SPEC-11 |
| Version | 1.0.1 |
| Status | DRAFT / Implementation Specification |
| Author | Ramy Bella |
| Classification | Confidential / Enterprise Proprietary |
| Target Audience | Security Architects, Integration Architects, IAM Engineers, Backend Engineers, Platform Engineers, DevSecOps, QA Architects, Reliability Engineers |
| Parent Document | PE-SPEC-10 |
| Related Documents | PE-SPEC-01 through PE-SPEC-10, PE-SPEC-12 through PE-SPEC-20, Phase 4 Specifications |
| System | Restaurant AI System |
| Phase | Phase 5 — Integrations |
| Lifecycle Folder | SECURITY & DATA |
| Last Updated | August 2026 |
2. EXECUTIVE PURPOSE
The Restaurant AI System interfaces with a vast ecosystem of LLM providers, booking engines, email dispatchers, CRM platforms, internal services, webhooks, and edge clients. PE-SPEC-11 defines the enterprise architecture governing the technical authentication and authorization mechanisms for all actors communicating with or through the Phase 5 Integration Layer.
This specification enforces how identities are proven, how technical permissions are evaluated, how credentials and tokens are scoped, how authorization context is cryptographically propagated, and how unauthorized actions are deterministically blocked.
Core Invariant:
IDENTITY PROOF \neq TECHNICAL AUTHORIZATION \neq BUSINESS AUTHORITY
A valid API key, OAuth token, JWT, mTLS certificate, service identity, or authenticated session proves who or what is communicating. It does NOT automatically grant permission to perform an operation, nor does it define business policy. Phase 3 remains the ultimate, unconditional authority for business authorization, user intent, business policy, state transitions, and business meaning.
3. PURPOSE AND SCOPE
Scope: What PE-SPEC-11 Controls
 * Integration Actor Authentication: Mechanisms for verifying external and internal identities.
 * API Key Boundaries: Secure injection, validation, and usage rules.
 * OAuth 2.x Provider Authentication: Token lifecycles, client credentials, and refresh handling.
 * Service-to-Service Authentication: Machine identities and STS (Security Token Service) exchanges.
 * JWT Validation: Signature, issuer, algorithm, and audience enforcement.
 * mTLS Identity Authentication: Certificate chain and subject validation.
 * Credential Scope & Audience: Enforcing token boundaries.
 * Token Expiration & Rotation: TTL limits, lifecycle monitoring, and zero-downtime rotation.
 * Technical Authorization Enforcement: Capability-level permission checks (e.g., verifying a token has the BOOKING_CREATE scope).
 * Provider Credential Scoping: Ensuring credentials are bound to explicit environments and tenants.
 * Request Authorization Context: Secure propagation of identity claims.
 * Least-Privilege Enforcement: Restricting actors to the minimum required technical scopes.
 * Replay Protection & Clock Skew: JTI tracking, nonce validation, and strict timing bounds.
 * Authentication & Authorization Failure Taxonomy: Canonical mapping of IAM failures.
Scope: What PE-SPEC-11 Explicitly Does NOT Control
 * Master integration architecture \rightarrow PE-SPEC-01
 * Canonical contract structure \rightarrow PE-SPEC-02
 * Provider abstraction \rightarrow PE-SPEC-03
 * Provider-specific implementations \rightarrow PE-SPEC-04 through PE-SPEC-09
 * Integration security perimeter / SSRF / Transport security \rightarrow PE-SPEC-10
 * Data mapping and minimization \rightarrow PE-SPEC-12
 * Tenant/environment isolation \rightarrow PE-SPEC-13
 * Error recovery \rightarrow PE-SPEC-14
 * Retry/idempotency orchestration \rightarrow PE-SPEC-15
 * Webhooks/events \rightarrow PE-SPEC-16
 * Rate limits/resilience \rightarrow PE-SPEC-17
 * Observability/audit \rightarrow PE-SPEC-18
 * Testing/certification methodology \rightarrow PE-SPEC-19
 * Lifecycle/registry \rightarrow PE-SPEC-20
 * Business authorization and business policy \rightarrow Phase 3
Important: PE-SPEC-11 may enforce technical authorization prerequisites supplied by Phase 3, but it MUST NOT invent business authorization policy.
4. AUTHORITY MODEL
System authorization flows in a strict hierarchical order. The network is secured by PE-SPEC-10, the technical identity and capability execution rights are governed by PE-SPEC-11, but the decision of whether an action should happen belongs entirely to Phase 3.
[PHASE 3 / BUSINESS AUTHORITY] (Evaluates business rules & user intent)
        ↓
[TECHNICAL AUTHORIZATION CONTEXT] (Assembles scoped claims)
        ↓
[PE-SPEC-11 / AUTHENTICATION & AUTHORIZATION] (Validates identity & technical capability rights)
        ↓
[PE-SPEC-10 / SECURITY PERIMETER] (Enforces transport, SSRF & payload safety)
        ↓
[PE-SPEC-03 / PROVIDER ABSTRACTION] (Routes to target provider)
        ↓
[PROVIDER ADAPTER] (Translates & applies technical credentials)
        ↓
[EXTERNAL PROVIDER] (Executes the operation)

Architectural Boundaries:
 * Phase 3 says whether the user/business action is authorized.
 * PE-SPEC-11 validates technical identity and ensures the execution context holds the technical permission to invoke the provider.
 * PE-SPEC-10 protects the network and security perimeter.
 * Provider adapters perform technical translation.
 * Providers execute tasks; they do not become system authorities.
5. AUTHENTICATION MECHANISMS
PE-SPEC-11 defines the supported technical methods for establishing identity within the Phase 5 layer.
5.1. JSON Web Tokens (JWT)
When JWTs are used for incoming requests (e.g., from edge clients, Phase 3, or webhooks), validation MUST be deterministic:
 * Signature Verification: Must use explicitly approved algorithms (e.g., RS256, ES256). The none algorithm MUST be deterministically rejected.
 * Issuer (iss) & Audience (aud): Must strictly match expected registered values.
 * Expiration (exp) & Not Before (nbf): Time-based claims must be strictly enforced.
 * Clock Skew: A maximum acceptable clock skew (e.g., 5 minutes) must be configured to prevent acceptance of wildly out-of-sync tokens.
5.2. OAuth 2.x & Machine-to-Machine (M2M)
For interactions with providers requiring OAuth (e.g., Microsoft Graph, Salesforce):
 * Tokens MUST be obtained through an explicitly approved OAuth flow appropriate to the integration, with all token acquisition and refresh operations performed through the authorized server-side authentication mechanism.
 * Refresh tokens MUST be protected by a secure, centralized token manager.
 * Access tokens MUST NEVER be exposed to the LLM, the frontend widget (PE-SPEC-07), or written to application logs or telemetry.
5.3. API Keys
 * API keys MUST be retrieved from an authorized Secret Manager at runtime.
 * API keys MUST be injected into integration requests out-of-band by the adapter.
 * API keys MUST NOT be utilized as business authorization identities; they solely represent the system's technical identity to the provider.
5.4. Mutual TLS (mTLS)
Where mTLS is configured:
 * The adapter MUST validate the Certificate Authority (CA) chain.
 * The subject Alternative Name (SAN) MUST be verified against the explicitly registered provider identity.
6. TECHNICAL AUTHORIZATION ENFORCEMENT
Proving identity is insufficient. The system MUST evaluate if the authenticated actor possesses the technical rights to invoke the requested capability.
6.1. Capability-Level Permission Checks
Every Phase 5 capability (e.g., BOOKING_CREATE, EMAIL_SEND, CRM_CONTACT_UPDATE) must be protected by a technical authorization gate.
 * The execution context MUST present a valid claim or token scope mapping to the requested capability.
 * If the token lacks the scope, the request MUST fail closed (ERR_AUTHZ_01), regardless of whether Phase 3 requested it.
6.2. Tenant and Provider Scoping
 * Credentials and authorization contexts MUST be bound to a specific tenant_scope and environment.
 * Ownership Clarity: PE-SPEC-11 authenticates the actor and validates that the authorization context is compatible with the requested technical operation. PE-SPEC-13 owns the detailed tenant/environment isolation policy and enforcement boundary. The two specifications MUST work together.
 * A token granting BOOKING_CREATE for tenant_A MUST deterministically fail if the request attempts to load provider credentials for tenant_B.
6.3. Least-Privilege Enforcement
Integration actors (including internal background workers and external automation triggers) MUST be granted the narrowest possible technical scope. Universal "admin" tokens traversing the Phase 5 layer are strictly prohibited.
7. AUTHORIZATION CONTEXT PROPAGATION
Identity and scope must traverse distributed system boundaries securely.
Rules for Context Propagation:
 * The system MUST utilize a cryptographically secure, tamper-evident object (e.g., a signed contextual JWT or internal secure context object) to propagate identity from Phase 3 into Phase 5.
 * No Blind Trust: Claims originating from untrusted clients (e.g., HTTP headers like X-User-Role: Admin sent by a browser) MUST be stripped or ignored. The context MUST be assembled and signed server-side.
 * The authorization context MUST be bound to the correlation_id to ensure auditability across the lifecycle of the request.
8. CREDENTIAL LIFECYCLE & ROTATION
Provider credentials degrade over time and must be actively managed.
 * Retrieval: Integration adapters MUST pull secrets at execution time (or cache them securely in memory with a strict TTL) from the Secret Manager.
 * Zero-Downtime Rotation: The architecture MUST support overlapping credential versions (e.g., primary and secondary keys) to allow seamless rotation of provider API keys without dropping requests.
 * Revocation Handling: If a provider signals that an API key or token is revoked/invalid, the system MUST immediately flush the credential from active memory caches and transition the adapter to a FAILED state until a new valid secret is supplied.
 * Expiration Monitoring: The system SHOULD monitor credential expiration and scheduled rotation windows and SHOULD generate operational signals before expiration where technically supported. Credentials approaching expiration MUST NOT silently fail into runtime behavior without observability. Where provider APIs expose expiration metadata, that metadata SHOULD be tracked through the governed credential lifecycle. Observability of these events remains governed by PE-SPEC-18.
9. TOKEN REPLAY & TIMING PROTECTION
Stolen or intercepted tokens pose a critical threat if replayed. Replay prevention mechanisms depend on the credential/request type and threat level.
 * JTI / Nonce Tracking: Where the credential or operation requires replay protection (e.g., single-use tokens, high-privilege operations, signed requests, or webhook authentication), the system MUST validate applicable JTI, nonce, request identifier, timestamp, or equivalent replay-control metadata against the approved replay policy.
 * Strict Time Bounds: Tokens with excessively long exp (Expiration) claims SHOULD be rejected.
 * Event Replay: Webhook and event authentication MUST include timestamp validation to ensure payloads are not replayed hours or days later (PE-SPEC-16 owns the detailed event integrity boundaries).
10. FAILURE ARCHITECTURE
Authentication and Authorization failures map deterministically to the canonical Phase 5 error classes. They MUST instantly halt execution and FAIL CLOSED.
Taxonomy Invariant:
AUTHENTICATION FAILURE \neq AUTHORIZATION FAILURE \neq TENANT / ENVIRONMENT VIOLATION \neq BUSINESS DENIAL
| Failure ID | Condition | Canonical Error Class | Severity |
|---|---|---|---|
| ERR_AUTH_01 | Missing, malformed, or unparseable token/key | AUTHENTICATION | Critical |
| ERR_AUTH_02 | Token signature validation failed | AUTHENTICATION | Critical |
| ERR_AUTH_03 | Token expired (exp) or used before active (nbf) | AUTHENTICATION | Critical |
| ERR_AUTH_04 | Issuer (iss) or Audience (aud) mismatch | AUTHENTICATION | Critical |
| ERR_AUTH_05 | Prohibited signature algorithm detected (e.g., none) | AUTHENTICATION | Critical |
| ERR_AUTH_06 | External provider rejected injected credentials | AUTHENTICATION | Critical |
| ERR_AUTHZ_01 | Context lacks required capability scope | AUTHORIZATION | Critical |
| ERR_AUTHZ_02 | Context attempts cross-tenant access or env mismatch | TENANT_VIOLATION | Critical |
| ERR_AUTHZ_03 | Token replay detected (JTI/Nonce collision) | AUTHORIZATION | Critical |
| ERR_AUTHZ_04 | Clock skew outside acceptable bounds | AUTHENTICATION | High |
Note: Canonical error classes are separated from result semantics. An IAM or Tenant failure results in an execution state of FAILED or REJECTED, not UNKNOWN.
11. SECURITY / THREAT MODEL
| Threat | Attack Surface | Preventive Control | Detection | Response | Owner | Severity |
|---|---|---|---|---|---|---|
| Token Forgery | Inbound Requests | Strict signature & algorithm validation; Reject alg: none | Token Validator | ERR_AUTH_02 | SecOps | Critical |
| Token Replay | Async / Webhooks | Strict metadata tracking based on replay policy; Clock-skew enforcement | Replay Cache | ERR_AUTHZ_03 | Arch | Critical |
| Audience Spoofing | Token Validation | Strict verification of aud against requested environment | Token Validator | ERR_AUTH_04 | SecOps | Critical |
| Privilege Escalation | Capability Exec | Least-privilege capability mapping; Server-side contexts only | Authz Gate | ERR_AUTHZ_01 | Platform | Critical |
| Cross-Tenant Access | Authz Context | Context verification matched to environment scope | Scope Validator | ERR_AUTHZ_02 | Arch | Critical |
| Credential Stagnation | Provider Adapters | Rotation policies; Active expiration monitoring | Secret Manager | Map ERR_AUTH_06 | SecOps | High |
12. PRODUCTION ACCEPTANCE CRITERIA
| AC ID | Category | Requirement | Verification Method | Expected Result | Pass/Fail | Severity |
|---|---|---|---|---|---|---|
| AC-IAM-01 | Algorithm Enf. | Submitting a token signed with the none algorithm deterministically fails. | Token Mock Test | ERR_AUTH_05 | Required | Critical |
| AC-IAM-02 | Time Bounds | Submitting an otherwise valid token with an exp claim in the past fails authentication. | Expiration Test | ERR_AUTH_03 | Required | Critical |
| AC-IAM-03 | Audience | A token minted for the staging audience fails authorization if submitted to a production endpoint. | Audience Mock | ERR_AUTH_04 | Required | Critical |
| AC-IAM-04 | Capability | A validly authenticated service account lacking the BOOKING_CREATE scope is blocked from executing the integration. | Authz Gate Test | ERR_AUTHZ_01 | Required | Critical |
| AC-IAM-05 | Credential Iso. | The adapter successfully retrieves and utilizes a provider API key without exposing it to the canonical request/response payloads or telemetry. | Payload Audit | Secrets Absent | Required | Critical |
| AC-IAM-06 | Cross-Tenant | A token issued for tenant_A attempting to load provider credentials for tenant_B fails closed. | Context Mock | ERR_AUTHZ_02 (TENANT_VIOLATION) | Required | Critical |
| AC-IAM-07 | Replay Prot. | Submitting an identical single-use token or protected webhook event twice fails authorization based on the configured replay policy. | Replay Simulation | ERR_AUTHZ_03 | Required | High |
| AC-IAM-08 | Business Sep. | An authenticated technical request is not permitted to mutate business state if Phase 3 validation rejects the underlying intent. | Arch Review | Intent Blocked | Required | Critical |
13. INTEGRATION AUTHORITY MATRIX
| Component | Owns | Consumes | Produces | Must Not Override |
|---|---|---|---|---|
| Phase 3 | Business Authz & Policy | Integration Results | Business Commands | Technical Identity Rules |
| PE-SPEC-10 | Security Perimeter (SSRF/TLS) | Outbound Network | Secured Transit | IAM Evaluation Logic |
| PE-SPEC-11 | IAM & Context Validation | Identity Claims | Authz Contexts | Phase 3 Business Rules |
| PE-SPEC-13 | Tenant / Environment Isolation | Evaluated Contexts | Isolation Boundaries | IAM Actor Validation |
| Provider Adapter | Provider Credential Usage | Authz Context | Authorized Calls | Token Validation Policies |
| Secret Manager | Credential Storage/Rotation | Adapter Requests | Provider Secrets | Adapter Routing |
14. INTEGRATION CONTRACTS / RELATIONSHIPS
PE-SPEC-11 operates as the Identity and Access Management (IAM) and technical authorization authority for Phase 5.
 * PE-SPEC-01/02/03: Defines the integration capabilities that PE-SPEC-11 enforces permissions against.
 * PE-SPEC-10 (Security): Relies on PE-SPEC-11 to handle the specific token semantics (JWT, OAuth) so PE-SPEC-10 can focus purely on network borders, SSRF, and TLS enforcement.
 * PE-SPEC-04–09 (Adapters): Depend on PE-SPEC-11 to provide the mechanisms for retrieving out-of-band secrets safely.
 * PE-SPEC-12 (Data Mapping): Assumes PE-SPEC-11 has already proven the identity before semantic mapping begins.
 * PE-SPEC-13 (Isolation): Owns the detailed tenant/environment isolation policy and enforcement boundary, utilizing the identity context authenticated by PE-SPEC-11.
 * PE-SPEC-16 (Webhooks): Uses PE-SPEC-11 signature and timing validation rules to process incoming provider events.
 * PE-SPEC-18 (Observability): Relies on PE-SPEC-11 ensuring that credentials are kept out of context properties that are forwarded to telemetry.
15. FINAL NON-NEGOTIABLE PRINCIPLES
 * PHASE 3 REMAINS THE ULTIMATE BUSINESS AUTHORITY.
 * IDENTITY PROOF DOES NOT EQUAL TECHNICAL AUTHORIZATION.
 * TECHNICAL AUTHORIZATION DOES NOT EQUAL BUSINESS AUTHORITY.
 * API KEYS AND SECRETS MUST NEVER BE LOGGED, ECHOED, OR PASSED TO UNTRUSTED CLIENTS.
 * CLIENT-SUPPLIED AUTHORIZATION CLAIMS MUST BE IGNORED; CONTEXT MUST BE ASSEMBLED SERVER-SIDE.
 * TOKEN SIGNATURES, AUDIENCES, ISSUERS, AND EXPIRATIONS MUST BE STRICTLY VALIDATED.
 * LEAST-PRIVILEGE SCOPES MUST BE ENFORCED AT THE CAPABILITY LEVEL.
 * CREDENTIALS MUST SUPPORT ROTATION AND IMMEDIATE REVOCATION.
 * CROSS-TENANT AUTHORIZATION ATTEMPTS MUST DETERMINISTICALLY FAIL CLOSED AS TENANT VIOLATIONS.
 * AUTHENTICATION AND AUTHORIZATION FAILURES MUST NOT BE SILENTLY RETRIED.
16. VERSION HISTORY
| Version | Date | Description | Author | Approval Status |
|---|---|---|---|---|
| 1.0.0 | August 2026 | Initial Integration Authentication & Authorization specification. Established clear boundaries between identity proof, technical authorization, and business authority. Defined strict validation requirements for JWTs, OAuth, API keys, and mTLS. Established capability-level permissions, context propagation rules, token replay protection, and canonical failure taxonomies for all Phase 5 operations. | Ramy Bella | DRAFT / Implementation Specification |
| 1.0.1 | August 2026 | Targeted IAM precision pass: refined tenant/environment error classification, generalized OAuth flow requirements, clarified replay-protection applicability, strengthened PE-SPEC-11/13 ownership boundaries, added credential-expiration monitoring requirements, and removed overclaimed mathematical-separation language. | Ramy Bella | DRAFT / Implementation Specification |
ARCHITECTURAL VERDICT
APPROVED — DRAFT / IMPLEMENTATION SPECIFICATION
Implementation Readiness Statement:
PE-SPEC-11 provides an enforceable, deterministic authentication and technical-authorization architecture for the Phase 5 Integration Layer. By structurally and deterministically separating identity verification and technical authorization from Phase 3 business authority, and by establishing rigorous technical scope enforcement (JWT validation, audience checking, capability gates), this specification ensures that no actor—internal or external—can bypass security controls. It seamlessly integrates with the PE-SPEC-10 security perimeter and defers tenant isolation (PE-SPEC-13), error recovery (PE-SPEC-14), and data mapping (PE-SPEC-12) to their respective specifications, creating a secure, fail-closed IAM foundation ready for implementation.
