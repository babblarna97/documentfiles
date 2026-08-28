# INT-SPEC-17: Rate Limits, Quotas & Resilience

## 1. DOCUMENT CONTROL

| Attribute | Value |
|---|---|
| Document Title | 17 Rate Limits, Quotas & Resilience.md |
| Document ID | INT-SPEC-17 |
| Version | 1.0.1 |
| Status | DRAFT / Implementation Specification |
| Author | Ramy Bella |
| Classification | Confidential / Enterprise Proprietary |
| Target Audience | Reliability Engineers, Platform Engineers, Integration Architects, Distributed Systems Engineers, Security Architects, Database Engineers |
| Parent Document | INT-SPEC-01 |
| Related Documents | INT-SPEC-01 through INT-SPEC-16, INT-SPEC-18 through INT-SPEC-20, Phase 4 Specifications |
| System | Restaurant AI System |
| Phase | Phase 5 — Integrations |
| Lifecycle Folder | RELIABILITY & EXECUTION |
| Last Updated | August 2026 |

---

## 2. EXECUTIVE PURPOSE

External third-party API providers enforce rigid rate limits, and breaching these limits triggers API key revocation, IP bans, financial overage penalties, and global service blackouts. Conversely, downstream integration endpoints operated by restaurant venues (e.g., local POS systems, legacy booking engines, or on-premise gateways) possess limited processing capacity and are easily paralyzed by sudden traffic spikes or burst concurrency.

`INT-SPEC-17` defines the complete enterprise architecture for rate limiting, tenant quota governance, distributed circuit breaking, concurrency backpressure, load shedding, and graceful degradation across the Phase 5 integration layer. It establishes a multi-tiered defense matrix that prevents rate breaches against upstream providers, protects internal worker infrastructure from thundering herd cascades, shields fragile downstream venue systems, and guarantees system resilience under severe load.

### Core Architectural Invariants:
* `RATE LIMIT BREACH \implies DETERMINISTIC HTTP 429 (RATE_LIMIT)`
* `CIRCUIT BREAKER OPEN \implies IMMEDIATE FAIL-FAST DISPATCH (< 5ms)`
* `BACKPRESSURE SATURATION \implies CONTROLLED PRIORITY LOAD SHEDDING`
* `RESILIENCE & FALLBACK \neq SILENT BUSINESS DATA FABRICATION`
* `TENANT QUOTA EXHAUSTION \neq GLOBAL PLATFORM OUTAGE`
* `CIRCUIT EVALUATION MUST EXECUTE ATOMICALLY IN MEMORY`

---

## 3. PURPOSE AND SCOPE

### Scope: What INT-SPEC-17 Controls
* **Upstream Rate Limit Enforcer:** Sliding Window Counter and Token Bucket algorithms governing outbound requests to third-party APIs (OpenAI, Resy, SevenRooms, SendGrid).
* **Downstream Load Protector:** Concurrency bounds and semaphores shielding venue-specific infrastructure from API flooding.
* **Distributed Circuit Breaker Engine:** Tripping state machine (`CLOSED` $\rightarrow$ `OPEN` $\rightarrow$ `HALF_OPEN`) preventing cascading thread pool exhaustion during remote outages.
* **Multi-Tenant Quota Governance:** Tiered execution budgets (Requests Per Minute/Day) bound per tenant and venue context (`INT-SPEC-13`).
* **Backpressure & Load Shedding Engine:** System-wide concurrency saturation detection and priority-based request shedding.
* **Fallback & Degraded Mode Execution Matrix:** Governed execution routes when non-critical integration dependencies trip or rate-limit.
* **Atomic Redis Lua Scripting:** Sub-millisecond distributed state evaluation for rate counters and circuit breaker state locks.

### Scope: What INT-SPEC-17 Explicitly Does NOT Control
* **Error Normalization & Classification:** Governed unconditionally by `INT-SPEC-14`.
* **Retry Execution & Backoff Mathematics:** Governed by `INT-SPEC-15`.
* **Asynchronous Webhook Ingestion & Ingestion Buffers:** Governed by `INT-SPEC-16`.
* **Telemetry, Trace Storage & APM Metrics Sinks:** Governed by `INT-SPEC-18`.
* **Business State Transitions & Workflow Rules:** Governed unconditionally by Phase 3 (Conversation Engine).

---

## 4. ARCHITECTURAL POSITION

`INT-SPEC-17` wraps all outgoing and incoming capability dispatch boundaries within Phase 5, evaluating rate limits, tenant quotas, and circuit breaker health *prior* to network transport execution.


