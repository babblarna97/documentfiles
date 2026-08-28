# INT-SPEC-18: Integration Observability & Audit

## 1. DOCUMENT CONTROL

| Attribute | Value |
|---|---|
| Document Title | 18 Integration Observability & Audit.md |
| Document ID | INT-SPEC-18 |
| Version | 1.0.0 |
| Status | DRAFT / Implementation Specification |
| Author | Ramy Bella |
| Classification | Confidential / Enterprise Proprietary |
| Target Audience | Site Reliability Engineers (SRE), Integration Architects, Security Architects, Platform Engineers, Database Engineers, DevOps Engineers |
| Parent Document | INT-SPEC-01 |
| Related Documents | INT-SPEC-01 through INT-SPEC-17, INT-SPEC-19, INT-SPEC-20, Phase 4 Specifications |
| System | Restaurant AI System |
| Phase | Phase 5 — Integrations |
| Lifecycle Folder | LIFECYCLE & OPERATIONS |
| Last Updated | August 2026 |

---

## 2. EXECUTIVE PURPOSE

A distributed, multi-tenant integration architecture executing asynchronous calls across third-party LLMs, booking engines, POS systems, and communication gateways cannot operate safely without continuous, deterministic observability. Without strict tracing and immutable audit trails, system bottlenecks become invisible, distributed race conditions cannot be debugged, and security breaches or data leaks pass undetected.

`INT-SPEC-18` defines the enterprise architecture for telemetry generation, distributed context propagation, structured logging, real-time APM metrics, SLI/SLO monitoring, and immutable security audit trails across the Phase 5 integration layer. It guarantees end-to-end trace lineage across all synchronous and asynchronous execution boundaries while enforcing zero-leakage scrubbing to prevent telemetry sinks from becoming data exfiltration vectors.

### Core Architectural Invariants:
* `TELEMETRY SINK \neq DATA EXFILTRATION VECTOR`
* `AUDIT TRAIL \neq MUTABLE LOG`
* `CORRELATION LINEAGE MUST SURVIVE ALL ASYNC BOUNDARIES`
* `OBSERVABILITY METRICS \neq BUSINESS AUTHORITY`
* `SCRUBBING FAILURE \implies SPAN DISCARD (FAIL-CLOSED)`

---

## 3. PURPOSE AND SCOPE

### Scope: What INT-SPEC-18 Controls
* **Distributed Tracing Engine:** W3C Trace Context specification enforcement (`traceparent`, `tracestate`) across all HTTP, RPC, queue, and thread boundaries.
* **Universal Correlation Lineage:** System-wide binding of `correlation_id` across prompt executions, integration dispatches, retries, and webhooks.
* **Canonical Telemetry Schema:** Structured, machine-readable JSON logging contracts (`IntegrationSpan@1.0.0`).
* **Zero-Leakage Scrubbing Pipeline:** In-memory redaction engine sanitizing credentials, API keys, bearer tokens, connection strings, PII, and PHI prior to log emission.
* **Cryptographic Immutable Audit Ledger:** Append-only, tamper-evident hash-chained ledger ($H_n = \text{SHA256}(H_{n-1} \mathbin{\Vert} \text{Payload}_n)$) for stateful mutation auditing.
* **SLI/SLO & APM Metrics Architecture:** Latency percentiles ($P_{50}, P_{95}, P_{99}$), error rates, circuit status, and token consumption metrics.

### Scope: What INT-SPEC-18 Explicitly Does NOT Control
* **Error Normalization & Classification:** Governed unconditionally by `INT-SPEC-14`.
* **Retry Execution & Backoff Scheduling:** Governed by `INT-SPEC-15`.
* **Asynchronous Webhook Ingestion & Signing:** Governed by `INT-SPEC-16`.
* **Rate Limits, Quotas & Circuit Breakers:** Governed by `INT-SPEC-17`.
* **Test Automation & Adapter Certification:** Governed by `INT-SPEC-19`.
* **Business State Transitions & Intent Precedence:** Governed unconditionally by Phase 3 (Conversation Engine).

---

## 4. ARCHITECTURAL POSITION

`INT-SPEC-18` acts as an asynchronous sidecar and interception pipeline wrapping all Phase 5 execution components, capturing telemetry spans and audit events without blocking primary execution paths.


