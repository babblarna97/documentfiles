## TEST-SPEC-18: E2E Adapter & Gateway Pipeline Tests

1. **DOCUMENT CONTROL**

Attribute| Value
Document Title| 18 E2E Adapter & Gateway Pipeline Tests.md
Document ID| TEST-SPEC-18
Version| 1.0.1
Status| APPROVED FOR IMPLEMENTATION
Author| Ramy Bella
Classification| Confidential / Enterprise Proprietary
Target Audience| Integration Engineers, SREs, QA Architects, POS Specialists
Parent Document| TEST-SPEC-01
Related Documents| Phase 5 Specs (INT-SPEC), TEST-SPEC-01, TEST-SPEC-02, TEST-SPEC-15
System| Restaurant AI System
Phase| Phase 6 — Validation & Verification (V&V)
Lifecycle Folder| 06 Integration & End-to-End
Last Updated| August 2026

---

2. EXECUTIVE PURPOSE

The Phase 5 Integration Adapters translate internal conversational states ("TEST-SPEC-15") into concrete API payloads directed at third-party Point-of-Sale (POS) systems, reservation systems, and SMS gateways. Testing these integrations directly against live, external third-party production APIs during CI/CD is prohibited—it causes rate-limiting, pollutes external databases with test data, and creates flaky, non-deterministic test failures.

"TEST-SPEC-18" defines the E2E Adapter & Gateway Pipeline Test Architecture. Executed in Stage 3 ("TEST-SPEC-02"), it establishes a hermetic, deterministic testing environment using API mocking (e.g., WireMock). It verifies adapter payload formatting, tenant credential isolation, timeout/circuit-breaker behaviors, and external error state mapping (e.g., handling HTTP 429 Rate Limits from a POS). It guarantees that the system perfectly speaks the language of its external integrations without actually touching them during CI/CD.

Core Testing Invariants:

* "LIVE EXTERNAL API CALL IN CI/CD \implies IMMEDIATE BUILD FAILURE"
* "ADAPTER PAYLOAD MALFORMED \implies CONTRACT MISMATCH ERROR"
* "TIMEOUT THRESHOLD EXCEEDED \implies MANDATORY CIRCUIT BREAKER OPEN STATE"
* "EXTERNAL API FAILURE (HTTP 500) \implies GRACEFUL FALLBACK (NO USER-FACING CRASH)"
* "TENANT A CREDENTIALS DISPATCHED FOR TENANT B \implies SEV-0 ISOLATION BREACH"

---

3. HERMETIC ADAPTER TESTING ARCHITECTURE

The test harness isolates the core system from the external internet using a Hermetic Mock Gateway.

[PHASE 3: STATE MACHINE] ──(BOOKING_EXECUTED)──► [PHASE 5: ADAPTER ROUTER]
                                                            │
                                  ┌─────────────────────────┴─────────────────────────┐
                                  │  (CI/CD Pipeline Network Boundary)                │
                                  ▼                                                   ▼
                     [HERMETIC WIREMOCK GATEWAY]                        [LIVE EXTERNAL API]
                     (Used for all E2E Tests)                             (PROHIBITED IN CI)
                     - Simulates HTTP 200 Success
                     - Simulates HTTP 429 Rate Limit
                     - Simulates HTTP 504 Timeout
                     - Simulates HTTP 500 Server Error

External API contract schemas used for adapter verification MUST be pinned to versioned, approved contract artifacts for each integration. CI/CD tests MUST validate against the pinned contract version and MUST NOT dynamically retrieve or trust an unversioned live vendor schema during test execution.

---

4. ADAPTER VERIFICATION CONTRACT ("AdapterE2EReport@1.0.0")

Every E2E adapter test MUST produce a structured schema detailing the dispatch status, circuit breaker state, internal outcome, tenant isolation state, and payload validation.

