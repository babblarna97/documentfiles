# DEP-SPEC-02: Tenant Onboarding & Provisioning Pipeline

## 1. DOCUMENT CONTROL

| Attribute         | Value                                                                                                       |
| ----------------- | ----------------------------------------------------------------------------------------------------------- |
| Document Title    | 02 Tenant Onboarding & Provisioning Pipeline.md                                                             |
| Document ID       | DEP-SPEC-02                                                                                                 |
| Version           | 1.0.3                                                                                                       |
| Status            | APPROVED FOR IMPLEMENTATION                                                                                 |
| Author            | Ramy Bella                                                                                                  |
| Classification    | Confidential / Enterprise Proprietary                                                                       |
| Target Audience   | Solutions Engineers, Platform Engineers, AI Config Specialists, QA Leads                                    |
| Parent Document   | DEP-SPEC-01                                                                                                 |
| Related Documents | DEP-SPEC-01, DEP-SPEC-03, DEP-SPEC-08, TEST-SPEC-09, TEST-SPEC-12, 01 AI Identity.md, 02 AI Constitution.md |
| System            | Restaurant AI System                                                                                        |
| Phase             | Phase 7 — Deployment & Production Operations                                                                |
| Lifecycle Folder  | 07 Deployment & Infrastructure                                                                              |
| Last Updated      | August 2026                                                                                                 |

---

## 2. EXECUTIVE PURPOSE

DEP-SPEC-01 establishes that the target environment is reproducible, isolated, and safe to operate. That proof becomes operationally meaningful only when an actual restaurant is converted into a configured, isolated tenant running inside that verified environment.

DEP-SPEC-02 defines the deterministic pipeline that converts a validated onboarding trigger into a configured tenant that is eligible for guest-facing traffic.

The pipeline:

1. establishes a unique authoritative tenant identity;
2. resolves the foundational tenant variables defined by "01 AI Identity.md";
3. ingests and verifies restaurant menu and knowledge content through "TEST-SPEC-12";
4. hands tenant-scoped secret and configuration requirements to "DEP-SPEC-08";
5. independently verifies the tenant isolation boundary through "TEST-SPEC-09"; and
6. evaluates and atomically commits the pre-go-live gate before guest traffic may be admitted.

No guest-facing conversation is permitted until all mandatory go-live conditions have been satisfied and the runtime traffic-control layer independently accepts the tenant as eligible.

### Scope Boundary

This document owns:

- the tenant onboarding sequence;
- tenant identity issuance;
- onboarding state management;
- onboarding idempotency and retry semantics;
- onboarding auditability;
- the pre-go-live eligibility gate; and
- the authoritative "TenantOnboardingRecord" for onboarding state.

This document does not own:

- the meaning of foundational variables, which is owned by "01 AI Identity.md";
- the environment provisioning or secrets platform, which is owned by "DEP-SPEC-01";
- the specific per-tenant secret and configuration values, which are owned by "DEP-SPEC-08";
- menu/knowledge schemas, chunking, or cryptographic provenance mechanics, which are owned by "TEST-SPEC-12";
- tenant isolation enforcement mechanics, which are owned by "TEST-SPEC-09";
- progressive rollout or traffic migration mechanics, which are owned by "DEP-SPEC-03"; or
- contract signature and CRM handling, which belong to the Phase 5 integrations layer.

The pipeline begins when a validated onboarding-trigger event is received from the owning integration layer.

### Core Operational Invariants

The following invariants are non-negotiable:

- "TENANT_ID COLLISION WITH ANY EXISTING TENANT" ⇒ PROHIBITED; onboarding MUST halt.
- "ONBOARDING STATUS = LIVE" without all required go-live conditions satisfied ⇒ PROHIBITED.
- Guest-facing traffic routed before menu provenance is verified by "TEST-SPEC-12" ⇒ SEV-0 safety violation.
- Guest-facing traffic routed before tenant isolation is independently verified by "TEST-SPEC-09" ⇒ SEV-0 isolation violation.
- Every onboarding execution MUST be auditable and reproducible per tenant and onboarding attempt.
- Every onboarding stage MUST be idempotent or expose an authoritative idempotency mechanism.
- Retries MUST NOT create duplicate or conflicting tenant resources.
- A stale or previously successful gate evaluation MUST NOT be sufficient to authorize a new LIVE transition.
- "TenantOnboardingRecord" is authoritative for onboarding state but MUST NOT be treated as the sole runtime authorization mechanism for guest traffic.

