# TEST-SPEC-20: Shadow Traffic Replay & Validation

## 1. DOCUMENT CONTROL

| Attribute | Value |
|---|---|
| Document Title | 20 Shadow Traffic Replay & Validation.md |
| Document ID | TEST-SPEC-20 |
| Version | 1.0.1 |
| Status | APPROVED FOR IMPLEMENTATION |
| Author | Ramy Bella |
| Classification | Confidential / Enterprise Proprietary |
| Target Audience | MLOps Engineers, Data Privacy Officers, QA Architects, SREs |
| Parent Document | TEST-SPEC-01 |
| Related Documents | Phase 5 Specs (INT-SPEC), TEST-SPEC-01, TEST-SPEC-02, TEST-SPEC-17, TEST-SPEC-18, TEST-SPEC-19 |
| System | Restaurant AI System |
| Phase | Phase 6 — Validation & Verification (V&V) |
| Lifecycle Folder | 07 Performance & Reliability |
| Last Updated | August 2026 |

---

## 2. EXECUTIVE PURPOSE

Synthetic tests, unit tests, and red-teaming are essential but cannot capture the full entropy of real-world human behavior (typos, slang, chaotic multi-intent requests). The final verification before a new AI model or prompt architecture reaches production is **Shadow Traffic Replay (Dark Launching)**.

`TEST-SPEC-20` defines the architecture for capturing real production user traffic, sanitizing it of PII (Personally Identifiable Information), and replaying it against the Release Candidate (Shadow) environment. It measures if the new system generates semantically equivalent or superior responses compared to the live Production system, without causing unauthorized side-effects (e.g., executing fake bookings against a live POS). Executed as the final Stage 4 validation (`TEST-SPEC-02`), it provides empirical proof of non-regression under real-world conditions.

### Core Testing Invariants:
* `SHADOW EXECUTION MUTATES PROD DATABASE \implies SEV-0 CRITICAL ISOLATION BREACH`
* `PII DETECTED IN SHADOW REPLAY LOGS \implies SEV-0 GDPR/PRIVACY VIOLATION`
* `SEMANTIC DRIFT > 2% ON REAL TRAFFIC \implies PIPELINE DEPLOYMENT HALT`
* `SHADOW LATENCY > PROD LATENCY BY 20% \implies PERFORMANCE REGRESSION`
* `SHADOW REPLAY VERDICTS MUST BE AUTOMATED VIA LLM JUDGE (TEST-SPEC-17)`

---

## 3. SHADOW REPLAY ARCHITECTURE & ISOLATION BOUNDARIES

The architecture relies on a read-only fork of production traffic that is asynchronously scrubbed and replayed.

```text
[LIVE PRODUCTION GATEWAY]
           │
           ├─ (1. Synchronous Response to Real User) ──► [User Device]
           │
           ▼ (2. Asynchronous Fork)
[PII SANITIZATION ENGINE]
  ├─ Strips real phone numbers, emails, and names.
  ├─ Replaces with synthetic equivalents (e.g., `+46700000000`).
           │
           ▼ (3. Replay Queue)
[SHADOW ENVIRONMENT (Release Candidate)]
  ├─ Context: Uses same System Prompt / Model as upcoming release.
  ├─ Isolation: Integrations strictly bound to Mock Gateways (`TEST-SPEC-18`).
           │
           ▼ (4. Output Comparison)
[SEMANTIC COMPARATOR (TEST-SPEC-17 JUDGE)]
  ├─ Compares $Response_{\text{prod}}$ vs $Response_{\text{shadow}}$.
  └─ Evaluates intent accuracy, tone, and deterministic tool calls.
```

---

## 4. SHADOW REPLAY VERDICT CONTRACT (`ShadowReplayVerdict@1.0.0`)