{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "title": "AdapterE2EReport@1.0.0",
  "type": "object",
  "properties": {
    "test_id": {
      "type": "string"
    },
    "adapter_type": {
      "type": "string",
      "enum": [
        "POS_BOOKING",
        "SMS_GATEWAY",
        "PAYMENT_GATEWAY"
      ]
    },
    "mock_scenario": {
      "type": "string",
      "enum": [
        "HAPPY_PATH_200",
        "TIMEOUT_504",
        "RATE_LIMIT_429",
        "SERVER_ERROR_500"
      ]
    },
    "circuit_breaker_state": {
      "type": "string",
      "enum": [
        "CLOSED",
        "OPEN",
        "HALF_OPEN"
      ]
    },
    "internal_outcome": {
      "type": "string",
      "enum": [
        "SUCCESS",
        "RETRYING",
        "FALLBACK",
        "FAIL_FAST",
        "CIRCUIT_OPEN"
      ]
    },
    "payload_validated": {
      "type": "boolean"
    },
    "tenant_isolation_verified": {
      "type": "boolean"
    },
    "executed_at": {
      "type": "string",
      "format": "date-time"
    }
  },
  "required": [
    "test_id",
    "adapter_type",
    "mock_scenario",
    "circuit_breaker_state",
    "internal_outcome",
    "payload_validated",
    "tenant_isolation_verified",
    "executed_at"
  ]
}

---

5. AUTOMATED HERMETIC ADAPTER TEST HARNESS

The test suite forces the adapters through diverse failure modes to ensure graceful degradation.

Representative Automated Hermetic Adapter Test Harness

@pytest.mark.rtm(req_id="INT-SPEC-18-ADP-001")
def test_pos_adapter_circuit_breaker_on_timeout():
    # Setup: Configure WireMock to simulate a 10-second timeout on the POS API
    mock_gateway.stub_request(
        endpoint="/api/pos/v1/booking",
        tenant_id="tenant_demo",
        delay_ms=10000,
        status_code=504
    )

    # Read the authoritative circuit-breaker failure threshold used by the
    # adapter runtime so the test does not assume an arbitrary failure count.
    failure_threshold = adapter_router.circuit_breaker.failure_threshold

    # Act: Dispatch the required number of consecutive failing requests.
    reports = []

    for _ in range(failure_threshold):
        reports.append(
            adapter_router.dispatch(
                intent="BOOKING_CREATE",
                state_payload=valid_booking_state
            )
        )

    report = reports[-1]

    # Assert 0: Every produced report is schema-valid
    assert all(
        validate_schema(r.json(), "AdapterE2EReport@1.0.0")
        for r in reports
    )

    # Assert 1: Every request observed the simulated timeout scenario
    assert all(
        r.mock_scenario == "TIMEOUT_504"
        for r in reports
    )

    # Assert 2: Circuit breaker tripped after the configured failure threshold
    assert report.circuit_breaker_state == "OPEN"

    # Assert 3: Internal outcome reflects the circuit-open state
    assert report.internal_outcome == "CIRCUIT_OPEN"

    # Assert 4: Tenant context remained correctly isolated
    assert all(
        r.tenant_isolation_verified
        for r in reports
    )

---

6. SECURITY & THREAT MODEL

Threat| Attack Vector| Preventive Control| Severity
Cross-Tenant API Key Bleed| Adapter accidentally attaches Tenant A's Bearer token to Tenant B's payload.| E2E Harness actively asserts tenant_id match on the generated HTTP header prior to dispatch.| CRITICAL (SEV-0)
Cascading System Failure| External SMS gateway goes down; blocked threads consume all worker memory.| Circuit breakers ("INT-SPEC-17") trip after the configured failure threshold, failing fast to protect worker resources.| HIGH (SEV-1)
Live Data Pollution| CI test suite accidentally connects to live POS and creates 10,000 fake reservations.| VPC egress rules in CI/CD environments strictly block external internet traffic.| HIGH (SEV-1)

---

7. FAILURE ARCHITECTURE & ERROR TAXONOMY

Error Code| Description| Canonical Class| Severity
ERR_TEST_18_01| Adapter dispatch attempted network connection to live external IP during CI.| SECURITY_BOUNDARY| CRITICAL (SEV-0)
ERR_TEST_18_02| Outbound adapter payload failed validation against the pinned target external API contract schema.| CONTRACT_MISMATCH| HIGH (SEV-1)
ERR_TEST_18_03| Circuit breaker failed to open after the configured sequence of simulated external timeouts.| RESOURCE_EXHAUSTION| HIGH (SEV-1)
ERR_TEST_18_04| AdapterE2EReport failed JSON Schema validation ("AdapterE2EReport@1.0.0").| VALIDATION| HIGH (SEV-1)
ERR_TEST_18_05| Adapter failed to gracefully handle external HTTP 4xx/5xx error codes or produce the required internal outcome.| LOGIC_FAIL| MEDIUM (SEV-2)

