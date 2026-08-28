# INT-SPEC-14: Integration Error Handling

## 1. DOCUMENT CONTROL

| Attribute | Value |
|---|---|
| Document Title | 14 Integration Error Handling.md |
| Document ID | INT-SPEC-14 |
| Version | 1.0.0 |
| Status | DRAFT / Implementation Specification |
| Author | Ramy Bella |
| Classification | Confidential / Enterprise Proprietary |
| Target Audience | Integration Architects, Reliability Engineers, Backend Engineers, Security Architects, Platform Engineers, QA Architects |
| Parent Document | INT-SPEC-01 |
| Related Documents | INT-SPEC-01 through INT-SPEC-13, INT-SPEC-15 through INT-SPEC-20, Phase 4 Specifications |
| System | Restaurant AI System |
| Phase | Phase 5 — Integrations |
| Lifecycle Folder | RELIABILITY & EXECUTION |
| Last Updated | August 2026 |

---

## 2. EXECUTIVE PURPOSE

The Restaurant AI System operates in a distributed environment interacting with non-deterministic LLM APIs, volatile third-party booking systems, external email gateways, CRM platforms, and client-side browser surfaces. In such an ecosystem, unhandled, inconsistently mapped, or insecurely logged errors trigger cascading system failures, corrupt business state, leak sensitive credentials, and destroy guest trust.

`INT-SPEC-14` defines the enterprise architecture governing error handling across the entire Phase 5 integration layer. It establishes a deterministic, unified error lifecycle that catches, normalizes, sanitizes, classifies, and routes all integration exceptions before they can compromise core business authority or leak sensitive context.

### Core Architectural Invariants:
* `ERROR NORMALIZATION \neq ERROR SUPPRESSION \neq BUSINESS AUTHORITY`
* `RAW PROVIDER EXCEPTION \neq SYSTEM ERROR CONTRACT`
* `UNKNOWN EXECUTION STATE \neq DEFINITIVE FAILURE \neq DEFINITIVE SUCCESS`
* `SENSITIVE CONTEXT MUST NEVER LEAK ACROSS ERROR SURFACES`

Error handling normalizes technical execution failures and evaluates recovery policies. It does NOT invent missing business facts, modify Phase 3 business state directly, or bypass tenant isolation boundaries.

---

## 3. PURPOSE AND SCOPE

### Scope: What INT-SPEC-14 Controls
* **Canonical Error Taxonomy:** Standardized classification of all integration failures into immutable canonical classes.
* **Error Normalization Pipeline:** Deterministic translation of proprietary HTTP status codes, SDK exceptions, and network drops into `INT-SPEC-02` error contracts.
* **Sanitization & Privacy Boundary:** Automated scrubbing of secrets, API keys, bearer tokens, DB connection strings, and PII from error objects prior to logging or client propagation.
* **Error Severity & Impact Classification:** Categorizing exceptions by operational impact (`CRITICAL`, `HIGH`, `MEDIUM`, `LOW`).
* **Retryability Determination Boundary:** Explicit declaration of whether an error is transient (eligible for retry) or fatal (permanent rejection).
* **Unknown Execution State Governance:** Managing ambiguous timeouts post-network-dispatch without corrupting state.
* **User-Facing vs. Telemetry Mapping:** Severing internal debugging stack traces from safe, localized client error payloads.
* **Integration Fault Isolation:** Preventing a single failing provider or adapter from crashing parent execution threads.
* **Error Testing & Injection Models:** Requirements for chaos engineering and adapter error certification.

### Scope: What INT-SPEC-14 Explicitly Does NOT Control
* **Master Integration Architecture:** Governed by `INT-SPEC-01`.
* **Canonical Schema Models:** Governed by `INT-SPEC-02`.
* **Retry Execution Algorithms & Backoff Calculations:** Governed by `INT-SPEC-15`.
* **Circuit Breaker State & Thresholds:** Governed by `INT-SPEC-17`.
* **Telemetry Delivery & Audit Log Storage:** Governed by `INT-SPEC-18`.
* **Business State Transitions & Intent Precedence:** Governed unconditionally by Phase 3 (Conversation Engine).

---

## 4. ARCHITECTURAL POSITION

`INT-SPEC-14` sits directly on the execution path between concrete provider adapters and canonical system interfaces, intercepting all exceptions and raw responses before handoff to Phase 3 runtime logic.


