# TEST-SPEC-19: Concurrency, Latency, & Backpressure Soak Tests

## 1. DOCUMENT CONTROL

| Attribute | Value |
|---|---|
| Document Title | 19 Concurrency, Latency, & Backpressure Soak Tests.md |
| Document ID | TEST-SPEC-19 |
| Version | 1.0.2 |
| Status | APPROVED FOR IMPLEMENTATION |
| Author | Ramy Bella |
| Classification | Confidential / Enterprise Proprietary |
| Target Audience | SREs, Performance Engineers, Backend Architects, QA Leads |
| Parent Document | TEST-SPEC-01 |
| Related Documents | Phase 5 Specs (INT-SPEC-17), TEST-SPEC-01, TEST-SPEC-02, TEST-SPEC-09 |
| System | Restaurant AI System |
| Phase | Phase 6 — Validation & Verification (V&V) |
| Lifecycle Folder | 07 Performance & Reliability |
| Last Updated | August 2026 |

---

## 2. EXECUTIVE PURPOSE

Generative AI systems and multi-tenant async workers are highly susceptible to resource exhaustion, thread lock contention, and latency degradation under load. A system that works perfectly for one user can suffer catastrophic Out-Of-Memory (OOM) crashes, database connection pool exhaustion, or memory leaks when serving 500 concurrent restaurant guests over a 24-hour period.

`TEST-SPEC-19` defines the **Concurrency, Latency, & Backpressure Soak Test Architecture**. Executed as the core component of the Stage 4 Staging Gate (`TEST-SPEC-02`), it establishes deterministic non-functional requirements (NFRs). It enforces strict SLAs for $P_{95}$ latency, verifies graceful load shedding (HTTP 429) when capacity is exceeded, and executes mandatory 24-hour soak tests to mathematically prove the absence of memory leaks and thread stagnation prior to production deployment.

### Core Testing Invariants:
* `P95 LATENCY > 500ms \implies PERFORMANCE REGRESSION (STAGE 4 PROMOTION HALT)`
* `WORKER OOM (OUT OF MEMORY) DURING SOAK \implies CRITICAL PLATFORM FAILURE`
* `CONCURRENCY > MAX_CAPACITY \implies GRACEFUL LOAD SHEDDING (HTTP 429), NO 500s`
* `MEMORY DELTA > 5% AFTER 24H SOAK \implies MEMORY LEAK DETECTED (SEV-1)`
* `THREAD LOCK CONTENTION \implies ASYNC WORKER PANIC (HARD BUILD BLOCK)`

---

## 3. PERFORMANCE SLAS & THRESHOLD ARCHITECTURE

The system is evaluated against strict latency percentiles and resource ceilings. Averages (Mean) are explicitly rejected as performance metrics in favor of $P_{95}$ and $P_{99}$ percentiles to capture tail-end degradation.

### 3.1. Latency & Resource Thresholds

| Metric | Target Threshold | Degradation Action | Measurement Boundary |
|---|---|---|---|
| **$P_{50}$ (Median) Latency** | $\le 200\text{ ms}$ | Telemetry Warning | Turn ingestion to final token emitted (excluding network I/O). |
| **$P_{95}$ Latency** | $\le 500\text{ ms}$ | `SEV-1` Build Halt | 95% of all turns must process within 500ms under nominal load. |
| **$P_{99}$ Latency** | $\le 1200\text{ ms}$ | `SEV-1` Build Halt | Hard cap for heavy RAG/Multi-Intent DAG resolutions. |
| **TTFT (Time to First Token)** | $\le 300\text{ ms}$ | Telemetry Warning | Applies strictly to streaming audio/chat responses. |
| **Max Memory Delta (Soak)**| $\le 5\%$ variance | `SEV-0` Leak Halt | Delta between Base RAM (Hour 1) and Final RAM (Hour 24). |

---

## 4. LOAD PROFILE & BACKPRESSURE CONTRACT (`LoadTestVerdict@1.0.0`)

