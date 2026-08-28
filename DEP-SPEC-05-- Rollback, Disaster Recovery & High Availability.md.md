# DEP-SPEC-05: Rollback, Disaster Recovery & High Availability

## 1. DOCUMENT CONTROL

| Attribute         | Value                                                                         |
| ----------------- | ----------------------------------------------------------------------------- |
| Document Title    | 05 Rollback, Disaster Recovery & High Availability.md                         |
| Document ID       | DEP-SPEC-05                                                                   |
| Version           | 1.0.0                                                                         |
| Status            | APPROVED FOR IMPLEMENTATION                                                   |
| Author            | Ramy Bella                                                                    |
| Classification    | Confidential / Enterprise Proprietary                                         |
| Target Audience   | SREs, Incident Commanders, Platform Engineers, Database Reliability Engineers |
| Parent Document   | DEP-SPEC-01                                                                   |
| Related Documents | DEP-SPEC-01, DEP-SPEC-03, DEP-SPEC-04, DEP-SPEC-06, DEP-SPEC-07, TEST-SPEC-16 |
| System            | Restaurant AI System                                                          |
| Phase             | Phase 7 — Deployment & Production Operations                                  |
| Lifecycle Folder  | 07 Deployment & Infrastructure                                                |
| Last Updated      | August 2026                                                                   |

---

## 2. EXECUTIVE PURPOSE & SCOPE BOUNDARIES

`DEP-SPEC-03` rolls a bad release back safely — as long as no guest data exists only in the new environment. `DEP-SPEC-04` defines the exact instant that stops being true: the Point of No Return. `DEP-SPEC-05` is what takes over the moment that instant passes and something still goes wrong — along with the two failure modes that have nothing to do with a release rollout at all: an LLM provider going down mid-conversation, and a region or database failing outright on an ordinary Tuesday.

**Owns:** RTO/RPO SLA definitions per disaster class; the post-PONR data reconciliation mechanism; automatic LLM provider fallback/failover; multi-region and cross-AZ high-availability architecture; the `DisasterRecoveryVerdict@1.0.0` contract.

**Delegates:** pre-PONR canary rollback → `DEP-SPEC-03`; PONR detection and cutover sequencing → `DEP-SPEC-04`; infrastructure and VPC peering → `DEP-SPEC-01`; telemetry and threshold configuration → `DEP-SPEC-06`; PagerDuty and on-call/manual escalation → `DEP-SPEC-07`.

This document never re-litigates whether PONR was crossed — that determination belongs entirely to `DEP-SPEC-04`. It begins exactly where that determination ends.

### Core Operational Invariants

- `POST-PONR FAILURE ⇒ DATA RECONCILIATION VIA THIS DOCUMENT, NEVER A DEP-SPEC-03 CANARY ROLLBACK`
- `SEV-0 TOTAL REGION FAILURE ⇒ RTO ≤ 15 MIN AND RPO ≤ 1 S, NO EXCEPTION`
- `SEV-0 LLM PROVIDER OUTAGE ⇒ AUTOMATIC FAILOVER WITHIN RTO ≤ 5 S, NO HUMAN APPROVAL REQUIRED TO INITIATE`
- `SEV-1 DATABASE SPLIT-BRAIN ⇒ RPO = 0 S, ZERO TOLERANCE FOR LOST BOOKINGS`
- `DISASTER RECOVERY VERDICT = 100% SCHEMA-VALIDATED PROOF OF RTO/RPO ATTAINMENT, NEVER A SELF-ATTESTED CLAIM`

---

## 3. RECOVERY ARCHITECTURE

### 3.1 Disaster Classification & RTO/RPO SLA Matrix

| Disaster Class | Scenario | RTO | RPO | Recovery Mechanism |
|---|---|---|---|---|
| SEV-0 | Total Region Failure | ≤ 15 min | ≤ 1 s | Synchronous cross-region replication; automated regional failover |
| SEV-0 | LLM Provider Outage | ≤ 5 s | N/A | Automatic circuit-breaker failover to secondary LLM provider |
| SEV-1 | Database Split-Brain | ≤ 30 min | 0 s | Quorum-based conflict resolution + confirmed reconciliation |