[PHASE 5 DISPATCH / EXECUTION BOUNDARY]
│
├─► (Execution Thread) ──► [PROVIDER ADAPTER TRANSPORT]
│
└─► (Telemetry Collector Interceptor)
↓
[INT-SPEC-18 / OBSERVABILITY PIPELINE]
├─ 1. W3C Context & Correlation Lineage Capture
├─ 2. In-Memory PII & Secret Scrubbing Engine
│      └─ Scrubbing Failed? ──► Discard Span (ERR_INT_18_01)
├─ 3. Emit Structured Span Payload (IntegrationSpan@1.0.0)
└─ 4. Route Stateful Mutation Events to Audit Engine
├───────────────────────────┬───────────────────────────┐
▼                           ▼                           ▼
[APM SINK (OTel/Datadog)]   [METRICS SINK (Prometheus)]  [IMMUTABLE AUDIT SINK]
(Traces & Telemetry)        (SLI/SLO Counters)           (WORM Storage / Hash Chain)

---

## 5. DISTRIBUTED TRACING & CONTEXT PROPAGATION

To trace a single conversational turn across multiple internal microservices and external provider APIs, `INT-SPEC-18` mandates strict compliance with the **W3C Trace Context** specification.

### 1. W3C Trace Context Standard Headers
All outbound integration dispatches and internal IPCs MUST inject and propagate W3C standard headers:

* `traceparent`: `version-trace_id-parent_id-trace_flags`
  * Example: `00-4bf92f3577b34da6a3ce929d0e0e4736-00f067aa0ba902b7-01`
* `tracestate`: Vendor-specific state key-value pairs preserved across hops.

### 2. Universal Correlation Lineage Equation
In addition to the W3C trace ID, every telemetry item MUST carry the immutable system correlation ID:

$$\text{Telemetry Context} = \Big\{\text{trace\_id}, \text{span\_id}, \text{correlation\_id}, \text{tenant\_id}, \text{environment}\Big\}$$

The `correlation_id` MUST survive:
1. HTTP client-to-server request/response cycles (`INT-SPEC-07`).
2. Asynchronous job queue serialization and worker execution (`INT-SPEC-15`).
3. Outbound and inbound webhook delivery loops (`INT-SPEC-16`).
4. Retries and backoff sleep delays (`INT-SPEC-15`).

---

## 6. CANONICAL TELEMETRY SCHEMA (`IntegrationSpan@1.0.0`)

All integration logs, spans, and execution traces MUST be emitted as structured, single-line JSON payloads conforming to `IntegrationSpan@1.0.0`. Unstructured text logs are explicitly prohibited in production.