To pass Stage 4, the test orchestrator runs three distinct profiles (Spike, Step-Up, and Soak) and emits a schema-validated verdict report.

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "title": "LoadTestVerdict@1.0.0",
  "type": "object",
  "properties": {
    "test_run_id": { "type": "string" },
    "load_profile": { 
      "type": "string",
      "enum": ["SPIKE_TEST", "STEP_UP_CAPACITY", "24H_SOAK"] 
    },
    "metrics": {
      "type": "object",
      "properties": {
        "p95_latency_ms": { "type": "number", "minimum": 0.0 },
        "p99_latency_ms": { "type": "number", "minimum": 0.0 },
        "error_rate_5xx_percent": { "type": "number", "minimum": 0.0, "maximum": 100.0 },
        "shed_rate_429_percent": { "type": "number", "minimum": 0.0, "maximum": 100.0 },
        "memory_leak_delta_percent": { "type": "number" }
      },
      "required": ["p95_latency_ms", "p99_latency_ms", "error_rate_5xx_percent", "shed_rate_429_percent", "memory_leak_delta_percent"]
    },
    "backpressure_verified": { "type": "boolean", "enum": [true] },
    "verdict": { 
      "type": "string",
      "enum": ["PASS", "FAIL_LATENCY_SLA", "FAIL_MEMORY_LEAK", "FAIL_BACKPRESSURE_COLLAPSE"] 
    },
    "executed_at": { "type": "string", "format": "date-time" }
  },
  "required": ["test_run_id", "load_profile", "metrics", "backpressure_verified", "verdict", "executed_at"]
}
```

---

## 5. AUTOMATED LOAD & BACKPRESSURE TEST HARNESS

The harness executes concurrency spikes to guarantee that INT-SPEC-17 backpressure rules safely shed excess load rather than crashing the worker node.

```python
# Representative Automated Backpressure & Load Shedding Test
@pytest.mark.rtm(req_id="PERF-SPEC-19-LOAD-001")
@pytest.mark.asyncio
async def test_graceful_backpressure_load_shedding(load_generator):
    # Setup: System configured for max 100 concurrent requests per node
    MAX_CAPACITY = 100
    OVERSATURATION_TARGET = 150
    
    # Act: Blast 150 concurrent requests simultaneously (Spike Profile)
    responses = await load_generator.fire_concurrent_requests(
        endpoint="/api/v1/chat/turn",
        count=OVERSATURATION_TARGET
    )
    
    # Analyze responses
    success_200s = [r for r in responses if r.status == 200]
    shed_429s = [r for r in responses if r.status == 429]
    crashed_500s = [r for r in responses if r.status >= 500]

    # Compute metrics for the LoadTestVerdict report
    p95_latency = calculate_percentile([r.latency for r in success_200s], 95)
    p99_latency = calculate_percentile([r.latency for r in success_200s], 99)
    error_rate = (len(crashed_500s) / OVERSATURATION_TARGET) * 100
    shed_rate = (len(shed_429s) / OVERSATURATION_TARGET) * 100

    # Emit the schema-validated verdict report (AC-PRF-05 requires every
    # performance evaluation to produce one; the SLA pass/fail judgment
    # itself lives here in the test logic, not in the schema — see Section 4).
    verdict_report = load_orchestrator.generate_verdict(
        test_id="run_19_001",
        profile="SPIKE_TEST",
        p95=p95_latency,
        p99=p99_latency,
        error_rate=error_rate,
        shed_rate=shed_rate,
        memory_delta=0.0  # Not evaluated during the Spike profile; see the 24H_SOAK profile.
    )

    # Assert 0: Report conforms strictly to the LoadTestVerdict@1.0.0 schema
    assert validate_schema(verdict_report.json(), "LoadTestVerdict@1.0.0"), "ERR_TEST_19_05: Verdict report failed schema validation"

    # Assert 1: Zero 5xx Server Errors (System MUST NOT crash under load)
    assert len(crashed_500s) == 0, "ERR_TEST_19_01: Node crashed under oversaturation"
    
    # Assert 2: System successfully processed up to its capacity
    assert len(success_200s) <= MAX_CAPACITY
    
    # Assert 3: System gracefully shed excess load via HTTP 429 (Too Many Requests)
    assert len(shed_429s) > 0, "ERR_TEST_19_02: Backpressure failed to shed load"
    assert (len(success_200s) + len(shed_429s)) == OVERSATURATION_TARGET
    
    # Assert 4: P95 Latency for successful requests remained under SLA
    assert p95_latency <= 500.0, "ERR_TEST_19_03: P95 Latency SLA breached under load"

    # Assert 5: The verdict itself reflects a passing run (evaluated in code, not schema)
    assert verdict_report.verdict == "PASS"
