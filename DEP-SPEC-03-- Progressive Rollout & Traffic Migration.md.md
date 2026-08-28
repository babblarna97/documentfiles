# DEP-SPEC-03: Progressive Rollout & Traffic Migration

## 1. DOCUMENT CONTROL

| Attribute         | Value                                                              |
| ----------------- | ------------------------------------------------------------------ |
| Document Title    | 03 Progressive Rollout & Traffic Migration.md                      |
| Document ID       | DEP-SPEC-03                                                        |
| Version           | 1.0.1                                                              |
| Status            | APPROVED FOR IMPLEMENTATION                                        |
| Author            | Ramy Bella                                                         |
| Classification    | Confidential / Enterprise Proprietary                              |
| Target Audience   | Platform Engineers, SREs, Release Engineers, QA Leads              |
| Parent Document   | DEP-SPEC-01                                                        |
| Related Documents | DEP-SPEC-01, DEP-SPEC-02, TEST-SPEC-02, TEST-SPEC-08, TEST-SPEC-09, TEST-SPEC-17, TEST-SPEC-20 |
| System            | Restaurant AI System                                               |
| Phase             | Phase 7 — Deployment & Production Operations                       |
| Lifecycle Folder  | 07 Deployment & Infrastructure                                     |
| Last Updated      | August 2026                                                        |

---

## 2. EXECUTIVE PURPOSE

DEP-SPEC-01 proves the environment a release runs in is reproducible and isolated. DEP-SPEC-02 proves a given *tenant* is safe to receive guest-facing traffic at all. Neither answers a different question: when a new release — a code change, an updated prompt template (`TEST-SPEC-16`), or a new LLM model version — is deployed, how much of the live guest population is exposed to it, in what order, and what happens automatically the instant it starts behaving worse than what it replaced?

A "big bang" deployment answers that question badly: 100% of tenants, 100% of guests, all at once, with no automated path back except a human noticing and acting fast enough. `DEP-SPEC-03` replaces that with a staged, metrics-gated rollout that only ever exposes a small, bounded slice of traffic to an unproven release, and that reverses itself automatically — without waiting for a human — the moment monitored metrics regress.

**Scope boundary.** This document owns the traffic-layer progressive rollout mechanism: canary staging, feature-flag/traffic-splitting logic, automatic circuit-breaker rollback, and the Runtime Admission Gateway that actually routes a guest request to a specific release version. It does **not** own: environment provisioning or the secrets platform (`DEP-SPEC-01`); tenant identity, onboarding, or the pre-go-live LIVE gate (`DEP-SPEC-02`); the Shadow environment or the mechanics of replay verdict computation (`TEST-SPEC-20` — this document only *consumes* that verdict as a hard precondition); or prompt semantic-drift scoring (`TEST-SPEC-17` — consumed here as one circuit-breaker input for prompt/model release types, not computed here).

**Relationship to DEP-SPEC-02.** DEP-SPEC-02 answers "is this tenant allowed to receive guest traffic at all?" and produces a `LIVE` tenant. DEP-SPEC-03 answers a separate question: "of the traffic this tenant is allowed to receive, which *release version* actually serves it right now?" A tenant being `LIVE` under DEP-SPEC-02 is a precondition for this document's gateway to consider it — never a substitute for the release-level routing decision made here.

### Core Operational Invariants

- `NEW RELEASE CANDIDATE ⇒ MANDATORY PASSING TEST-SPEC-20 SHADOW REPLAY VERDICT BEFORE ANY LIVE TRAFFIC ADMISSION`
- `CANARY STAGE ADVANCEMENT ⇒ ONLY VIA AUTOMATED METRIC-WINDOW EVALUATION, NEVER ELAPSED TIME ALONE`
- `CIRCUIT-BREAKER THRESHOLD BREACH ⇒ AUTOMATIC ROLLBACK TO LAST KNOWN-GOOD VERSION, NO HUMAN APPROVAL REQUIRED TO INITIATE`
- `TENANT MARKED LIVE BY DEP-SPEC-02 ⇒ NECESSARY BUT NOT SUFFICIENT FOR RECEIVING ANY SPECIFIC RELEASE'S TRAFFIC SHARE`
- `ROLLBACK TARGET = LAST VERIFIED-STABLE VERSION, NEVER A PARTIALLY-ROLLED-OUT INTERMEDIATE STATE`