```json
{
  "timestamp": "2026-08-19T08:15:30.125Z",
  "trace_id": "4bf92f3577b34da6a3ce929d0e0e4736",
  "span_id": "00f067aa0ba902b7",
  "parent_span_id": "5c12ef349ab801c3",
  "correlation_id": "corr_987654321",
  "isolation_context": {
    "tenant_id": "tenant_restaurant_a",
    "venue_id": "venue_main_dining",
    "environment": "production"
  },
  "execution_target": {
    "capability": "BOOKING_CREATE",
    "provider": "resy_primary",
    "adapter_version": "1.2.0"
  },
  "performance": {
    "duration_ms": 245.8,
    "attempt_number": 1,
    "circuit_breaker_state": "CLOSED"
  },
  "result": {
    "status": "SUCCESS",
    "canonical_error_class": null,
    "provider_http_code": 200
  },
  "token_usage": {
    "prompt_tokens": 0,
    "completion_tokens": 0,
    "total_tokens": 0
  },
  "attributes": {
    "idempotency_key": "idem_sha256_abcd1234...",
    "scrubbed_fields_count": 2
  }
}

7. ZERO-LEAKAGE PII & SECRET SCRUBBING ENGINE
Telemetry sinks are frequent targets for data exfiltration. INT-SPEC-18 mandates an in-memory sanitization pipeline that intercept and scrub payloads before serialization to transport sinks.
Raw Execution Span Data
        │
        ▼
[Stage 1: Header & Key Redaction] ──► Matches Authorization, API Keys, Bearer, Passwords
        │                                └─ Replace with [REDACTED_SECRET]
        ▼
[Stage 2: PII Pattern Masking] ──► Matches Emails, Phone Numbers, Credit Cards, Guest Names
        │                            └─ Replace with [MASKED_PII:hash_prefix]
        ▼
[Stage 3: Verification Check] ──► Scans Output for Residual High-Entropy Strings / Keys
        │
        ├─ Verification Passed ──► Emit to Log Pipeline / APM
        └─ Verification Failed ──► DISCARD SPAN & Trigger ERR_INT_18_01 (CRITICAL)

Non-Negotiable Redaction Rules:
 * Secrets & Credentials: Authorization, X-API-Key, Cookie, Set-Cookie, db_connection_string, and provider private keys MUST be replaced with [REDACTED_SECRET].
 * Guest PII: Email addresses MUST be masked as j***n@domain.com. Phone numbers MUST be obfuscated to show only the last 4 digits (+1-XXX-XXX-1234).
 * Fail-Closed Principle: If the sanitization engine encounters an unparseable payload or high-entropy string ambiguity, it MUST discard the telemetry span entirely (ERR_INT_18_01) rather than risk leaking sensitive data.
8. CRYPTOGRAPHIC IMMUTABLE AUDIT TRAIL
For stateful mutations (BOOKING_CREATE, BOOKING_CANCEL, EMAIL_SEND, CREDENTIAL_ROTATION), INT-SPEC-18 maintains a tamper-evident, append-only cryptographic audit ledger.
9. Hash-Chained Audit Ledger Formula
Each audit log entry A_n includes the cryptographic hash of the previous record A_{n-1}, forming an immutable chain:
Where:
 * H_0 = \text{SHA256("GENESIS_BLOCK_INT_SPEC_18")}.
 * \text{PayloadHash}_n = \text{SHA256}(\text{Canonical Normalized Result Payload}).
2. Audit Trail Guarantees
 * Storage Tier: Written to write-once-read-many (WORM) storage or append-only PostgreSQL tables with row-level security blocking UPDATE and DELETE operations.
 * Non-Repudiation: Proves unequivocally that a specific external API call was dispatched by a specific tenant context at a precise time.
9. APM METRICS, SLI/SLO BOUNDARIES & ALERTING
INT-SPEC-18 mandates real-time metric aggregation across three Service Level Indicators (SLIs).
10. Key Performance Metrics (Prometheus / OTel Standard)
 * integration_requests_total: Counter tracking execution volume by provider, capability, tenant_id, and status.
 * integration_request_duration_ms: Histogram measuring latency percentiles (P_{50}, P_{95}, P_{99}).
 * integration_circuit_breaker_state: Gauge tracking circuit status (0 = \text{CLOSED}, 1 = \text{HALF\_OPEN}, 2 = \text{OPEN}).
 * integration_scrubbing_failures_total: Counter tracking dropped un-scrubbed spans.
2. Operational SLO Thresholds & Alerting Matrix
| Metric / SLI | Target SLO | Warning Alert Threshold | Critical Alert Threshold |
|---|---|---|---|
| Integration Availability | \ge 99.9\% Success | < 99.5\% over 5 mins | < 98.0\% over 5 mins (Page SRE) |
| P95 Latency (Booking API) | \le 1{,}500\text{ ms} | > 2{,}000\text{ ms} over 5 mins | > 4{,}000\text{ ms} over 5 mins |
| Circuit Breaker Tripped | 0 Open Circuits | 1 Circuit HALF_OPEN | \ge 1 Circuit OPEN (Page SRE) |
| DLQ Depth (INT-SPEC-16) | 0 Poison Payloads | > 5 Messages in DLQ | > 20 Messages in DLQ |
| Telemetry Scrubbing Failures | 0 Failures | \ge 1 Discarded Span | \ge 5 Discarded Spans (SecOps Alert) |
3. SECURITY & THREAT MODEL
| Threat | Attack Vector | Preventive Control | Severity |
|---|---|---|---|
| Data Leakage via Telemetry | API keys or guest PII logged to third-party APM (Datadog/Loggly). | Multi-stage in-memory scrubbing pipeline with fail-closed span discard (ERR_INT_18_01). | CRITICAL |
| Log Injection Attack | Attacker inserts newline characters (\r\n) in input to forge log entries. | Mandatory JSON structure encoding; raw strings sanitised against control chars. | HIGH |
| Audit Trail Tampering | Malicious admin alters database logs to erase proof of unauthorized booking. | SHA256 cryptographic hash-chaining (H_n) + WORM database permissions. | CRITICAL |
| Log Flooding DoS | High-frequency API calls generate terabytes of logs to exhaust disk/cost. | Adaptive sampling on high-volume read spans (GET); 100% logging on mutations. | HIGH |
4. FAILURE ARCHITECTURE & ERROR CODES
| Error Code | Description | Canonical Class | Severity |
|---|---|---|---|
| ERR_INT_18_01 | Telemetry scrubbing verification failed; span discarded to prevent secret leak. | DATA_BOUNDARY | CRITICAL |
| ERR_INT_18_02 | Distributed trace context propagation failed (traceparent header corrupted). | VALIDATION | MEDIUM |
| ERR_INT_18_03 | Cryptographic audit ledger hash-chain mismatch (Tampering detected). | INTERNAL | CRITICAL |
| ERR_INT_18_04 | Audit log storage engine (WORM DB) write failure. | PROVIDER_UNAVAILABLE | CRITICAL |
| ERR_INT_18_05 | Telemetry payload size exceeded maximum ingestion boundary (500 KB). | VALIDATION | MEDIUM |
5. PRODUCTION ACCEPTANCE CRITERIA
| AC ID | Category | Requirement | Verification Method | Pass/Fail |
|---|---|---|---|---|
| AC-OBS-01 | Secret Scrubbing | Passing an API key in a raw error message results in [REDACTED_SECRET] in emitted log. | Log Inspection Test | REQUIRED |
| AC-OBS-02 | Span Discard | Unscrubbed span containing raw private key is completely discarded with ERR_INT_18_01 alert. | Fault Injection Simulation | REQUIRED |
| AC-OBS-03 | Trace Propagation | Outbound HTTP call correctly formats and transmits valid W3C traceparent header. | Header Inspection Test | REQUIRED |
| AC-OBS-04 | Audit Integrity | Modifying a historical row in the PostgreSQL audit log causes hash-chain validation to fail (ERR_INT_18_03). | Tamper Verification Test | REQUIRED |
| AC-OBS-05 | Metric Accuracy | 100 failed booking requests accurately increment integration_requests_total{status="error"} by 100. | Prometheus Metric Audit | REQUIRED |
| AC-OBS-06 | Correlation Lineage | A background retry job preserves the exact correlation_id of the original trigger request. | Trace Lineage Verification | REQUIRED |
6. INTEGRATION AUTHORITY MATRIX
| Component | Owns | Consumes | Produces |
|---|---|---|---|
| INT-SPEC-18 | Tracing Engine, W3C Headers, Scrubbing Pipeline, Immutable Audit Ledger, APM Metrics | Raw Execution Traces, Spans, Mutation Events | Sanitized JSON Spans (IntegrationSpan@1.0.0), Hash-Chained Audit Logs, APM Alerts |
| INT-SPEC-13 | Tenant & Environment Scopes | System Context | Validated Telemetry Context (env:tenant:) |
| INT-SPEC-14 | Error Taxonomy & Classes | Raw Exceptions | Canonical Error Indicators for Telemetry |
| Phase 3 (CE) | Business Truth & State Transitions | Telemetry & Audit Query Signals | Authoritative Business Actions |
7. FINAL NON-NEGOTIABLE PRINCIPLES
 * TELEMETRY SINKS MUST NEVER BECOME DATA EXFILTRATION PATHS; SCRUBBING IS MANDATORY.
 * IF TELEMETRY SCRUBBING CANNOT GUARANTEE SECRET REMOVAL, THE SPAN MUST BE DISCARDED IMMEDIATELY.
 * ALL LOGS MUST BE EMITTED AS STRUCTURED JSON (IntegrationSpan@1.0.0); TEXT LOGS ARE PROHIBITED.
 * W3C TRACE CONTEXT AND CORRELATION IDS MUST SURVIVE ALL SYNCHRONOUS AND ASYNC BOUNDARIES.
 * STATEFUL MUTATIONS MUST BE WRITTEN TO AN APPEND-ONLY, CRYPTOGRAPHICALLY HASH-CHAINED AUDIT LEDGER.
 * OBSERVABILITY IS AN ASYNCHRONOUS SIDECAR; TELEMETRY FAILURES MUST NEVER BLOCK PRIMARY BUSINESS EXECUTION.
15. VERSION HISTORY & ARCHITECTURAL VERDICT
| Version | Date | Description | Author | Status |
|---|---|---|---|---|
| 1.0.0 | August 2026 | Initial specification of Integration Observability & Audit architecture (INT-SPEC-18). | Ramy Bella | APPROVED |
ARCHITECTURAL VERDICT:
APPROVED — DRAFT / IMPLEMENTATION SPECIFICATION
INT-SPEC-18 establishes the mandatory W3C distributed tracing, zero-leakage scrubbing, canonical JSON telemetry, and cryptographic audit trail engine for Phase 5. Ready for implementation.

