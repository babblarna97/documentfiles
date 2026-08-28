# DEP-SPEC-04: Cutover Day Execution & Go/No-Go Sequencing

## 1. DOCUMENT CONTROL

| Attribute         | Value                                                                        |
| ----------------- | ---------------------------------------------------------------------------- |
| Document Title    | 04 Cutover Day Execution & Go-No-Go Sequencing.md                            |
| Document ID       | DEP-SPEC-04                                                                  |
| Version           | 1.0.1                                                                        |
| Status            | APPROVED FOR IMPLEMENTATION                                                  |
| Author            | Ramy Bella                                                                   |
| Classification    | Confidential / Enterprise Proprietary                                        |
| Target Audience   | Release Engineers, SREs, Incident Commanders, Go/No-Go Approvers             |
| Parent Document   | DEP-SPEC-01                                                                  |
| Related Documents | DEP-SPEC-01, DEP-SPEC-02, DEP-SPEC-03, DEP-SPEC-05, DEP-SPEC-06, DEP-SPEC-09 |
| System            | Restaurant AI System                                                         |
| Phase             | Phase 7 — Deployment & Production Operations                                 |
| Lifecycle Folder  | 07 Deployment & Infrastructure                                               |
| Last Updated      | August 2026                                                                  |

---

## 2. EXECUTIVE PURPOSE & SCOPE BOUNDARIES

Every other Phase 7 document proves one piece of the machine works: the environment is reproducible (`DEP-SPEC-01`), a tenant is safe to serve (`DEP-SPEC-02`), a release can be exposed to traffic gradually and reversed automatically (`DEP-SPEC-03`). None of them says *when* to pull the trigger, in what order, or what to do if pulling it goes wrong after the point where "wrong" stops being a traffic problem and becomes a data problem. `DEP-SPEC-04` is the executable, timestamped run-book for Cutover Day itself — T-minus through T-plus — including the formal Go/No-Go decision boundary and the Point of No Return (PONR).

**Owns:** the cutover execution sequence, the T-minus/T-plus timeline matrix, the PONR boundary definition, the formal Go/No-Go sign-off requirement, the atomic pre-PONR environment revert, and the `CutoverExecutionRecord@1.0.0` receipt.

**Delegates:** infrastructure and environment state (`DEP-SPEC-01`); tenant readiness (`DEP-SPEC-02`); disaster recovery once PONR has been passed (`DEP-SPEC-05`); telemetry and alerting (`DEP-SPEC-06`); pre-cutover smoke testing (`DEP-SPEC-09`).

**Trigger scope.** A Cutover Day event is a full production-environment or platform migration — infrastructure replatforming, a region move, or a change to the environment `DEP-SPEC-01` provisions. It is **not** the mechanism for bringing an individual new restaurant tenant online. Routine tenant onboarding is fully handled by `DEP-SPEC-02` (tenant creation and `LIVE` eligibility) followed by `DEP-SPEC-03` (gradual traffic exposure to that tenant); neither step requires a Go/No-Go verdict, a T-minus/T-plus sequence, or a PONR evaluation, and this document is not invoked for it. This ceremony exists because a Cutover Day event's blast radius is the entire platform across every tenant, not one restaurant.

**Not the same mechanism as DEP-SPEC-03.** `DEP-SPEC-03`'s canary rollout and circuit-breaker rollback operate on individual software releases *within* a single stable environment, routing a percentage of guest traffic between a release candidate and `stable_version_ref`. A Cutover Day migration is binary — an entire environment either is or is not serving production — and is not a release. `DEP-SPEC-03` is neither invoked by, nor a dependency of, a Cutover Day event; the pre-PONR revert described in Section 3.3 is a self-contained mechanism owned by this document instead.

This document does not re-define any of the mechanisms above. It defines the sequence in which they are invoked, the human sign-off required before that sequence may begin, and the single boundary — PONR — past which the correct response to a failure changes from "roll back" to "declare a disaster."

### Core Operational Invariants