[EXTERNAL PROVIDER / TRANSPORT]
↓ (Raw Exception / HTTP Error / Timeout)
[PROVIDER ADAPTER]
↓
[INT-SPEC-14 / ERROR NORMALIZATION PIPELINE]
├─ 1. Interception & Context Capture
├─ 2. Secret & PII Scrubbing
├─ 3. Classification to Canonical Class
├─ 4. Severity & Retryability Assertion
└─ 5. Canonical Error Object Generation
↓
[INT-SPEC-15 / RETRY & IDEMPOTENCY EVALUATOR]
├─ If Retryable & Idempotent → Trigger Backoff
└─ If Non-Retryable / Exhausted → Escalate
↓
[INT-SPEC-02 / CANONICAL ERROR RESULT]
↓
[PHASE 3 / BUSINESS INTERPRETATION] (Determines User/Workflow Action)

---

## 5. ERROR AUTHORITY & TRUST MODEL

In Phase 5, errors carry distinct trust levels depending on their point of origin:

1. **INTERNAL SYSTEM ERRORS:** High trust regarding operational semantics; must be protected against external exposure.
2. **PROVIDER-NATIVE ERRORS:** Untrusted; third-party APIs can return malformed error JSON, invalid HTTP status codes, or misleading text messages.
3. **CLIENT-SURFACE ERRORS:** Untrusted; frontend inputs or tampered browser signals must not dictate backend error states.

### Core Invariants of Error Trust:
* Third-party provider error messages MUST NOT be passed raw to user-facing widget surfaces.
* An external API returning `HTTP 200 OK` with an error payload in the body MUST be intercepted and converted into a canonical integration failure.
* An external API returning `HTTP 500` MUST NOT trigger core system crashes or unhandled stack trace dumps.

---

## 6. CANONICAL ERROR TAXONOMY

All integration errors across all providers MUST map to exactly one of the fifteen canonical error classes defined below:

| Canonical Error Class | Code | Description | Default Retryability |
|---|---|---|---|
| `VALIDATION` | `ERR_CLASS_VALIDATION` | Malformed input, schema mismatch, or unparseable payload. | **NON-RETRYABLE** |
| `AUTHENTICATION` | `ERR_CLASS_AUTHN` | Invalid API keys, expired tokens, or failed transport handshakes. | **NON-RETRYABLE** |
| `AUTHORIZATION` | `ERR_CLASS_AUTHZ` | Provider or venue lacks permission to execute capability. | **NON-RETRYABLE** |
| `TRANSIENT` | `ERR_CLASS_TRANSIENT` | Temporary network drop, socket reset, or DNS glitch prior to dispatch. | **RETRYABLE** |
| `TIMEOUT` | `ERR_CLASS_TIMEOUT` | Provider failed to respond within capability timeout budget. | **CONDITIONAL** |
| `RATE_LIMIT` | `ERR_CLASS_RATE_LIMIT` | Provider quota or throttling threshold breached (HTTP 429). | **RETRYABLE** |
| `PROVIDER_UNAVAILABLE` | `ERR_CLASS_UNAVAILABLE` | Provider service down, returning HTTP 502/503/504. | **RETRYABLE** |
| `PROVIDER_REJECTED` | `ERR_CLASS_REJECTED` | Provider executed request but explicitly refused action (e.g., fully booked). | **NON-RETRYABLE** |
| `CONTRACT_MISMATCH` | `ERR_CLASS_CONTRACT` | Provider response violated expected `INT-SPEC-02` structure. | **NON-RETRYABLE** |
| `DATA_BOUNDARY` | `ERR_CLASS_DATA_BOUNDARY` | Payload contains prohibited secrets, unmapped PII, or raw PCI. | **NON-RETRYABLE** |
| `TENANT_VIOLATION` | `ERR_CLASS_TENANT` | Scope mismatch or cross-tenant resource attempt (`INT-SPEC-13`). | **NON-RETRYABLE** |
| `IDEMPOTENCY` | `ERR_CLASS_IDEMPOTENCY` | Duplicate execution collision or missing mandatory idempotency key. | **NON-RETRYABLE** |
| `WEBHOOK_INTEGRITY` | `ERR_CLASS_WEBHOOK` | Failed cryptographic signature or timestamp replay check (`INT-SPEC-16`). | **NON-RETRYABLE** |
| `CONFIGURATION` | `ERR_CLASS_CONFIG` | Adapter unpinned, missing configuration, or bad routing table. | **NON-RETRYABLE** |
| `INTERNAL` | `ERR_CLASS_INTERNAL` | Unhandled exception inside adapter or framework execution logic. | **NON-RETRYABLE** |

---

## 7. ERROR NORMALIZATION & MAPPING PIPELINE

Every exception entering the error pipeline passes through a deterministic 5-stage transformation sequence:


