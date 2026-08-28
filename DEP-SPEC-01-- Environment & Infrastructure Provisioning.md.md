# DEP-SPEC-01: Environment & Infrastructure Provisioning

## 1. DOCUMENT CONTROL

| Attribute         | Value                                                                                                                                                                         |
| ----------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Document Title    | 01 Environment & Infrastructure Provisioning.md                                                                                                                               |
| Document ID       | DEP-SPEC-01                                                                                                                                                                   |
| Version           | 1.0.1                                                                                                                                                                         |
| Status            | APPROVED FOR IMPLEMENTATION                                                                                                                                                   |
| Author            | Ramy Bella                                                                                                                                                                    |
| Classification    | Confidential / Enterprise Proprietary                                                                                                                                         |
| Target Audience   | Platform Engineers, SREs, Security Engineers, DevOps Leads                                                                                                                    |
| Parent Document   | 01 AI Identity.md                                                                                                                                                             |
| Related Documents | Phase 6 Specs (TEST-SPEC), TEST-SPEC-01, TEST-SPEC-02, TEST-SPEC-10, TEST-SPEC-18, TEST-SPEC-20, DEP-SPEC-02, DEP-SPEC-03, DEP-SPEC-04, DEP-SPEC-06, DEP-SPEC-07, DEP-SPEC-08 |
| System            | Restaurant AI System                                                                                                                                                          |
| Phase             | Phase 7 — Deployment & Production Operations                                                                                                                                  |
| Lifecycle Folder  | 07 Deployment & Infrastructure                                                                                                                                                |
| Last Updated      | August 2026                                                                                                                                                                   |

---

## 2. EXECUTIVE PURPOSE

Phases 1 through 6 prove that the system, its knowledge, and its safety guardrails work correctly against a controlled test corpus. None of that proof means anything if the environment it runs in cannot be reproduced, audited, and trusted. An untracked manual change to a production server, a secret committed to source control, or an environment that silently drifts from what was tested makes every upstream guarantee — allergen safety, grounding, prompt integrity — unverifiable in practice, because nobody can be certain production actually matches what TEST-SPEC-12 through TEST-SPEC-20 validated.

`DEP-SPEC-01` defines the **Environment & Infrastructure Provisioning Architecture**: the four-tier environment model (dev/staging/shadow/prod), the Infrastructure-as-Code (IaC) pipeline that is the only permitted way to change any of them, the foundational secrets management platform, and continuous drift detection. It is the substrate every other Phase 7 document is built on top of.

**Scope boundary.** This document owns the secrets management *platform* — storage, encryption at rest/in transit, rotation mechanism, and IAM access boundaries. It does **not** own the per-tenant operational process of setting specific secret or configuration *values* (e.g., a given restaurant's `{{DISCOUNT_AUTHORITY}}`, a tenant's POS API key) — that process is owned by `DEP-SPEC-08` (Configuration & Secrets Management) and executes entirely on top of the platform defined here.

### Core Operational Invariants

* `ANY INFRASTRUCTURE CHANGE OUTSIDE THE IAC PIPELINE \implies PROHIBITED ("CLICKOPS" BANNED)`
* `SECRET VALUE IN SOURCE CONTROL, IAC STATE, PLAN OUTPUT, OR LOGS \implies SEV-0 CRITICAL BREACH`
* `LIVE INFRASTRUCTURE STATE DIVERGES FROM DECLARED IAC SOURCE \implies DRIFT, MANDATORY REMEDIATION`
* `CROSS-ENVIRONMENT ACCESS (E.G. STAGING REACHING PROD SECRETS OR DATA) \implies SEV-0 ISOLATION BREACH`
* `ENVIRONMENT PROVISIONING = 100% REPRODUCIBLE FROM VERSION-CONTROLLED IAC SOURCE`

---

## 3. ENVIRONMENT TIERS & PROVISIONING ARCHITECTURE

