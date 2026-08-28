PE-SPEC-10: Integration Security
1. DOCUMENT CONTROL
| Attribute | Value |
|---|---|
| Document Title | 10 Integration Security.md |
| Document ID | PE-SPEC-10 |
| Version | 1.0.1 |
| Status | DRAFT / Implementation Specification |
| Author | Ramy Bella |
| Classification | Confidential / Enterprise Proprietary |
| Target Audience | Security Architects, Integration Architects, Platform Engineers, Backend Engineers, SecOps, DevOps |
| Parent Document | PE-SPEC-01 |
| Related Documents | PE-SPEC-01 through PE-SPEC-09, PE-SPEC-11 through PE-SPEC-20, Phase 4 Specifications |
| System | Restaurant AI System |
| Phase | Phase 5 — Integrations |
| Lifecycle Folder | SECURITY & DATA |
| Last Updated | August 2026 |
2. EXECUTIVE PURPOSE
The Restaurant AI System connects to diverse external providers (Booking, CRM, Email, OpenAI) via the Phase 5 Integration Layer. These connections represent critical attack surfaces. If an external provider is compromised, or if an internal adapter is misconfigured, the system could suffer credential theft, data exfiltration, Server-Side Request Forgery (SSRF), or malicious payload ingestion.
The Integration Security Architecture (PE-SPEC-10) defines the overarching security boundaries, trust zones, and defensive mechanisms required to securely operate the Phase 5 integration layer.
Core Invariant:
EXTERNAL PROVIDERS = UNTRUSTED.
An integration adapter MUST protect the core system from the provider, and the provider from the core system. PE-SPEC-10 deterministically enforces that integration traffic is encrypted, authenticated, bounded by strict egress policies, and sanitized before crossing into core business logic or returning to the LLM.
3. PURPOSE AND SCOPE
Scope: What PE-SPEC-10 Controls
 * Trust Boundaries & Zones: Demarcation between Core, Adapters, and External Providers.
 * SSRF & Egress Protection: Network-level restrictions preventing adapters from accessing unauthorized internal or external IPs.
 * Credential & Secret Hygiene: Enforcing out-of-band secret injection and prohibiting secret leakage in logs, payloads, or prompts.
 * Transport Security: Enforcing explicitly approved encrypted transport profiles for all provider communications.
 * Provider Payload Sanitization: Treating all incoming API responses, webhooks, and errors as potentially hostile data.
 * Dependency & SDK Security: Supply-chain security requirements for third-party provider libraries.
 * Injection Defense from Integrations: Preventing external data from executing as internal prompt injections.
 * Integration Security Failure Modes: Deterministic fail-closed architecture for security anomalies mapped to canonical Phase 5 error classes.
Scope: What PE-SPEC-10 Explicitly Does NOT Control
 * Phase 4 Prompt Security: Governed by Phase 4 (PE-SPEC-11 Prompt Security Architecture).
 * Authentication & Authorization Logic: The specific OAuth, API Key, or JWT implementations are governed by Phase 5 PE-SPEC-11 (Integration Authentication & Authorization).
 * Data Mapping & Minimization: Governed by PE-SPEC-12.
 * Tenant Isolation: Governed by PE-SPEC-13.
 * Retry & Resilience: Governed by PE-SPEC-15 and PE-SPEC-17.
 * Webhook & Event Integrity: Governed by PE-SPEC-16.
 * Business Authorization: Governed exclusively by Phase 3.
4. ARCHITECTURAL POSITION
PE-SPEC-10 establishes a hardened perimeter around the provider adapter layer.
[PHASE 3 / BUSINESS CORE] (High Trust)
        ↕
[PE-SPEC-02 CANONICAL CONTRACTS] (Validation Boundary)
        ↕
=====================================================
[PE-SPEC-10 / INTEGRATION SECURITY BOUNDARY]
  - Credential Injection (Out-of-band)
  - SSRF Egress Filtering
  - Payload Sanitization
  - Encrypted Transport Enforcement
=====================================================
        ↕
