# INT-SPEC-16: Webhooks & Event Handling

## 1. DOCUMENT CONTROL

| Attribute | Value |
|---|---|
| Document Title | 16 Webhooks & Event Handling.md |
| Document ID | INT-SPEC-16 |
| Version | 1.0.0 |
| Status | DRAFT / Implementation Specification |
| Author | Ramy Bella |
| Classification | Confidential / Enterprise Proprietary |
| Target Audience | Integration Architects, Security Architects, Distributed Systems Engineers, Backend Engineers, QA Architects, SecOps |
| Parent Document | INT-SPEC-01 |
| Related Documents | INT-SPEC-01 through INT-SPEC-15, INT-SPEC-17 through INT-SPEC-20, Phase 4 Specifications |
| System | Restaurant AI System |
| Phase | Phase 5 — Integrations |
| Lifecycle Folder | RELIABILITY & EXECUTION |
| Last Updated | August 2026 |

---

## 2. EXECUTIVE PURPOSE

Asynchronous webhooks and event streams allow external platforms (booking providers, POS systems, CRM engines, email dispatchers) to notify the Restaurant AI System of out-of-band state changes (e.g., table ready, reservation cancelled, email bounced, customer profile updated). Because inbound webhooks arrive via public internet endpoints, they represent high-risk attack surfaces vulnerable to signature forgery, payload poisoning, replay attacks, cross-tenant injection, and resource exhaustion DoS.

`INT-SPEC-16` defines the complete enterprise architecture for ingesting, validating, processing, and emitting webhooks and event streams. It enforces strict cryptographic signature verification, constant-time verification against timing attacks, timestamp tolerance windows, event-level deduplication, decoupled fast-ACK ingestion buffers, and tenant-bound processing boundaries.

### Core Architectural Invariants:
* `UNVERIFIED WEBHOOK \neq AUTHORITATIVE EVENT`
* `HTTP 202 ACCEPTANCE \neq BUSINESS STATE MUTATION`
* `INVALID SIGNATURE \implies IMMEDIATE DISCARD (HTTP 401/403)`
* `EVENT REPLAY \implies ATOMIC SUPPRESSION`
* `WEBHOOK PAYLOAD \neq DIRECT PHASE 3 AUTHORITY`

---

## 3. PURPOSE AND SCOPE

### Scope: What INT-SPEC-16 Controls
* **Cryptographic Inbound Signature Verification:** Standardized validation algorithms (HMAC-SHA256, Ed25519, RSA-SHA256) executed in constant-time.
* **Replay Protection Engine:** Timestamp tolerance checking ($\pm 300\text{ seconds}$) and atomic event deduplication via distributed lock tables.
* **Tenant & Environment Target Resolution:** Binding unauthenticated inbound webhook calls to verified `INT-SPEC-13` isolation scopes via immutable endpoint routing or provider account metadata.
* **Decoupled Ingestion & Fast-ACK Pipeline:** Buffer architecture accepting validated webhooks with `HTTP 202 Accepted` in $< 100\text{ ms}$ prior to asynchronous processing.
* **Event Payload Normalization & Schema Validation:** Mapping raw provider JSON/XML structures to canonical `INT-SPEC-02` event contracts via `INT-SPEC-12`.
* **Outbound Webhook Dispatch & Signing Engine:** Generating, signing, and dispatching outgoing system events to external venue webhooks with retry guarantees (`INT-SPEC-15`).
* **Dead Letter Queue (DLQ) & Poison Payload Governance:** Handling unparseable, malformed, or continuously failing event streams.

### Scope: What INT-SPEC-16 Explicitly Does NOT Control
* **Canonical Schema Contract Definitions:** Governed by `INT-SPEC-02`.
* **Data Transformation Rules:** Governed by `INT-SPEC-12`.
* **Tenant Isolation Context Enforcement:** Governed by `INT-SPEC-13`.
* **Retry Backoff & Deduplication Ledgers:** Governed by `INT-SPEC-15`.
* **Throttling & Concurrency Rate Limits:** Governed by `INT-SPEC-17`.
* **Business State Transitions & Workflow Rules:** Governed unconditionally by Phase 3 (Conversation Engine).