- `CUTOVER EXECUTION WITHOUT A SIGNED GO/NO-GO VERDICT ⇒ PROHIBITED`
- `PRE-CUTOVER SMOKE TEST FAILURE (DEP-SPEC-09) ⇒ AUTOMATED ABORT & ROLLBACK, NO MANUAL OVERRIDE`
- `EXCEEDING POINT OF NO RETURN (PONR) WITH UNHEALTHY TELEMETRY ⇒ SEV-0 DISASTER DECLARATION (DEP-SPEC-05)`
- `T-MINUS SEQUENCE STEP SKIPPED OR EXECUTED OUT OF ORDER ⇒ AUTOMATED HALT, NO MANUAL BYPASS`
- `CUTOVER EXECUTION RECORD = 100% AUDITABLE AND SCHEMA-VALIDATED`

---

## 3. EXECUTION ARCHITECTURE: T-MINUS/T-PLUS TIMELINE & PONR BOUNDARY

### 3.1 Timeline Matrix

| Marker | Phase | Actions | Owning Spec(s) |
|---|---|---|---|
| T-60 min | Pre-Flight | Source code and configuration freeze; full automated smoke suite executes. | DEP-SPEC-09 |
| T-15 min | State Lock & Read-Only Check | Database synchronization verified; system enters read-only/state-locked mode; health endpoints verified across all target services. | DEP-SPEC-01, DEP-SPEC-06 |
| T-0 | Cutover Switch | Atomic DNS/API gateway switch executes as a single indivisible operation. | DEP-SPEC-04 |
| T+15 min | Post-Cutover Audit (First Window) | Telemetry analysis: P95 latency, 5xx error rate, against pre-cutover baseline. | DEP-SPEC-06 |
| T+30 min | Post-Cutover Audit (Confirmation Window) | Second telemetry window confirms sustained stability; formal `COMPLETED` sign-off becomes eligible. | DEP-SPEC-06 |

Every marker above executes strictly in the order shown. A step that has not been reached MUST NOT be skipped ahead of, and a step already passed MUST NOT be re-entered — either condition halts the cutover automatically pending manual investigation (see `ERR_DEP_04_01`).

### 3.2 Go/No-Go Decision Boundary

A single Go/No-Go verdict, signed by an authorized approver and referenced on the `CutoverExecutionRecord` (Section 4), is the only mechanism authorized to move a cutover out of `PRE_FLIGHT`. There is no code path that begins a cutover sequence without a verdict already recorded. If the T-60 smoke suite (`DEP-SPEC-09`) fails, that decision is reversed automatically: the cutover aborts before the T-0 switch ever executes, with no manual override available, regardless of what the original Go/No-Go verdict authorized — a signed verdict authorizes an *attempt*, not a guaranteed execution.

### 3.3 Point of No Return (PONR)

**PONR is a state-based boundary, not a fixed clock offset.** It is reached at the instant the target environment commits its first live, guest-facing write that has no corresponding record in the prior stable environment (for example, the first new guest reservation accepted post-switch). Before that instant, an atomic revert of the T-0 switch — pointing DNS/API gateway traffic back to the environment that was serving production immediately beforehand — is safe: no guest data exists uniquely in the new environment, so reverting traffic loses nothing. This revert is a self-contained mechanism owned by this document, distinct from and independent of `DEP-SPEC-03`'s release-level canary and circuit-breaker, which are not invoked during a Cutover Day event (see Section 2). After PONR, a simple revert would silently lose or duplicate that guest's data — the only safe response to a subsequent failure is full data reconciliation via disaster recovery (`DEP-SPEC-05`), not a revert.

Because PONR is state-based, it may be reached before, at, or after the T+15 marker depending on when the first qualifying write actually occurs — it MUST be evaluated continuously from T-0 onward and MUST NOT be assumed to align with any single T-marker.

`ponr_passed` (Section 4) is set `true` only once telemetry has remained within nominal bounds continuously from the PONR crossing instant through the T+30 confirmation window. Any breach at any point within that window — not only at the instant of crossing — triggers `ERR_DEP_04_03` and a SEV-0 disaster declaration rather than confirming `ponr_passed`.

---