RPO is marked N/A for LLM Provider Outage because an inference call carries no persisted guest state of its own — recovery for that class is measured by RTO alone. Every other class MUST report a numeric RPO.

### 3.2 Post-PONR Data Reconciliation Engine

Activated exclusively by a `DEP-SPEC-04` PONR-breach signal. Runs in strict order:

```text
[DEP-SPEC-04 SEV-0 DISASTER SIGNAL RECEIVED]
           │
           ▼
[1. FREEZE WRITES] on the failed environment side to stop further divergence
           │
           ▼
[2. ENUMERATE POST-PONR WRITES]
  Every guest-facing write committed on the new environment since the
  PONR crossing instant (DEP-SPEC-04 §3.3) with no corresponding record
  on the prior stable environment.
           │
           ▼
[3. RECONCILE] each write into the recovery target using idempotent,
  guest-transaction-ID-keyed merge semantics
           │
     ┌─────┴─────┐
     ▼           ▼
[CLEAN MERGE]  [GENUINE CONFLICT]
     │       (e.g., two guests double-booked
     │        the same table/time-slot across
     │        environments — not a technical
     │        problem this engine may silently
     │        resolve on its own)
     │           │
     │           ▼
     │  [FLAG TO RECONCILIATION EXCEPTION QUEUE]
     │   routed to on-call via DEP-SPEC-07
     │           │
     └─────┬─────┘
           ▼
[4. RECOVERED ELIGIBLE ONLY WHEN unreconciled_writes_count == 0]
  every enumerated write is either reconciled or explicitly flagged —
  never silently dropped, never silently left pending
```

### 3.3 LLM Provider Fallback & Failover Architecture

A circuit breaker continuously measures the primary LLM provider's error rate, latency, and explicit outage signals from the provider's own status endpoint (thresholds owned and configured by `DEP-SPEC-06`). On breach:

1. Traffic re-routes automatically to the secondary LLM provider — no human approval required to initiate, matching the same automation posture as `DEP-SPEC-03`'s circuit-breaker rollback.
2. The already-compiled, provider-agnostic prompt payload (`CompiledPromptPayload@1.0.0`, `TEST-SPEC-16`) is re-routed as-is. It is never recompiled or reconstructed for the secondary provider — failover changes the destination, not the payload.
3. Fail-back to the primary provider occurs only after primary health has remained within nominal bounds for a sustained evaluation window — never on a single passing health check — to avoid rapid oscillation between providers mid-incident.

### 3.4 Multi-Region & Cross-AZ High Availability

Database replication is synchronous across Availability Zones within a region and configured per the applicable RPO bound (Section 3.1) across regions. This document owns the failover *decision logic* — when a standby is eligible for promotion, when regional traffic redirects — while `DEP-SPEC-01` owns the underlying VPC peering and network resources that logic operates over. A standby is eligible for promotion only once its replication lag is confirmed within the disaster class's RPO bound; a lagging standby is never promoted, regardless of how long the primary has been down (Section 6, `ERR_DEP_05_03`). Regional failover triggers on a sustained regional health-check failure window, not a single failed check.

---