---

## 3. TENANT ONBOARDING PIPELINE ARCHITECTURE

```text
[PRECONDITION]
EnvironmentProvisioningRecord from DEP-SPEC-01
for target PROD environment

Required:
    drift_detected = false
    secrets_scan_passed = true

Runtime execution environment MUST match
target_environment_id.

Any mismatch MUST fail onboarding before
tenant identity issuance.

Onboarding MUST NOT begin against an
unverified or mismatched environment.
                         │
                         ▼
┌──────────────────────────────────────────────────────┐
│ STAGE 1 — TENANT IDENTITY ISSUANCE                  │
│                                                      │
│ Authoritative tenant registry generates a new        │
│ tenant_id atomically and performs collision checks.  │
│                                                      │
│ Caller-supplied tenant IDs are NOT accepted as       │
│ authoritative identity.                              │
│                                                      │
│ External tenant references may be retained only as   │
│ non-authoritative correlation metadata.              │
└──────────────────────────────────────────────────────┘
                         │
                         ▼
┌──────────────────────────────────────────────────────┐
│ STAGE 2 — FOUNDATIONAL VARIABLE RESOLUTION          │
│                                                      │
│ Resolve tenant-specific values for variables defined │
│ by 01 AI Identity.md:                                │
│                                                      │
│ {{PLATFORM_NAME}}                                    │
│ {{RESTAURANT_NAME}}                                  │
│ {{AI_NAME}}                                          │
│ {{BRAND_VOICE}}                                      │
│ {{SUPPORTED_LANGUAGES}}                              │
│                                                      │
│ This stage populates values. It MUST NOT redefine    │
│ their meaning.                                       │
│                                                      │
│ {{DISCOUNT_AUTHORITY}} and secret values are         │
│ explicitly outside this stage.                       │
└──────────────────────────────────────────────────────┘
                         │
                         ▼
┌──────────────────────────────────────────────────────┐
│ STAGE 3 — MENU & KNOWLEDGE BASE INGESTION           │
│                                                      │
│ Invoke TEST-SPEC-12:                                │
│                                                      │
│ KBMenuIngestSchema@1.0.0                             │
│ ChunkProvenance@1.0.0                                │
│                                                      │
│ Required outcomes include successful schema         │
│ validation, ingestion, and provenance verification.  │
│                                                      │
│ Ingested content remains inert and MUST NOT create   │
│ guest-facing effect until Stage 6 authorizes LIVE.   │
│                                                      │
│ Stage execution MUST be idempotent or use an         │
│ authoritative idempotency key.                       │
└──────────────────────────────────────────────────────┘
                         │
                         ▼
┌──────────────────────────────────────────────────────┐
│ STAGE 4 — SECRETS & CONFIGURATION HANDOFF           │
│                                                      │
│ Hand tenant_id and the required configuration        │
│ contract to DEP-SPEC-08.                             │
│                                                      │
│ DEP-SPEC-08 owns secret/configuration values.        │
│                                                      │
│ This pipeline MUST NOT:                              │
│   • set secrets                                      │
│   • read secret values                               │
│   • persist secret values                            │
│   • log secret values                                │
│                                                      │
│ secrets_handoff_complete means only that DEP-SPEC-08 │
│ has returned an authoritative completion signal.     │
└──────────────────────────────────────────────────────┘
                         │
                         ▼
┌──────────────────────────────────────────────────────┐
│ STAGE 5 — ISOLATION BOUNDARY VERIFICATION           │
│                                                      │
│ Run TEST-SPEC-09 against this specific tenant_id.   │
│                                                      │
│ Verify that:                                         │
│   • existing tenants cannot reach this tenant;      │
│   • this tenant cannot reach existing tenants;      │
│   • tenant-scoped resources remain within boundary. │
│                                                      │
│ Tenant ID uniqueness MUST NOT be treated as proof    │
│ of isolation.                                        │
└──────────────────────────────────────────────────────┘
                         │
                         ▼
┌──────────────────────────────────────────────────────┐
│ STAGE 6 — PRE-GO-LIVE GATE                          │
│                                                      │
│ Re-evaluate current authoritative state for all      │
│ mandatory dependencies immediately before LIVE.      │
│                                                      │
│ Required:                                            │
│   foundational_variables_resolved = true             │
│   menu_ingestion_status = COMPLETE                   │
│   menu_provenance_verified = true                    │
│   secrets_handoff_complete = true                    │
│   isolation_boundary_verified = true                 │
│   required authorization conditions = valid          │
│                                                      │
│ LIVE MUST be atomically committed.                   │
│                                                      │
│ A stale previous gate result MUST NOT authorize LIVE.│
└──────────────────────────────────────────────────────┘
                         │
                         ▼
                  [TENANT LIVE]
                         │
                         ▼
        Runtime traffic-control layer independently
        verifies tenant eligibility before admitting
        guest-facing traffic.
```