---

## 3. PROGRESSIVE ROLLOUT PIPELINE ARCHITECTURE

```text
[RELEASE CANDIDATE] (code build, prompt template version, or model version)
           │
           ▼
[TEST-SPEC-20 SHADOW REPLAY GATE]
  MUST return a PASSING verdict against real mirrored production
  traffic before this pipeline may admit a single guest request.
  A missing, failing, or stale verdict blocks progression — see
  Section 4, ERR_DEP_03_01.
           │
           ▼ (shadow_gate_passed = true)
┌──────────────────────────────────────────────────────────────┐
│ STAGE 0 — 1% CANARY                                          │
│ STAGE 1 — 5% CANARY                                          │
│ STAGE 2 — 25% CANARY                                         │
│ STAGE 3 — 100% (FULL EXPOSURE)                                │
│                                                                │
│ Each stage requires, before advancing to the next:            │
│   1. minimum bake duration elapsed, AND                      │
│   2. minimum guest-request sample size observed, AND         │
│   3. all monitored circuit-breaker metrics within threshold  │
│      for the full evaluation window.                         │
│ Time elapsed alone satisfies none of the above.               │
└──────────────────────────────┬────────────────────────────────┘
                               │
                     ┌─────────┴─────────┐
                     ▼                   ▼
         [ALL STAGES CLEAN]     [BREACH AT ANY STAGE]
                     │                   │
                     ▼                   ▼
              [STABLE]          [AUTOMATIC ROLLBACK]
     becomes new rollback   traffic reverts to prior
     target for the NEXT    stable_version_ref; no
     release candidate      human approval to initiate
                     │                   │
                     └─────────┬─────────┘
                                ▼
                 [CIRCUIT BREAKER MONITOR — always active]
        Watches P95 latency delta, guest-facing error rate,
        continuous safety-suite regression signals, and (for
        PROMPT_TEMPLATE / MODEL candidates) TEST-SPEC-17 semantic
        drift. A breach at ANY stage — including 100% — triggers
        the same automatic rollback path.
```

**Traffic splitting mechanism.** Traffic is split along three independent dimensions, evaluated in this order by the Runtime Admission Gateway:

1. **Explicit tenant pin** — a tenant may be explicitly pinned to the stable version (e.g., for a compliance or contractual reason) regardless of global canary stage. A pin always wins.
2. **Deterministic cohort assignment** — absent a pin, each tenant/guest-session is deterministically hashed into or out of the current stage's traffic percentage, so a given tenant's assignment is stable across requests within a stage rather than re-randomized per request.
3. **Region-scoped override** — a rollout may additionally be restricted to specific regions before considering global percentage, for staged geographic exposure.

**Runtime Admission Gateway.** This is the actual routing layer standing between a guest request and a release version. For every request it evaluates, in order: (a) is this tenant currently `LIVE` per `DEP-SPEC-02`? If not, the request never reaches this document's logic at all. (b) Is this tenant pinned to stable? (c) Does this tenant's deterministic cohort assignment fall inside the release candidate's current traffic percentage? Only if (a) is true and either (b) resolves to "not pinned" and (c) resolves to "in cohort" does the request route to the release candidate; otherwise it routes to `stable_version_ref`.

**No direct-to-100% path.** A release candidate MUST pass through Stages 0–2 in order. The only way a release reaches 100% exposure without having individually cleared 5% and 25% is if it is *itself* the rollback target of a failed later release — i.e., it was already `STABLE` before the failed rollout began.

---

## 4. STATE TRANSITION & ROLLBACK RULES

### 4.1 Canary State Machine

```text
SHADOW_GATE_PENDING
        │
        ▼
   STAGE_0_1PCT
        │
        ▼
   STAGE_1_5PCT
        │
        ▼
   STAGE_2_25PCT
        │
        ▼
  STAGE_3_100PCT
        │
        ▼
      STABLE
```

Terminal states: `STABLE` (success) and `ROLLED_BACK` (failure exit).

`ROLLED_BACK` is reachable from any non-terminal state (`STAGE_0_1PCT` through `STAGE_3_100PCT`) — a circuit-breaker breach at 100% exposure rolls back exactly as a breach at 1% does. No stage is "past the point of automatic rollback."