## 4. CONTRACT SCHEMA (`CutoverExecutionRecord@1.0.0`)

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "title": "CutoverExecutionRecord@1.0.0",
  "type": "object",
  "additionalProperties": false,
  "properties": {
    "cutover_id": { "type": "string", "minLength": 1 },
    "target_environment_id": { "type": "string", "minLength": 1 },
    "status": {
      "type": "string",
      "enum": ["PRE_FLIGHT", "IN_PROGRESS", "PONR_REACHED", "COMPLETED", "ABORTED"]
    },
    "pre_flight_passed": { "type": "boolean" },
    "ponr_passed": { "type": "boolean" },
    "executed_by": { "type": "string", "minLength": 1 },
    "go_no_go_reference": { "type": "string", "minLength": 1 },
    "started_at": { "type": "string", "format": "date-time" },
    "completed_at": { "type": "string", "format": "date-time" }
  },
  "required": [
    "cutover_id",
    "target_environment_id",
    "status",
    "pre_flight_passed",
    "ponr_passed",
    "executed_by",
    "started_at"
  ],
  "allOf": [
    {
      "if": {
        "properties": { "status": { "const": "COMPLETED" } },
        "required": ["status"]
      },
      "then": {
        "properties": {
          "pre_flight_passed": { "const": true },
          "ponr_passed": { "const": true }
        },
        "required": ["pre_flight_passed", "ponr_passed", "completed_at"]
      }
    },
    {
      "if": {
        "properties": { "status": { "enum": ["PONR_REACHED", "COMPLETED"] } },
        "required": ["status"]
      },
      "then": {
        "properties": {
          "pre_flight_passed": { "const": true }
        },
        "required": ["pre_flight_passed"]
      }
    },
    {
      "if": {
        "properties": { "status": { "const": "ABORTED" } },
        "required": ["status"]
      },
      "then": {
        "required": ["completed_at"]
      }
    }
  ]
}
```

### Contract Semantics

`completed_at` and `go_no_go_reference` are deliberately absent from the top-level `required` list: a record in `PRE_FLIGHT` or `IN_PROGRESS` has legitimately not completed yet, and a record created before a verdict is fully attached should not be forced to fabricate one. Their presence is enforced conditionally instead — `completed_at` becomes mandatory the instant a cutover reaches either terminal state (`COMPLETED` or `ABORTED`), which is the only point at which "when did this finish" is a meaningful question.

`pre_flight_passed` and `ponr_passed` remain unconditionally required booleans (default `false` at record creation) because, unlike a timestamp or a reference string, a gate flag always has a defined true/false state — there is no "not yet applicable" case for a gate that simply hasn't been evaluated yet.

A record MUST NOT reach `PONR_REACHED` or `COMPLETED` with `pre_flight_passed = false` — PONR, by definition, cannot occur before T-0, and T-0 cannot occur before a passing pre-flight gate.

---

## 5. AUTOMATED CUTOVER TEST HARNESS

```python
# Representative Automated Pre-Flight Abort Test
@pytest.mark.rtm(req_id="DEP-SPEC-04-CUT-001")
def test_failed_smoke_test_triggers_automated_abort():
    cutover = cutover_engine.start_cutover(
        target_environment_id="PROD-EU-01",
        executed_by="release-engineer-042",
        go_no_go_reference="GNG-2026-08-21-001"
    )

    # Simulate the DEP-SPEC-09 pre-flight smoke suite reporting a failure
    smoke_test_feed.report_result(
        cutover_id=cutover.cutover_id,
        suite_passed=False
    )

    cutover_engine.evaluate_pre_flight(cutover_id=cutover.cutover_id)

    record = cutover_engine.get_record(cutover_id=cutover.cutover_id)
    abort_event = cutover_engine.get_last_abort_event(cutover.cutover_id)

    assert record.status == "ABORTED"
    assert record.pre_flight_passed is False
    assert abort_event.error_code == "ERR_DEP_04_01"

    # Abort must already be complete, not awaiting a manual decision
    assert cutover_engine.has_pending_human_decision(
        cutover.cutover_id
    ) is False


# Representative Automated COMPLETED-Transition Gate Test
@pytest.mark.rtm(req_id="DEP-SPEC-04-CUT-002")
def test_completed_transition_blocked_without_all_gates_validated():
    cutover = cutover_engine.start_cutover(
        target_environment_id="PROD-EU-01",
        executed_by="release-engineer-042",
        go_no_go_reference="GNG-2026-08-21-002"
    )

    smoke_test_feed.report_result(
        cutover_id=cutover.cutover_id,
        suite_passed=True
    )
    cutover_engine.evaluate_pre_flight(cutover_id=cutover.cutover_id)

    # PONR has not yet been reached; ponr_passed is still false
    with pytest.raises(SchemaValidationException) as exc_info:
        cutover_engine.transition_status(
            cutover_id=cutover.cutover_id,
            new_status="COMPLETED"
        )

    assert exc_info.value.error_code == "ERR_DEP_04_04"

    record = cutover_engine.get_record(cutover_id=cutover.cutover_id)
    assert record.status != "COMPLETED"
    assert record.pre_flight_passed is True
    assert record.ponr_passed is False