```text
[IaC SOURCE REPOSITORY] (version-controlled, PR-gated, single source of truth)
           │
           ▼
   [IaC PLAN STAGE] ── computes diff against remote state (locked, not local)
           │
           ▼
   [MANDATORY HUMAN REVIEW + APPROVAL] (required for STAGING/SHADOW/PROD; no auto-apply)
           │
           ▼
   [IaC APPLY STAGE] ── executes only against the target environment reviewed
           │
     ┌─────┴──────┬───────────────┬────────────────┐
     ▼             ▼               ▼                ▼
  [DEV]        [STAGING]       [SHADOW]          [PROD]
  ephemeral,   prod-parity,    prod-parity,      live tenants,
  per-branch,  hermetic E2E    receives ONLY     real guest
  no tenant    test target     mirrored/replayed traffic
  data         (TEST-SPEC-18)  traffic, never    (this is the
                                serves a real     only tier that
                                guest response    is user-facing)
                                (TEST-SPEC-20)
           │
           ▼
  [CONTINUOUS DRIFT DETECTION] ── scheduled diff: live state vs. declared IaC
           │
           ▼
  [DRIFT DETECTED] → SEV-1 alert; remediation plan generated; human approval
                      required before any corrective apply
```

**Environment parity.** STAGING and SHADOW MUST be infrastructure-identical to PROD (same instance types, network topology, IAM model, and secrets platform configuration) except for the traffic and data they carry. Parity gaps invalidate the predictive value of TEST-SPEC-18 (which runs against STAGING) and TEST-SPEC-20 (which runs against SHADOW) — if either tier drifts from PROD's shape, a passing E2E or shadow-replay result no longer proves what it claims to prove.

**Shadow tier containment.** SHADOW exists solely to receive mirrored or replayed production traffic for validation (TEST-SPEC-20) and MUST NEVER have an outbound network path capable of reaching a real guest, a live payment processor, or a live third-party integration. A response generated in SHADOW is discarded after evaluation; it is never delivered.

**DEV tier.** DEV is ephemeral and provisioned per feature branch. It MUST NOT be provisioned with real tenant data or production secrets under any circumstance — synthetic fixtures only.

---

## 4. ENVIRONMENT PROVISIONING CONTRACT (`EnvironmentProvisioningRecord@1.0.0`)

Every provisioning or re-verification event MUST produce a structured, schema-validated record. `secrets_platform_ref` identifies the secrets platform instance backing the environment; it never contains a secret value itself, only a reference/ARN-style pointer.

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "title": "EnvironmentProvisioningRecord@1.0.0",
  "type": "object",
  "additionalProperties": false,
  "properties": {
    "environment_id": { "type": "string", "minLength": 1 },
    "environment_tier": {
      "type": "string",
      "enum": ["DEV", "STAGING", "SHADOW", "PROD"]
    },
    "iac_source_commit_hash": { "type": "string", "minLength": 1 },
    "iac_state_checksum": { "type": "string", "minLength": 1 },
    "secrets_platform_ref": { "type": "string", "minLength": 1 },
    "drift_detected": { "type": "boolean" },
    "secrets_scan_passed": { "type": "boolean" },
    "network_isolation_verified": { "type": "boolean" },
    "tenant_data_present": { "type": "boolean" },
    "provisioned_at": { "type": "string", "format": "date-time" },
    "last_verified_at": { "type": "string", "format": "date-time" }
  },
  "required": [
    "environment_id",
    "environment_tier",
    "iac_source_commit_hash",
    "iac_state_checksum",
    "secrets_platform_ref",
    "drift_detected",
    "secrets_scan_passed",
    "network_isolation_verified",
    "tenant_data_present",
    "provisioned_at",
    "last_verified_at"
  ]
}
```

A record with `drift_detected: true` or `secrets_scan_passed: false` MUST NOT be treated as a valid deployment target by any downstream Phase 7 process (`DEP-SPEC-02` onboarding, `DEP-SPEC-03` rollout) until remediated.

---

## 5. AUTOMATED PROVISIONING VERIFICATION HARNESS

The harness verifies environment isolation, secrets hygiene, and drift as part of every scheduled check and every IaC apply.

```python
# Representative Automated Environment Isolation & Drift Test
@pytest.mark.rtm(req_id="DEP-SPEC-01-ENV-001")
def test_staging_cannot_access_production_secrets_and_no_state_drift():
    # Arrange: resolve the currently provisioned staging and prod environments
    staging_env = provisioning_engine.get_environment(tier="STAGING")
    prod_env = provisioning_engine.get_environment(tier="PROD")

    # Act: attempt to resolve a production-scoped secret using the staging
    # environment's own IAM identity — this MUST be denied.
    with pytest.raises(AccessDeniedError):
        secrets_client.get_secret(
            secret_id=f"{prod_env.secrets_namespace}/llm_provider_api_key",
            requesting_identity=staging_env.iam_role
        )

    # Assert 1: The provisioning record is schema-valid
    record = provisioning_engine.get_provisioning_record(staging_env.environment_id)
    assert validate_schema(record.json(), "EnvironmentProvisioningRecord@1.0.0")

    # Assert 2: No plaintext secret value appears anywhere in the exported
    # IaC state or plan output — only secrets-platform references are
    # permitted to appear.
    iac_state_dump = provisioning_engine.export_state(staging_env.environment_id)
    for known_secret_value in secrets_client.list_known_plaintext_values():
        assert known_secret_value not in iac_state_dump

    # Assert 3: Live environment state matches declared IaC source exactly
    assert record.drift_detected is False

    # Assert 4: Network isolation between staging and prod is enforced
    assert record.network_isolation_verified is True