A rollout MUST NOT transition directly from `SHADOW_GATE_PENDING` to any `STAGE_*` state without `shadow_gate_passed = true` recorded by the `TEST-SPEC-20` verdict-ingestion service specifically (see Section 6, `ERR_DEP_03_06`).

### 4.2 Automated Stage Advancement Rule

A canary stage may advance to the next stage only when **all** of the following hold:

1. the stage's minimum bake duration has elapsed;
2. the stage's minimum guest-request sample size has been observed;
3. every monitored circuit-breaker metric remained within its configured threshold for the entire evaluation window.

Specific bake-duration, sample-size, and metric-threshold values are configured per release type and per deployment tier by the monitoring/evaluation layer; they are deliberately not hardcoded in this document or in the `RolloutState` schema (Section 5), so that tightening a threshold is a configuration change, not a specification amendment. Elapsed time satisfying (1) alone, without (2) and (3), MUST NOT authorize advancement.

### 4.3 Circuit Breaker Inputs

The circuit breaker continuously evaluates, for the active release candidate at its current stage:

- P95 latency delta against the current stable baseline.
- Guest-facing error rate against the current stable baseline.
- Continuous safety/security regression signal from the always-on canary-scoped subset of `TEST-SPEC-08` / `TEST-SPEC-09` checks.
- For `PROMPT_TEMPLATE` and `MODEL` release types only: semantic drift signal from `TEST-SPEC-17`.

A breach of any single monitored input is sufficient to trip the breaker; inputs are not averaged against one another.

### 4.4 Rollback Target Integrity

The rollback target, `stable_version_ref`, MUST always resolve to the version that was `STABLE` immediately before the current rollout began — never to an intermediate canary stage of the *current* rollout, and never to a different in-flight rollout. A rollback MUST NOT be satisfied by simply halting progression in place; it MUST actively return 100% of the affected traffic to `stable_version_ref`.

---

## 5. ROLLOUT CONTRACT (`RolloutState@1.0.0`)

Every rollout MUST produce and continuously update a structured, schema-validated record.

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "title": "RolloutState@1.0.0",
  "type": "object",
  "additionalProperties": false,
  "properties": {
    "rollout_id": { "type": "string", "minLength": 1 },
    "release_candidate_id": { "type": "string", "minLength": 1 },
    "release_type": {
      "type": "string",
      "enum": ["CODE", "PROMPT_TEMPLATE", "MODEL"]
    },
    "canary_stage": {
      "type": "string",
      "enum": [
        "SHADOW_GATE_PENDING",
        "STAGE_0_1PCT",
        "STAGE_1_5PCT",
        "STAGE_2_25PCT",
        "STAGE_3_100PCT",
        "STABLE",
        "ROLLED_BACK"
      ]
    },
    "traffic_percentage": { "type": "integer", "minimum": 0, "maximum": 100 },
    "shadow_gate_passed": { "type": "boolean" },
    "circuit_breaker_tripped": { "type": "boolean" },
    "stable_version_ref": { "type": "string", "minLength": 1 },
    "rollback_reason": { "type": "string", "minLength": 1 },
    "started_at": { "type": "string", "format": "date-time" },
    "last_transition_at": { "type": "string", "format": "date-time" }
  },
  "required": [
    "rollout_id",
    "release_candidate_id",
    "release_type",
    "canary_stage",
    "traffic_percentage",
    "shadow_gate_passed",
    "circuit_breaker_tripped",
    "stable_version_ref",
    "started_at",
    "last_transition_at"
  ]
}
```

`rollback_reason` is intentionally absent from `required`: it is only meaningful once `canary_stage = ROLLED_BACK` and MUST NOT be populated, even as an empty string, while a rollout is still progressing normally.

### Contract Semantics

For every state other than `ROLLED_BACK`: `circuit_breaker_tripped = false`.

For `ROLLED_BACK`: `circuit_breaker_tripped = true`, `traffic_percentage = 0`, and `rollback_reason` is present and non-empty.

`shadow_gate_passed` MUST be written exclusively by the `TEST-SPEC-20` verdict-ingestion service. No component owned by this document has a direct write path to that field (Section 6, `ERR_DEP_03_06`).

---

## 6. AUTOMATED ROLLOUT & ROLLBACK TEST HARNESS

```python
# Representative Automated Shadow-Gate Precondition Test
@pytest.mark.rtm(req_id="DEP-SPEC-03-ROL-001")
def test_rollout_blocked_without_passing_shadow_gate():
    candidate = release_candidate_factory(
        release_type="PROMPT_TEMPLATE",
        shadow_gate_passed=False
    )

    with pytest.raises(ShadowGateNotPassedException) as exc_info:
        rollout_engine.start_rollout(
            release_candidate_id=candidate.release_candidate_id
        )

    assert exc_info.value.error_code == "ERR_DEP_03_01"

    state = rollout_engine.get_state(candidate.release_candidate_id)
    assert state.canary_stage == "SHADOW_GATE_PENDING"
    assert state.traffic_percentage == 0