Every shadow replay batch MUST emit a schema-validated verdict. The JSON schema enforces types and bounded enums; the test harness evaluates the threshold SLAs.

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "title": "ShadowReplayVerdict@1.0.0",
  "type": "object",
  "additionalProperties": false,
  "properties": {
    "replay_batch_id": { "type": "string", "minLength": 1 },
    "traffic_sample_size": { "type": "integer", "minimum": 100 },
    "isolation_verified": { "type": "boolean" },
    "pii_leak_detected": { "type": "boolean" },
    "metrics": {
      "type": "object",
      "properties": {
        "semantic_match_percent": { "type": "number", "minimum": 0.0, "maximum": 100.0 },
        "tool_call_match_percent": { "type": "number", "minimum": 0.0, "maximum": 100.0 },
        "shadow_p95_latency_ms": { "type": "number", "minimum": 0.0 },
        "prod_p95_baseline_ms": { "type": "number", "minimum": 0.0 }
      },
      "required": ["semantic_match_percent", "tool_call_match_percent", "shadow_p95_latency_ms", "prod_p95_baseline_ms"]
    },
    "verdict": { 
      "type": "string",
      "enum": ["PASS", "FAIL_SEMANTIC_DRIFT", "FAIL_TOOL_MISMATCH", "FAIL_ISOLATION_BREACH", "FAIL_PII_LEAK", "FAIL_LATENCY_REGRESSION"] 
    },
    "executed_at": { "type": "string", "format": "date-time" }
  },
  "required": ["replay_batch_id", "traffic_sample_size", "isolation_verified", "pii_leak_detected", "metrics", "verdict", "executed_at"]
}
```

---

## 5. AUTOMATED SHADOW REPLAY TEST HARNESS

The test harness pulls a sanitized traffic log, executes the replay, runs the semantic judge, and validates the schema contract.

```python
# Representative Automated Shadow Replay Test Harness
@pytest.mark.rtm(req_id="PRF-SPEC-20-SHD-001")
def test_shadow_traffic_semantic_parity(shadow_orchestrator, pii_scanner, mock_adapters):
    # Setup: Fetch sanitized traffic batch
    traffic_batch = shadow_orchestrator.fetch_sanitized_batch(size=1000)
    
    # Act: Replay traffic against the shadow environment
    replay_results = shadow_orchestrator.execute_replay(traffic_batch)
    
    # Act: Judge comparisons using TEST-SPEC-17 semantic engine
    semantic_match_rate = calculate_semantic_parity(replay_results)
    tool_match_rate = calculate_tool_call_parity(replay_results)

    # Act: Measure shadow latency against the current production baseline
    shadow_p95 = calculate_percentile([r.latency for r in replay_results], 95)
    prod_p95_baseline = shadow_orchestrator.fetch_prod_p95_baseline()
    
    # Act: Generate Verdict Schema
    verdict_report = shadow_orchestrator.generate_verdict(
        batch_id="shadow_run_001",
        sample_size=len(traffic_batch),
        isolation_verified=mock_adapters.verify_zero_live_calls(),
        pii_leak_detected=pii_scanner.scan_logs_for_pii(replay_results),
        semantic_match=semantic_match_rate,
        tool_match=tool_match_rate,
        p95_latency=shadow_p95,
        prod_p95_baseline=prod_p95_baseline
    )

    # Assert 0: Report strictly conforms to the JSON Schema contract
    assert validate_schema(verdict_report.json(), "ShadowReplayVerdict@1.0.0"), "ERR_TEST_20_04: Verdict failed schema validation"

    # Assert 1: Absolute Data Privacy & Isolation
    assert verdict_report.pii_leak_detected == False, "ERR_TEST_20_01: PII detected in shadow replay logs"
    assert verdict_report.isolation_verified == True, "ERR_TEST_20_02: Shadow environment contacted live production API"

    # Assert 2: Semantic & Tool Call SLA Verification (Evaluated here, not in schema)
    assert verdict_report.metrics.semantic_match_percent >= 98.0, "ERR_TEST_20_03: Semantic drift exceeded 2% threshold"
    assert verdict_report.metrics.tool_call_match_percent == 100.0, "ERR_TEST_20_03: Tool call execution diverged from production"

    # Assert 3: Shadow latency must not regress more than 20% versus the production baseline
    latency_regression_percent = (
        (verdict_report.metrics.shadow_p95_latency_ms - verdict_report.metrics.prod_p95_baseline_ms)
        / verdict_report.metrics.prod_p95_baseline_ms
    ) * 100
    assert latency_regression_percent <= 20.0, "ERR_TEST_20_05: Shadow P95 latency regressed more than 20% versus production baseline"

    # Assert 4: Verdict pass
    assert verdict_report.verdict == "PASS"