[PE-SPEC-03 / PROVIDER ABSTRACTION & ADAPTERS] (Medium Trust)
        ↕
[EXTERNAL VENDOR API / NETWORK] (Untrusted)

Boundary Integrity: Core business logic and the LLM MUST NEVER bypass the PE-SPEC-10 security perimeter to access the network directly.
5. TRUST BOUNDARIES & ZONES
The integration architecture operates across distinct trust zones. Data crossing these zones MUST undergo explicit security transitions.
 * Zone 1: Core System (High Trust): Phase 3 runtime, Phase 4 orchestration, and canonical PE-SPEC-02 contracts. Assumed safe, provided strict input validation is maintained.
 * Zone 2: Adapter Layer (Medium Trust): The code bridging the canonical contract to the provider API. It handles secrets and raw external data. It is highly restricted in its network capabilities.
 * Zone 3: External Provider (Zero Trust): Third-party APIs, webhooks, and public networks. Data originating here is strictly treated as hostile until cryptographically verified and structurally validated.
Rule: Zone 3 data MUST NOT enter Zone 1 without being validated and normalized by Zone 2 under PE-SPEC-10 constraints.
6. SSRF & EGRESS PROTECTION
Server-Side Request Forgery (SSRF) is a critical threat when an AI system is authorized to dispatch network requests. The system MUST provide enforceable architectural controls to prevent the LLM or malicious user input from dictating the network destination.
Mandatory Controls:
 * Destination Allowlisting: Integration adapters MUST dispatch requests ONLY to explicitly registered provider identities and hostnames.
 * No Dynamic URL Construction from Untrusted Input: An LLM tool proposal MUST NOT be able to specify a raw URL (e.g., url: "[http://169.254.169.254/metadata](http://169.254.169.254/metadata)"). The integration contract MUST use semantic identifiers (e.g., provider: "booking_vendor_A"), which the adapter securely resolves to the explicitly configured endpoint.
 * DNS Resolution & Validation: The transport client MUST validate DNS resolution results and block loopback, link-local, private/internal, and metadata-service address ranges where appropriate.
 * DNS Rebinding Protection: The architecture MUST protect against DNS rebinding by revalidating resolved destinations where required and preventing subsequent resolution changes from bypassing egress controls.
Core Invariant:
UNTRUSTED INPUT MUST NOT CONTROL NETWORK DESTINATION.
7. CREDENTIAL & SECRET HYGIENE
Integration adapters require API keys, client secrets, and certificates to authenticate against external providers.
Non-Negotiable Secret Handling:
 * Out-of-Band Storage: Secrets MUST NOT be hardcoded in application code, prompt templates, or repository configuration. They must be injected at runtime via an explicitly approved Secret Manager (PE-SPEC-11).
 * Adapter-Level Scope: The adapter minimizes secret lifetime, confines it to the smallest possible execution scope, avoids unnecessary copying, and never serializes it into persistent application state.
 * No Echoing: Provider API keys, tokens, or basic auth strings MUST NEVER be exposed through logs, traces, exceptions, prompts, canonical responses, or API responses to the frontend.
 * Telemetry Scrubbing: All outbound and inbound transport headers/bodies MUST be scrubbed of authentication tokens before being sent to PE-SPEC-18 (Observability).
Architectural Intent:
SECRETS MUST NEVER ESCAPE THE AUTHORIZED TRANSPORT/SECRET BOUNDARY.
8. TRANSPORT SECURITY
All data traversing the boundary between the adapter and the external provider MUST be cryptographically protected.
 * Encrypted Transport Profile: All provider communications MUST use an explicitly approved encrypted transport security profile. TLS 1.3 SHOULD be preferred where supported; TLS 1.2 MAY be permitted only where explicitly approved and configured with secure, non-deprecated cipher suites. Plaintext HTTP MUST be prohibited in production.
 * Certificate Validation: Certificate chain validation is mandatory. Certificate validation bypass (e.g., verify=False) in production environments is strictly PROHIBITED.
 * No Protocol Downgrade: Insecure protocol downgrades are structurally prohibited.
 * mTLS (Mutual TLS): Where supported by internal or high-security external providers, mTLS MUST be implemented to cryptographically prove the identity of the Restaurant AI System to the provider. Provider-specific transport requirements may be stricter.