### No Partial Go-Live

A tenant is either:

- fully eligible and atomically transitioned to "LIVE"; or
- not eligible and blocked from guest-facing traffic.

There is no supported intermediate state in which some guest traffic is served while mandatory onboarding conditions remain incomplete.

An interrupted, partially failed, stale, or under-configured onboarding attempt MUST NOT silently become guest-facing.

---

## 4. STATE TRANSITION & IDEMPOTENCY RULES

### 4.1 Authoritative State Machine

The canonical onboarding state sequence is:

```text
DRAFT
  │
  ▼
IDENTITY_ISSUED
  │
  ▼
VARIABLES_RESOLVED
  │
  ▼
MENU_INGESTING
  │
  ▼
SECRETS_PENDING
  │
  ▼
VERIFYING
  │
  ▼
LIVE
```

Terminal or exceptional states:

- FAILED
- SUSPENDED

#### Legal Transitions

The following forward transitions are legal:

- DRAFT → IDENTITY_ISSUED
- IDENTITY_ISSUED → VARIABLES_RESOLVED
- VARIABLES_RESOLVED → MENU_INGESTING
- MENU_INGESTING → SECRETS_PENDING
- SECRETS_PENDING → VERIFYING
- VERIFYING → LIVE

"FAILED" and "SUSPENDED" are exceptional states and MUST NOT be treated as successful onboarding states.

Recovery transitions MUST be explicitly implemented and documented by the applicable recovery policy. Arbitrary backward transitions MUST NOT be permitted.

A tenant MUST NOT transition directly to "LIVE" from any state other than "VERIFYING".

### 4.2 SUSPENDED Semantics

"SUSPENDED" means the tenant is intentionally prevented from receiving guest-facing traffic.

A suspended tenant:

- MUST NOT be runtime-eligible;
- MUST NOT receive guest-facing traffic;
- MUST retain sufficient state for audit and recovery;
- MUST undergo revalidation before returning to "LIVE".

A prior successful onboarding state MUST NOT automatically restore guest eligibility.

### 4.3 FAILED Semantics

"FAILED" means the onboarding attempt did not complete successfully.

A "FAILED" attempt:

- MUST NOT be considered "LIVE";
- MUST NOT authorize guest traffic;
- MUST NOT be silently reused as a completed tenant;
- MUST either resume through an explicitly supported recovery path or execute the applicable cleanup/compensation procedure.

Any orphaned resources created by the failed attempt MUST be:

- deallocated;
- quarantined; or
- explicitly retained under an operator-approved recovery state.

A new onboarding attempt MUST NOT be permitted to create conflicting resources until the applicable prior-attempt cleanup or recovery requirement has been satisfied.

### 4.4 Idempotency

Every stage MUST satisfy at least one of the following:

1. the operation is inherently idempotent; or
2. the operation requires an authoritative idempotency key and completion record.

The minimum onboarding idempotency identity is:

```text
tenant_id + onboarding_attempt_id + stage
```

The originating event is additionally tracked through:

```text
source_event_id
```

Retries caused by network failures, process crashes, duplicate delivery, or timeout MUST NOT create:

- duplicate tenant identities;
- duplicate tenant-scoped resources;
- duplicate knowledge-base versions;
- duplicate secret/configuration mutations;
- conflicting state transitions; or
- multiple successful LIVE activations.

A completed stage replay MUST return or reference the existing authoritative completion result rather than execute a second logical mutation.

### 4.5 Duplicate and Replay Protection

The system MUST reject or safely deduplicate:

- duplicate "source_event_id";
- duplicate "onboarding_attempt_id";
- replayed stage requests;
- stale stage completion callbacks;
- duplicate dependency completion signals.

If the same source event is delivered more than once, the second delivery MUST resolve to the existing onboarding attempt rather than create a new tenant identity.

### 4.6 Zombie Tenant Control

Every non-terminal onboarding attempt MUST have a maximum allowed dwell time for its current state.