# Representative Automated Circuit Breaker Rollback Test
@pytest.mark.rtm(req_id="DEP-SPEC-03-ROL-002")
def test_automatic_rollback_on_circuit_breaker_breach():
    candidate = release_candidate_factory(
        release_type="CODE",
        shadow_gate_passed=True
    )

    rollout_engine.start_rollout(
        release_candidate_id=candidate.release_candidate_id
    )
    rollout_engine.force_stage(
        release_candidate_id=candidate.release_candidate_id,
        stage="STAGE_1_5PCT"
    )

    # Simulate a sustained P95 latency regression during the canary window
    metrics_feed.inject_latency_regression(
        release_candidate_id=candidate.release_candidate_id,
        p95_delta_pct=45
    )

    rollout_engine.evaluate_circuit_breaker(
        release_candidate_id=candidate.release_candidate_id
    )

    state = rollout_engine.get_state(candidate.release_candidate_id)

    assert state.circuit_breaker_tripped is True
    assert state.canary_stage == "ROLLED_BACK"
    assert state.traffic_percentage == 0
    assert state.stable_version_ref == candidate.previous_stable_version_ref

    # Rollback must already be complete, not waiting on a human approval step
    assert rollout_engine.has_pending_human_approval(
        candidate.release_candidate_id
    ) is False


# Representative Automated Stage Advancement Test
@pytest.mark.rtm(req_id="DEP-SPEC-03-ROL-003")
def test_stage_advancement_requires_metric_window_not_just_time():
    candidate = release_candidate_factory(
        release_type="CODE",
        shadow_gate_passed=True
    )

    rollout_engine.start_rollout(
        release_candidate_id=candidate.release_candidate_id
    )
    rollout_engine.force_stage(
        release_candidate_id=candidate.release_candidate_id,
        stage="STAGE_0_1PCT"
    )

    # Advance the clock past the minimum bake duration, but do not supply
    # a sufficient guest-request sample size for this stage.
    clock.advance(rollout_engine.stage_min_bake_duration("STAGE_0_1PCT"))
    metrics_feed.set_sample_size(
        release_candidate_id=candidate.release_candidate_id,
        observed_requests=1
    )

    rollout_engine.evaluate_stage_advancement(
        release_candidate_id=candidate.release_candidate_id
    )

    state = rollout_engine.get_state(candidate.release_candidate_id)

    # Elapsed time alone must not be sufficient; the stage must not advance.
    assert state.canary_stage == "STAGE_0_1PCT"


# Representative Automated DEP-SPEC-02 Separation Test
@pytest.mark.rtm(req_id="DEP-SPEC-03-ROL-004")
def test_dep_spec_02_eligibility_necessary_but_not_sufficient():
    # Tenant is fully LIVE per DEP-SPEC-02
    tenant = onboarding_engine.get_record(tenant_id="TENANT-9001")
    assert tenant.onboarding_status == "LIVE"

    candidate = release_candidate_factory(
        release_type="CODE",
        shadow_gate_passed=True
    )
    rollout_engine.start_rollout(
        release_candidate_id=candidate.release_candidate_id
    )
    rollout_engine.force_stage(
        release_candidate_id=candidate.release_candidate_id,
        stage="STAGE_0_1PCT"
    )

    # Tenant is explicitly pinned to stable, independent of canary cohort
    admission_gateway.pin_tenant_to_stable(tenant_id="TENANT-9001")

    decision = admission_gateway.route(
        tenant_id="TENANT-9001",
        release_candidate_id=candidate.release_candidate_id
    )

    # LIVE status alone did not admit this tenant to the new release
    assert decision.routed_version == candidate.previous_stable_version_ref
    assert decision.routed_to_candidate is False
