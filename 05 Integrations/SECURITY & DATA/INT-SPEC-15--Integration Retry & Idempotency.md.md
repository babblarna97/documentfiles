# INT-SPEC-15: Integration Retry & Idempotency

## 1. DOCUMENT CONTROL

| Attribute | Value |
|---|---|
| Document Title | 15 Integration Retry & Idempotency.md |
| Document ID | INT-SPEC-15 |
| Version | 1.0.1 |
| Status | DRAFT / Implementation Specification |
| Author | Ramy Bella |
| Classification | Confidential / Enterprise Proprietary |
| Target Audience | Reliability Engineers, Backend Engineers, Integration Architects, Distributed Systems Engineers, Security Architects, Database Engineers |
| Parent Document | INT-SPEC-01 |
| Related Documents | INT-SPEC-01 through INT-SPEC-14, INT-SPEC-16 through INT-SPEC-20, Phase 4 Specifications |
| System | Restaurant AI System |
| Phase | Phase 5 — Integrations |
| Lifecycle Folder | RELIABILITY & EXECUTION |
| Last Updated | August 2026 |

---

## 2. EXECUTIVE PURPOSE

Transient errors in distributed third-party APIs are inevitable. Network sockets disconnect, remote endpoints undergo transient throttling, and TCP packets drop post-dispatch. Without a mathematically bounded recovery mechanism and a stateful deduplication ledger, automated retry loops trigger catastrophic side-effects: duplicate table bookings, multiple email dispatches to guests, corrupted financial/CRM logs, and severe thundering herd problems on downstream servers.

`INT-SPEC-15` defines the complete, deterministic enterprise architecture for retries, exponential backoff, stochastic jitter, and distributed idempotency across the Phase 5 integration layer. It guarantees that any stateful mutation (`POST`, `PUT`, `DELETE`) is executed **exactly once** in the external provider system, regardless of network retries, worker crashes, or concurrent execution attempts.

### Core Architectural Invariants:
* `RETRY EXECUTION \neq UNBOUNDED REPEAT`
* `STATEFUL MUTATION \implies MANDATORY IDEMPOTENCY KEY`
* `IDEMPOTENCY LEDGER RECORD \neq BUSINESS STATE`
* `UNKNOWN EXECUTION STATE \implies RECONCILIATION BEFORE MUTATION RE-DISPATCH`
* `EXPIRED execution_budget \implies PERMANENT FAILURE ESCALATION`

---

## 3. PURPOSE AND SCOPE

### Scope: What INT-SPEC-15 Controls
* **Mathematical Backoff & Jitter Engine:** Bounded algorithms for exponential backoff with stochastic full jitter to prevent Thundering Herd alignment.
* **Idempotency Key Synthesis:** Cryptographic, multi-variate hashing pipeline generating deterministic keys for all mutating payloads.
* **Atomic Idempotency Ledger State Machine:** Atomic state lifecycle (`NON_EXISTENT` $\rightarrow$ `PENDING` $\rightarrow$ `COMPLETED` / `FAILED`) governing concurrent executions.
* **Redis Lua Script Execution:** Atomic lock acquisition and state transition scripts preventing race conditions under sub-millisecond concurrency.
* **Retry Envelope Protocol:** Canonical JSON schema encapsulating transient error state, execution counts, budgets, and `INT-SPEC-13` tenant context for queue propagation.
* **Reconciliation Bridge:** Protocol for verifying ambiguous (`UNKNOWN`) execution outcomes prior to dispatching secondary attempts.
* **Ledger Memory & TTL Governance:** Storage tiers, namespace prefixing, and memory eviction safeguards for Redis and PostgreSQL.

### Scope: What INT-SPEC-15 Explicitly Does NOT Control
* **Error Normalization & Classification:** Governed unconditionally by `INT-SPEC-14`.
* **Circuit Breaker State Tripping & Thresholds:** Governed by `INT-SPEC-17`.
* **Telemetry & Trace Emission Schema:** Governed by `INT-SPEC-18`.
* **Business Authorization & Cancellation Policy:** Governed unconditionally by Phase 3 (Conversation Engine).