## 4. CONTRACT SCHEMA (`DisasterRecoveryVerdict@1.0.0`)

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "title": "DisasterRecoveryVerdict@1.0.0",
  "type": "object",
  "additionalProperties": false,
  "properties": {
    "verdict_id": { "type": "string", "minLength": 1 },
    "disaster_class": {
      "type": "string",
      "enum": ["SEV0_REGION_FAILURE", "SEV0_LLM_PROVIDER_OUTAGE", "SEV1_DB_SPLIT_BRAIN"]
    },
    "triggering_cutover_id": { "type": "string", "minLength": 1 },
    "status": {
      "type": "string",
      "enum": ["DETECTED", "RECOVERING", "RECONCILING", "RECOVERED", "RTO_BREACHED", "RPO_BREACHED"]
    },
    "detected_at": { "type": "string", "format": "date-time" },
    "recovered_at": { "type": "string", "format": "date-time" },
    "measured_rto_seconds": { "type": "number", "minimum": 0 },
    "measured_rpo_seconds": { "type": "number", "minimum": 0 },
    "rto_target_seconds": { "type": "number", "minimum": 0 },
    "rpo_target_seconds": { "type": ["number", "null"], "minimum": 0 },
    "rto_met": { "type": "boolean" },
    "rpo_met": { "type": "boolean" },
    "unreconciled_writes_count": { "type": "integer", "minimum": 0 },
    "reconciliation_exceptions_flagged": { "type": "integer", "minimum": 0 }
  },
  "required": [
    "verdict_id",
    "disaster_class",
    "status",
    "detected_at",
    "measured_rto_seconds",
    "measured_rpo_seconds",
    "rto_target_seconds",
    "rpo_target_seconds",
    "rto_met",
    "rpo_met",
    "unreconciled_writes_count",
    "reconciliation_exceptions_flagged"
  ],
  "allOf": [
    {
      "if": {
        "properties": { "status": { "const": "RECOVERED" } },
        "required": ["status"]
      },
      "then": {
        "properties": {
          "rto_met": { "const": true },
          "rpo_met": { "const": true },
          "unreconciled_writes_count": { "const": 0 }
        },
        "required": ["rto_met", "rpo_met", "unreconciled_writes_count", "recovered_at"]
      }
    },
    {
      "if": {
        "properties": { "status": { "const": "RTO_BREACHED" } },
        "required": ["status"]
      },
      "then": {
        "properties": { "rto_met": { "const": false } },
        "required": ["rto_met", "recovered_at"]
      }
    },
    {
      "if": {
        "properties": { "status": { "const": "RPO_BREACHED" } },
        "required": ["status"]
      },
      "then": {
        "properties": { "rpo_met": { "const": false } },
        "required": ["rpo_met", "recovered_at"]
      }
    }
  ]
}
```

### Contract Semantics

`triggering_cutover_id` is intentionally absent from `required`: it links a verdict back to the `CutoverExecutionRecord` (`DEP-SPEC-04`) only when the disaster arose from a post-PONR breach. An LLM provider outage on an ordinary day has no associated cutover and MUST NOT be forced to invent one.

`recovered_at` is required only once a verdict reaches a terminal state (`RECOVERED`, `RTO_BREACHED`, or `RPO_BREACHED`) — the same reasoning applied to `completed_at` in `DEP-SPEC-04`: a value with no honest answer yet must not be required yet.

`unreconciled_writes_count` and `reconciliation_exceptions_flagged` remain unconditionally required, defaulting to `0` at verdict creation, because — like the gate booleans in `DEP-SPEC-04` — a count always has a defined value from the moment the record exists; there is no "not yet applicable" state for a counter.

A verdict MUST NOT reach `RECOVERED` while `unreconciled_writes_count` is nonzero, even if every other metric is within target. A clean RTO/RPO with an open reconciliation conflict is not a recovered system.

---

## 5. AUTOMATED DR TEST HARNESS

```python
# Representative Automated LLM Provider Failover Test
@pytest.mark.rtm(req_id="DEP-SPEC-05-DR-001")
def test_llm_provider_outage_triggers_failover_within_five_seconds():
    # Simulate a hard outage on the primary LLM provider
    llm_provider_feed.simulate_outage(provider="PRIMARY")

    dr_engine.evaluate_circuit_breaker(provider="PRIMARY")

    verdict = dr_engine.get_verdict(disaster_class="SEV0_LLM_PROVIDER_OUTAGE")

    assert verdict.status == "RECOVERED"
    assert verdict.measured_rto_seconds <= 5
    assert verdict.rto_met is True

    # No guest-facing data loss is possible for a stateless inference call,
    # but the reconciliation counters must still read clean.
    assert verdict.unreconciled_writes_count == 0
    assert verdict.rpo_met is True

    # Failover must have completed automatically, not via a paged human
    assert dr_engine.has_pending_human_decision(verdict.verdict_id) is False

    # The already-compiled prompt payload was re-routed, not recompiled
    routing_event = dr_engine.get_last_failover_event(verdict.verdict_id)
    assert routing_event.target_provider == "SECONDARY"
    assert routing_event.prompt_recompiled is False