```

---

## 6. SECURITY & THREAT MODEL

| Threat | Attack Vector | Preventive Control | Severity |
|---|---|---|---|
| Live Database Mutation | Shadow environment executes a booking API call, creating phantom reservations in the live restaurant POS. | Shadow deployment strictly inherits Mock Adapters (TEST-SPEC-18); network egress to live POS endpoints is blocked via VPC rules. | CRITICAL (SEV-0) |
| PII Data Leakage | Production chat logs containing guest phone numbers are mirrored to lower-security staging logs. | PII Sanitization Engine runs deterministically on the prod-side before logs enter the replay queue. | CRITICAL (SEV-0) |
| Silent API Contract Break | A new model outputs JSON tool calls correctly, but uses a date format the POS rejects. | 100% Tool Call Match SLA ensures the shadow output is structurally identical to the validated prod output. | HIGH (SEV-1) |

---

## 7. FAILURE ARCHITECTURE & ERROR TAXONOMY

| Error Code | Description | Canonical Class | Severity |
|---|---|---|---|
| ERR_TEST_20_01 | Un-sanitized PII (e.g., real phone numbers) detected in shadow environment context or logs. | DATA_LEAKAGE | CRITICAL (SEV-0) |
| ERR_TEST_20_02 | Shadow environment attempted or succeeded in executing a side-effect on a live production API. | SECURITY_BOUNDARY | CRITICAL (SEV-0) |
| ERR_TEST_20_03 | Semantic match or Tool Call match fell below the strict parity SLAs during replay. | PROBABILISTIC_DRIFT | HIGH (SEV-1) |
| ERR_TEST_20_04 | ShadowReplayVerdict payload failed JSON Schema validation (ShadowReplayVerdict@1.0.0). | VALIDATION | HIGH (SEV-1) |
| ERR_TEST_20_05 | Shadow environment P95 latency exceeded the production baseline P95 latency by more than 20%. | PERFORMANCE_DEGRADATION | HIGH (SEV-1) |

---

## 8. PRODUCTION ACCEPTANCE CRITERIA

| AC ID | Category | Requirement | Verification Method | Pass/Fail |
|---|---|---|---|---|
| AC-SHD-01 | Zero State Mutation | 100% of shadow traffic tool calls are routed to mock adapters (TEST-SPEC-18); 0 live DB mutations. | VPC / Adapter Audit | REQUIRED |
| AC-SHD-02 | Zero PII Leakage | 0 instances of real PII (names, numbers, emails) migrate from production into shadow logs. | PII Scanner Audit | REQUIRED |
| AC-SHD-03 | Semantic Parity | $\ge 98\%$ semantic similarity between production outputs and shadow outputs on identical sanitized inputs. | TEST-SPEC-17 Judge | REQUIRED |
| AC-SHD-04 | Tool Call Exactness | 100% structural match required for all underlying API tool calls/JSON payloads generated during replay. | Payload Diff Checker | REQUIRED |
| AC-SHD-05 | Schema Enforcement | 100% of shadow evaluation verdicts validate against ShadowReplayVerdict@1.0.0. | Schema Inspector | REQUIRED |
| AC-SHD-06 | Latency Regression Bound | Shadow environment $P_{95}$ latency does not exceed the production baseline $P_{95}$ latency by more than 20%. | Latency Delta Audit | REQUIRED |

---

## 9. INTEGRATION AUTHORITY MATRIX

| Component | Owns | Consumes | Produces |
|---|---|---|---|
| TEST-SPEC-20 | Shadow Traffic Replay Orchestration, PII Stripping Validation, Replay Verdicts | Sanitized Prod Logs | ShadowReplayVerdict, Stage 4 Sign-off |
| TEST-SPEC-17 | Semantic Drift Judging Formulas | Prod vs Shadow Outputs | Semantic Match Scores |
| TEST-SPEC-18 | Hermetic Mock Gateways | Shadow Tool Calls | Mocked API Responses (No Side Effects) |
| TEST-SPEC-19 | Production $P_{95}$ Latency Baseline | Live Traffic Metrics | Prod Latency Baseline for Regression Comparison |
| TEST-SPEC-02 | CI/CD Stage 4 Gate Enforcement | Shadow Verdicts (this doc) | Production Deployment Authorization |

---

## 10. FINAL NON-NEGOTIABLE PRINCIPLES

SHADOW TRAFFIC MUST NEVER CAUSE SIDE-EFFECTS OR MUTATIONS IN LIVE PRODUCTION DATABASES.

ALL PRODUCTION TRAFFIC MUST BE DETERMINISTICALLY STRIPPED OF PII BEFORE ENTERING REPLAY QUEUES.

TOOL CALL PAYLOADS GENERATED IN SHADOW MUST MATCH PRODUCTION PAYLOADS WITH 100% EXACTNESS.

SHADOW VERDICTS MUST BE LOGGED AS STRICTLY TYPED, SCHEMA-VALIDATED REPORTS.

JSON SCHEMAS SHALL ONLY ENFORCE DATA TYPES; SLA BUSINESS RULES MUST BE EVALUATED IN TEST LOGIC.

SHADOW $P_{95}$ LATENCY MUST NEVER EXCEED THE PRODUCTION BASELINE BY MORE THAN 20% WITHOUT TRIGGERING A REGRESSION HALT.

---

## 11. VERSION HISTORY & ARCHITECTURAL VERDICT

| Version | Date | Description | Author | Status |
|---|---|---|---|---|
| 1.0.0 | August 2026 | Initial specification of Shadow Traffic Replay & Validation (TEST-SPEC-20). | Ramy Bella | SUPERSEDED |
| 1.0.1 | August 2026 | Review fix pass. Fixed a malformed `$schema` URL in `ShadowReplayVerdict@1.0.0` (markdown link syntax had been baked into the JSON string, making it invalid JSON as written) — the same recurring defect class already fixed in TEST-SPEC-07/12/13/15/16/19. Closed a gap where the "Shadow latency must not exceed production baseline by more than 20%" Core Testing Invariant had no corresponding enforcement path anywhere in the document: added `prod_p95_baseline_ms` to the `ShadowReplayVerdict@1.0.0` schema, added `FAIL_LATENCY_REGRESSION` to the verdict enum, added `ERR_TEST_20_05`, added `AC-SHD-06`, added the corresponding Final Non-Negotiable Principle, and added the latency-regression computation and assertion to the representative test harness. No change to the PII isolation architecture, semantic/tool-call parity thresholds, or the existing error codes/acceptance criteria. | Ramy Bella | APPROVED FOR IMPLEMENTATION |

**ARCHITECTURAL VERDICT:**
APPROVED FOR IMPLEMENTATION. TEST-SPEC-20 v1.0.1 establishes the dark launching architecture, PII sanitization boundaries, strict mock-adapter isolation, production-baseline latency regression enforcement, and the `ShadowReplayVerdict@1.0.0` contract for Phase 6. Ready for implementation.