[CANONICAL INTEGRATION REQUEST / DISPATCH]
↓
[INT-SPEC-17 / RESILIENCE & GATEWAY ENGINE]
├─ 1. Tenant Quota Check (RPM/RPD per INT-SPEC-13 Context)
│      └─ Exceeded? ──► Fail Fast (ERR_INT_17_03: TENANT_QUOTA_EXCEEDED)
├─ 2. System Backpressure Check (Thread Saturation Level)
│      └─ Saturation > 90%? ──► Execute Priority Load Shedding
├─ 3. Upstream Provider Rate Limiter (Redis Sliding Window)
│      └─ Breached? ──► Return HTTP 429 (ERR_INT_17_01: RATE_LIMIT_EXCEEDED)
├─ 4. Distributed Circuit Breaker Gate (CLOSED / OPEN / HALF_OPEN)
│      ├─ If OPEN ──► Immediate Fail-Fast (<5ms) / Execute Fallback
│      └─ If CLOSED / HALF_OPEN ──► Allocate Concurrency Semaphore & Pass
└─ 5. Downstream Venue Concurrency Guard (Semaphore)
↓
[PROVIDER ADAPTER NETWORK TRANSPORT DISPATCH]
↓
[INT-SPEC-17 / METRICS FEEDBACK LOOP]
└─ Intercept Latency, Status Code & Error Class (INT-SPEC-14)
└─ Feed Error/Success Metrics back to Circuit Breaker Sliding Window

---

## 5. DISTRIBUTED RATE LIMITING MECHANICS

To guarantee sub-millisecond precision across distributed worker nodes without race conditions, rate limits are tracked in Redis using the **Sliding Window Counter** algorithm with weighted time-frame decay.

### 1. Mathematical Sliding Window Formulation

The estimated request volume $N_{\text{current}}$ over a sliding window frame of duration $W$ is calculated as:

$$N_{\text{current}} = C_{\text{current\_bucket}} + C_{\text{previous\_bucket}} \times \left(1 - \frac{t_{\text{elapsed}}}{W}\right)$$

Where:
* $C_{\text{current\_bucket}}$: Total request count in the active time bucket.
* $C_{\text{previous\_bucket}}$: Total request count in the immediately preceding time bucket.
* $t_{\text{elapsed}}$: Elapsed time (in milliseconds) within the current time bucket.
* $W$: Total window frame size in milliseconds (e.g., $60{,}000\text{ ms}$ for a 1-minute window).

If $N_{\text{current}} \ge N_{\text{limit}}$, the dispatch MUST be blocked immediately, returning `ERR_INT_17_01` (`RATE_LIMIT_EXCEEDED`).

### 2. Atomic Redis Lua Rate Limiter Script (`evaluate_sliding_rate_limit.lua`)