---

## 4. ARCHITECTURAL POSITION

`INT-SPEC-15` operates as the execution reliability layer directly downstream of the `INT-SPEC-14` Error Normalization Pipeline.



[INT-SPEC-14 / CANONICAL ERROR EVALUATED] ↓ (Condition: retryable == true) [INT-SPEC-15 / RETRY & IDEMPOTENCY ENGINE] ├─ 1. Evaluate Capability Mutation Type (Stateful vs Read-Only) ├─ 2. Compute Payload Digest & Assert Idempotency Key ├─ 3. Query Distributed Idempotency Ledger (Redis Lua Script) │ ├─ If PENDING (In-Flight Lock) → Requeue / Suppress Duplicate │ ├─ If COMPLETED → Return Cached Response (Zero Provider Call) │ └─ If FAILED / NON-EXISTENT → Acquire Lock & Proceed ├─ 4. Evaluate Execution Budget & Attempt Limits ├─ 5. Calculate Delay (Exponential Backoff + Full Jitter) └─ 6. Wrap Payload in INT-SPEC-15 Retry Envelope & Dispatch ↓ [IDEMPOTENCY LEDGER (Redis SETNX + Postgres)] ↓ [PROVIDER ADAPTER DISPATCH]
---

## 5. MATHEMATICAL BACKOFF & JITTER MODEL

To prevent thundering herd collisions when an upstream service recovers from an outage, all retries MUST apply exponential backoff combined with stochastic **Full Jitter**.

### 1. Mathematical Backoff Formulation

$$T_{\text{sleep}} = \min\left(T_{\text{max}}, T_{\text{base}} \times 2^{\text{attempt}}\right)$$

$$T_{\text{backoff}} = \text{random\_uniform}\left(0, T_{\text{sleep}}\right)$$

* **$T_{\text{base}}$:** Base initial delay (Default: $500\text{ ms}$).
* **$T_{\text{max}}$:** Maximum upper bound delay cap (Default: $30{,}000\text{ ms}$).
* **$\text{attempt}$:** Current zero-based attempt counter ($0, 1, 2, \dots, N_{\text{max}} - 1$).
* **$\text{random\_uniform}(0, X)$:** Pseudo-random value drawn uniformly from the continuous interval $[0, X]$.

### 2. Execution Budgets & Attempt Boundaries

| Execution Surface | Capability Type | Max Attempts ($N_{\text{max}}$) | Max Execution Budget ($T_{\text{budget}}$) | Backoff Formula |
|---|---|---|---|---|
| **Synchronous (Widget / API)** | Read / Query (`GET`) | 2 Attempts | $3{,}000\text{ ms}$ | Full Jitter ($T_{\text{base}}=200\text{ms}$) |
| **Synchronous (Widget / API)** | Stateful (`POST`/`PUT`) | 1 Attempt (0 Retries)* | $4{,}000\text{ ms}$ | No Retry (Reconcile / Re-query) |
| **Asynchronous (Queue Worker)** | Stateful Mutation | 5 Attempts | $86{,}400\text{ s}$ (24 Hours) | Full Jitter ($T_{\text{base}}=1000\text{ms}$) |
| **Asynchronous (Webhooks)** | Event Ingestion | 8 Attempts | $172{,}800\text{ s}$ (48 Hours) | Exponential Backoff + Jitter |

*\*Note: Synchronous mutation calls do not execute inline network retries unless protected by an explicit idempotency key confirmed by the distributed ledger.*

---

## 6. IDEMPOTENCY KEY SYNTHESIS PIPELINE

Every state-changing capability dispatch MUST synthesize a deterministic, non-colliding idempotency key prior to transport execution.

### 1. Cryptographic Key Formula

$$\text{idempotency\_key} = \text{SHA256}\Big(\text{environment} \mathbin{\Vert} \text{tenant\_id} \mathbin{\Vert} \text{capability} \mathbin{\Vert} \text{correlation\_id} \mathbin{\Vert} \text{payload\_digest}\Big)$$