```

---

## 6. SECURITY & THREAT MODEL

| Threat | Attack Vector | Preventive Control | Severity |
|---|---|---|---|
| Slowloris / Connection Exhaustion | Attacker opens thousands of slow HTTP connections to tie up async worker threads. | Reverse proxy (NGINX/Gateway) enforce strict client timeout & connection limits before traffic reaches application workers. | HIGH (SEV-1) |
| Cross-Tenant Resource Starvation | Tenant A blasts traffic, consuming 100% of CPU, starving Tenant B. | Fair-share token buckets (INT-SPEC-17) enforce per-tenant quotas; aggressive 429 shedding per tenant ID. | CRITICAL (SEV-0) |
| OOM DDoS Attack | Attacker sends highly complex multi-intent DAG requests specifically designed to spike RAM. | Hard token limits (TEST-SPEC-11) + DAG depth limits restrict per-request memory allocation bounds. | HIGH (SEV-1) |

---

## 7. FAILURE ARCHITECTURE & ERROR TAXONOMY

| Error Code | Description | Canonical Class | Severity |
|---|---|---|---|
| ERR_TEST_19_01 | System returned HTTP 5xx Server Errors during load saturation instead of gracefully shedding traffic. | RESOURCE_EXHAUSTION | CRITICAL (SEV-0) |
| ERR_TEST_19_02 | Backpressure mechanism (INT-SPEC-17) failed to emit HTTP 429 Too Many Requests when capacity exceeded. | LOGIC_FAIL | HIGH (SEV-1) |
| ERR_TEST_19_03 | $P_{95}$ Latency SLA ($\le 500\text{ ms}$) breached during nominal or step-up load profiles. | PERFORMANCE_DEGRADATION | HIGH (SEV-1) |
| ERR_TEST_19_04 | Memory leak detected: Baseline RAM vs Final RAM delta exceeded 5% during 24-hour soak test. | MEMORY_LEAK | CRITICAL (SEV-0) |
| ERR_TEST_19_05 | LoadTestVerdict payload failed JSON Schema validation (LoadTestVerdict@1.0.0). | VALIDATION | HIGH (SEV-1) |

---

## 8. PRODUCTION ACCEPTANCE CRITERIA

| AC ID | Category | Requirement | Verification Method | Pass/Fail |
|---|---|---|---|---|
| AC-PRF-01 | $P_{95}$ SLA Compliance | 100% of Stage 4 test runs maintain $P_{95} \le 500\text{ ms}$ and $P_{99} \le 1200\text{ ms}$. | Metric Aggregator Audit | REQUIRED |
| AC-PRF-02 | Zero 5xx Load Crashes | 0 HTTP 5xx errors recorded during standard 150% capacity spike tests. | Spike Load Harness | REQUIRED |
| AC-PRF-03 | Backpressure Shedding | Traffic exceeding worker concurrency limits correctly triggers immediate HTTP 429 shedding. | Saturation Test Suite | REQUIRED |
| AC-PRF-04 | 24-Hour Soak Stability | Memory consumption variance remains $\le 5\%$ across a continuous 24-hour standard soak test. | Soak Profiler | REQUIRED |
| AC-PRF-05 | Schema Enforcement | 100% of performance evaluation verdicts validate against LoadTestVerdict@1.0.0. | Schema Inspector | REQUIRED |

---

## 9. INTEGRATION AUTHORITY MATRIX

| Component | Owns | Consumes | Produces |
|---|---|---|---|
| TEST-SPEC-19 | Latency SLAs, Load Test Profiles (Spike, Soak), Memory Leak Detection, Load Verdicts | Simulated API Traffic | LoadTestVerdict, Stage 4 Pass/Fail Signals |
| INT-SPEC-17 | Authoritative Rate Limiting, Backpressure & Tenant Quotas | Ingress Traffic Metrics | HTTP 429 Shedding Responses |
| TEST-SPEC-02 | CI/CD Stage 4 Gate Enforcement | Load Verdicts (this document) | Production Release Authorization |
| TEST-SPEC-11 | DoW & Token Exhaustion Defenses | Token Budget Counters | Truncation / Fast-Fail Events |

---

## 10. FINAL NON-NEGOTIABLE PRINCIPLES

AVERAGE (MEAN) LATENCIES ARE INVALID FOR PERFORMANCE SIGN-OFF; ONLY P95 AND P99 PERCENTILES MAY BE USED.

THE SYSTEM MUST NEVER CRASH (HTTP 5XX) UNDER EXCESSIVE LOAD; IT MUST SHED LOAD GRACEFULLY VIA HTTP 429.

NO SYSTEM BUILD SHALL BE PROMOTED TO PRODUCTION WITHOUT PASSING A DETERMINISTIC MEMORY SOAK TEST.

PERFORMANCE VERDICTS MUST BE LOGGED AS STRICTLY TYPED, SCHEMA-VALIDATED REPORTS.

CROSS-TENANT RESOURCE STARVATION IS A SEV-0 BREACH; HEAVY LOAD FROM TENANT A MUST NEVER DEGRADE TENANT B.

---

## 11. VERSION HISTORY & ARCHITECTURAL VERDICT

| Version | Date | Description | Author | Status |
|---|---|---|---|---|
| 1.0.0 | August 2026 | Initial specification of Concurrency, Latency, & Backpressure Soak Tests (TEST-SPEC-19). | Ramy Bella | SUPERSEDED |
| 1.0.1 | August 2026 | Review fix pass. Added the missing top-level document title (the file previously opened directly at Section 1 with no `#` title line) — the same defect class already fixed in TEST-SPEC-14. Fixed a malformed `$schema` URL in `LoadTestVerdict@1.0.0` (markdown link syntax had been baked into the JSON string, making it invalid JSON as written) — the same defect class already fixed in TEST-SPEC-07/12/13/15/16. No change to the performance thresholds, load profile architecture, error codes, or acceptance criteria. | Ramy Bella | SUPERSEDED |
| 1.0.2 | August 2026 | External technical review fix pass. Resolved a schema/prose contract mismatch in `LoadTestVerdict@1.0.0` where the SLA thresholds themselves ($P_{95} \le 500\text{ms}$, $P_{99} \le 1200\text{ms}$, 0% error rate, $\le 5\%$ memory delta) were hardcoded as hard JSON Schema `maximum` constraints — meaning a report describing an actual SLA breach (e.g. `"verdict": "FAIL_LATENCY_SLA"`) could never itself validate against the schema, silently masking every real performance failure as a schema error (`ERR_TEST_19_05`) instead. Metrics are now type-bounded only (non-negative latencies, 0-100% rates); SLA pass/fail judgment is evaluated by the test logic and AC-PRF-01/02/04, not the schema — the same fix class already applied to `token_count` in TEST-SPEC-16 v1.0.3. Added the missing `LoadTestVerdict` construction and `Assert 0` schema-validation step to the representative test harness in Section 5, closing a gap where AC-PRF-05 required every performance evaluation to produce a schema-validated verdict but the representative test never actually built or validated one — the same `Assert 0` pattern already established in TEST-SPEC-17/18. No change to the latency/memory thresholds themselves, error codes, or acceptance criteria. | Ramy Bella | APPROVED FOR IMPLEMENTATION |

**ARCHITECTURAL VERDICT:**
APPROVED FOR IMPLEMENTATION. TEST-SPEC-19 v1.0.2 establishes the deterministic $P_{95}$ latency SLAs, backpressure (HTTP 429) load shedding verification, memory leak soak harnesses, and the LoadTestVerdict@1.0.0 contract — with SLA judgment cleanly separated from schema type-validation — for Phase 6. Ready for implementation.
