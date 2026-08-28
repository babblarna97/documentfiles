# DEP-SPEC-07: Incident Response, Severity Taxonomy & Playbooks

## 1. DOCUMENT CONTROL

| Attribute         | Value                                                                                                                               |
| ----------------- | ----------------------------------------------------------------------------------------------------------------------------------- |
| Document Title    | 07 Incident Response, Severity Taxonomy & Playbooks.md                                                                              |
| Document ID       | DEP-SPEC-07                                                                                                                         |
| Version           | 1.0.1                                                                                                                               |
| Status            | APPROVED FOR IMPLEMENTATION                                                                                                         |
| Author            | Ramy Bella                                                                                                                          |
| Classification    | Confidential / Enterprise Proprietary                                                                                               |
| Target Audience   | SREs, Incident Commanders, Security Lead, On-Call Engineers                                                                         |
| Parent Document   | DEP-SPEC-01                                                                                                                         |
| Related Documents | DEP-SPEC-01, DEP-SPEC-03, DEP-SPEC-04, DEP-SPEC-05, DEP-SPEC-06, TEST-SPEC-02, TEST-SPEC-08, TEST-SPEC-09, INT-SPEC-15, INT-SPEC-18 |
| System            | Restaurant AI System                                                                                                                |
| Phase             | Phase 7 — Deployment & Production Operations                                                                                        |
| Lifecycle Folder  | 07 Deployment & Infrastructure                                                                                                      |
| Last Updated      | August 2026                                                                                                                         |

---

## 2. EXECUTIVE PURPOSE & SCOPE BOUNDARIES

Every other Phase 7 document either generates a signal — a telemetry breach (`DEP-SPEC-06`), a circuit-breaker trip (`DEP-SPEC-03`), a disaster declaration (`DEP-SPEC-05`) — or executes an automated response to one. None of them tells a human what that signal actually means in terms of urgency, who gets woken up, how fast, or what to do once they're awake. `DEP-SPEC-07` is that layer: the severity taxonomy, the on-call escalation ladder, the break-glass protocol for direct production access, the auto-remediation playbooks for known failure patterns, and the structured postmortem contract that closes the loop.

**Owns:** the production severity taxonomy (SEV-0 through SEV-3); the incident lifecycle and on-call routing/escalation; the break-glass access protocol; auto-remediation playbooks; the `IncidentResolutionRecord@1.0.0` contract.