If the dwell-time threshold is exceeded:

1. the condition MUST be recorded;
2. an explicit stall event MUST be emitted;
3. the onboarding attempt MUST transition according to the recovery policy;
4. guest traffic eligibility MUST remain false;
5. applicable cleanup or operator review MUST be initiated.

A stalled onboarding attempt MUST NEVER become "LIVE" solely because a timeout or retry eventually completes without revalidation.

### 4.7 Gate Freshness

A previous successful gate evaluation is not sufficient for activation.

Immediately before the "LIVE" transition, the orchestrator MUST re-evaluate the authoritative state of all mandatory dependencies.

This includes, where applicable:

- foundational variable validity;
- menu ingestion state;
- menu provenance state;
- secrets/configuration handoff state;
- tenant isolation state;
- deployment/environment validity;
- authorization conditions;
- any dependency version or validity state required by the current deployment.

If any mandatory dependency changed, expired, was revoked, became invalid, or can no longer be trusted, the tenant MUST remain non-LIVE until the applicable condition is revalidated.

### 4.8 Runtime Authorization Separation

"TenantOnboardingRecord.onboarding_status = LIVE" is necessary for onboarding completion but MUST NOT by itself be treated as the complete runtime authorization decision.

The runtime traffic-control layer MUST independently evaluate tenant eligibility before admitting guest-facing traffic.

This separation protects against:

- stale onboarding records;
- authorization races;
- runtime configuration changes;
- suspension after onboarding;
- dependency revocation;
- deployment rollback;
- isolation changes; and
- other conditions that can invalidate traffic eligibility after onboarding.

---

## 5. TENANT ONBOARDING CONTRACT — TenantOnboardingRecord@1.0.0

Every onboarding attempt MUST produce a structured, schema-validated "TenantOnboardingRecord".

This record is the authoritative state record for the onboarding pipeline.

It is NOT, by itself, the runtime traffic authorization mechanism.

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "title": "TenantOnboardingRecord@1.0.0",
  "type": "object",
  "additionalProperties": false,
  "properties": {
    "tenant_id": { "type": "string", "minLength": 1 },
    "onboarding_attempt_id": { "type": "string", "minLength": 1 },
    "source_event_id": { "type": "string", "minLength": 1 },
    "contract_reference_id": { "type": "string", "minLength": 1 },
    "target_environment_id": { "type": "string", "minLength": 1 },
    "onboarding_status": {
      "type": "string",
      "enum": ["DRAFT", "IDENTITY_ISSUED", "VARIABLES_RESOLVED", "MENU_INGESTING", "SECRETS_PENDING", "VERIFYING", "LIVE", "FAILED", "SUSPENDED"]
    },
    "foundational_variables_resolved": { "type": "boolean" },
    "menu_ingestion_status": {
      "type": "string",
      "enum": ["NOT_STARTED", "IN_PROGRESS", "COMPLETE", "FAILED"]
    },
    "menu_provenance_verified": { "type": "boolean" },
    "secrets_handoff_complete": { "type": "boolean" },
    "isolation_boundary_verified": { "type": "boolean" },
    "pre_golive_gate_passed": { "type": "boolean" },
    "created_at": { "type": "string", "format": "date-time" },
    "last_updated_at": { "type": "string", "format": "date-time" }
  },
  "required": [
    "tenant_id", "onboarding_attempt_id", "source_event_id", "contract_reference_id",
    "target_environment_id", "onboarding_status", "foundational_variables_resolved",
    "menu_ingestion_status", "menu_provenance_verified", "secrets_handoff_complete",
    "isolation_boundary_verified", "pre_golive_gate_passed", "created_at", "last_updated_at"
  ],
  "allOf": [
    {
      "if": {
        "properties": { "onboarding_status": { "const": "LIVE" } },
        "required": ["onboarding_status"]
      },
      "then": {
        "properties": {
          "foundational_variables_resolved": { "const": true },
          "menu_ingestion_status": { "const": "COMPLETE" },
          "menu_provenance_verified": { "const": true },
          "secrets_handoff_complete": { "const": true },
          "isolation_boundary_verified": { "const": true },
          "pre_golive_gate_passed": { "const": true }
        }
      }
    },
    {
      "if": {
        "not": {
          "properties": { "onboarding_status": { "const": "LIVE" } },
          "required": ["onboarding_status"]
        }
      },
      "then": {
        "properties": {
          "pre_golive_gate_passed": { "const": false }
        }
      }
    }
  ]
}
```

### Contract Semantics

"pre_golive_gate_passed" is derived state. It MUST NOT be manually treated as an independent authorization flag.

For every non-LIVE state: `pre_golive_gate_passed = false`

For "LIVE":

- foundational_variables_resolved = true
- menu_ingestion_status = COMPLETE
- menu_provenance_verified = true
- secrets_handoff_complete = true
- isolation_boundary_verified = true
- pre_golive_gate_passed = true

These requirements are enforced directly by the schema's conditional (`allOf`/`if`/`then`) rules above, not by prose alone — a record marked "LIVE" that violates one or more of them fails JSON Schema validation itself and MUST be rejected by the orchestrator before it can be persisted as authoritative state.

---

## 6. AUTOMATED ONBOARDING GATE TEST HARNESS

```python
# Representative Automated Pre-Go-Live Gate Enforcement Test
@pytest.mark.rtm(req_id="DEP-SPEC-02-ONB-001")
def test_golive_blocked_when_menu_provenance_unverified():
    onboarding = onboarding_engine.start(
        contract_reference_id="CTR-2026-00842",
        source_event_id="EVT-2026-00842",
        target_environment_id="prod-eu-west-1"
    )

    onboarding_engine.issue_tenant_identity(
        onboarding.onboarding_attempt_id
    )

    onboarding_engine.resolve_foundational_variables(
        onboarding.onboarding_attempt_id
    )

    record = onboarding_engine.get_record(
        onboarding.onboarding_attempt_id
    )

    assert record.menu_provenance_verified is False

    with pytest.raises(GoLiveBlockedException) as exc_info:
        onboarding_engine.mark_live(
            onboarding.onboarding_attempt_id
        )

    assert exc_info.value.error_code == "ERR_DEP_02_02"

    record = onboarding_engine.get_record(
        onboarding.onboarding_attempt_id
    )

    assert record.onboarding_status != "LIVE"
    assert record.pre_golive_gate_passed is False