```

---

## 7. SECURITY & THREAT MODEL

| Threat | Attack Vector | Preventive Control | Severity |
|---|---|---|---|
| Uncontrolled Blast Radius | A release is pushed directly to 100% traffic, bypassing staged canary exposure. | No code path exists to set `traffic_percentage = 100` except via sequential stage advancement or as a verified prior `STABLE` rollback target. | CRITICAL (SEV-0) |
| Rollback Suppression | An engineer manually pauses or disables the circuit breaker during a live regression to "give it more time." | Circuit-breaker evaluation runs in a service independent of the release-owner's control plane; suppressing it requires a separate, dual-authorized, fully audited action, itself logged as a SEV-1 event regardless of outcome. | CRITICAL (SEV-0) |
| Tenant Pin Violation | A tenant explicitly pinned to stable (e.g., for a compliance reason) is incorrectly routed canary traffic due to a routing bug. | Admission Gateway evaluates the explicit pin before deterministic cohort assignment on every request; a pin always overrides stage-percentage routing. | HIGH (SEV-1) |
| Shadow Gate Bypass | `shadow_gate_passed` is manually set `true` without an actual passing `TEST-SPEC-20` verdict, to skip verification. | The field has exactly one authorized writer: the `TEST-SPEC-20` verdict-ingestion service. The rollout orchestrator has no direct write path to it. | CRITICAL (SEV-0) |

---

## 8. FAILURE ARCHITECTURE & ERROR TAXONOMY

| Error Code | Description | Canonical Class | Severity |
|---|---|---|---|
| ERR_DEP_03_01 | Rollout initiated without a passing TEST-SPEC-20 shadow gate verdict. | PROCESS_VIOLATION | CRITICAL (SEV-0) |
| ERR_DEP_03_02 | Canary stage advanced without a completed automated metric evaluation window. | PROCESS_VIOLATION | HIGH (SEV-1) |
| ERR_DEP_03_03 | Circuit breaker threshold breached but automatic rollback did not execute. | SAFETY_VIOLATION | CRITICAL (SEV-0) |
| ERR_DEP_03_04 | Rollback target could not be verified as a previously stable version. | VALIDATION | CRITICAL (SEV-0) |
| ERR_DEP_03_05 | Tenant-level traffic pin was not honored by the admission gateway. | ISOLATION_VIOLATION | HIGH (SEV-1) |
| ERR_DEP_03_06 | `shadow_gate_passed` mutated by a source other than the TEST-SPEC-20 verdict-ingestion service. | SECURITY_VIOLATION | CRITICAL (SEV-0) |
| ERR_DEP_03_07 | RolloutState record failed JSON Schema validation. | CONTRACT_MISMATCH | HIGH (SEV-1) |
| ERR_DEP_03_08 | Traffic admitted for a release candidate to a tenant not currently marked LIVE by DEP-SPEC-02. | RUNTIME_AUTHORIZATION | CRITICAL (SEV-0) |

---

## 9. PRODUCTION ACCEPTANCE CRITERIA

| AC ID | Category | Requirement | Verification Method | Pass/Fail |
|---|---|---|---|---|
| AC-ROL-01 | Shadow Gate Enforcement | 100% of rollouts blocked from any live traffic until `shadow_gate_passed = true` is recorded by TEST-SPEC-20. | Gate Enforcement Test Suite | REQUIRED |
| AC-ROL-02 | Staged Progression | 100% of rollouts traverse 1% → 5% → 25% → 100% in order; 0 direct-to-100% transitions outside a verified rollback-to-stable. | Stage Sequence Audit | REQUIRED |
| AC-ROL-03 | Automatic Rollback | 100% of circuit-breaker threshold breaches trigger rollback within the configured detection-to-rollback window, with 0 pending human-approval steps. | Circuit Breaker Test Suite | REQUIRED |
| AC-ROL-04 | Tenant Pin Integrity | 100% of tenants explicitly pinned to stable are never routed canary traffic, at any stage. | Pin Integrity Test Suite | REQUIRED |
| AC-ROL-05 | Schema Compliance | 100% of RolloutState records validate against RolloutState@1.0.0. | Schema Inspector | REQUIRED |
| AC-ROL-06 | DEP-SPEC-02 Precondition | 0 instances of any traffic (canary or stable) routed to a tenant not currently marked LIVE by DEP-SPEC-02. | Cross-Spec Admission Audit | REQUIRED |
| AC-ROL-07 | Rollback Target Integrity | 100% of rollbacks resolve to a verified previously-stable version, never a partially-rolled-out intermediate state. | Rollback Target Audit | REQUIRED |

---

## 10. INTEGRATION AUTHORITY MATRIX

| Component | Owns | Consumes | Produces |
|---|---|---|---|
| DEP-SPEC-03 | Canary staging, traffic splitting, circuit breaker & automatic rollback, Runtime Admission Gateway, RolloutState | Release candidates, Shadow replay verdicts (TEST-SPEC-20), Tenant LIVE eligibility (DEP-SPEC-02) | Routed live traffic, RolloutState records, Rollback events |
| TEST-SPEC-20 | Shadow traffic replay & verdict computation | Release candidate | Shadow replay verdict (consumed here as a hard precondition) |
| DEP-SPEC-02 | Tenant onboarding, identity, pre-go-live gate | — | Tenant LIVE eligibility (consumed here as a precondition, not a routing decision) |
| DEP-SPEC-01 | Environment tiers & secrets platform | — | Provisioned PROD environment this pipeline operates within |
| TEST-SPEC-02 | CI/CD stage gate enforcement | RolloutState signals | Deployment gate go/no-go decisions |
| TEST-SPEC-17 | Prompt regression & semantic drift scoring | PROMPT_TEMPLATE / MODEL release candidates | Drift scores (consumed here as one circuit-breaker input) |

---

## 11. FINAL NON-NEGOTIABLE PRINCIPLES

NO RELEASE CANDIDATE MAY RECEIVE ANY LIVE GUEST TRAFFIC WITHOUT A PASSING TEST-SPEC-20 SHADOW REPLAY VERDICT.

CANARY STAGE ADVANCEMENT MUST BE DRIVEN EXCLUSIVELY BY AUTOMATED METRIC EVALUATION, NEVER BY ELAPSED TIME ALONE.

AUTOMATIC ROLLBACK ON CIRCUIT BREAKER BREACH MUST NEVER REQUIRE HUMAN APPROVAL TO INITIATE.

A TENANT'S DEP-SPEC-02 LIVE STATUS IS A PRECONDITION FOR RECEIVING TRAFFIC, NOT AN AUTHORIZATION TO RECEIVE ANY SPECIFIC RELEASE'S TRAFFIC SHARE.

ROLLBACK MUST ALWAYS TARGET A VERIFIED LAST-KNOWN-STABLE VERSION, NEVER A PARTIALLY-ROLLED-OUT INTERMEDIATE STATE.

---

## 12. VERSION HISTORY & ARCHITECTURAL VERDICT

| Version | Date | Description | Author | Status |
|---|---|---|---|---|
| 1.0.0 | August 2026 | Initial specification of Progressive Rollout & Traffic Migration (DEP-SPEC-03). Establishes the four-stage canary model (1%/5%/25%/100%), the mandatory TEST-SPEC-20 shadow-gate precondition, deterministic traffic splitting with tenant-pin precedence, automatic circuit-breaker rollback with no human-approval dependency, the RolloutState@1.0.0 contract, and the explicit separation between DEP-SPEC-02 tenant eligibility and release-level traffic routing. | Ramy Bella | SUPERSEDED |
| 1.0.1 | August 2026 | Consistency pass. Added TEST-SPEC-08 and TEST-SPEC-09 to Related Documents — both are cited in Section 4.3 as circuit-breaker safety/security regression inputs but were missing from the header field. No change to the rollout mechanics, schema, error taxonomy, or acceptance criteria. | Ramy Bella | APPROVED FOR IMPLEMENTATION |

**ARCHITECTURAL VERDICT:**
APPROVED FOR IMPLEMENTATION. DEP-SPEC-03 establishes the staged canary rollout model, the shadow-gate precondition inherited from TEST-SPEC-20, deterministic traffic-splitting and tenant-pin rules, fully automatic circuit-breaker rollback, and the Runtime Admission Gateway that bridges DEP-SPEC-02 tenant eligibility to actual release-version routing. Ready for implementation.