**Delegates:** alert generation and threshold detection → `DEP-SPEC-06` (which pages this document — not the reverse); the technical execution of a rollback → `DEP-SPEC-03` and `DEP-SPEC-04` (this document only dictates when a human must intervene if the automation itself fails); the disaster-recovery mechanism and data reconciliation → `DEP-SPEC-05` (this document owns communication and the manual-declare path for cases automation didn't catch, not the reconciliation engine); telemetry storage → `INT-SPEC-18`.

### Two Severity Taxonomies, on Purpose

`DEP-SPEC-05`'s `disaster_class` values (e.g., `SEV0_LLM_PROVIDER_OUTAGE`) classify *which automated recovery mechanism applies* — they are not human-paging severities. This document's SEV-0 through SEV-3 taxonomy classifies *human response urgency* instead, and the two do not always align: an LLM provider outage is a `SEV0` disaster class in `DEP-SPEC-05` because it demands an automatic sub-5-second failover with zero tolerance for delay, but it pages as **SEV-1** here (Section 3) — a human isn't needed for the common case, since the automation is expected to resolve it faster than a person could act anyway. The two taxonomies reconnect at exactly one point: if a `DEP-SPEC-05` recovery ever breaches its own RTO/RPO target — meaning the automation failed — that failure escalates automatically to this document's **SEV-0** (Section 3.2).

### Core Operational Invariants

- `SEV-0 DECLARATION ⇒ ON-CALL MUST ACKNOWLEDGE WITHIN 5 MINUTES, NO EXCEPTION`
- `BREAK-GLASS PRODUCTION ACCESS ⇒ MANDATORY DUAL-CONTROL KMS AUTHORIZATION (TEST-SPEC-02), NEVER SINGLE-OPERATOR`
- `ANY DEP-SPEC-03/04/05 AUTOMATED RECOVERY THAT BREACHES ITS OWN SLA ⇒ AUTOMATIC ESCALATION TO SEV-0 HUMAN PAGING`
- `INCIDENT CANNOT REACH RESOLVED WITHOUT A RECORDED MITIGATION; CANNOT REACH POSTMORTEM CLOSURE WITHOUT ROOT CAUSE AND ACTION ITEMS`
- `INCIDENTRESOLUTIONRECORD = STRUCTURED, SCHEMA-VALIDATED POSTMORTEM, NEVER FREE-TEXT PROSE`

---

## 3. SEVERITY TAXONOMY, PAGING MATRIX & INCIDENT STATE MACHINE

### 3.1 Severity Definitions

| Severity | Definition | Examples | Paged | Ack SLA |
|---|---|---|---|---|
| SEV-0 | Critical system or security failure | Prompt injection leak (`TEST-SPEC-08`), cross-tenant data breach (`TEST-SPEC-09`), total region down (`DEP-SPEC-05` `SEV0_REGION_FAILURE`) | SRE Lead + Security Lead, any hour | ≤ 5 min |
| SEV-1 | Major disruption | LLM provider outage in the common self-healing case, backend booking API unreachable | Primary On-Call | ≤ 15 min |
| SEV-2 | Degraded performance | Error budget flagged CONSUMED (`DEP-SPEC-06`) without a full outage; elevated latency not yet at SLO breach | Primary On-Call, business-hours-aware | ≤ 60 min during business hours; next business day otherwise |
| SEV-3 | Non-urgent | A minor memory leak caught before it caused a crash; an isolated schema-validation error in a rare webhook | Handled during business hours | No night paging |

### 3.2 Escalation Rule (Automation-Failure Bridge)

If any `DEP-SPEC-03`, `DEP-SPEC-04`, or `DEP-SPEC-05` automated mechanism breaches its own configured SLA — a circuit breaker that should have rolled back and didn't, a cutover abort that didn't complete, a disaster recovery that exceeded its RTO/RPO target — that breach escalates automatically to **SEV-0** in this taxonomy, regardless of what severity the underlying event would otherwise have carried. Automation failing silently is never acceptable; if the machine can't fix it, a human must be paged immediately.

### 3.3 Escalation Ladder

`L1 (Primary On-Call) → L2 (Secondary On-Call) → SRE Lead`. An incident unacknowledged past its severity's SLA auto-escalates to the next tier on an independent timer — the escalation clock is not owned or pausable by the on-call engineer it is paging (Section 8, `ERR_DEP_07_04` is not the applicable code here; see Section 7's Escalation Ladder Bypass control).

### 3.4 Incident State Machine

```text
DETECTED → ACKNOWLEDGED → INVESTIGATING → MITIGATED → RESOLVED → POSTMORTEM
```

Linear and immutable: no state may be skipped, and no transition may move backward. `RESOLVED` closes guest-facing impact; `POSTMORTEM` closes the incident organizationally, and only once the structured analysis in Section 5 is complete.

---

## 4. BREAK-GLASS ACCESS & AUTO-REMEDIATION ARCHITECTURE

### 4.1 Break-Glass Access Protocol

At SEV-0, direct database or infrastructure access is sometimes unavoidable. This document reuses `TEST-SPEC-02`'s existing Dual-Control Auth mechanism rather than defining a separate one: a temporary, time-limited production key is issued only on simultaneous KMS signature from two authorized roles (System Architect + Security Lead, or SRE Lead + Security Lead, depending on the incident), exactly as `TEST-SPEC-02` already requires for emergency deployment bypasses. No single operator may self-authorize break-glass access under any circumstance.