# Representative Automated Tenant Identity Generation
# and Caller-Supplied Identity Rejection Test
@pytest.mark.rtm(req_id="DEP-SPEC-02-ONB-002")
def test_tenant_identity_generation_unique_and_caller_supplied_rejected():
    first_tenant_id = onboarding_engine.issue_tenant_identity(
        onboarding_attempt_id="ONB-2026-0001"
    )

    second_tenant_id = onboarding_engine.issue_tenant_identity(
        onboarding_attempt_id="ONB-2026-0002"
    )

    assert first_tenant_id != second_tenant_id

    with pytest.raises(TenantIdentityInputRejectedException) as exc_info:
        onboarding_engine.issue_tenant_identity(
            onboarding_attempt_id="ONB-2026-0003",
            requested_tenant_id=first_tenant_id
        )

    assert exc_info.value.error_code == "ERR_DEP_02_01"


# Representative Duplicate Event Protection Test
@pytest.mark.rtm(req_id="DEP-SPEC-02-ONB-003")
def test_duplicate_source_event_does_not_create_second_tenant():
    first = onboarding_engine.start(
        contract_reference_id="CTR-2026-00843",
        source_event_id="EVT-2026-00843",
        target_environment_id="prod-eu-west-1"
    )

    second = onboarding_engine.start(
        contract_reference_id="CTR-2026-00843",
        source_event_id="EVT-2026-00843",
        target_environment_id="prod-eu-west-1"
    )

    assert second.onboarding_attempt_id == first.onboarding_attempt_id


# Representative LIVE Semantic Integrity Test
@pytest.mark.rtm(req_id="DEP-SPEC-02-ONB-004")
def test_live_record_requires_all_gate_conditions():
    invalid_record = {
        "tenant_id": "TENANT-001",
        "onboarding_attempt_id": "ONB-001",
        "source_event_id": "EVT-001",
        "contract_reference_id": "CTR-001",
        "target_environment_id": "prod-eu-west-1",
        "onboarding_status": "LIVE",
        "foundational_variables_resolved": True,
        "menu_ingestion_status": "COMPLETE",
        "menu_provenance_verified": False,
        "secrets_handoff_complete": True,
        "isolation_boundary_verified": True,
        "pre_golive_gate_passed": True,
        "created_at": "2026-08-20T12:00:00Z",
        "last_updated_at": "2026-08-20T12:01:00Z"
    }

    with pytest.raises(SchemaValidationException):
        validate_tenant_onboarding_record(invalid_record)