9. PROVIDER PAYLOAD SANITIZATION
External systems can be compromised. A compromised CRM or Booking vendor could return malicious payloads designed to execute XSS on the Restaurant AI widget, exploit JSON parsers, or trigger prompt injection.
Defensive Requirements:
 * Strict Schema Validation: Incoming proprietary JSON/XML MUST be structurally validated. Extraneous fields MUST be stripped during normalization.
 * Content Sanitization: String values retrieved from external providers (e.g., "Guest Notes" from a CRM) MUST be treated as untrusted data. They must not contain executable control characters.
 * Prompt Injection Defense: If external data (e.g., an email reply or CRM note) is fed back into the LLM context, it MUST be wrapped in Phase 4 security fencing (PE-SPEC-11 Prompt Security) to prevent the external system from hijacking the AI prompt.
 * Size Limits & Rejection Semantics: Payloads exceeding the configured maximum security size (e.g., 1MB) MUST be rejected before normalization or downstream processing.
Security Invariant:
OVERSIZED UNTRUSTED PAYLOAD \neq SAFE PAYLOAD AFTER TRUNCATION.
10. DEPENDENCY & SDK SECURITY
Provider adapters often utilize vendor-supplied SDKs (e.g., openai-python, stripe-node). These dependencies are supply-chain attack vectors.
Supply Chain Integrity:
 * Explicit Pinning: All third-party SDKs and dependencies MUST be strictly version-pinned.
 * Vulnerability Scanning: Adapters MUST NOT be promoted to production if their dependencies contain known critical/high CVEs.
 * Minimal Privileges: Adapters importing third-party SDKs must operate in environments with the minimum necessary OS and network permissions.
 * SDK Isolation: As established in PE-SPEC-03, no third-party SDK may be imported into the core business application; they remain structurally isolated in the adapter layer.
11. INJECTION & LOGGING DEFENSE
Adversaries may attempt to exploit the integration layer's logging and error handling.
 * Log Forging Prevention: Provider responses and error strings mapped to canonical integration errors MUST be sanitized for newline (\n, \r) and control characters to prevent log injection/forging.
 * Error Omission: Detailed provider error strings (e.g., database syntax errors leaked by a third-party vendor) MUST NOT be passed through to the untrusted frontend widget (PE-SPEC-07). The widget receives a generic canonical failure; the detailed error is safely routed to structured telemetry (PE-SPEC-18).
