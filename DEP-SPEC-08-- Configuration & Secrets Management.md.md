# DEP-SPEC-08: Configuration & Secrets Management

## 1. DOCUMENT CONTROL

| Attribute         | Value                                                                                                                                           |
| ----------------- | ----------------------------------------------------------------------------------------------------------------------------------------------- |
| Document Title    | 08 Configuration & Secrets Management.md                                                                                                        |
| Document ID       | DEP-SPEC-08                                                                                                                                     |
| Version           | 1.1.2                                                                                                                                           |
| Status            | APPROVED FOR IMPLEMENTATION                                                                                                                     |
| Author            | Ramy Bella                                                                                                                                      |
| Classification    | Confidential / Enterprise Proprietary                                                                                                           |
| Target Audience   | Security Architects, Platform Engineers, SREs, Integration Engineers                                                                            |
| Parent Document   | DEP-SPEC-01                                                                                                                                     |
| Related Documents | DEP-SPEC-01, DEP-SPEC-02, DEP-SPEC-03, DEP-SPEC-06, DEP-SPEC-07, INT-SPEC-10, INT-SPEC-11, INT-SPEC-13, INT-SPEC-18, TEST-SPEC-09, TEST-SPEC-10 |
| System            | Restaurant AI System                                                                                                                            |
| Phase             | Phase 7 — Deployment & Production Operations                                                                                                    |
| Lifecycle Folder  | 07 Deployment & Infrastructure                                                                                                                  |
| Last Updated      | August 2026                                                                                                                                     |

---

## 2. EXECUTIVE PURPOSE & SCOPE BOUNDARIES

DEP-SPEC-01 provisions the underlying secure secret-management infrastructure, KMS resources, environment boundaries, and platform-level controls. DEP-SPEC-08 governs the lifecycle and runtime handling of the actual secret and configuration values stored within that infrastructure.

This document owns the configuration and secret control plane, not the infrastructure that hosts it.

**Owns:**
- Tenant-to-secret binding and authorization rules
- Runtime secret retrieval and ephemeral injection
- Secret versioning and lifecycle state
- Automated rotation, validation, activation, rollback, and retirement
- Critical tenant configuration mutation controls
- TenantConfigAuditRecord@1.0.0
- Secret/configuration access telemetry and failure signaling

**Delegates:**
- KMS/Vault infrastructure and environment provisioning → DEP-SPEC-01
- Tenant onboarding and provisioning workflow → DEP-SPEC-02
- Canary deployment and rollback decisions → DEP-SPEC-03
- Cutover sequencing and production promotion → DEP-SPEC-04
- LLM/API telemetry and telemetry storage → DEP-SPEC-06
- Human incident response, break-glass access, and escalation → DEP-SPEC-07
- mTLS/network transport controls → INT-SPEC-11
- Authoritative tenant isolation architecture → INT-SPEC-13
- Tenant-isolation verification → TEST-SPEC-09
- PII/privacy controls → TEST-SPEC-10
- Long-term audit storage → INT-SPEC-18

### Core Operational Invariants

- SECRET RETRIEVAL WITHOUT VALID ISOLATION CONTEXT ⇒ ACCESS DENIED (SEV-0)
- CROSS-TENANT SECRET MAPPING ⇒ SEV-0 SECURITY VIOLATION
- SECRET VALUE IN ENVIRONMENT VARIABLE ⇒ PROHIBITED DEPLOYMENT STATE
- SECRET VALUE IN TELEMETRY / LOG / AUDIT RECORD ⇒ SEV-0 SCRUBBING FAILURE
- SECRET ROTATION PAST CONFIGURED MAXIMUM AGE ⇒ ROTATION REQUIRED
- SECRET VERSION ACTIVATION WITHOUT VALIDATED REPLACEMENT ⇒ PROHIBITED
- FAILED SECRET ROTATION ⇒ OLD ACTIVE VERSION REMAINS IN SERVICE UNTIL SAFE CUTOVER
- CONFIGURATION MUTATION WITHOUT VALID AUDIT RECORD ⇒ MUTATION REJECTED

---

## 3. CONFIGURATION & SECRET CLASSIFICATION