[Raw Exception]
→ Stage 1: Interception (Capture raw code, response body, transport metadata)
→ Stage 2: Sanitization (Scrub API keys, bearer tokens, headers, PII)
→ Stage 3: Taxonomy Mapping (Match native error to Canonical Class)
→ Stage 4: Envelope Enrichment (Attach correlation_id, tenant_scope, provenance)
→ Stage 5: Canonical Output (Emit INT-SPEC-02 IntegrationError object)

### Canonical Error Contract Schema (`IntegrationError@1.0.0`):
```json
{
  "error_id": "err_01HXYZ1234567890",
  "error_code": "ERR_BOOKING_05",
  "error_class": "TIMEOUT",
  "severity": "HIGH",
  "retryable": true,
  "message": "Provider failed to respond within configured budget of 2500ms.",
  "provenance": {
    "provider": "resy_primary",
    "adapter_version": "1.2.0",
    "capability": "BOOKING_CREATE"
  },
  "context": {
    "tenant_id": "tenant_restaurant_a",
    "venue_id": "venue_main_dining",
    "environment": "production",
    "correlation_id": "corr_987654321",
    "execution_state": "UNKNOWN"
  },
  "provider_raw_reference": {
    "native_code": "HTTP_504",
    "vendor_request_id": "req_resy_abc123"
  },
  "timestamp": "2026-08-19T07:15:00.000Z"
}

8. ERROR CLASSIFICATION & SEVERITY MODEL
Errors are assigned an operational severity level to drive alerts, fallback behavior, and circuit breakers:
 * CRITICAL: Complete loss of core integration capabilities, tenant isolation breaches, credential leakage attempts, or corrupted internal state. Requires immediate engineering intervention.
 * HIGH: Primary provider unavailable for a core capability (e.g., booking API down) without seamless fallback, or persistent rate limiting.
 * MEDIUM: Transient network timeout on non-blocking queries, soft bounces on emails, or localized formatting mismatches.
 * LOW: Expected rejection states (e.g., guest requested a table size that the restaurant doesn't support).
9. SANITIZATION & PRIVACY BOUNDARY
To prevent secret exfiltration via telemetry or logging, INT-SPEC-14 enforces automated scrubbing prior to error object instantiation.
Non-Negotiable Sanitization Rules:
 * Secrets & Keys: Headers containing Authorization, X-API-Key, Bearer, or connection strings MUST be replaced with [REDACTED].
 * PII Redaction: Phone numbers, email addresses, and guest names present in raw HTTP error bodies MUST be scrubbed or tokenized based on INT-SPEC-12 policies.
 * Stack Traces: Raw language stack traces containing local file system paths, database credentials, or framework internals MUST NEVER be propagated outside the internal secure logging boundary.
10. UNKNOWN EXECUTION STATE HANDLING
When a network timeout or connection drop occurs after a state-changing mutation payload (e.g., BOOKING_CREATE, EMAIL_SEND) has been dispatched across the wire, the system faces an ambiguous state:
Mandatory Governance Rules:
 * INT-SPEC-14 MUST mark the result execution_state as UNKNOWN.
 * The pipeline MUST NOT convert UNKNOWN into SUCCESS or FAILURE.
 * The error MUST be flagged with retryable: false for direct execution paths unless an authoritative idempotency key (INT-SPEC-15) was transmitted with the original request.
 * Phase 3 MUST be informed that the business outcome is unverified, triggering reconciliation protocols (e.g., background status polling or webhook listening).
11. USER-FACING VS TELEMETRY ERROR MAPPING
INT-SPEC-14 enforces total decoupling between detailed technical logs and client-facing error payloads.
+-----------------------------------------------------------------------+
|                         INTERNAL TELEMETRY SINK                       |
| { "error_code": "ERR_BOOKING_05", "provider_code": "HTTP_504", ... }  |
+-----------------------------------------------------------------------+
                                    ▲
                                    │ (Interception Boundary)
+-----------------------------------------------------------------------+
|                         CLIENT SURFACE (WIDGET)                       |
| "We are currently checking table availability with the restaurant.    |
|  Please wait a moment or refresh."                                    |
+-----------------------------------------------------------------------+

 * Client Surface Rule: Users and website widgets receive sanitized, localized, friendly status messages. Technical error codes, provider names, vendor endpoints, and raw internal statuses MUST NEVER be exposed to the client DOM (INT-SPEC-07).
12. ERROR SECURITY & THREAT MODEL
| Threat | Attack Vector | Preventive Control | Severity |
|---|---|---|---|
| Secret Exfiltration | API key leaked in raw provider error response payload logged to APM. | Mandatory pattern scrubbing & header redaction in Stage 2 pipeline. | CRITICAL |
| Information Disclosure | Stack traces returned to website widget revealing server OS and paths. | Strict decoupling of internal errors from user-facing responses. | HIGH |
| State Corruption | Silently converting network timeout on BOOKING_CREATE to FAILURE, causing double booking. | Enforcement of explicit UNKNOWN state semantics. | CRITICAL |
| Error Flooding / DoS | Malicious third-party returning massive error JSON strings to exhaust memory. | Payload size limits (max 10KB) on raw provider error bodies. | HIGH |
| Cross-Tenant Leakage | Telemetry logs mixing correlation IDs and tenant IDs without scope bounds. | Explicit tenant_scope binding on all generated error contracts. | CRITICAL |
13. FAILURE ARCHITECTURE & ERROR CODES
| Error Code | Description | Canonical Class | Severity |
|---|---|---|---|
| ERR_INT_14_01 | Adapter returned raw unhandled exception. | INTERNAL | HIGH |
| ERR_INT_14_02 | Error mapping rule missing for native provider status. | CONTRACT_MISMATCH | MEDIUM |
| ERR_INT_14_03 | Secret detected in raw error payload during sanitization. | DATA_BOUNDARY | CRITICAL |
| ERR_INT_14_04 | Ambiguous execution state on stateful mutation. | TIMEOUT | HIGH |
| ERR_INT_14_05 | Error payload size exceeded memory boundary. | VALIDATION | MEDIUM |
14. PRODUCTION ACCEPTANCE CRITERIA
| AC ID | Category | Requirement | Verification Method | Pass/Fail |
|---|---|---|---|---|
| AC-ERR-01 | Normalization | Native HTTP 503 from booking vendor maps to PROVIDER_UNAVAILABLE. | Fault Injection Test | REQUIRED |
| AC-ERR-02 | Sanitization | API keys in raw HTTP 401 response headers are replaced with [REDACTED]. | Log Audit Inspection | REQUIRED |
| AC-ERR-03 | Isolation | Stack traces are completely absent from widget-facing API payloads. | Edge Response Scan | REQUIRED |
| AC-ERR-04 | Ambiguity | Network drops post-POST return UNKNOWN state without fabricating failure. | Simulated Socket Drop | REQUIRED |
| AC-ERR-05 | Provenance | All generated IntegrationError objects contain valid correlation_id and tenant_id. | Contract Validation | REQUIRED |
15. INTEGRATION AUTHORITY MATRIX
| Component | Owns | Consumes | Produces |
|---|---|---|---|
| INT-SPEC-14 | Error Taxonomy, Sanitization Rules, Error Contracts | Raw Provider Exceptions, Transport Failures | Normalized IntegrationError Objects |
| INT-SPEC-15 | Retry Schedules, Backoff Algorithms, Idempotency Ledgers | Normalized IntegrationError Objects | Retry Actions / Failure Escalations |
| INT-SPEC-18 | Telemetry Ingestion, Structured Logging, Audit Storage | Normalized IntegrationError Objects | APM Traces, Alert Triggers |
| Phase 3 (CE) | Business Interpretation, Workflow State Transitions | Normalized IntegrationError Results | User Recovery Conversations |
16. FINAL NON-NEGOTIABLE PRINCIPLES
 * PHASE 3 REMAINS THE ULTIMATE BUSINESS AUTHORITY.
 * ALL PROVIDER EXCEPTION SURFACES MUST BE INTERCEPTED AND NORMALIZED.
 * SECRETS, API KEYS, AND PII MUST NEVER LEAK INTO ERROR LOGS OR CLIENT RESPONSES.
 * UNKNOWN EXECUTION STATES MUST NEVER BE SILENTLY CONVERTED TO SUCCESS OR FAILURE.
 * CLIENT SURFACES MUST ONLY RECEIVE SANITIZED, LOCALIZED STATUS MESSAGES.
 * NO IMPLICIT ERROR HANDLING; ALL UNMAPPED NATIVE CODES FAIL CLOSED TO CONTRACT_MISMATCH.
17. VERSION HISTORY & ARCHITECTURAL VERDICT
| Version | Date | Description | Author | Status |
|---|---|---|---|---|
| 1.0.0 | August 2026 | Initial specification of Integration Error Handling architecture. | Ramy Bella | APPROVED |
ARCHITECTURAL VERDICT:
APPROVED — DRAFT / IMPLEMENTATION SPECIFICATION
INT-SPEC-14 establishes the mandatory error normalization, privacy sanitization, and taxonomy boundary for the Phase 5 integration layer. Ready for implementation.