```lua
-- KEYS[1]: Current Bucket Key (env:tenant_id:provider:capability:current_bucket)
-- KEYS[2]: Previous Bucket Key (env:tenant_id:provider:capability:prev_bucket)
-- ARGV[1]: Max Permitted Requests (N_limit)
-- ARGV[2]: Window Size in Seconds (W = 60)
-- ARGV[3]: Current Unix Timestamp in Seconds (t_now)
-- ARGV[4]: Elapsed Seconds in Current Bucket (t_elapsed)

local current_count = tonumber(redis.call("GET", KEYS[1]) or "0")
local prev_count = tonumber(redis.call("GET", KEYS[2]) or "0")
local limit = tonumber(ARGV[1])
local window = tonumber(ARGV[2])
local elapsed = tonumber(ARGV[4])

-- Calculate weighted sliding count
local weight = (window - elapsed) / window
local estimated_count = current_count + (prev_count * weight)

if estimated_count >= limit then
    local ttl = redis.call("TTL", KEYS[1])
    return {0, tostring(math.ceil(estimated_count)), tostring(ttl)} -- Blocked: 0, Count, Reset TTL
else
    redis.call("INCR", KEYS[1])
    redis.call("EXPIRE", KEYS[1], window * 2)
    return {1, tostring(math.ceil(estimated_count + 1)), "0"} -- Permitted: 1, Count, 0
end

3. RFC 6585 & RFC 7231 Rate Limit Header Emission
Every rate-limited capability interface MUST emit standard RFC 6585 / RFC 7231 HTTP response headers:
 * X-RateLimit-Limit: Maximum requests permitted within the window (N_{\text{limit}}).
 * X-RateLimit-Remaining: Remaining capacity in current window (\max(0, N_{\text{limit}} - \lceil N_{\text{current}} \rceil)).
 * X-RateLimit-Reset: UTC Unix timestamp indicating when the current window expires.
 * Retry-After: Mandatory header on HTTP 429 responses specifying exact sleep duration in seconds before secondary attempt.
6. DISTRIBUTED CIRCUIT BREAKER STATE MACHINE
To prevent cascading system collapse when a third-party vendor experiences a partial or full outage, INT-SPEC-17 enforces a distributed 3-state Circuit Breaker per (environment, provider, capability) tuple.
                  ┌──────────────────┐
                  │      CLOSED      │  ← Normal Execution (Metrics Collected)
                  └────────┬─────────┘
                           │ Failure Rate >= 50% AND Volume >= 20 in 60s
                           ▼
                  ┌──────────────────┐
                  │       OPEN       │  ← Immediate Fail-Fast (<5ms)
                  └───┬──────────┬───┘
                      │          │
    Cooldown Expired  │          │ Hard Reset /
    (T_cooldown=30s)  ▼          │ Admin Intervention
                  ┌──────────┐   │
                  │HALF_OPEN │   │
                  └────┬─────┘   │
                       │         │
      Trial Requests   │         │ Trial Request
      Succeed (N=5)    ▼         ▼ Failed (N>=1)
             ┌───────────┐    ┌───────────┐
             │  CLOSED   │    │   OPEN    │
             └───────────┘    └───────────┘

7. State Definitions & Operational Rules
 * CLOSED: Normal operation. Outbound network requests execute. Response statuses, latencies, and INT-SPEC-14 error classes are recorded in a sliding 60-second execution window.
 * OPEN: The provider or capability is classified as unreachable. All outbound dispatches fail fast immediately in < 5\text{ ms} with ERR_INT_17_02 (CIRCUIT_OPEN) without executing network transport. Governed fallback routes are executed if registered.
 * HALF_OPEN: After a configurable cooldown interval (T_{\text{cooldown}} = 30\text{ seconds}), the circuit transitions to HALF_OPEN. A strict trial semaphore permits N_{\text{trial}} = 5 consecutive requests to touch the provider API.
   * If all 5 trial requests succeed, the circuit resets to CLOSED.
   * If any trial request fails (returning PROVIDER_UNAVAILABLE, TIMEOUT, or INTERNAL per INT-SPEC-14), the circuit instantly reverts to OPEN for another T_{\text{cooldown}} cycle.
2. Tripping Threshold Mathematics
A circuit breaker transitions from CLOSED to OPEN if ALL THREE mathematical criteria are satisfied within a sliding evaluation window of W_{\text{cb}} = 60\text{ seconds}:
 * Minimum Request Volume Guard (V_{\text{min}}):
   
 * Failure Rate Threshold (E_{\text{rate}}):
   
   
   Where E_{\text{canonical}} comprises errors classified by INT-SPEC-14 as PROVIDER_UNAVAILABLE, TIMEOUT, INTERNAL, or CONTRACT_MISMATCH.
 * High Latency Percentile Cap (P_{99}):
   
7. TENANT QUOTA GOVERNANCE & MULTI-TENANT ISOLATION
To prevent a single venue from consuming the platform's shared API quotas (e.g., exhausting shared OpenAI TPM/RPM limits), INT-SPEC-17 enforces multi-tenant execution tiers strictly bound to INT-SPEC-13 IsolationContext scopes.
8. Multi-Tenant Execution Tier Matrix
| Quota Tier | Max Requests / Min (RPM) | Max Requests / Day (RPD) | Max Concurrent Threads (C_{\text{max}}) | Token-Per-Min Cap (TPM) |
|---|---|---|---|---|
| Standard Venue | 60 RPM | 10,000 RPD | 5 Threads | 40,000 TPM |
| Enterprise Chain | 600 RPM | 100,000 RPD | 50 Threads | 400,000 TPM |
| Platform System Core | 3,000 RPM | Unlimited | 200 Threads | 2,000,000 TPM |
9. Isolation Namespacing
Tenant quotas are enforced completely independently from provider-level rate limits using Redis key isolation:
A tenant breaching their tier limit receives an immediate ERR_INT_17_03 (TENANT_QUOTA_EXCEEDED) response. Other tenants operating on the same platform infrastructure remain completely unaffected.
10. BACKPRESSURE & LOAD SHEDDING ENGINE
When internal worker queues, memory limits, or database connection pools approach exhaustion, INT-SPEC-17 triggers automated Priority Load Shedding to keep core platform capabilities alive.
System Resource Saturation Level (CPU / Memory / Thread Pool / DB Concurrency)
   0% ──────────────────────── 70% ──────────────────────── 85% ──────────────────────── 100%
             [GREEN]                    [YELLOW]                   [RED: LOAD SHEDDING]
      Execute all requests      Queue non-critical events    Drop Low-Priority Traffic
                                                             Reserve 100% capacity for
                                                             Stateful Mutation (Bookings)

Priority Classification & Shedding Hierarchy:
| Priority Level | Capability Examples | Shedding Threshold | Action Under Saturation |
|---|---|---|---|
| P0: CRITICAL_MUTATION | BOOKING_CREATE, BOOKING_CANCEL | > 95\% Saturation | Never shed. Always processed or queued with high priority. |
| P1: REALTIME_CONVERSATION | RESPONSE_GENERATION, WIDGET_STREAM | > 85\% Saturation | Shed if execution budget < 1000\text{ms}; return localized fallback response. |
| P2: ASYNC_NOTIFICATION | EMAIL_SEND, CRM_SYNC | > 70\% Saturation | Defer execution to secondary persistent queues (INT-SPEC-15/16). |
| P3: BATCH_ANALYTICS | PROSPECT_SCRAPER, AUDIT_EXPORT | > 60\% Saturation | Drop execution immediately with ERR_INT_17_04 (SYSTEM_BACKPRESSURE). |
9. FALLBACK MECHANICS & DEGRADED MODES
When a circuit breaker trips to OPEN or an integration capability is rate-limited, INT-SPEC-17 enforces governed fallback behaviors.
Fallback Execution Matrix:
[Capability Request Failed / Circuit OPEN]
        │
        ├─ Capability: BOOKING_CREATE
        │    ├─ Check Alternate Compatible Adapter (INT-SPEC-03 / INT-SPEC-05)
        │    │    ├─ Fallback Adapter Available ──► Reroute Dispatch
        │    │    └─ No Fallback Available ──► Return State: UNKNOWN_AVAILABILITY
        │    └─ Invariant: NEVER fabricate fake booking confirmations.
        │
        ├─ Capability: RESPONSE_GENERATION (LLM)
        │    ├─ Check Secondary Model Reference (INT-SPEC-04)
        │    │    ├─ Pinned Secondary Model Ready ──► Reroute to Backup Model
        │    │    └─ Backup Model Unavailable ──► Return Localized System Fallback Dialog
        │    └─ Invariant: NEVER expose raw stack traces or vendor names.
        │
        └─ Capability: EMAIL_SEND / CRM_SYNC
             ├─ Buffer Payload in Persistent Dead Letter Queue (INT-SPEC-16)
             └─ Invariant: Re-dispatch automatically when circuit resets to CLOSED.

10. SECURITY & THREAT MODEL
| Threat | Attack Vector | Preventive Control | Severity |
|---|---|---|---|
| Thundering Herd Attack | Upstream recovery causes 1,000 concurrent retries to flood provider. | Circuit Breaker HALF_OPEN trial throttling (N_{\text{trial}}=5) + INT-SPEC-15 Full Jitter backoff. | CRITICAL |
| Denial of Wallet (DoW) | Malicious traffic bursts exploit LLM/SMS APIs to inflate costs. | Strict daily quota caps (RPD) per tenant with automated cost threshold alerts. | CRITICAL |
| Tenant Resource Starvation | Single tenant spams API endpoint, consuming global system capacity. | Multi-tenant quota governance (INT-SPEC-13 tenant-bound Redis sliding window limits). | HIGH |
| Cascade Infrastructure Collapse | Unresponsive venue POS stalls server thread pool indefinitely. | Concurrency semaphores per venue (C_{\text{max}}=5) + strict capability timeouts (INT-SPEC-01). | CRITICAL |
| Rate Limiter Poisoning | Attacker tampers with Redis rate counter keys via injected headers. | Server-authoritative key generation; header inputs are treated as untrusted data. | HIGH |
11. FAILURE ARCHITECTURE & ERROR CODES
| Error Code | Description | Canonical Class | Severity |
|---|---|---|---|
| ERR_INT_17_01 | Upstream provider rate limit breached (HTTP 429). | RATE_LIMIT | HIGH |
| ERR_INT_17_02 | Circuit breaker OPEN; request failed fast without network transport. | PROVIDER_UNAVAILABLE | HIGH |
| ERR_INT_17_03 | Tenant execution quota or tier limit exceeded. | RATE_LIMIT | MEDIUM |
| ERR_INT_17_04 | System backpressure threshold tripped; priority load shedding active. | RATE_LIMIT | HIGH |
| ERR_INT_17_05 | Distributed rate limiter / circuit breaker storage engine (Redis) unreachable. | PROVIDER_UNAVAILABLE | CRITICAL |
| ERR_INT_17_06 | Concurrency semaphore allocation timeout for downstream venue. | TIMEOUT | HIGH |
12. PRODUCTION ACCEPTANCE CRITERIA
| AC ID | Category | Requirement | Verification Method | Pass/Fail |
|---|---|---|---|---|
| AC-RES-01 | Sliding Window | Request N+1 in window exceeding limit returns ERR_INT_17_01 with valid Retry-After header. | Rate Limit Load Test | REQUIRED |
| AC-RES-02 | Circuit Tripping | 50% failure rate over 20 requests trips circuit to OPEN; next dispatch fails in < 5\text{ ms}. | Fault Injection Simulation | REQUIRED |
| AC-RES-03 | Half-Open Recovery | Circuit transitions to HALF_OPEN after 30s cooldown and allows exactly 5 trial requests. | State Machine Verification | REQUIRED |
| AC-RES-04 | Tenant Isolation | Tenant A exceeding 60 RPM gets rate-limited; Tenant B on same endpoint operates with 0 interference. | Multi-Tenant Concurrency Test | REQUIRED |
| AC-RES-05 | Load Shedding | System saturation > 90\% drops low-priority batch jobs while preserving 100% of BOOKING_CREATE calls. | Resource Saturation Test | REQUIRED |
| AC-RES-06 | Header Emission | Rate-limited responses emit RFC 6585 compliant X-RateLimit-Limit, Remaining, and Reset headers. | Header Audit Test | REQUIRED |
13. INTEGRATION AUTHORITY MATRIX
| Component | Owns | Consumes | Produces |
|---|---|---|---|
| INT-SPEC-17 | Rate Limit Counters, Circuit Breaker State Machine, Quotas, Load Shedding Rules, Semaphores | Execution Dispatches, INT-SPEC-14 Error Signals | Execution Approvals, Fail-Fast Rejections (HTTP 429), Fallback Triggers |
| INT-SPEC-13 | Tenant & Environment Scopes | System Context | Validated Tenant Quota Keys (env:tenant:) |
| INT-SPEC-15 | Backoff Delay Calculation | Rate Limit Rejections (HTTP 429) | Delayed Retry Schedules |
| Phase 3 (CE) | Business Meaning & Fallback Messaging | Degraded Execution Signals | Authoritative User Fallback Dialogues |
14. FINAL NON-NEGOTIABLE PRINCIPLES
 * NO OUTBOUND API CALL MAY BE DISPATCHED WITHOUT RATE LIMIT AND CIRCUIT BREAKER EVALUATION.
 * CIRCUIT BREAKER OPEN STATES MUST FAIL FAST IMMEDIATELY (< 5ms) WITHOUT TOUCHING NETWORK TRANSPORTS.
 * TENANT QUOTAS MUST BE STRICTLY ISOLATED; ONE TENANT'S TRAFFIC BURST MUST NEVER IMPACT ANOTHER TENANT.
 * FALLBACK MODES MUST NEVER SILENTLY FABRICATE BUSINESS CONFIRMATIONS OR ALTER PHASE 3 STATE.
 * LOAD SHEDDING MUST PRESERVE CORE MUTATION CAPABILITIES (BOOKING_CREATE) AT ALL COSTS.
 * ALL RATE LIMIT AND CIRCUIT STATE EVALUATIONS MUST EXECUTE ATOMICALLY IN DISTRIBUTED MEMORY (REDIS LUA).
15. VERSION HISTORY & ARCHITECTURAL VERDICT
| Version | Date | Description | Author | Status |
|---|---|---|---|---|
| 1.0.0 | August 2026 | Initial specification of Rate Limits, Quotas & Resilience architecture. | Ramy Bella | APPROVED |
| 1.0.1 | August 2026 | Comprehensive enterprise specification pass: added complete Sliding Window formulas, atomic Redis Lua rate limiter scripts, mathematical circuit breaker tripping rules, multi-tenant quota tiering, backpressure load shedding hierarchy, and governed fallback execution matrix. | Ramy Bella | APPROVED |
ARCHITECTURAL VERDICT:
APPROVED — DRAFT / IMPLEMENTATION SPECIFICATION
INT-SPEC-17 establishes the mandatory rate limiting, circuit breaking, tenant quota governance, and resilience engine for Phase 5. Ready for implementation.