Where:
* $\text{payload\_digest} = \text{SHA256}(\text{Canonical JSON Payload with sorted keys})$.
* $\mathbin{\Vert}$ represents string concatenation using an explicit colon delimiter (`:`).

### 2. Redis Key Namespacing Schema
To satisfy `INT-SPEC-13` Tenant & Environment Isolation, the key MUST be formatted in Redis as:

$$\text{Redis Key} = \text{env} : \text{tenant\_id} : \text{capability} : \text{idempotency\_key}$$

---

## 7. DISTRIBUTED IDEMPOTENCY LEDGER STATE MACHINE

The idempotency ledger enforces strict execution deduplication using a 4-state lifecycle.



┌──────────────────┐ │ NON_EXISTENT │ └────────┬─────────┘ │ │ Atomic Acquisition (SETNX) ▼ ┌──────────────────┐ │ PENDING │ ← In-flight Lock (TTL: 60s) └───┬──────────┬───┘ │ │ Execution │ │ Execution Fatal Error Success │ │ OR Budget Exhausted ▼ ▼ ┌───────────┐ ┌───────────┐ │ COMPLETED │ │ FAILED │ └───────────┘ └───────────┘
### State Definitions & Behavioral Rules:
1. **`NON_EXISTENT`:** No prior record exists. The system attempts to acquire an execution lock.
2. **`PENDING`:** Execution is in-flight. The key is locked in Redis with a 60-second TTL. Concurrent duplicate dispatches receive an `ERR_INT_15_01` (`IDEMPOTENCY_IN_PROGRESS`) response or are queued in background workers.
3. **`COMPLETED`:** The capability executed successfully. The canonical `INT-SPEC-02` response payload is cached in the ledger for 24 hours. Any subsequent request with the same idempotency key **bypasses the external provider completely** and returns the stored result.
4. **`FAILED`:** The execution failed fatally or exceeded its attempt budget. The lock is cleared or marked failed, allowing a new attempt under a distinct correlation context.

---

## 8. ATOMIC REDIS LUA SCRIPTING FOR STATE TRANSITIONS

To prevent race conditions under high-concurrency environments, ledger state checks and acquisitions MUST execute atomically via Lua scripts on the Redis cluster.

### 1. Atomic Lock Acquisition Script (`acquire_idempotency_lock.lua`)