This does not violate `DEP-SPEC-01`'s infrastructure-as-code rules — it is the same exception path `TEST-SPEC-02` already defines for emergency hotfixes: a break-glass session is temporary by construction and MUST be filed for retroactive IaC reconciliation within 24 hours of issuance, the identical window `TEST-SPEC-02` mandates for emergency deployment compliance. A break-glass key that expires without a linked `incident_id` or without retroactive filing is itself a SEV-0 finding (Section 8, `ERR_DEP_07_05`).

### 4.2 Auto-Remediation Playbooks

Distinct from break-glass: an auto-remediation playbook is a codified, version-controlled script an authenticated on-call engineer runs against a known failure pattern — it does not grant raw production access. Two representative playbooks:

- **Force a circuit breaker to `HALF_OPEN`** — for cases where a `DEP-SPEC-03`/`DEP-SPEC-05` circuit breaker remains tripped after the underlying condition has genuinely cleared, allowing a controlled re-evaluation rather than waiting out the full automatic cool-down.
- **Clear a dead Redis lock** — for cases where an `INT-SPEC-15` integration adapter's idempotency lock was orphaned by a crashed process and is now blocking legitimate retries.

Every playbook execution requires an authenticated on-call identity, is logged against the triggering `incident_id`, and is itself schema-validated as part of the `IncidentResolutionRecord`'s mitigation trail — it is never an untracked shell command.

---

## 5. CONTRACT SCHEMA (`IncidentResolutionRecord@1.0.0`)

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "title": "IncidentResolutionRecord@1.0.0",
  "type": "object",
  "additionalProperties": false,
  "properties": {
    "incident_id": { "type": "string", "minLength": 1 },
    "severity": { "type": "string", "enum": ["SEV-0", "SEV-1", "SEV-2", "SEV-3"] },
    "status": {
      "type": "string",
      "enum": ["DETECTED", "ACKNOWLEDGED", "INVESTIGATING", "MITIGATED", "RESOLVED", "POSTMORTEM"]
    },
    "triggering_signal_source": { "type": "string", "minLength": 1 },
    "detected_at": { "type": "string", "format": "date-time" },
    "acknowledged_by": { "type": "string", "minLength": 1 },
    "acknowledged_at": { "type": "string", "format": "date-time" },
    "mitigation_summary": { "type": "string", "minLength": 1 },
    "resolved_at": { "type": "string", "format": "date-time" },
    "root_cause_analysis": { "type": "string", "minLength": 1 },
    "action_items": {
      "type": "array",
      "items": {
        "type": "object",
        "additionalProperties": false,
        "properties": {
          "description": { "type": "string", "minLength": 1 },
          "owner": { "type": "string", "minLength": 1 },
          "due_date": { "type": "string", "format": "date" }
        },
        "required": ["description", "owner"]
      }
    }
  },
  "required": [
    "incident_id",
    "severity",
    "status",
    "triggering_signal_source",
    "detected_at"
  ],
  "allOf": [
    {
      "if": {
        "properties": { "status": { "enum": ["ACKNOWLEDGED", "INVESTIGATING", "MITIGATED", "RESOLVED", "POSTMORTEM"] } },
        "required": ["status"]
      },
      "then": {
        "required": ["acknowledged_by", "acknowledged_at"]
      }
    },
    {
      "if": {
        "properties": { "status": { "enum": ["MITIGATED", "RESOLVED", "POSTMORTEM"] } },
        "required": ["status"]
      },
      "then": {
        "required": ["mitigation_summary"]
      }
    },
    {
      "if": {
        "properties": { "status": { "const": "RESOLVED" } },
        "required": ["status"]
      },
      "then": {
        "required": ["resolved_at"]
      }
    },
    {
      "if": {
        "properties": { "status": { "const": "POSTMORTEM" } },
        "required": ["status"]
      },
      "then": {
        "properties": {
          "action_items": { "minItems": 1 }
        },
        "required": ["root_cause_analysis", "action_items", "resolved_at"]
      }
    }
  ]
}
```

### Contract Semantics

The conditionals above are deliberately cumulative, mirroring the linear state machine in Section 3.4: reaching `ACKNOWLEDGED` or later requires acknowledgment fields; reaching `MITIGATED` or later additionally requires a mitigation summary; `RESOLVED` additionally requires a resolution timestamp; `POSTMORTEM` requires all of the above plus a non-empty `root_cause_analysis` and at least one `action_items` entry. Each stage only ever *adds* requirements — no rule here overrides or contradicts an earlier one, which keeps every reachable status combination satisfiable.

The 5-minute SEV-0 acknowledgment SLA (Core Invariant 1) is intentionally **not** encoded as a schema constraint: JSON Schema cannot compute a duration between two `date-time` values. That timing requirement is verified at the test-harness and acceptance-criteria level instead (Section 6 and Section 9, `AC-INC-01`), not the contract level.

A record with `severity = "SEV-0"` is not otherwise structurally distinguished from other severities in this schema — severity does not change which fields are required, only how fast a human must respond and who gets paged, both of which are process rules (Section 3), not data-shape rules.

---

## 6. AUTOMATED INCIDENT TEST HARNESS

```python
# Representative Automated Chaos Engineering Test
@pytest.mark.rtm(req_id="DEP-SPEC-07-INC-001")
def test_synthetic_sev1_triggers_oncall_within_sla():
    incident = pagerduty_chaos_harness.trigger_synthetic_alert(
        severity="SEV-1",
        source="chaos-engineering",
        environment="STAGING"
    )

    oncall_rotation.wait_for_activation(incident_id=incident.incident_id)

    record = incident_engine.get_record(incident_id=incident.incident_id)

    assert record.acknowledged_by is not None
    assert record.status != "DETECTED"

    ack_latency_minutes = (
        record.acknowledged_at - record.detected_at
    ).total_seconds() / 60
    assert ack_latency_minutes <= 15

    # The staging on-call rotation, not a hardcoded individual, was paged
    active_rotation = oncall_rotation.get_active_rotation("STAGING")
    assert active_rotation.rotation_type == "PRIMARY"