---

## 4. ARCHITECTURAL POSITION

`INT-SPEC-16` functions as the ingress/egress gateway for all asynchronous event traffic crossing the Phase 5 boundary.


[EXTERNAL PROVIDER / WEBHOOK DISPATCHER]
↓ (Public Internet HTTP POST)
[INT-SPEC-16 / INGESTION EDGE]
├─ 1. Rate Limiting & Payload Size Check (< 1MB)
├─ 2. Cryptographic Signature Verification (HMAC / Ed25519)
├─ 3. Timestamp Window & Replay Protection Check
├─ 4. Tenant & Environment Resolution (INT-SPEC-13)
└─ 5. Fast-ACK Response (HTTP 202 Accepted < 100ms)
↓ (Push to Internal Event Buffer / Queue)
[ASYNC WORKER / INGESTION PIPELINE]
↓
[INT-SPEC-12 / EVENT PAYLOAD NORMALIZATION]
↓
[INT-SPEC-02 / CANONICAL EVENT CONTRACT]
↓
[PHASE 3 / BUSINESS INTERPRETATION]

---

## 5. CRYPTOGRAPHIC SIGNATURE & AUTHENTICITY BOUNDARY

All inbound webhooks MUST be cryptographically verified prior to parsing or queueing. Unsigned or improperly signed payloads MUST be dropped at the edge.

### 1. Verification Algorithm Standard
The primary verification protocol relies on HMAC-SHA256 over a canonical signature string comprising the request timestamp and raw payload bytes.

$$\text{Signature}_{\text{computed}} = \text{HMAC-SHA256}\Big(K_{\text{secret}}, t_{\text{header}} \mathbin{\Vert} \text{"."} \mathbin{\Vert} \text{Raw\_Body\_Bytes}\Big)$$

Where:
* $K_{\text{secret}}$: The tenant-and-provider-scoped shared secret retrieved via `INT-SPEC-11` / `INT-SPEC-13`.
* $t_{\text{header}}$: The Unix timestamp string extracted from the provider's signature header (e.g., `X-Provider-Timestamp`).
* $\text{Raw\_Body\_Bytes}$: The unparsed, exact byte sequence received in the HTTP request body.

### 2. Constant-Time Comparison Invariant
To eliminate side-channel timing attacks, string comparison between computed and received signatures MUST execute using a constant-time comparison function:

$$\text{Time}(\text{Compare}(S_{\text{computed}}, S_{\text{received}})) = C \quad \forall S_{\text{received}}$$

An invalid signature comparison MUST immediately return `HTTP 401 Unauthorized` without exposing whether the failure was due to bad timestamp, bad secret, or bad body format.

---

## 6. REPLAY ATTACK & TIMESTAMP TOLERANCE MODEL

To prevent malicious actors from capturing and replaying legitimate provider webhooks, `INT-SPEC-16` enforces dual-layer replay protection.


Incoming Webhook Request
│
▼
Is |t_current - t_header| <= 300s ? ─────► [NO] ───► REJECT (ERR_INT_16_02: Timestamp Out of Range)
│
[YES]
▼
Is event_id in Redis Lock Table? ────────► [YES] ──► SUPPRESS & ACK (HTTP 200/202: Duplicate Ignored)
│
[NO]
▼
Write event_id to Redis (TTL: 7 Days) ───► PROCEED TO INGESTION QUEUE

### 1. Timestamp Drift Boundary
The request timestamp $t_{\text{header}}$ MUST fall within an explicit tolerance window $\Delta t_{\text{max}}$ relative to system clock $t_{\text{system}}$:

$$|t_{\text{system}} - t_{\text{header}}| \le 300\text{ seconds}$$

Requests outside this 5-minute window MUST be rejected immediately (`ERR_INT_16_02`).

### 2. Atomic Event-ID Deduplication Table
Every processed webhook event ID (`event_id` or `provider_event_id`) MUST be recorded in a distributed Redis deduplication table:

$$\text{Redis Deduplication Key} = \text{env} : \text{tenant\_id} : \text{provider} : \text{event\_id}$$

* **TTL:** 604,800 seconds (7 Days).
* If a duplicate `event_id` arrives within the TTL window, the edge suppresses processing and returns `HTTP 200 OK` (or `HTTP 202 Accepted`) to acknowledge the provider without dispatching secondary queue events.

---

## 7. TENANT & ENVIRONMENT RESOLUTION PROTOCOL

Inbound webhooks do not carry internal session tokens. `INT-SPEC-16` must resolve the target `INT-SPEC-13` `IsolationContext` deterministically using one of two authorized mechanisms:

1. **Path-Based Routing (Preferred):** The incoming webhook URL contains an immutable, cryptographically random tenant endpoint token:
   `https://api.restaurantai.com/v1/webhooks/ingest/tpl_01HXYZ_tenant_token_abc123`
2. **Header/Metadata Binding:** The provider transmits a verified Account ID or Venue Reference in signed headers mapped in the database to a specific `tenant_id`.

### Resolution Invariants:
* A webhook whose payload claims `tenant_id = "restaurant_a"` BUT arrives on an endpoint bound to `tenant_b` MUST trigger a `TENANT_VIOLATION` error (`ERR_INT_16_03`) and be rejected immediately.
* Webhooks CANNOT resolve to multiple tenants simultaneously.

---

## 8. DECOUPLED FAST-ACK INGESTION ARCHITECTURE

To prevent external provider timeouts (which typically occur if an HTTP response is not received within 2,000–5,000 ms), `INT-SPEC-16` decouples network ingestion from business processing.


[Ingestion Edge] ──► Signature OK? ──► Write Raw Event to Queue ──► Return HTTP 202 (<100ms)
│
▼
[Background Event Queue]
│
▼
[Async Processing Worker]
├─ 1. INT-SPEC-12 Mapping
├─ 2. INT-SPEC-02 Schema Val
└─ 3. Phase 3 Workflow

### SLA & Processing Rules:
* **Ingestion SLA:** Time-to-ACK ($T_{\text{ACK}}$) MUST be $< 100\text{ ms}$.
* **Payload Limit:** Maximum HTTP body size is **1 MB**. Payloads exceeding 1 MB MUST be dropped immediately with `HTTP 413 Payload Too Large`.
* **Response Status:** Validated webhooks pushed to the ingestion queue MUST return `HTTP 202 Accepted` with a tracking payload:
  `{"status": "ACCEPTED", "tracking_id": "tr_01HXYZ...", "timestamp": "2026-08-19T08:12:00.000Z"}`

---

## 9. EVENT CANONICAL SCHEMAS & NORMALIZATION

Raw webhook payloads are untrusted vendor JSON/XML objects. They MUST be normalized via `INT-SPEC-12` into canonical `INT-SPEC-02` event contracts before passing to Phase 3.