```

---

## 6. SECURITY & THREAT MODEL

| Threat | Attack Vector | Preventive Control | Severity |
|---|---|---|---|
| Go/No-Go Forgery | A cutover is started referencing a `go_no_go_reference` that does not correspond to a genuine, authorized sign-off. | `executed_by` and `go_no_go_reference` are cross-validated against an independent approval record at cutover start; a missing or unverifiable reference blocks `PRE_FLIGHT` from ever starting. | CRITICAL (SEV-0) |
| Stale Pre-Flight Reuse | A previous `pre_flight_passed = true` result is carried over into a new cutover attempt without re-running the DEP-SPEC-09 smoke suite. | `pre_flight_passed` is scoped to a single `cutover_id` and defaults `false` on every new record; it can only be set by a fresh smoke-suite evaluation against that specific `cutover_id`. | CRITICAL (SEV-0) |
| Silent PONR Crossing | The T-0 switch executes and guest writes begin without an explicit PONR evaluation ever being logged, leaving no record of whether the crossing criteria were checked. | PONR evaluation runs continuously and automatically from T-0 onward as an independent process, not as a step a human must remember to trigger; every write to the target environment is checked against the PONR criterion in Section 3.3. | CRITICAL (SEV-0) |
| Post-PONR Rollback Attempt | An operator, unaware PONR has passed, attempts the pre-PONR environment revert instead of a `DEP-SPEC-05` disaster declaration. | Once `status = PONR_REACHED`, this document's own pre-PONR revert path is programmatically disabled for that `cutover_id`; only the `DEP-SPEC-05` disaster path remains available. | HIGH (SEV-1) |

---

## 7. FAILURE ARCHITECTURE & ERROR TAXONOMY

| Error Code | Description | Canonical Class | Severity |
|---|---|---|---|
| ERR_DEP_04_01 | Pre-flight validation (source/config freeze or DEP-SPEC-09 smoke suite) failed prior to cutover. | PROCESS_VIOLATION | CRITICAL (SEV-0) |
| ERR_DEP_04_02 | Cutover execution attempted without a signed, verifiable Go/No-Go verdict. | RUNTIME_AUTHORIZATION | CRITICAL (SEV-0) |
| ERR_DEP_04_03 | Telemetry breach detected after PONR was reached, requiring disaster declaration rather than rollback. | SAFETY_VIOLATION | CRITICAL (SEV-0) |
| ERR_DEP_04_04 | CutoverExecutionRecord failed schema or conditional-validation rules (e.g., COMPLETED attempted without both gates passed). | CONTRACT_MISMATCH | HIGH (SEV-1) |

---

## 8. PRODUCTION ACCEPTANCE CRITERIA

| AC ID | Category | Requirement | Verification Method | Pass/Fail |
|---|---|---|---|---|
| AC-CUT-01 | Go/No-Go Enforcement | 100% of cutover executions blocked absent a signed, referenced Go/No-Go verdict. | Sign-Off Verification Audit | REQUIRED |
| AC-CUT-02 | Pre-Flight Gate | 100% of cutovers auto-abort on any DEP-SPEC-09 smoke test failure, with 0 manual override paths exercised. | Pre-Flight Test Suite | REQUIRED |
| AC-CUT-03 | PONR Integrity | 100% of PONR crossings are recorded with contemporaneous telemetry health state; 0 undocumented crossings. | PONR Audit Log Review | REQUIRED |
| AC-CUT-04 | Post-PONR Disaster Path | 100% of post-PONR telemetry breaches trigger DEP-SPEC-05 disaster declaration; 0 attempted post-PONR canary rollbacks. | Disaster Path Test Suite | REQUIRED |
| AC-CUT-05 | Schema Compliance | 100% of CutoverExecutionRecord instances validate against CutoverExecutionRecord@1.0.0, including all conditional rules. | Schema Inspector | REQUIRED |

---

## 9. INTEGRATION AUTHORITY MATRIX

| Component | Owns | Consumes | Produces |
|---|---|---|---|
| DEP-SPEC-04 | Cutover sequence, T-minus/T-plus matrix, PONR boundary, Go/No-Go sign-off, atomic switch execution, pre-PONR environment revert, CutoverExecutionRecord | Pre-flight results (DEP-SPEC-09), environment state (DEP-SPEC-01), tenant readiness (DEP-SPEC-02) | CutoverExecutionRecord, PONR crossing events, disaster-declaration triggers |
| DEP-SPEC-09 | Automated pre-flight smoke test suite | Target environment | Pre-flight pass/fail signal (consumed here as an abort trigger) |
| DEP-SPEC-01 | Environment tiers & secrets platform | — | Provisioned target environment this document cuts traffic into |
| DEP-SPEC-02 | Tenant onboarding & LIVE eligibility | — | Tenant readiness state (consumed as a pre-flight input) |
| DEP-SPEC-03 | Canary staging & release-level traffic routing | — | Not invoked during a Cutover Day event — see Section 2 scope boundary. Continues operating independently, at the release level, on whichever environment is currently live. |
| DEP-SPEC-05 | Disaster recovery & data reconciliation | PONR breach signal (this document) | Executes recovery once a SEV-0 disaster is declared |
| DEP-SPEC-06 | Telemetry & alerting | Post-cutover metrics | P95 latency / 5xx error rate signals consumed at T+15/T+30 audits and for PONR health validation |

---

## 10. FINAL NON-NEGOTIABLE PRINCIPLES

NO CUTOVER MAY EXECUTE WITHOUT A SIGNED, VERIFIABLE GO/NO-GO VERDICT.

A FAILED PRE-CUTOVER SMOKE TEST (DEP-SPEC-09) MUST TRIGGER AN AUTOMATED ABORT WITH ZERO MANUAL OVERRIDE PATH.

CROSSING THE POINT OF NO RETURN MUST BE RECORDED AGAINST CONTEMPORANEOUS TELEMETRY HEALTH, NEVER ASSUMED.

ANY UNHEALTHY TELEMETRY STATE DETECTED AFTER PONR MUST TRIGGER A SEV-0 DISASTER DECLARATION (DEP-SPEC-05), NEVER A CANARY ROLLBACK.

EVERY CUTOVER EXECUTION MUST PRODUCE A COMPLETE, SCHEMA-VALIDATED, AUDITABLE CUTOVEREXECUTIONRECORD.

---

## 11. VERSION HISTORY & ARCHITECTURAL VERDICT

| Version | Date | Description | Author | Status |
|---|---|---|---|---|
| 1.0.0 | August 2026 | Initial specification of Cutover Day Execution & Go/No-Go Sequencing (DEP-SPEC-04). Establishes the T-minus/T-plus timeline matrix, the state-based (not clock-based) PONR definition, the CutoverExecutionRecord@1.0.0 contract with conditional gate validation, and the mandatory automated abort/disaster-declaration paths. | Ramy Bella | SUPERSEDED |
| 1.0.1 | August 2026 | Review fix pass. Added an explicit trigger-scope boundary: a Cutover Day event is a rare, full-environment/platform migration — not the mechanism for routine tenant onboarding, which remains fully owned by DEP-SPEC-02 (tenant LIVE eligibility) and DEP-SPEC-03 (gradual per-tenant traffic exposure), neither of which requires this document's ceremony. Corrected the pre-PONR rollback mechanism's ownership: it is now a self-contained, atomic environment revert owned by this document, replacing an incorrect delegation to DEP-SPEC-03 — which operates on release-level canary traffic within a single environment and was never actually defined to expose a per-cutover-id rollback-disable interface. Updated the T-0 timeline row, Section 3.3, the post-PONR threat control, and the Integration Authority Matrix accordingly. No change to the Go/No-Go requirement, the PONR criterion itself, the schema, or the disaster-declaration path. | Ramy Bella | APPROVED FOR IMPLEMENTATION |

**ARCHITECTURAL VERDICT:**
APPROVED FOR IMPLEMENTATION. DEP-SPEC-04 establishes the executable Cutover Day run-book, the mandatory Go/No-Go sign-off boundary, the state-based PONR criterion that governs the switch from canary rollback to full disaster recovery, and the CutoverExecutionRecord@1.0.0 contract binding all of it to a single auditable trail. Ready for implementation.