# Representative Automated Postmortem Gate Test
@pytest.mark.rtm(req_id="DEP-SPEC-07-INC-002")
def test_postmortem_closure_blocked_without_root_cause_and_action_items():
    incident = incident_engine.create_incident(
        severity="SEV-2",
        source="DEP-SPEC-06"
    )
    incident_engine.transition_status(incident.incident_id, "ACKNOWLEDGED")
    incident_engine.transition_status(incident.incident_id, "INVESTIGATING")
    incident_engine.transition_status(incident.incident_id, "MITIGATED")
    incident_engine.transition_status(incident.incident_id, "RESOLVED")

    with pytest.raises(SchemaValidationException) as exc_info:
        incident_engine.transition_status(
            incident_id=incident.incident_id,
            new_status="POSTMORTEM"
        )

    assert exc_info.value.error_code == "ERR_DEP_07_03"

    record = incident_engine.get_record(incident.incident_id)
    assert record.status != "POSTMORTEM"

    # Supplying both required fields allows the same transition to succeed
    incident_engine.attach_postmortem_content(
        incident_id=incident.incident_id,
        root_cause_analysis="Redis connection pool exhaustion under a sustained retry storm.",
        action_items=[
            {"description": "Add a pool-size alert threshold", "owner": "platform-team"}
        ]
    )
    incident_engine.transition_status(
        incident_id=incident.incident_id,
        new_status="POSTMORTEM"
    )

    record = incident_engine.get_record(incident.incident_id)
    assert record.status == "POSTMORTEM"