Configuration and secrets MUST be distinguished by sensitivity and lifecycle requirements.

| Class | Example | Storage Requirement | Runtime Requirement |
|---|---|---|---|
| SECRET | POS API key, LLM provider credential, webhook signing secret | Secret manager / Vault backed by KMS | JIT retrieval only |
| SENSITIVE CONFIG | Tenant-specific discount authority, provider routing policy | Controlled configuration store | Tenant-scoped access |
| STANDARD CONFIG | Non-sensitive feature flags, display preferences | Versioned configuration store | Normal runtime access |

### 3.1 Classification Rules

- Secret values MUST NOT be stored in source code, Git history, container images, plaintext configuration files, environment variables, telemetry, or ordinary application logs.
- Secret references, version identifiers, and metadata MAY be stored outside the secret manager provided the referenced value cannot be reconstructed from them.
- Configuration sensitivity MUST be determined by the information represented, not merely by storage format.
- A configuration change MUST NOT reduce an established security classification merely by changing its representation.
- Tenant-scoped sensitive configuration MUST inherit the applicable tenant isolation requirements of INT-SPEC-13.
- Audit records MUST reference secret/configuration identities and versions, never plaintext secret values.

---

## 4. SECRET INJECTION & RUNTIME ACCESS ARCHITECTURE

Secret values MUST be retrieved as late as practical and retained in process memory only for the minimum execution period required by the consuming operation.

### 4.1 JIT Secret Fetch Protocol

1. The runtime operation establishes a validated IsolationContext.
2. The adapter or service requests a secret using a tenant-bound secret reference.
3. The secret manager validates:
   - requesting workload identity
   - tenant scope
   - environment scope
   - requested secret reference
   - requested secret version/state
4. The secret manager returns the active version only if all authorization checks succeed.

For controlled secret rotation, the authorized rotation workflow MAY retrieve a newly created pending version solely for provider-side validation and activation. Pending-version retrieval MUST be restricted to the rotation workflow and MUST NOT be exposed through ordinary runtime secret access.

5. The consuming operation uses the plaintext value without persisting it to configuration files, logs, traces, or application-level caches.
6. The plaintext value is released immediately after use. Where the runtime supports deterministic memory zeroization, the implementation SHOULD perform zeroization before release; otherwise secret lifetime and copy creation MUST be minimized.

```text
[AUTHORIZED WORKLOAD]
        │
        ▼
[VALID ISOLATION CONTEXT]
        │
        ▼
[SECRET ACCESS POLICY CHECK]
  ├─ Workload Identity
  ├─ Tenant Scope
  ├─ Environment Scope
  └─ Secret Reference / Version
        │
        ├── FAIL → ACCESS DENIED + AUDIT SIGNAL
        │
        ▼
[ACTIVE SECRET VERSION]
        │
        ▼
[EPHEMERAL RUNTIME USE]
        │
        ▼
[RELEASE / ZEROIZATION WHERE SUPPORTED]
```

### 4.2 Environment Variable Prohibition

Secret values MUST NOT be injected through application environment variables.

This prohibition applies to:

- POS credentials
- LLM provider API keys
- Database passwords
- Signing secrets
- Tenant-specific authentication tokens

Runtime configuration MAY contain non-secret identifiers such as secret references, version IDs, endpoint names, or configuration IDs.

---

## 5. SECRET VERSIONING & ROTATION ARCHITECTURE

Secrets are managed as explicitly versioned resources rather than mutable anonymous values.

### 5.1 Secret Lifecycle

```text
[CREATED]
    │
    ▼
[VALIDATED]
    │
    ▼
[ACTIVE]
    │
    ├──────────────► [ROTATION_REQUIRED]
    │                       │
    │                       ▼
    │               [NEW VERSION CREATED]
    │                       │
    │                       ▼
    │                [NEW VERSION VALIDATED]
    │                       │
    │                 ┌─────┴─────┐
    │                 ▼           ▼
    │            [ACTIVATE]   [ROTATION_FAIL]
    │                 │           │
    │                 │           └──► Keep old ACTIVE version
    │                 ▼
    │           [RETIRE OLD VERSION]
    │
    └──────────────────────────────► [REVOKED / DELETED]
```