12. SECURITY THREAT MODEL
| Threat | Attack Surface | Preventive Control | Detection | Response | Owner | Severity |
|---|---|---|---|---|---|---|
| SSRF | Adapter Transport | Destination allowlisting; block loopback/private IPs | Network Monitor | FAIL_CLOSED | SecOps | Critical |
| Credential Exfiltration | Adapter Memory / Logs | Secret Manager injection; Log scrubbing | Secret Scanner | Alert / Revoke | SecOps | Critical |
| Man-in-the-Middle (MITM) | Network Transport | Approved TLS profile / Mandatory cert validation | Cert Validator | Drop Connection | Platform | Critical |
| Provider Poisoning | External API Response | Strict Schema Validation; Input sanitization | Schema Audit | FAIL_CLOSED | QA Arch | High |
| Prompt Hijacking | CRM Notes / External Data | Phase 4 prompt fencing applied to fetched data | Eval Testing | Block Content | Prompt Eng | Critical |
| Supply Chain Compromise | Third-Party SDKs | Dependency pinning; Vulnerability scanning | CI/CD Scanner | Block Build | SecOps | Critical |
| Log Forging | Error Handling | Newline/Control character stripping in errors | Log Auditor | Sanitize | Platform | Medium |
| DNS Rebinding | Transport Resolution | Destination allowlisting; DNS validation; Anti-rebinding | DNS Monitor | Drop Request | NetSec | High |
13. FAILURE ARCHITECTURE
Deterministic integration security errors map into the established canonical Phase 5 error taxonomy (PE-SPEC-01 through PE-SPEC-09). These errors instantly halt the execution path and FAIL CLOSED.
Taxonomy Rule:
PE-SPEC-10 SECURITY EVENT \neq CANONICAL PHASE 5 ERROR CLASS \neq BUSINESS OUTCOME.
| Failure ID | Condition | Canonical Error Class | Result State | Severity |
|---|---|---|---|---|
| ERR_SEC_INT_01 | Unauthorized Egress Attempt (SSRF blocked) | AUTHORIZATION | FAILED | Critical |
| ERR_SEC_INT_02 | Missing or corrupted integration credentials | AUTHENTICATION | FAILED | Critical |
| ERR_SEC_INT_03 | Encrypted transport/Certificate validation failure | CONFIGURATION | FAILED | Critical |
| ERR_SEC_INT_04 | Malicious payload / executable content detected | CONTRACT_MISMATCH | FAILED | Critical |
| ERR_SEC_INT_05 | Payload size exceeds maximum security bounds | VALIDATION | FAILED | High |
| ERR_SEC_INT_06 | Dependency integrity failure (SDK tampered/CVE) | CONFIGURATION | FAILED | Critical |
| ERR_SEC_INT_07 | Log injection attempt / Security sanitization fail | DATA_BOUNDARY | FAILED | Medium |
Note: Security failures are NEVER retryable automatically. They require SecOps intervention or definitive provider remediation. PE-SPEC-15 enforces retry boundaries.
14. PRODUCTION ACCEPTANCE CRITERIA
| AC ID | Category | Requirement | Verification Method | Expected Result | Pass/Fail | Severity |
|---|---|---|---|---|---|---|
| AC-SEC-I01 | SSRF Prevent. | Attempting to dispatch a canonical request to an unauthorized destination deterministically fails. | Egress Mock Test | ERR_SEC_INT_01 (AUTHORIZATION) | Required | Critical |
| AC-SEC-I02 | Credential Iso | API Keys are fully absent from system logs, LLM context, and Phase 3 state after a successful provider call. | Context Audit | Secrets Absent | Required | Critical |
| AC-SEC-I03 | TLS Enforce. | The adapter drops the connection and fails closed if the provider's encrypted transport profile violates approved TLS configuration or if certificate validation fails. | Cert Mock Test | ERR_SEC_INT_03 (CONFIGURATION) | Required | Critical |
| AC-SEC-I04 | Payload Bounds | A provider response exceeding the configured maximum security size limit is rejected before normalization or downstream processing. | Overflow Inject | ERR_SEC_INT_05 (VALIDATION) | Required | High |
| AC-SEC-I05 | Poison Defense | A CRM string containing prompt injection syntax is safely fenced and does not hijack the next LLM turn. | Injection Test | Attack Neutralized | Required | Critical |
| AC-SEC-I06 | Log Forging | An external provider error containing \n\r[FAKE_LOG] is sanitized before hitting the audit sink. | Error String Test | Newlines scrubbed | Required | High |
| AC-SEC-I07 | SDK Isolation | Integration adapter dependencies are strictly isolated and do not exist in the dependency tree of the Phase 3 or Phase 4 core runtimes. | Dependency Graph | Isolation verified | Required | Critical |
15. INTEGRATION AUTHORITY MATRIX
| Component | Owns | Consumes | Produces | Must Not Override |
|---|---|---|---|---|
| Phase 3 | Business Authorization | Secured Normal Results | Business Commands | Integration Security Rules |
| PE-SPEC-01/02/03 | Architecture & Contracts | Execution Intents | Abstract Dispatch | PE-SPEC-10 Security Gates |
| PE-SPEC-10 | Integration Security Boundaries | Outbound/Inbound Data | Secured Transit | Phase 3 Business Logic |
| PE-SPEC-11 | Auth & Authz Logic | Credentials / Tokens | Auth Headers | Egress Security Rules |
| PE-SPEC-12 | Data Mapping/Minimization | Sanitized Responses | Mapped Contracts | Security Validation |
| Provider Adapter | Transport Execution | Secured Config | Raw Responses | PE-SPEC-10 Perimeter |
16. INTEGRATION CONTRACTS / RELATIONSHIPS
PE-SPEC-10 governs the security perimeter for all Phase 5 implementations.
 * Phase 4 Prompt Security (PE-SPEC-11): PE-SPEC-10 protects the network layer; Phase 4 protects the cognitive layer (preventing prompt injection). If PE-SPEC-10 fetches data, Phase 4 fences it.
 * PE-SPEC-04 through PE-SPEC-09 (Adapters): Must operate entirely within the PE-SPEC-10 egress and payload sanitization boundaries.
 * PE-SPEC-11 (Auth & Authz): Implements the specific credential workflows that PE-SPEC-10 keeps out-of-band and protects from logging.
 * PE-SPEC-12 (Data Mapping): Executes semantic data transformations after PE-SPEC-10 has proven the payload is structurally secure and non-hostile.
 * PE-SPEC-13 (Isolation): Handles tenant isolation, while PE-SPEC-10 handles network/vendor isolation.
 * PE-SPEC-15 / PE-SPEC-17: Own retry orchestration and resilience; PE-SPEC-10 ensures security failures do not silently become retryable operations.
 * PE-SPEC-16 (Webhooks): Owns webhook and event integrity.
 * PE-SPEC-18 (Observability): Consumes logs only after PE-SPEC-10 mandates credential and log-forging scrubbing.
 * PE-SPEC-20 (Lifecycle/Registry): Owns provider registration.