```

---

## 7. SECURITY & THREAT MODEL

| Threat | Attack Vector | Preventive Control | Severity |
|---|---|---|---|
| Break-Glass Abuse | An operator uses emergency production access for non-emergency convenience rather than a genuine incident. | Dual-control KMS authorization plus mandatory 24-hour retroactive compliance filing (`TEST-SPEC-02`); any session without a linked `incident_id` is flagged automatically. | CRITICAL (SEV-0) |
| Postmortem Whitewashing | `root_cause_analysis` is filled with vague, non-actionable text solely to satisfy the schema's presence check. | The schema enforces presence and shape, not quality — it is a floor, not a substitute for the owning team's human review before an incident is considered organizationally closed. | MEDIUM (SEV-2) |
| Severity Self-Downgrading | An on-call engineer downgrades a SEV-0 to a lower tier to reduce paging noise. | Severity is set by the triggering system (`DEP-SPEC-06`, `DEP-SPEC-03`/`04`/`05`, or `TEST-SPEC-08`/`09`'s own error taxonomy) at detection time, never self-declared by the responder; any downgrade requires a second, independent sign-off. | HIGH (SEV-1) |
| Escalation Ladder Bypass | An incident sits unacknowledged past its SLA without auto-escalating to the next tier. | The escalation timer runs independently of the paged engineer's own action — it cannot be paused, silenced, or reset by the person it is escalating past. | HIGH (SEV-1) |

---

## 8. FAILURE ARCHITECTURE & ERROR TAXONOMY

| Error Code | Description | Canonical Class | Severity |
|---|---|---|---|
| ERR_DEP_07_01 | A SEV-0 incident was not acknowledged within the 5-minute SLA. | PROCESS_VIOLATION | CRITICAL (SEV-0) |
| ERR_DEP_07_02 | Break-glass production access was granted without dual-control KMS authorization. | SECURITY_VIOLATION | CRITICAL (SEV-0) |
| ERR_DEP_07_03 | An incident reached POSTMORTEM status without both root_cause_analysis and a populated action_items array. | CONTRACT_MISMATCH | HIGH (SEV-1) |
| ERR_DEP_07_04 | An incident's status transitioned out of linear order (e.g., DETECTED directly to RESOLVED). | PROCESS_VIOLATION | HIGH (SEV-1) |
| ERR_DEP_07_05 | Break-glass access exceeded its authorized window without a retroactive compliance filing within 24 hours. | SECURITY_VIOLATION | CRITICAL (SEV-0) |

---

## 9. PRODUCTION ACCEPTANCE CRITERIA

| AC ID | Category | Requirement | Verification Method | Pass/Fail |
|---|---|---|---|---|
| AC-INC-01 | SEV-0 Ack SLA | 100% of SEV-0 incidents acknowledged within 5 minutes across simulated chaos tests. | Chaos Test Harness | REQUIRED |
| AC-INC-02 | Break-Glass Integrity | 100% of break-glass sessions carry dual-control KMS authorization and a linked incident_id; 0 orphaned sessions. | Break-Glass Audit | REQUIRED |
| AC-INC-03 | Postmortem Completeness | 0 IncidentResolutionRecords reach POSTMORTEM status missing root_cause_analysis or action_items. | Schema Inspector | REQUIRED |
| AC-INC-04 | State Machine Integrity | 100% of incidents traverse DETECTED→ACKNOWLEDGED→INVESTIGATING→MITIGATED→RESOLVED→POSTMORTEM in strict order; 0 skipped states. | State Transition Audit | REQUIRED |
| AC-INC-05 | Escalation Reliability | 100% of unacknowledged incidents past their SLA auto-escalate to the next tier without manual initiation. | Escalation Ladder Test | REQUIRED |
| AC-INC-06 | Retroactive Compliance | 100% of break-glass sessions are filed for retroactive IaC reconciliation within 24 hours. | Compliance Audit | REQUIRED |

---

## 10. INTEGRATION AUTHORITY MATRIX

| Component | Owns | Consumes | Produces |
|---|---|---|---|
| DEP-SPEC-07 | Severity taxonomy, incident lifecycle, on-call routing, break-glass protocol, auto-remediation playbooks, IncidentResolutionRecord | Alert signals (DEP-SPEC-06), rollback-automation-failure signals (DEP-SPEC-03/04), DR communication trigger (DEP-SPEC-05) | IncidentResolutionRecord, escalation events, break-glass audit entries |
| DEP-SPEC-06 | Telemetry, SLO/error-budget engine | — | Alert signals this document pages on; this document never generates its own telemetry |
| DEP-SPEC-03 | Canary rollback | Human override decision (this document, only if automation fails) | Rollback execution |
| DEP-SPEC-04 | Cutover sequencing | Human Go/No-Go input; this document's escalation path for pre-flight failures | Cutover execution |
| DEP-SPEC-05 | DR reconciliation & failover mechanics | Manual disaster-declaration trigger (this document, for cases automation didn't catch) | Recovery execution |
| DEP-SPEC-01 | IaC & environment provisioning | Break-glass retroactive compliance filings (this document) | The IaC state break-glass sessions must reconcile back into within 24 hours |
| TEST-SPEC-02 | KMS dual-control signature governance | — | The dual-control authorization mechanism this document's break-glass protocol reuses |
| INT-SPEC-15 | POS/booking integration adapters | — | Redis lock state this document's auto-remediation playbooks may clear |
| INT-SPEC-18 | Audit log & telemetry storage (Datadog/Splunk/WORM) | IncidentResolutionRecord, break-glass audit entries (this document) | Immutable long-term audit storage |

---

## 11. FINAL NON-NEGOTIABLE PRINCIPLES

SEV-0 INCIDENTS MUST BE ACKNOWLEDGED WITHIN 5 MINUTES, WITH NO EXCEPTION FOR TIME OF DAY.

BREAK-GLASS PRODUCTION ACCESS MUST NEVER BE GRANTED WITHOUT DUAL-CONTROL KMS AUTHORIZATION, AND MUST ALWAYS BE TIME-LIMITED AND FULLY AUDITED.

ANY AUTOMATED RECOVERY MECHANISM (DEP-SPEC-03/04/05) THAT BREACHES ITS OWN SLA MUST ESCALATE TO SEV-0 HUMAN PAGING; AUTOMATION FAILURE MUST NEVER BE SILENT.

AN INCIDENT MUST NEVER REACH POSTMORTEM CLOSURE WITHOUT A DOCUMENTED ROOT CAUSE AND AT LEAST ONE ACTION ITEM.

THE INCIDENT STATE MACHINE MUST NEVER SKIP A STATE; EVERY TRANSITION MUST BE SEQUENTIAL AND AUDITABLE.

---

## 12. VERSION HISTORY & ARCHITECTURAL VERDICT

| Version | Date | Description | Author | Status |
|---|---|---|---|---|
| 1.0.0 | August 2026 | Initial specification of Incident Response, Severity Taxonomy & Playbooks (DEP-SPEC-07). Establishes the SEV-0 through SEV-3 human-paging taxonomy (explicitly reconciled against DEP-SPEC-05's separate disaster_class taxonomy via the automation-failure escalation bridge), the linear incident state machine, the break-glass protocol reusing TEST-SPEC-02's dual-control KMS mechanism, auto-remediation playbooks, and the IncidentResolutionRecord@1.0.0 contract with cumulative conditional validation. | Ramy Bella | SUPERSEDED |
| 1.0.1 | August 2026 | Consistency pass. Fixed three internal citations that pointed to Section 6 (the Test Harness) for content that actually lives elsewhere: `ERR_DEP_07_04` and `ERR_DEP_07_05` are defined in Section 8 (Failure Architecture), not Section 6; `AC-INC-01` is defined in Section 9 (Acceptance Criteria), not Section 6 — likely drift from an earlier section order. No change to the severity taxonomy, state machine, schema, error codes, or acceptance criteria themselves. | Ramy Bella | APPROVED FOR IMPLEMENTATION |

**ARCHITECTURAL VERDICT:**
APPROVED FOR IMPLEMENTATION. DEP-SPEC-07 establishes the human-response layer sitting on top of every automated Phase 7 mechanism: a severity taxonomy that explicitly reconciles with DEP-SPEC-05 rather than silently colliding with it, a linear and auditable incident lifecycle, a break-glass protocol grounded in TEST-SPEC-02's existing dual-control authorization rather than a new one-off mechanism, and the IncidentResolutionRecord@1.0.0 contract proving every postmortem carries a real root cause and action items rather than closing on vague prose. Ready for implementation.