---

8. PRODUCTION ACCEPTANCE CRITERIA

AC ID| Category| Requirement| Verification Method| Pass/Fail
AC-ADP-01| Hermetic Execution| 100% of E2E adapter tests execute against local mock gateways; 0 live external calls.| VPC Network Audit| REQUIRED
AC-ADP-02| Payload Integrity| 100% of generated adapter payloads match the exact pinned, versioned schema required by the target external API contract.| Payload Schema Test| REQUIRED
AC-ADP-03| Circuit Breaker SLA| Simulated external API timeouts (> 3000 ms) successfully trip circuit breakers into OPEN state after the configured failure threshold is reached.| Fault Injection Test| REQUIRED
AC-ADP-04| Error Mapping| External HTTP 400/500 errors are gracefully caught and mapped to the expected internal fallback outcome, as recorded in "AdapterE2EReport@1.0.0" ("internal_outcome").| Mock Scenario Suite| REQUIRED
AC-ADP-05| Schema Compliance| 100% of E2E test reports validate against "AdapterE2EReport@1.0.0", bounded by strict enums.| Schema Inspector| REQUIRED

---

9. INTEGRATION AUTHORITY MATRIX

Component| Owns| Consumes| Produces
TEST-SPEC-18| Hermetic API Testing, Circuit Breaker Verification, Payload Schema Checks| FSM States (TEST-SPEC-15), Mock API Scenarios| AdapterE2EReport, Outbound Validation Signals
Phase 5 (INT-SPEC)| Authoritative Adapter Implementations & Retry Logic| FSM State Contexts| Outbound HTTP Payloads
INT-SPEC-17| Resiliency, Circuit Breakers & Backpressure| Adapter Error Rates| Circuit Breaker State (OPEN / CLOSED)
TEST-SPEC-02| CI/CD Stage 3 Gate Enforcement| E2E Adapter Reports (this document)| Staging Promotion Blocks

---

10. FINAL NON-NEGOTIABLE PRINCIPLES

NO CI/CD PIPELINE SHALL EVER EXECUTE INTEGRATION TESTS AGAINST LIVE EXTERNAL PRODUCTION APIS.

ALL EXTERNAL INTEGRATIONS MUST BE TESTED USING HERMETIC, DETERMINISTIC MOCK GATEWAYS.

ADAPTER PAYLOADS MUST BE SCHEMA-VALIDATED AGAINST PINNED, VERSIONED CONTRACT ARTIFACTS BEFORE NETWORK DISPATCH TO PREVENT CONTRACT MISMATCHES.

EXTERNAL TIMEOUTS OR SERVER ERRORS MUST TRIGGER THE CONFIGURED CIRCUIT-BREAKER BEHAVIOR AND PREVENT CASCADING THREAD LOCKS.

ADAPTER EVALUATION VERDICTS MUST BE LOGGED AS STRICTLY TYPED, ENUM-BOUNDED SCHEMA REPORTS.

---

11. VERSION HISTORY & ARCHITECTURAL VERDICT

Version| Date| Description| Author| Status
1.0.0| August 2026| Initial specification of E2E Adapter & Gateway Pipeline Tests (TEST-SPEC-18).| Ramy Bella| SUPERSEDED
1.0.1| August 2026| Review fix pass. Added the missing "internal_outcome" field to "AdapterE2EReport@1.0.0", corrected the circuit-breaker timeout harness to execute the configured failure threshold before asserting "OPEN", pinned external API contract schemas for reproducible CI validation, and aligned AC-ADP-04 with the structured internal outcome. No unrelated architectural changes introduced.| Ramy Bella| APPROVED FOR IMPLEMENTATION

ARCHITECTURAL VERDICT:

APPROVED FOR IMPLEMENTATION

TEST-SPEC-18 v1.0.1 establishes the hermetic E2E mocking framework, "AdapterE2EReport@1.0.0" contract, circuit-breaker fault injection and verification, pinned external contract validation, and zero-live-call policy for Phase 6. Ready for implementation.