### 5.2 Rotation Rules

- Each secret MUST have a configured maximum age appropriate to its secret class and operational risk.
- The default maximum age for production integration credentials is 90 days, unless a stricter policy applies.
- Rotation MUST create and validate the replacement version before switching the active pointer.
- The replacement credential MUST be verified against the consuming provider before activation.
- Activation MUST be atomic from the application's perspective: a request sees either the previously active validated version or the newly active validated version, never a partially written credential.
- The previous version MAY remain temporarily available only when required for controlled overlap or rollback.
- Retirement of the previous version MUST occur only after successful activation and the configured overlap/rollback window.
- A failed rotation MUST NOT deactivate a known-good active credential solely because the replacement failed.
- Successful rotation MUST emit a SECRET_ROTATED audit record.
- Emergency revocation MAY occur outside the normal rotation schedule and MUST be audited.

### 5.3 Rotation Trigger Classes

Rotation MAY be triggered by:

- Scheduled maximum-age threshold
- Provider compromise or suspected credential exposure
- Security incident response
- Manual authorized administrative action
- Provider-mandated credential renewal

Incident-triggered revocation and break-glass handling remain owned by DEP-SPEC-07.

---

## 6. CONFIGURATION & SECRET MUTATION CONTRACT

Every creation, update, rotation, revocation, or deletion of a secret or critical tenant configuration MUST generate a schema-valid TenantConfigAuditRecord@1.0.0.

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "title": "TenantConfigAuditRecord@1.0.0",
  "type": "object",
  "additionalProperties": false,
  "properties": {
    "audit_id": { "type": "string", "minLength": 1 },
    "tenant_id": { "type": "string", "minLength": 1 },
    "mutation_id": { "type": "string", "minLength": 1 },
    "mutation_type": {
      "type": "string",
      "enum": [
        "SECRET_CREATED",
        "SECRET_ROTATED",
        "CONFIG_UPDATED",
        "SECRET_REVOKED",
        "SECRET_DELETED"
      ]
    },
    "secret_reference": { "type": "string", "minLength": 1 },
    "previous_version_id": { "type": ["string", "null"] },
    "new_version_id": { "type": ["string", "null"] },
    "actor_identity": { "type": "string", "minLength": 1 },
    "source": { "type": "string", "minLength": 1 },
    "executed_at": { "type": "string", "format": "date-time" }
  },
  "required": [
    "audit_id",
    "tenant_id",
    "mutation_id",
    "mutation_type",
    "secret_reference",
    "actor_identity",
    "source",
    "executed_at"
  ],
  "allOf": [
    {
      "if": {
        "properties": {
          "mutation_type": {
            "enum": ["SECRET_ROTATED", "CONFIG_UPDATED", "SECRET_REVOKED", "SECRET_DELETED"]
          }
        },
        "required": ["mutation_type"]
      },
      "then": {
        "required": ["previous_version_id"]
      }
    },
    {
      "if": {
        "properties": {
          "mutation_type": {
            "enum": ["SECRET_CREATED", "SECRET_ROTATED"]
          }
        },
        "required": ["mutation_type"]
      },
      "then": {
        "required": ["new_version_id"]
      }
    }
  ]
}
```

### Contract Rules

- `secret_reference` identifies the secret without revealing its value.
- Secret plaintext MUST never appear in this record.
- `mutation_id` MUST uniquely identify the administrative or automated mutation operation.
- Secret rotation/revocation/deletion and versioned protected-configuration mutations MUST reference the exact version being replaced, activated, or revoked.
- Audit generation failure MUST reject the originating mutation rather than silently proceed.
- Audit records are immutable after creation and are stored according to INT-SPEC-18.
- Version identifiers are required for versioned secret mutations and for protected configuration mutations that use versioned configuration state.

---

## 7. AUTOMATED CONFIGURATION & SECRETS TEST HARNESS

```python
# Representative Automated Tenant Isolation Secrets Test
@pytest.mark.rtm(req_id="DEP-SPEC-08-SEC-001")
def test_cross_tenant_secret_fetch_rejected_by_vault():
    vault = SecretsPlatformManager()

    tenant_a_ctx = IsolationContext(tenant_id="tenant_A")
    tenant_b_ctx = IsolationContext(tenant_id="tenant_B")

    with pytest.raises(VaultAccessDeniedException) as exc_info:
        vault.get_secret(
            path="tenant_B/pos_api_key",
            requesting_context=tenant_a_ctx
        )

    assert exc_info.value.error_code == "ERR_DEP_08_02"
    assert exc_info.value.secret_payload is None