```lua
-- KEYS[1]: Full Redis Key (env:tenant_id:capability:idempotency_key)
-- ARGV[1]: Lock Value (correlation_id)
-- ARGV[2]: Lock TTL in seconds (60)

local current_state = redis.call("HGET", KEYS[1], "state")

if not current_state then
    -- Key does not exist. Acquire PENDING lock.
    redis.call("HSET", KEYS[1], "state", "PENDING", "owner", ARGV[1], "created_at", redis.call("TIME")[1])
    redis.call("EXPIRE", KEYS[1], tonumber(ARGV[2]))
    return "ACQUIRED"
elseif current_state == "PENDING" then
    -- Execution currently in-flight.
    return "IN_PROGRESS"
elseif current_state == "COMPLETED" then
    -- Execution already succeeded. Return cached response.
    local response = redis.call("HGET", KEYS[1], "response_payload")
    return "COMPLETED:" .. response
elseif current_state == "FAILED" then
    -- Execution previously failed.
    return "FAILED"
end


2. Atomic Lock Completion Script (complete_idempotency_lock.lua)
-- KEYS[1]: Full Redis Key
-- ARGV[1]: Owner (correlation_id)
-- ARGV[2]: Normalized Response Payload (JSON String)
-- ARGV[3]: Response TTL in seconds (86400 = 24 Hours)

local owner = redis.call("HGET", KEYS[1], "owner")

if owner == ARGV[1] then
    redis.call("HSET", KEYS[1], "state", "COMPLETED", "response_payload", ARGV[2], "completed_at", redis.call("TIME")[1])
    redis.call("EXPIRE", KEYS[1], tonumber(ARGV[3]))
    return "SUCCESS"
else
    return "OWNERSHIP_MISMATCH"
end


9. RETRY ENVELOPE SCHEMA (RetryEnvelope@1.0.0)
When a transient error triggers an asynchronous retry, the payload MUST be wrapped in a standardized envelope that preserves trace context, attempt counters, and INT-SPEC-13 tenant bindings across queue boundaries.
{
  "envelope_id": "env_01HXYZ9876543210",
  "attempt_count": 2,
  "max_attempts": 5,
  "first_dispatched_at": "2026-08-19T08:00:00.000Z",
  "next_retry_at": "2026-08-19T08:00:04.250Z",
  "execution_budget_expires_at": "2026-08-20T08:00:00.000Z",
  "isolation_context": {
    "tenant_id": "tenant_restaurant_a",
    "venue_id": "venue_main_dining",
    "environment": "production"
  },
  "execution_metadata": {
    "capability": "BOOKING_CREATE",
    "provider": "resy_primary",
    "correlation_id": "corr_123456789",
    "idempotency_key": "idem_sha256_abcd1234efgh5678..."
  },
  "last_canonical_error": {
    "error_code": "ERR_BOOKING_05",
    "error_class": "TIMEOUT",
    "severity": "HIGH"
  },
  "canonical_payload": {
    "party_size": 4,
    "booking_date": "2026-08-20",
    "booking_time": "19:00:00"
  }
}


10. RECONCILIATION & AMBIGUOUS (UNKNOWN) EXECUTION PROTOCOL
When a network timeout occurs after a state-changing mutation payload has been transmitted over the wire, the system faces an ambiguous execution state (UNKNOWN). Issuing a blind retry risks duplicate side-effects if the provider actually processed the request before dropping the socket.
[TIMEOUT OCCURS POST-DISPATCH]
        ↓
[INT-SPEC-14 Marks Result: UNKNOWN]
        ↓
[INT-SPEC-15 Reconciliation Triggered]
        │
        ├─ Step 1: Query Idempotency Ledger for External Provider Ref
        ├─ Step 2: Issue Provider Status Lookup (BOOKING_GET / STATUS_CHECK)
        │            │
        │            ├─ If Record Exists in Provider → Mark Ledger COMPLETED & Return
        │            └─ If Record ABSENT in Provider → Safe to Issue Retry Dispatch
        └─ Step 3: If Status Lookup Unsupported → Escalate to Human/Phase 3 Recovery


11. SECURITY & THREAT MODEL
Threat
Attack Vector
Preventive Control
Severity
Replay Attack / Duplicate Booking
Malicious or buggy client sending identical payload repeatedly.
Atomic idempotency locking (acquire_idempotency_lock.lua) on all mutations.
CRITICAL
Ledger Poisoning
Injecting fake success responses into Redis.
Strict validation: only responses passing INT-SPEC-02 schema validation can transition state to COMPLETED.
HIGH
Cross-Tenant Key Collision
Identical payload hashes colliding between distinct venues.
Mandatory inclusion of tenant_id and environment in the Redis key namespace (INT-SPEC-13).
CRITICAL
Stranded Lock Deadlock
Worker process killed via SIGKILL while holding PENDING state.
Mandatory 60-second TTL on PENDING states using Redis automatic key expiration.
HIGH
Thundering Herd Burst
Provider outage recovery causes thousands of concurrent worker retries.
Mandatory Full Jitter stochastic randomization applied to all backoff calculations.
HIGH

12. FAILURE ARCHITECTURE & ERROR CODES
Error Code
Description
Canonical Class
Severity
ERR_INT_15_01
Idempotency lock contention (PENDING state currently held by active execution).
IDEMPOTENCY
HIGH
ERR_INT_15_02
Max retry attempts (N_{\text{max}}) or execution time budget (T_{\text{budget}}) exhausted.
TIMEOUT
HIGH
ERR_INT_15_03
Idempotency ledger storage engine (Redis/Postgres) unavailable.
PROVIDER_UNAVAILABLE
CRITICAL
ERR_INT_15_04
Stateful mutation payload submitted without mandatory idempotency metadata.
VALIDATION
HIGH
ERR_INT_15_05
Reconciliation check failed to determine outcome of UNKNOWN state.
CONTRACT_MISMATCH
HIGH

13. PRODUCTION ACCEPTANCE CRITERIA
AC ID
Category
Requirement
Verification Method
Pass/Fail
AC-RTY-01
Jitter Uniformity
1,000 concurrent retries distribute uniformly across the backoff window without burst spikes.
Concurrency Distribution Test
REQUIRED
AC-RTY-02
Atomic Deduplication
10 concurrent requests with identical idempotency keys execute provider API exactly once.
Race Condition Mock
REQUIRED
AC-RTY-03
Tenant Isolation
Identical payload hashes across Tenant A and Tenant B yield isolated, non-colliding Redis keys.
Multi-Tenant Verification Test
REQUIRED
AC-RTY-04
Stranded Lock Recovery
Killing a worker process holding a PENDING lock auto-releases the lock after 60 seconds.
Chaos Termination Test
REQUIRED
AC-RTY-05
Budget Enforcement
A background queue retry job exceeding its 24-hour expiration budget escalates to permanent failure.
Time-Travel Execution Test
REQUIRED
AC-RTY-06
Reconciliation
Ambiguous timeout on BOOKING_CREATE triggers status check before issuing secondary mutation attempt.
Fault Injection Simulation
REQUIRED

14. INTEGRATION AUTHORITY MATRIX
Component
Owns
Consumes
Produces
INT-SPEC-15
Backoff Formulas, Jitter Math, Idempotency Ledger, Lock State Machine
INT-SPEC-14 Error Objects, Payload Hashes
Deduplicated Execution Dispatches, RetryEnvelope Payloads
INT-SPEC-13
Tenant & Environment Isolation Namespaces
System Execution Context
Validated Redis Namespace Prefixes (env:tenant:)
INT-SPEC-18
Retry Metrics & Audit Trail Telemetry
Retry Attempts, Backoff Events, Ledger Transitions
Operational Metrics, APM Traces, Alert Signals
Phase 3 (CE)
Business Truth & Intent Authorization
Deduplicated Execution Outcomes, Permanent Failure Escalations
Authoritative Business State Updates

15. FINAL NON-NEGOTIABLE PRINCIPLES
ALL STATEFUL INTEGRATION MUTATIONS (POST, PUT, DELETE) MUST BE PROTECTED BY IDEMPOTENCY KEYS.
NO RETRIES MAY BE ISSUED WITHOUT STOCHASTIC FULL JITTER.
AN AMBIGUOUS UNKNOWN EXECUTION STATE MUST BE RECONCILED BEFORE DISPATCHING A NEW MUTATION.
IDEMPOTENCY KEYS AND LEDGER RECORDS MUST BE STRICTLY ISOLATED PER TENANT AND ENVIRONMENT.
MAX RETRY BUDGETS AND ATTEMPT COUNTS MUST BE STRICTLY ENFORCED; EXHAUSTED BUDGETS ESCALATE PERMANENTLY TO PHASE 3.
THE IDEMPOTENCY LEDGER IS AN EXECUTION DEDUPLICATION MECHANISM; IT DOES NOT OVERRIDE AUTHORITATIVE PHASE 3 BUSINESS STATE.
16. VERSION HISTORY & ARCHITECTURAL VERDICT
Version
Date
Description
Author
Status
1.0.0
August 2026
Initial specification of Integration Retry & Idempotency architecture.
Ramy Bella
APPROVED
1.0.1
August 2026
Expanded complete enterprise specification: added explicit Full Jitter math, Redis Lua scripts for atomic locks, atomic ledger state machine, RetryEnvelope@1.0.0 schema, and reconciliation protocol.
Ramy Bella
APPROVED

ARCHITECTURAL VERDICT:
APPROVED — DRAFT / IMPLEMENTATION SPECIFICATION INT-SPEC-15 establishes the complete mathematical backoff, atomic idempotency deduplication, and execution reliability engine for Phase 5. Ready for implementation.