# Representative Automated Post-PONR Reconciliation Gate Test
@pytest.mark.rtm(req_id="DEP-SPEC-05-DR-002")
def test_recovered_status_blocked_while_writes_remain_unreconciled():
    verdict = dr_engine.declare_disaster(
        disaster_class="SEV0_REGION_FAILURE",
        triggering_cutover_id="CUT-2026-08-21-001"
    )

    # Two post-PONR guest writes are found with no matching record on the
    # recovery target: one auto-reconciles cleanly, one is a genuine
    # double-booking conflict that requires human resolution.
    reconciliation_engine.enumerate_post_ponr_writes(verdict.verdict_id)
    reconciliation_engine.reconcile_write(
        verdict_id=verdict.verdict_id,
        write_id="WRITE-001"
    )
    reconciliation_engine.flag_exception(
        verdict_id=verdict.verdict_id,
        write_id="WRITE-002",
        reason="conflicting_table_assignment"
    )

    with pytest.raises(SchemaValidationException) as exc_info:
        dr_engine.transition_status(
            verdict_id=verdict.verdict_id,
            new_status="RECOVERED"
        )

    assert exc_info.value.error_code == "ERR_DEP_05_04"

    record = dr_engine.get_verdict_record(verdict.verdict_id)
    assert record.status != "RECOVERED"
    assert record.unreconciled_writes_count == 1
    assert record.reconciliation_exceptions_flagged == 1