# Representative Rotation Failure Safety Test
@pytest.mark.rtm(req_id="DEP-SPEC-08-ROT-001")
def test_failed_rotation_preserves_active_secret():
    secret = secrets_manager.get_secret(
        reference="tenant_A/pos_api_key"
    )

    old_version = secret.active_version

    rotation_result = rotation_engine.rotate(
        reference="tenant_A/pos_api_key",
        simulate_provider_validation_failure=True
    )

    assert rotation_result.success is False
    assert secrets_manager.get_active_version(
        reference="tenant_A/pos_api_key"
    ) == old_version


# Representative Audit Contract Test
@pytest.mark.rtm(req_id="DEP-SPEC-08-AUD-001")
def test_secret_rotation_emits_valid_audit_record():
    result = rotation_engine.rotate(
        reference="tenant_A/pos_api_key"
    )

    record = audit_store.get_record(result.mutation_id)

    assert validate_schema(
        record.json(),
        "TenantConfigAuditRecord@1.0.0"
    )

    assert record.mutation_type == "SECRET_ROTATED"
    assert record.previous_version_id is not None
    assert record.new_version_id is not None
    assert record.secret_value is None
```

---

## 8. SECURITY & THREAT MODEL

| Threat | Attack Vector | Preventive Control | Severity |
|---|---|---|---|
| Cross-Tenant Secret Theft | Tenant A requests a secret reference owned by Tenant B. | Secret manager enforces workload identity, tenant scope, environment scope, and reference ownership before release. | CRITICAL (SEV-0) |
| Environment Variable Exposure | Attacker obtains `/proc/<pid>/environ` or equivalent runtime environment data. | Production secret values are prohibited from environment variables and injected only through authorized runtime retrieval. | CRITICAL (SEV-0) |
| Secret Leakage in Telemetry | Secret value appears in logs, traces, exceptions, or debugging output. | TEST-SPEC-10 / INT-SPEC-18 scrubbing boundaries plus verification-layer detection; any confirmed secret leak is escalated as a SEV-0 security event. | CRITICAL (SEV-0) |
| Rotation Race Condition | Concurrent administrative operations rotate or delete the wrong secret version. | Version-aware mutation records require exact previous_version_id; mutation conflicts are rejected. | HIGH (SEV-1) |
| Failed Rotation | Replacement credential is invalid or rejected by the external provider. | Old validated version remains ACTIVE until replacement verification succeeds. | HIGH (SEV-1) |
| Stale Credential | Secret remains active beyond configured maximum age. | Scheduled rotation compliance sweep and automatic ROTATION_REQUIRED state. | HIGH (SEV-1) |
| Unauthorized Configuration Mutation | Operator changes discount authority or sensitive tenant configuration without authorization. | RBAC, tenant scoping, mutation audit records, and independent approval for high-impact configuration classes. | CRITICAL (SEV-0) |
| Audit Tampering | Actor modifies or deletes mutation history after the fact. | Immutable audit storage through INT-SPEC-18; mutation records are append-only. | CRITICAL (SEV-0) |

---

## 9. FAILURE ARCHITECTURE & ERROR TAXONOMY

| Error Code | Description | Canonical Class | Severity |
|---|---|---|---|
| ERR_DEP_08_01 | Secret access attempted without a valid IsolationContext or workload authorization context. | SECURITY_BOUNDARY | CRITICAL (SEV-0) |
| ERR_DEP_08_02 | Cross-tenant secret access attempt detected. | SECURITY_VIOLATION | CRITICAL (SEV-0) |
| ERR_DEP_08_03 | Scheduled or incident-triggered secret rotation failed and no validated replacement became ACTIVE. | PROCESS_STALL | HIGH (SEV-1) |
| ERR_DEP_08_04 | TenantConfigAuditRecord failed schema validation or required mutation metadata was missing. | CONTRACT_MISMATCH | HIGH (SEV-1) |
| ERR_DEP_08_05 | Secret value detected in telemetry, logs, traces, or audit payloads. | DATA_LEAKAGE | CRITICAL (SEV-0) |
| ERR_DEP_08_06 | Secret activation attempted without successful provider-side credential validation. | VALIDATION | HIGH (SEV-1) |
| ERR_DEP_08_07 | Configuration mutation attempted without required authorization or approval. | ACCESS_DENIED | CRITICAL (SEV-0) |
| ERR_DEP_08_08 | Secret exceeded its configured maximum age without a completed rotation. | CONFIGURATION_DRIFT | HIGH (SEV-1) |

---

## 10. PRODUCTION ACCEPTANCE CRITERIA

| AC ID | Category | Requirement | Verification Method | Pass/Fail |
|---|---|---|---|---|
| AC-CFG-01 | JIT Secret Fetch | 0 production secret values are injected through application environment variables; 100% use approved runtime secret retrieval. | Infrastructure Audit | REQUIRED |
| AC-CFG-02 | Tenant Isolation | 100% of cross-tenant secret access attempts are denied. | Penetration Harness | REQUIRED |
| AC-CFG-03 | Secret Rotation | 100% of production secrets remain within their configured maximum age, defaulting to ≤ 90 days where applicable. | Configuration Sweep | REQUIRED |
| AC-CFG-04 | Rotation Safety | 100% of failed rotations preserve the last validated ACTIVE secret version. | Rotation Fault Injection | REQUIRED |
| AC-CFG-05 | Provider Validation | 100% of replacement credentials are validated against the consuming provider before activation. | Rotation Integration Test | REQUIRED |
| AC-CFG-06 | Audit Integrity | Every secret/configuration mutation generates a valid TenantConfigAuditRecord@1.0.0; mutation fails closed if the audit record cannot be created. | Schema & Mutation Harness | REQUIRED |
| AC-CFG-07 | Telemetry Hygiene | 0 plaintext secret values appear in logs, traces, telemetry, or audit records across the production test suite. | Telemetry Security Audit | REQUIRED |
| AC-CFG-08 | Version Safety | Concurrent version mutations cannot overwrite or delete a version without exact version binding. | Concurrency Mutation Test | REQUIRED |
| AC-CFG-09 | Authorization | 100% of protected configuration mutations enforce required role/approval policy. | RBAC Penetration Test | REQUIRED |

---

## 11. INTEGRATION AUTHORITY MATRIX

| Component | Owns | Consumes | Produces |
|---|---|---|---|
| DEP-SPEC-08 | Secret lifecycle, tenant-secret binding, runtime injection, rotation, configuration mutation governance, TenantConfigAuditRecord | IsolationContext, secret references, provider credentials, authorized configuration requests | Secret versions, rotation signals, configuration mutation records, secret-access failure signals |
| DEP-SPEC-01 | KMS/Vault infrastructure, environment boundaries, secret-manager platform | DEP-SPEC-08 provisioning requirements | Secure secret-management infrastructure |
| DEP-SPEC-02 | Tenant onboarding and provisioning pipeline | Tenant configuration inputs | Provisioned tenant resources and identifiers |
| DEP-SPEC-03 | Canary rollback and release automation | Rotation/configuration health signals | Rollback/cutover decisions |
| DEP-SPEC-04 | Production cutover sequencing and Go/No-Go validation | Configuration/secret readiness signals | Promotion decisions |
| DEP-SPEC-06 | LLM/API telemetry, tracing, SLO and cost monitoring | Secret/configuration failure signals where operationally relevant | Telemetry and alert signals |
| DEP-SPEC-07 | Human incident response, escalation, break-glass access | SEV-0/SEV-1 secret incidents, rotation failures requiring intervention | Incident records, escalation actions |
| INT-SPEC-11 | mTLS and network transport security | Approved service credentials and certificates | Secure network transport |
| INT-SPEC-13 | Authoritative tenant isolation | IsolationContext | Tenant scope contracts |
| INT-SPEC-18 | Audit and telemetry storage | TenantConfigAuditRecord and secret-security events | Immutable audit storage |
| TEST-SPEC-09 | Multi-tenant isolation verification | Secret access scenarios and IsolationContext | Isolation test verdicts |
| TEST-SPEC-10 | PII/secret scrubbing verification | Telemetry and log outputs | Privacy/security test verdicts |

---

## 12. FINAL NON-NEGOTIABLE PRINCIPLES

SECRET VALUES MUST NEVER BE STORED IN SOURCE CONTROL, CONTAINER IMAGES, APPLICATION ENVIRONMENT VARIABLES, LOGS, TELEMETRY, OR STANDARD AUDIT RECORDS.

SECRET ACCESS MUST ALWAYS BE BOUND TO A VALID WORKLOAD IDENTITY, TENANT ISOLATION CONTEXT, ENVIRONMENT, AND SECRET REFERENCE.

CROSS-TENANT SECRET ACCESS ATTEMPTS MUST FAIL CLOSED AND CONSTITUTE AN IMMEDIATE SEV-0 SECURITY EVENT.

SECRET ROTATION MUST VALIDATE THE REPLACEMENT CREDENTIAL BEFORE ACTIVATION; A FAILED ROTATION MUST NEVER DISPLACE A LAST-KNOWN-GOOD ACTIVE VERSION.

EVERY SECRET OR PROTECTED CONFIGURATION MUTATION MUST PRODUCE A SCHEMA-VALIDATED, IMMUTABLE AUDIT RECORD WITHOUT EXPOSING THE SECRET VALUE.

SECRET MAXIMUM-AGE POLICIES MUST BE ENFORCED AUTOMATICALLY; STALE ACTIVE CREDENTIALS MUST NEVER REMAIN UNDETECTED.

BREAK-GLASS SECRET ACCESS AND EMERGENCY REVOCATION MUST USE THE EXISTING INCIDENT AND DUAL-CONTROL GOVERNANCE PATHS RATHER THAN DEFINING A PARALLEL EMERGENCY MODEL.

---

## 13. VERSION HISTORY & ARCHITECTURAL VERDICT

| Version | Date        | Description                                                                                                                                                                                                                                                                                                          | Author     | Status                      |
| ------- | ----------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ---------- | --------------------------- |
| 1.0.0   | August 2026 | Initial specification of Configuration & Secrets Management (DEP-SPEC-08).                                                                                                                                                                                                                                           | Ramy Bella | SUPERSEDED                  |
| 1.1.0   | August 2026 | Hardened enterprise implementation specification. Clarified ownership/delegation boundaries; added explicit configuration classification; strengthened versioned rotation and failed-rotation behavior; expanded the mutation/audit contract; added secret-leak telemetry controls and expanded acceptance criteria. | Ramy Bella | SUPERSEDED                  |
| 1.1.1   | August 2026 | Formatting and consistency pass. Removed duplicated Document Control block; normalized table and section structure to match the rest of the DEP-SPEC series; tightened language for consistency. No architectural or semantic changes.                                                                               | Ramy Bella | APPROVED FOR IMPLEMENTATION |
| 1.1.2   | August 2026 | Targeted fix pass. Clarified pending-version retrieval for authorized rotation workflow (Section 4.1). Clarified that version identifiers apply to both secret mutations and versioned protected-configuration mutations (Section 6).                                                                                | Ramy Bella | APPROVED FOR IMPLEMENTATION |

**ARCHITECTURAL VERDICT**

APPROVED FOR IMPLEMENTATION

DEP-SPEC-08 establishes the authoritative lifecycle and runtime-control layer for tenant-scoped secrets and protected configuration. It defines JIT retrieval, isolation-bound access, version-safe rotation, failed-rotation preservation, mutation auditing, secret-leak prevention, and explicit integration boundaries with the surrounding Phase 7 architecture.

Status: READY FOR IMPLEMENTATION.