### Canonical Event Contract Schema (`CanonicalEvent@1.0.0`):
```json
{
  "event_id": "evt_01HXYZ1234567890",
  "provider_event_id": "resy_evt_abc98765",
  "event_type": "BOOKING_CANCELLED",
  "provider": "resy_primary",
  "isolation_context": {
    "tenant_id": "tenant_restaurant_a",
    "venue_id": "venue_main_dining",
    "environment": "production"
  },
  "provenance": {
    "adapter_version": "1.2.0",
    "ingested_at": "2026-08-19T08:12:00.100Z",
    "signature_algorithm": "HMAC-SHA256"
  },
  "normalized_data": {
    "booking_reference": "bk_resy_998877",
    "cancellation_reason": "GUEST_REQUESTED",
    "cancelled_at": "2026-08-19T08:11:50.000Z"
  },
  "correlation_id": "corr_987654321"
}

10. OUTBOUND WEBHOOK DISPATCH & SIGNING ENGINE
When the Restaurant AI System emits asynchronous notifications to external client systems (e.g., notifying a venue's POS that an AI booking occurred), INT-SPEC-16 governs the outbound signing and dispatch pipeline.
[Phase 3 System Event] ──► [INT-SPEC-16 Outbound Engine]
                                    │
                                    ├─ 1. Fetch Target Endpoint & Secret
                                    ├─ 2. Append X-RestaurantAI-Timestamp
                                    ├─ 3. Generate HMAC-SHA256 Signature
                                    └─ 4. Dispatch with INT-SPEC-15 Retry Engine

Outbound Header Standard:
Outbound webhooks emitted by the platform MUST include the following cryptographic headers:
 * X-RestaurantAI-Signature: Computed HMAC-SHA256 signature string.
 * X-RestaurantAI-Timestamp: UTC Unix timestamp of dispatch.
 * X-RestaurantAI-Delivery-ID: Unique UUID for the delivery attempt.
 * User-Agent: RestaurantAI-WebhookDispatcher/1.0.
Outbound failures are retried automatically using INT-SPEC-15 exponential backoff with full jitter across a 24-hour delivery budget.
11. DEAD LETTER QUEUE (DLQ) & POISON PAYLOAD MANAGEMENT
Webhooks that pass edge signature verification but fail during asynchronous processing (e.g., due to schema corruption, database lock exhaustion, or adapter crashes) MUST be isolated to prevent queue blockages.
[Async Worker] ──► Processing Fails (Attempt N) ──► INT-SPEC-15 Retry Engine
                                                            │
                                                            ├─ Attempts < 5 ──► Requeue with Backoff
                                                            └─ Attempts >= 5 ─► Route to DLQ
                                                                                      │
                                                                                      ▼
                                                                            [DLQ Storage & Alerting]

DLQ Governance Rules:
 * Payloads exceeding maximum retry attempts (N_{\text{max}} = 5) MUST be moved to the Dead Letter Queue (DLQ).
 * DLQ records MUST retain: raw body, received headers, verification metadata, stack traces, and correlation_id.
 * DLQ records MUST be retained for 30 days to permit manual replay after bug fixes.
 * An automated alert (HIGH severity) MUST trigger whenever the DLQ depth exceeds 10 messages in a 5-minute window.
12. SECURITY & THREAT MODEL
| Threat | Attack Vector | Preventive Control | Severity |
|---|---|---|---|
| Webhook Forgery | Attacker posts fake booking cancellations to public endpoint. | Mandatory constant-time HMAC-SHA256 / Ed25519 signature verification at edge. | CRITICAL |
| Replay Attack | Attacker intercepts valid payload and re-posts it 100 times. | Strict 5-minute timestamp window check + Redis atomic event_id lock table. | CRITICAL |
| Cross-Tenant Injection | Provider Webhook claims Tenant A payload on Tenant B endpoint. | Strict path token & metadata binding via INT-SPEC-13 before processing. | CRITICAL |
| Amplification DoS | Attacker sends 10MB JSON payloads to exhaust server memory. | Hard 1MB payload size limit enforced at reverse proxy / edge layer. | HIGH |
| Timing Attack | Attacker measures byte comparison execution time to forge signatures. | Mandatory constant-time string comparison (crypto.timingSafeEqual). | HIGH |
| Poison Payload Lockup | Corrupted JSON payload causes worker thread to crash in endless loop. | Isolated execution sandboxing + DLQ routing after max retry exhaustion. | HIGH |
13. FAILURE ARCHITECTURE & ERROR CODES
| Error Code | Description | Canonical Class | Severity |
|---|---|---|---|
| ERR_INT_16_01 | Inbound webhook signature verification failed (Invalid key/hash). | WEBHOOK_INTEGRITY | CRITICAL |
| ERR_INT_16_02 | Request timestamp drift exceeded tolerance (\pm 300\text{s}). | WEBHOOK_INTEGRITY | HIGH |
| ERR_INT_16_03 | Inbound webhook target endpoint mismatch with tenant scope. | TENANT_VIOLATION | CRITICAL |
| ERR_INT_16_04 | Webhook HTTP body size exceeded maximum limit (1 MB). | VALIDATION | HIGH |
| ERR_INT_16_05 | Inbound event payload failed canonical schema normalization. | CONTRACT_MISMATCH | HIGH |
| ERR_INT_16_06 | Outbound webhook dispatch failed permanent delivery budget. | PROVIDER_UNAVAILABLE | HIGH |
14. PRODUCTION ACCEPTANCE CRITERIA
| AC ID | Category | Requirement | Verification Method | Pass/Fail |
|---|---|---|---|---|
| AC-WHK-01 | Signature Valid. | An inbound webhook with a single corrupted signature byte returns HTTP 401 in < 20\text{ ms}. | Signature Tamper Test | REQUIRED |
| AC-WHK-02 | Replay Reject | Re-sending an identical valid webhook payload 10 seconds later returns HTTP 202 but emits 0 duplicate queue events. | Duplicate Dispatch Test | REQUIRED |
| AC-WHK-03 | Timestamp Drift | Inbound webhooks carrying a timestamp older than 300 seconds are rejected (HTTP 401/403). | Time-Shift Simulation | REQUIRED |
| AC-WHK-04 | Fast-ACK SLA | Ingestion edge returns HTTP 202 in < 100\text{ ms} under 500 RPS load. | Load Performance Test | REQUIRED |
| AC-WHK-05 | Tenant Isolation | Webhook sent to Endpoint A containing Tenant B payload fails closed (ERR_INT_16_03). | Cross-Tenant Mock Test | REQUIRED |
| AC-WHK-06 | DLQ Routing | Poison payload failing schema normalization 5 times is safely moved to DLQ without crashing worker. | Chaos Poison Test | REQUIRED |
15. INTEGRATION AUTHORITY MATRIX
| Component | Owns | Consumes | Produces |
|---|---|---|---|
| INT-SPEC-16 | Signature Verification, Replay Deduction, Ingestion Edge, Outbound Signing, DLQ Rules | Raw HTTP Requests, Outbound System Events | Validated Queue Payloads, Signed Outbound Webhooks |
| INT-SPEC-12 | Payload Schema Mapping | Raw Ingested Webhook JSON | Canonical Event Objects (INT-SPEC-02) |
| INT-SPEC-13 | Tenant & Environment Namespaces | Webhook Endpoint Tokens | Validated IsolationContext |
| INT-SPEC-15 | Outbound Webhook Retries | Failed Outbound Dispatches | Backoff Dispatch Schedules |
| Phase 3 (CE) | Business Meaning & State Updates | Canonical Event Objects | Authoritative State Mutations |
16. FINAL NON-NEGOTIABLE PRINCIPLES
 * NO UNVERIFIED WEBHOOK MAY EVER ENTER INTERNAL EVENT QUEUES OR PHASE 3 LOGIC.
 * ALL SIGNATURE COMPARISONS MUST EXECUTE IN CONSTANT TIME.
 * INBOUND WEBHOOK ACCEPTANCE (HTTP 202) IS A TRANSPORT ACKNOWLEDGEMENT, NOT A BUSINESS CONFIRMATION.
 * REPLAY PROTECTION MUST ENFORCE BOTH TIMESTAMP TOLERANCE AND ATOMIC EVENT DEDUPLICATION.
 * POISON PAYLOADS MUST BE AUTOMATICALLY ISOLATED TO THE DEAD LETTER QUEUE (DLQ) AFTER RETRY EXHAUSTION.
 * OUTBOUND WEBHOOKS EMITTED BY THE PLATFORM MUST ALWAYS BE CRYPTOGRAPHICALLY SIGNED.
17. VERSION HISTORY & ARCHITECTURAL VERDICT
| Version | Date | Description | Author | Status |
|---|---|---|---|---|
| 1.0.0 | August 2026 | Initial specification of Webhooks & Event Handling architecture (INT-SPEC-16). | Ramy Bella | APPROVED |
ARCHITECTURAL VERDICT:
APPROVED — DRAFT / IMPLEMENTATION SPECIFICATION
INT-SPEC-16 establishes the mandatory cryptographic signature verification, replay protection, fast-ACK ingestion architecture, and outbound event signing engine for Phase 5. Ready for implementation.