```

---

## 6. SECURITY & THREAT MODEL

| Threat | Attack Vector | Preventive Control | Severity |
|---|---|---|---|
| Silent Data Loss During Failover | A region failover promotes a standby that was behind on replication, silently losing recent bookings. | Promotion requires confirmed replication lag within the applicable RPO bound; a lagging standby is never eligible for promotion regardless of primary downtime duration. | CRITICAL (SEV-0) |
| Reconciliation Bypass | An operator manually forces `status` to `RECOVERED` to close an incident before all flagged writes are resolved. | `RECOVERED` is schema-gated on `unreconciled_writes_count == 0`; no write path exists to force the transition while the count is nonzero. | CRITICAL (SEV-0) |
| Prompt Context Corruption on Failover | The secondary LLM provider receives a re-derived or reformatted prompt that differs from what the primary would have received, changing model behavior mid-incident. | Failover re-routes the exact already-compiled, provider-agnostic prompt payload; it is never recompiled or reconstructed for the secondary provider. | HIGH (SEV-1) |
| Failover Flapping | Primary provider or region recovers briefly then fails again, causing rapid oscillation that itself degrades guest experience. | Fail-back requires a sustained healthy evaluation window, never a single passing health check. | MEDIUM (SEV-2) |

---

## 7. FAILURE ARCHITECTURE & ERROR TAXONOMY

| Error Code | Description | Canonical Class | Severity |
|---|---|---|---|
| ERR_DEP_05_01 | A declared disaster exceeded its disaster class's RTO target before recovery completed. | SAFETY_VIOLATION | CRITICAL (SEV-0) |
| ERR_DEP_05_02 | A declared disaster exceeded its disaster class's RPO target (data loss beyond the tolerated window). | SAFETY_VIOLATION | CRITICAL (SEV-0) |
| ERR_DEP_05_03 | A standby was promoted for region failover without confirmed replication lag within the applicable RPO bound. | DATA_INTEGRITY | CRITICAL (SEV-0) |
| ERR_DEP_05_04 | DisasterRecoveryVerdict was transitioned toward RECOVERED while unreconciled_writes_count remained nonzero. | CONTRACT_MISMATCH | HIGH (SEV-1) |

---

## 8. PRODUCTION ACCEPTANCE CRITERIA

| AC ID | Category | Requirement | Verification Method | Pass/Fail |
|---|---|---|---|---|
| AC-DR-01 | RTO Compliance | 100% of declared disasters recover within their disaster class's RTO target; 0 unflagged RTO breaches. | RTO Audit | REQUIRED |
| AC-DR-02 | RPO Compliance | 100% of declared disasters recover within their disaster class's RPO target; 0 tolerated lost bookings for SEV-1 Database Split-Brain. | RPO Audit | REQUIRED |
| AC-DR-03 | LLM Failover Speed | 100% of simulated primary LLM provider outages fail over to the secondary provider within 5 seconds, with the original compiled prompt payload preserved unchanged. | DR Test Harness | REQUIRED |
| AC-DR-04 | Reconciliation Completeness | 0 DisasterRecoveryVerdict records reach RECOVERED status with a nonzero unreconciled_writes_count. | Schema Inspector | REQUIRED |
| AC-DR-05 | Replication Safety | 100% of region-failover standby promotions occur only with replication lag confirmed within the applicable RPO bound. | Promotion Safety Audit | REQUIRED |

---

## 9. INTEGRATION AUTHORITY MATRIX

| Component | Owns | Consumes | Produces |
|---|---|---|---|
| DEP-SPEC-05 | RTO/RPO SLA matrix, post-PONR reconciliation engine, LLM provider fallback/failover, multi-region & cross-AZ HA, DisasterRecoveryVerdict | Post-PONR disaster signal (DEP-SPEC-04), infra/VPC topology (DEP-SPEC-01), telemetry thresholds (DEP-SPEC-06) | DisasterRecoveryVerdict records, reconciliation exception queue entries |
| DEP-SPEC-03 | Pre-PONR canary staging & rollback | — | Pre-PONR rollback mechanism (this document's scope begins only where that one ends) |
| DEP-SPEC-04 | Cutover sequencing & PONR detection | — | PONR crossing / SEV-0 disaster declaration trigger (this document's activation signal) |
| DEP-SPEC-01 | Environment tiers, VPC peering, secrets platform | — | Multi-region/cross-AZ network topology this document fails over across |
| DEP-SPEC-06 | Telemetry & alerting | Circuit-breaker/HA trigger conditions | Latency, error-rate, and provider-health signals feeding disaster detection |
| DEP-SPEC-07 | PagerDuty & incident management | Reconciliation exception queue entries (this document) | On-call escalation & manual resolution of flagged conflicts |

---

## 10. FINAL NON-NEGOTIABLE PRINCIPLES

POST-PONR FAILURES MUST BE HANDLED BY THIS DOCUMENT'S DATA RECONCILIATION ENGINE, NEVER BY A DEP-SPEC-03 CANARY ROLLBACK.

SEV-0 TOTAL REGION FAILURE MUST RECOVER WITHIN 15 MINUTES WITH NO MORE THAN 1 SECOND OF DATA LOSS.

SEV-0 LLM PROVIDER OUTAGE MUST FAIL OVER TO A SECONDARY PROVIDER WITHIN 5 SECONDS WITHOUT HUMAN APPROVAL TO INITIATE.

SEV-1 DATABASE SPLIT-BRAIN MUST RESOLVE WITH ZERO TOLERANCE FOR LOST GUEST BOOKINGS.

A DISASTERRECOVERYVERDICT MUST BE SCHEMA-VALIDATED PROOF OF RTO/RPO ATTAINMENT, NEVER A SELF-ATTESTED CLAIM.

---

## 11. VERSION HISTORY & ARCHITECTURAL VERDICT

| Version | Date | Description | Author | Status |
|---|---|---|---|---|
| 1.0.0 | August 2026 | Initial specification of Rollback, Disaster Recovery & High Availability (DEP-SPEC-05). Establishes the RTO/RPO SLA matrix per disaster class, the post-PONR data reconciliation engine, automatic LLM provider fallback/failover with prompt-context preservation, multi-region/cross-AZ HA promotion rules, and the DisasterRecoveryVerdict@1.0.0 contract. | Ramy Bella | APPROVED FOR IMPLEMENTATION |

**ARCHITECTURAL VERDICT:**
APPROVED FOR IMPLEMENTATION. DEP-SPEC-05 establishes the RTO/RPO SLA matrix, the post-PONR reconciliation engine that takes over exactly where DEP-SPEC-04's PONR boundary ends, automatic LLM provider failover with preserved prompt context, and the DisasterRecoveryVerdict@1.0.0 contract proving every recovery against a schema rather than a claim. Ready for implementation.