17. FINAL NON-NEGOTIABLE PRINCIPLES
 * EXTERNAL PROVIDERS ARE UNTRUSTED.
 * UNTRUSTED INPUT MUST NOT CONTROL NETWORK DESTINATIONS; SSRF MUST BE STRUCTURALLY PREVENTED.
 * SECRETS MUST NEVER ESCAPE THE AUTHORIZED TRANSPORT/SECRET BOUNDARY.
 * ALL PROVIDER COMMUNICATION MUST BE EXPLICITLY ENCRYPTED AND CERTIFICATE-VALIDATED.
 * OVERSIZED OR HOSTILE DATA MUST BE REJECTED BEFORE DOWNSTREAM PROCESSING.
 * DEPENDENCIES AND VENDOR SDKS MUST BE ISOLATED FROM CORE DOMAIN CODE.
 * SECURITY ANOMALIES MUST ALWAYS FAIL CLOSED.
 * PHASE 3 REMAINS THE ULTIMATE BUSINESS AUTHORITY.
18. VERSION HISTORY
| Version | Date | Description | Author | Approval Status |
|---|---|---|---|---|
| 1.0.0 | August 2026 | Initial Integration Security specification. Established trust zones, SSRF/egress protection, out-of-band credential hygiene, TLS enforcement, provider payload sanitization, supply-chain constraints, and deterministic fail-closed security error mapping for all Phase 5 provider adapters. | Ramy Bella | DRAFT / Implementation Specification |
| 1.0.1 | August 2026 | Targeted security-precision pass: aligned PE-SPEC-10 failures with the canonical Phase 5 error taxonomy, refined transport-security requirements, replaced universal IP-pinning assumptions with DNS/egress controls, clarified secret lifetime handling, hardened payload-size rejection semantics, and removed overclaimed mathematical-guarantee language. | Ramy Bella | DRAFT / Implementation Specification |
ARCHITECTURAL VERDICT
APPROVED — DRAFT / IMPLEMENTATION SPECIFICATION
Implementation Readiness Statement:
PE-SPEC-10 establishes an enforceable, fail-closed security perimeter around the Phase 5 Integration Layer. By structurally decoupling credential management from business logic, enforcing strict destination allowlisting and DNS controls, and treating all external provider data as fundamentally hostile, this specification ensures that the integration layer is heavily constrained against attack. It seamlessly integrates with the established canonical Phase 5 error taxonomy and defers exact authentication workflows and data mapping rules to the subsequent Phase 5 specifications without claiming impossible absolute guarantees.