```

---

## 6. SECURITY & THREAT MODEL

| Threat | Attack Vector | Preventive Control | Severity |
|---|---|---|---|
| Secret Leakage | A secret value is committed to source control, written into IaC state, or logged in plaintext by an application error handler. | Pre-commit and CI-stage secrets scanning; IaC state contains only secrets-platform references, never values. | CRITICAL (SEV-0) |
| Cross-Environment Lateral Movement | A compromised or misconfigured STAGING identity attempts to read PROD secrets or data. | IAM roles are scoped per environment tier with default-deny; no role spans two tiers. | CRITICAL (SEV-0) |
| Silent Configuration Drift | An engineer makes a manual "quick fix" directly in the cloud console, bypassing IaC. | Continuous drift detection compares live state to declared IaC source on a fixed schedule; all changes require IaC pipeline review. | HIGH (SEV-1) |
| Shadow Traffic Leakage | A SHADOW environment is misconfigured with an outbound path to a real guest channel or live payment processor. | SHADOW network egress is allow-listed to evaluation/logging endpoints only; default-deny on all other egress. | CRITICAL (SEV-0) |

---

## 7. FAILURE ARCHITECTURE & ERROR TAXONOMY

| Error Code | Description | Canonical Class | Severity |
|---|---|---|---|
| ERR_DEP_01_01 | Secret value found in plaintext within IaC source, state, plan output, or logs. | SECURITY_VIOLATION | CRITICAL (SEV-0) |
| ERR_DEP_01_02 | Cross-environment access succeeded where isolation should have denied it. | SECURITY_BOUNDARY | CRITICAL (SEV-0) |
| ERR_DEP_01_03 | Live infrastructure state diverged from declared IaC source (undeclared drift). | CONFIGURATION_DRIFT | HIGH (SEV-1) |
| ERR_DEP_01_04 | Infrastructure change applied outside the IaC pipeline. | PROCESS_VIOLATION | HIGH (SEV-1) |
| ERR_DEP_01_05 | EnvironmentProvisioningRecord failed JSON Schema validation. | VALIDATION | HIGH (SEV-1) |
| ERR_DEP_01_06 | Shadow-tier environment had an outbound path capable of reaching a real guest or live third-party integration. | SAFETY_VIOLATION | CRITICAL (SEV-0) |

---

## 8. PRODUCTION ACCEPTANCE CRITERIA

| AC ID | Category | Requirement | Verification Method | Pass/Fail |
|---|---|---|---|---|
| AC-ENV-01 | IaC-Only Changes | 100% of infrastructure changes across all environment tiers originate from the version-controlled IaC pipeline; 0 manual console modifications per audit period. | IaC Audit Log Review | REQUIRED |
| AC-ENV-02 | Zero Secret Leakage | 0 plaintext secret values detected in source, state, plan output, or logs across the full secrets-scan corpus. | Secrets Scanner | REQUIRED |
| AC-ENV-03 | Environment Isolation | 100% of cross-environment access attempts that should be denied (staging→prod, dev→staging, shadow→live guest) are denied. | Isolation Penetration Test | REQUIRED |
| AC-ENV-04 | Zero Undeclared Drift | 100% of live environment states match their declared IaC source at every scheduled drift-detection interval. | Drift Detection Suite | REQUIRED |
| AC-ENV-05 | Schema Compliance | 100% of provisioning records validate against EnvironmentProvisioningRecord@1.0.0. | Schema Inspector | REQUIRED |
| AC-ENV-06 | Shadow Traffic Containment | 100% of shadow-tier environments verified to have zero outbound paths capable of reaching a real guest. | Network Path Audit | REQUIRED |

---

## 9. INTEGRATION AUTHORITY MATRIX

| Component | Owns | Consumes | Produces |
|---|---|---|---|
| DEP-SPEC-01 | Environment Tier Definitions, IaC Pipeline, Secrets Management Platform, Drift Detection | IaC Source Repository, Cloud Provider APIs | Provisioned Environments (DEV/STAGING/SHADOW/PROD), EnvironmentProvisioningRecord |
| TEST-SPEC-02 | CI/CD Stage Gate Enforcement | Provisioning Records (this document) | Deployment Gate Go/No-Go Decisions |
| TEST-SPEC-18 | Hermetic E2E Adapter Testing | Staging Environment (this document) | AdapterE2EReport |
| TEST-SPEC-20 | Shadow Traffic Replay & Validation | Shadow Environment (this document) | Replay Validation Verdicts |
| DEP-SPEC-08 | Per-Tenant Configuration & Secrets Management | Secrets Management Platform (this document) | Tenant-Scoped Secret & Configuration Values |
| DEP-SPEC-02 | Tenant Onboarding & Provisioning Pipeline | Provisioned PROD Environment (this document) | Configured Tenant Instances |
| DEP-SPEC-03 | Progressive Rollout & Traffic Migration | Provisioned PROD Environment (this document) | RolloutState, routed release traffic |
| DEP-SPEC-04 | Cutover Day Execution & Go/No-Go Sequencing | Environment State & Secrets Platform (this document) | CutoverExecutionRecord, PONR crossing events |

---

## 10. FINAL NON-NEGOTIABLE PRINCIPLES

ALL INFRASTRUCTURE MUST BE DEFINED AS VERSION-CONTROLLED CODE; NO ENVIRONMENT TIER MAY BE MODIFIED OUTSIDE THE IAC PIPELINE.

NO SECRET VALUE MAY EVER APPEAR IN PLAINTEXT IN SOURCE CONTROL, IAC STATE, PLAN OUTPUT, OR APPLICATION LOGS.

DEV, STAGING, AND SHADOW ENVIRONMENTS MUST NEVER BE GRANTED ACCESS TO PRODUCTION SECRETS OR PRODUCTION TENANT DATA.

LIVE INFRASTRUCTURE STATE MUST NEVER BE ALLOWED TO SILENTLY DIVERGE FROM ITS DECLARED IAC SOURCE.

THE SHADOW ENVIRONMENT MUST NEVER HAVE AN OUTBOUND PATH CAPABLE OF REACHING A REAL GUEST.

---

## 11. VERSION HISTORY & ARCHITECTURAL VERDICT

| Version | Date | Description | Author | Status |
|---|---|---|---|---|
| 1.0.0 | August 2026 | Initial specification of Environment & Infrastructure Provisioning (DEP-SPEC-01). Establishes the four-tier environment model (dev/staging/shadow/prod — shadow added beyond the original three-tier proposal, since TEST-SPEC-20's shadow traffic replay has no defined infrastructure to run against otherwise), the IaC-only change pipeline, the foundational secrets management platform, continuous drift detection, and an explicit scope boundary against DEP-SPEC-08 (which owns per-tenant secret *values*, not the secrets platform defined here). | Ramy Bella | SUPERSEDED |
| 1.0.1 | August 2026 | Consistency pass. Added DEP-SPEC-03 and DEP-SPEC-04 to the Integration Authority Matrix as explicit consumers — both now exist and explicitly depend on this document, which predates them. No change to environment tiers, the IaC pipeline, the schema, or non-negotiable principles. | Ramy Bella | APPROVED FOR IMPLEMENTATION |

**ARCHITECTURAL VERDICT:**
APPROVED FOR IMPLEMENTATION. DEP-SPEC-01 establishes the four-tier environment model, IaC-only provisioning discipline, the foundational secrets management platform, drift detection, and environment isolation guarantees that every subsequent Phase 7 document depends on. Ready for implementation.