```

## 7. SECURITY & THREAT MODEL

| Threat | Attack Vector | Preventive Control | Severity |
|---|---|---|---|
| Tenant Identity Collision | Overlapping tenant identity | Registry-generated identity + collision detection | CRITICAL (SEV-0) |
| Premature Go-Live | Force LIVE with incomplete conditions | Atomic pre-go-live gate | CRITICAL (SEV-0) |
| Stale Gate Result | Reuse old gate result | Mandatory re-evaluation before LIVE | HIGH (SEV-1) |
| Zombie Tenant | Stalled partial onboarding | Dwell-time + recovery policy | HIGH (SEV-1) |
| Unverified Menu Content | Serve without provenance | Provenance as hard gate | CRITICAL (SEV-0) |
| Secrets Handoff Race | LIVE before handoff complete | Require completion signal | HIGH (SEV-1) |
| Duplicate Event | Replay creates new tenant | source_event_id + idempotency | HIGH (SEV-1) |
| Secret Value Leakage | Secrets in records/logs | Pipeline never handles secret values | CRITICAL (SEV-0) |
| Isolation Failure | Cross-tenant access possible | Independent TEST-SPEC-09 | CRITICAL (SEV-0) |
| Runtime Authorization Drift | Eligibility changes after LIVE | Independent runtime traffic control | CRITICAL (SEV-0) |

---

## 8. FAILURE ARCHITECTURE & ERROR TAXONOMY

| Error Code    | Description                                                                                                                   | Canonical Class       | Severity         |
| ------------- | ----------------------------------------------------------------------------------------------------------------------------- | --------------------- | ---------------- |
| ERR_DEP_02_01 | Caller-supplied or colliding tenant ID                                                                                        | ISOLATION_VIOLATION   | CRITICAL (SEV-0) |
| ERR_DEP_02_02 | Go-live with unmet/stale gate conditions                                                                                      | PROCESS_VIOLATION     | CRITICAL (SEV-0) |
| ERR_DEP_02_03 | Menu ingestion/provenance failure                                                                                             | VALIDATION            | HIGH (SEV-1)     |
| ERR_DEP_02_04 | Pipeline stall beyond timeout                                                                                                 | PROCESS_STALL         | HIGH (SEV-1)     |
| ERR_DEP_02_05 | Secrets handoff not confirmed                                                                                                 | DEPENDENCY_FAILURE    | HIGH (SEV-1)     |
| ERR_DEP_02_06 | TenantOnboardingRecord schema/semantic failure                                                                                | CONTRACT_MISMATCH     | HIGH (SEV-1)     |
| ERR_DEP_02_07 | Invalid state transition                                                                                                      | PROCESS_VIOLATION     | HIGH (SEV-1)     |
| ERR_DEP_02_08 | Duplicate/replayed event                                                                                                      | PROCESS_VIOLATION     | HIGH (SEV-1)     |
| ERR_DEP_02_09 | Stale dependency used for go-live                                                                                             | SECURITY_BOUNDARY     | HIGH (SEV-1)     |
| ERR_DEP_02_10 | Environment mismatch                                                                                                          | PROCESS_VIOLATION     | HIGH (SEV-1)     |
| ERR_DEP_02_11 | Recovery or cleanup of a failed or zombie onboarding attempt did not complete before a conflicting new attempt was permitted. | PROCESS_VIOLATION     | HIGH (SEV-1)     |
| ERR_DEP_02_12 | Isolation verification failed                                                                                                 | ISOLATION_VIOLATION   | CRITICAL (SEV-0) |
| ERR_DEP_02_13 | Runtime traffic-control rejection                                                                                             | RUNTIME_AUTHORIZATION | CRITICAL (SEV-0) |

---

## 9. PRODUCTION ACCEPTANCE CRITERIA

| AC ID | Category | Requirement | Verification Method | Pass/Fail |
|---|---|---|---|---|
| AC-ONB-01 | Zero Identity Collisions | Zero collisions in tenant registry | Registry Collision Test | REQUIRED |
| AC-ONB-02 | Gate Enforcement | 100% of incomplete go-live attempts rejected | Gate Enforcement Test Suite | REQUIRED |
| AC-ONB-03 | No Partial Go-Live | Zero tenants serve traffic with incomplete conditions | Partial-State Traffic Audit | REQUIRED |
| AC-ONB-04 | Zombie Detection | 100% of stalled attempts detected and handled | Dwell-Time Simulation Suite | REQUIRED |
| AC-ONB-05 | Schema Compliance | 100% records validate against schema + LIVE semantics | Schema Inspector | REQUIRED |
| AC-ONB-06 | Auditability | Complete ordered stage history with zero secrets | Audit Trail Reconstruction Test | REQUIRED |
| AC-ONB-07 | Idempotent Recovery | Replay creates no duplicates | Replay/Retry Test Suite | REQUIRED |
| AC-ONB-08 | State Transition Enforcement | 100% invalid transitions rejected | State Machine Transition Audit | REQUIRED |
| AC-ONB-09 | Runtime Traffic Enforcement | Non-eligible tenants blocked from traffic | Runtime Admission Test | REQUIRED |
| AC-ONB-10 | Secret Non-Disclosure | Zero secret values in any onboarding artifact | Secrets Scanner | REQUIRED |
| AC-ONB-11 | Environment Binding | 100% executions match target_environment_id | Environment Binding Audit | REQUIRED |
| AC-ONB-12 | Stale Gate Rejection | 100% stale gate attempts rejected | Gate Freshness Test Suite | REQUIRED |
| AC-ONB-13 | Isolation Verification | 100% tenants pass TEST-SPEC-09 before LIVE | Cross-Spec Isolation Audit | REQUIRED |
| AC-ONB-14 | Duplicate Event Protection | Replays resolve to existing attempt | Duplicate Event Test Suite | REQUIRED |
| AC-ONB-15 | LIVE Contract Integrity | Zero LIVE records with unmet invariants | Schema Inspector | REQUIRED |
| AC-ONB-16 | Runtime Eligibility Separation | Runtime authorization independent of onboarding record | Runtime Admission Test | REQUIRED |

---

## 10. INTEGRATION AUTHORITY MATRIX

| Component | Owns | Consumes | Produces |
|---|---|---|---|
| DEP-SPEC-02 | Onboarding sequence, identity issuance, state machine, idempotency, pre-go-live gate, TenantOnboardingRecord | Provisioned PROD env, onboarding trigger, dependency verdicts | Configured + verified tenant eligible for runtime evaluation |
| 01 AI Identity.md | Meaning of foundational variables | — | Variable definitions |
| DEP-SPEC-01 | Environment + secrets platform | Infrastructure requirements | Provisioned target environment |
| TEST-SPEC-12 | Menu schema, chunking, provenance | Raw menu/knowledge data | Provenanced content + verdicts |
| TEST-SPEC-09 | Tenant isolation verification | New tenant identity | Isolation verdict |
| DEP-SPEC-08 | Per-tenant secret/config values | Tenant identity + secrets platform | Handoff completion signal |
| DEP-SPEC-03 | Progressive rollout / traffic migration | Configured tenants + eligibility signals | Rollout state |
| Phase 5 Integrations | Contract/CRM trigger generation | Commercial onboarding event | Onboarding-trigger event |

---

## 11. OBSERVABILITY & AUDIT REQUIREMENTS

Every onboarding attempt MUST produce sufficient telemetry to reconstruct the complete lifecycle without exposing secret values.

Minimum audit fields:

- onboarding_attempt_id, tenant_id, source_event_id, contract_reference_id, target_environment_id
- state entered / exited, timestamp, initiating principal/service
- stage outcome, error code, dependency reference
- retry/replay information, recovery action

**Secret Handling:** Raw secret values, API keys, tokens, credentials or private keys MUST NEVER appear in onboarding telemetry. Only status flags (e.g. `secrets_handoff_complete = true`) are permitted.

---

## 12. RECOVERY & COMPENSATION REQUIREMENTS

A failed stage MUST NOT automatically trigger unrestricted retry.

Recovery MUST first determine whether the preceding stage produced a durable authoritative completion record.

Distinguish between: NOT_EXECUTED | IN_PROGRESS | COMPLETED | FAILED | UNKNOWN

"UNKNOWN" is not equivalent to "FAILED". Query authoritative state before repeating a potentially non-idempotent operation.

Preferred recovery flow:

```text
VERIFY EXISTING STATE → REUSE AUTHORITATIVE RESULT → RETRY ONLY IF SAFE → COMPENSATE IF NECESSARY
```

Never: FAILURE → BLIND RETRY

---

## 13. PRE-GO-LIVE GATE DEFINITION

The final gate MUST evaluate against **current** authoritative state:

- Environment valid
- Foundational variables resolved
- Menu ingestion complete
- Menu provenance verified
- Secrets/configuration handoff complete
- Tenant isolation verified
- No unresolved critical onboarding failure
- No unresolved recovery condition
- Required authorization conditions valid
- Runtime eligibility prerequisites satisfied

If all true → atomically commit VERIFYING → LIVE  
If any false → reject LIVE transition

---

## 14. NON-NEGOTIABLE OPERATIONAL PRINCIPLES

1. No two tenants may ever share or collide on an authoritative tenant identity.
2. Caller-supplied tenant IDs are never accepted as authoritative identity.
3. No tenant may be marked "LIVE" while any mandatory pre-go-live condition is false, stale, unknown, or unverifiable.
4. Guest-facing traffic must never reach a tenant whose menu content lacks verified provenance.
5. Guest-facing traffic must never reach a tenant whose isolation boundary is unverified.
6. This pipeline resolves foundational variables; it never redefines their meaning.
7. Every onboarding attempt must produce a complete and auditable lifecycle record.
8. Every onboarding stage must be idempotent or expose an authoritative idempotency mechanism.
9. Retries must never create duplicate or conflicting tenant resources.
10. Duplicate onboarding events must resolve safely to the authoritative existing onboarding attempt.
11. A stale gate result must never authorize a new LIVE transition.
12. "LIVE" must be an atomic state transition evaluated against current authoritative dependency state.
13. "TenantOnboardingRecord" is authoritative onboarding state, not sole runtime traffic authorization.
14. Runtime traffic-control must independently enforce guest eligibility.
15. Secret values must never appear in onboarding records, logs, fixtures, or audit records.
16. Failed and stalled onboarding attempts must never silently become guest-facing.
17. Unknown execution state must be resolved through authoritative state inspection before a potentially non-idempotent operation is retried.
18. Isolation verification must remain independent from tenant identity uniqueness.
19. Successful onboarding must be reproducible from its authoritative records and dependency references without relying on undocumented operator knowledge.
20. Any condition capable of invalidating tenant eligibility after onboarding must be enforceable by the runtime authorization layer independently of the onboarding record.

---

## 15. VERSION HISTORY & ARCHITECTURAL VERDICT

| Version | Date        | Description                                                                                                                                                                                                                                                        | Author     | Status                      |
| ------- | ----------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ | ---------- | --------------------------- |
| 1.0.0   | August 2026 | Initial specification of the Tenant Onboarding & Provisioning Pipeline.                                                                                                                                                                                            | Ramy Bella | SUPERSEDED                  |
| 1.0.1   | August 2026 | Enterprise architecture revision. Formalized registry-generated identity, atomic gate evaluation, state machine, idempotency, zombie controls, runtime authorization separation, and recovery requirements.                                                        | Ramy Bella | APPROVED FOR IMPLEMENTATION |
| 1.0.2   | August 2026 | Review fix pass. Added missing ERR_DEP_02_11 (closing the gap between _10 and _12 in the error taxonomy). Replaced the abbreviated Section 6 test-name list with full runnable pytest code for all four representative tests. No architectural or semantic changes | Ramy Bella | SUPERSEDED |
| 1.0.3   | August 2026 | Consistency pass. Added `allOf`/`if`/`then` conditional validation to `TenantOnboardingRecord@1.0.0` so the LIVE-state semantic requirements in Section 5 are enforced by the schema itself, matching the pattern already established in DEP-SPEC-04's `CutoverExecutionRecord@1.0.0`, rather than remaining prose-only. Added the missing "Verification Method" column to the Section 9 Acceptance Criteria table, matching every other file in this series. Corrected a stale "v1.0.1" reference in this Architectural Verdict section. No change to the state machine, error codes, or non-negotiable principles. | Ramy Bella | APPROVED FOR IMPLEMENTATION |

**ARCHITECTURAL VERDICT**

APPROVED FOR IMPLEMENTATION

DEP-SPEC-02 v1.0.3 defines a deterministic and auditable tenant onboarding pipeline from validated onboarding trigger through tenant identity issuance, foundational configuration, knowledge ingestion, secrets/configuration handoff, isolation verification, and atomic pre-go-live activation.

The specification explicitly separates tenant identity from isolation, onboarding state from runtime traffic authorization, ingestion completion from provenance verification, previous gate results from current activation eligibility, secret handoff status from secret values, and retry behavior from blind re-execution.

Status: READY FOR IMPLEMENTATION.

