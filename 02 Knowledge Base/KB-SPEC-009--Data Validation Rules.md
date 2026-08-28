KB-SPEC-009: Data Validation Rules
1. DOCUMENT CONTROL
| Attribute | Value |
|---|---|
| Document Title | Data Validation Rules |
| Document ID | KB-SPEC-009 |
| Version | 1.1.2 |
| Status | 🔒 LOCKED / Approved for Implementation |
| Author | Ramy Bella |
| Classification | Confidential / Enterprise Proprietary |
| Target Audience | Backend Engineers, Database Engineers, Data Engineers, Booking Engine Engineers, AI/RAG Engineers, QA Engineers, Security Engineers, Platform Architects |
| Parent Document | KB-SPEC-002 — Master JSON Schema |
| Related Documents | KB-SPEC-001, KB-SPEC-002, KB-SPEC-003, KB-SPEC-004, KB-SPEC-005, KB-SPEC-006, KB-SPEC-007, KB-SPEC-008 |
| System | Restaurant AI System |
| Phase | Phase 2 — Knowledge Base Specification |
| Last Updated | August 2026 |
2. PURPOSE & SCOPE
Purpose
The Data Validation Rules specify the authoritative enforcement contract for all Knowledge Base records in the Restaurant AI System. This document serves as the implementation-grade standard for the validation stack that all data must pass before entering canonical operational storage, becoming VERIFIED, or influencing runtime decisions (e.g., Booking Engine execution, RAG retrieval).
Scope
 * Defines the 16-stage validation pipeline for all Knowledge Base domain records, explicitly separating pre-validation candidate capture from active canonical operational storage.
 * Sets deterministic failure behavior for malformed, contradictory, or unauthorized data.
 * Enforces the architectural boundaries between domains while strictly honoring domain-specific canonicalization pipelines.
 * Formalizes the multi-tier state-transition matrix, validation result taxonomy, validation-layer deployment boundaries (JSON/Pydantic vs. PostgreSQL vs. Application Service vs. Runtime), and explicit isolation quarantine semantics.
 * Establishes comprehensive testable acceptance criteria covering all enterprise safety and data integrity guarantees.
3. RELATIONSHIP TO PREVIOUS SPECIFICATIONS
This document acts as the structural enforcer for all prior specifications. It MUST NOT redesign the domains themselves, but rather provide the rules to ensure they remain consistent:
 * KB-SPEC-002: Enforces the Master JSON structure.
 * KB-SPEC-003/005/007/008: Enforces the specific fields, Enums, and invariants defined in these domain-specific documents.
 * KB-SPEC-004/006: Enforces safety-critical constraints, including temporal half-open interval logic and EU-14 allergen state exhaustiveness.
 * Cross-Specification Precedence: Domain-specific canonicalization and structural definitions established in KB-SPEC-005, KB-SPEC-007, and KB-SPEC-008 take absolute precedence over generic validation utility functions.
4. VALIDATION LAYERS & RESPONSIBILITY BOUNDARIES
To prevent implementation ambiguity, the validation stack is strictly distributed across four execution layers. Engineers MUST NOT blur these boundaries:
Layer 1: JSON Schema & Pydantic (Edge / Ingestion & Application Boundary)
 * Responsibility: Structural validation, primitive type checking, required/optional boundaries, null prohibition, exact ENUM matching, and JSON Schema conformance.
 * Scope: Operates synchronously during payload ingestion or administrative submission.
Layer 2: PostgreSQL Storage & Constraints (Persistence Boundary)
 * Responsibility: Multi-tenant isolation enforcement (Row-Level Security / RLS), foreign-key referential integrity, unicity constraints, and immutable database-level constraints.
 * Scope: Operates at the storage engine level.
Layer 3: Application Service Layer (Business Logic & Deterministic Engine)
 * Responsibility: Cross-record conflict detection, state-transition legality, provenance validation, domain boundary checks (e.g., preventing allergen claims in unstructured FAQs), and cryptographic identity hash verification.
 * Scope: Operates within backend orchestration services prior to granting operational activation status.
Layer 4: Runtime Environment (Evaluation Boundary)
 * Responsibility: Dynamic evaluation of temporal validity (EXPIRED derivation), real-time "open now" evaluations, and Booking Engine eligibility pipeline execution.
 * Scope: Operates transiently during active API calls or RAG retrieval requests.
5. VALIDATION PIPELINE
Data clearing the ingestion gateway must proceed through the 16 deterministic stages.
 * Stage 1 (Tenant Context): Bind request to venue_id and evaluate RLS/authorization claims.
 * Stage 2 (Structure): Validate JSON syntax and closed-property conformance.
 * Stage 3 (Type Validation): Validate primitive data types (Integers, Strings, Booleans).
 * Stage 4 (Null/Presence): Enforce omission-over-null; reject explicit null.
 * Stage 5 (Enum Validation): Validate fields against strict domain-specific ENUMs.
 * Stage 6 (Format Validation): Validate strings against required formats (RFC 3339 timestamps, HH:MM times, UUIDs).
 * Stage 7 (Domain Canonicalization): Apply domain-specific normalization (e.g., currency units, menu price minor units). Note: Normalization MUST NEVER repair invalid data or invent missing operational certainty into certainty.
 * Stage 8 (Deterministic Identity): Execute domain-specific canonicalization hashing (SHA-256) and verify that generated IDs match stored identifiers.
 * Stage 9 (Semantic Invariant): Validate intra-record mathematical and logical bounds (e.g., min_party_size <= max_party_size).
 * Stage 10 (Business Rule Invariant): Validate multi-field conditional rules (e.g., Cancellation penalty type invariants).
 * Stage 11 (Temporal Validation): Enforce half-open intervals [effective_from, effective_until) where applicable.
 * Stage 12 (Cross-Record Conflict Detection): Check for active, overlapping records sharing identical operational dimensions/scopes that trigger domain-specific conflict semantics per upstream schema definitions (e.g., KB-SPEC-007 and KB-SPEC-008).
 * Stage 13 (Provenance & Verification): Validate lineage structure conditional on source_type.
 * Stage 14 (Cross-Domain Boundary): Enforce architectural safety barriers (e.g., isolating allergen assertions).
 * Stage 15 (State Transition Validation): Verify that record state modifications comply with the legal state-transition matrix.
 * Stage 16 (Audit & Event Emission): Generate an immutable audit log entry for the validation outcome.
6. STORAGE VS. QUARANTINE VS. OPERATIONAL ACTIVATION
To prevent contradiction with ingestion pipelines, the system distinguishes four distinct data states:
 * Raw / Candidate Storage: Unverified or scraped payloads enter staging tables/queues. They do not require full operational verification to be stored for audit or review, but they MUST NOT be exposed to active RAG or Booking Engines.
 * Quarantine (QUARANTINED): Payloads that are structurally parseable but fail provenance, safety boundary, or conflict checks are isolated in a staging/isolation status (QUARANTINED) rather than acting as a canonical domain state. They are strictly invisible to operational workloads and RAG indices.
 * Persisted Domain State: The canonical database state of a record (VERIFIED, UNVERIFIED, CONFLICT, INACTIVE).
 * Runtime / Derived State: Transient states computed dynamically by the runtime engine (e.g., EXPIRED based on effective_until <= NOW). EXPIRED is purely runtime-derived and can never be written directly as a persistent domain state.
7. VALIDATION RESULT MODEL
{
  "correlation_id": "uuid-v4",
  "record_id": "string",
  "validation_result": "VALID | INVALID | CONFLICT | REJECTED",
  "isolation_status": "NONE | QUARANTINED",
  "persisted_state": "VERIFIED | UNVERIFIED | CONFLICT | INACTIVE",
  "runtime_state": "ACTIVE | EXPIRED | SYSTEM_CONFLICT",
  "errors": [
    {
      "code": "VAL-TEMP-001",
      "field_path": "booking_rules.records[0].effective_until",
      "severity": "CRITICAL",
      "message": "effective_until must be strictly greater than effective_from."
    }
  ],
  "timestamp": "2026-08-12T10:00:00Z"
}

8. CROSS-SPECIFICATION INTEGRITY & IDENTITY CANONICALIZATION
Identity & Canonicalization Precedence
Domain-specific canonicalization rules defined in upstream specifications take absolute precedence over any generic validation layer utility:
 * item_id (KB-SPEC-005): venue_id + "|" + catalog_id + "|" + category_id + "|" + normalized_name
 * policy_id (KB-SPEC-007): venue_id + "|" + category + "|" + operational_dimension + "|" + operational_scope
 * faq_id (KB-SPEC-007): venue_id + "|" + normalized_question_text (where normalization requires Unicode NFKC \rightarrow Unicode casefold \rightarrow trim \rightarrow punctuation removal \rightarrow whitespace collapse \rightarrow UTF-8 encoding).
 * booking_rule_id (KB-SPEC-008): venue_id + "|" + rule_type + "|" + operational_scope
Any generic NFKC/casefold utility in KB-009 MUST NOT alter or override these exact domain hashing specifications.
9. PROVENANCE VALIDATION (CONDITIONAL ON SOURCE TYPE)
Stage 13 enforces strict source-dependent lineage validation:
 * OFFICIAL_WEBSITE / OFFICIAL_POLICY_PAGE / OFFICIAL_BOOKING_PLATFORM / PDF_DOCUMENT: source_url is Mandatory. source_url must be a valid HTTPS URL.
 * OFFICIAL_BOOKING_PLATFORM: A booking platform is authoritative only when explicitly registered and recognized in the venue's tenant registry as an official venue-controlled or venue-authorized source. Third-party aggregator listings are automatically rejected as authoritative.
 * HUMAN_VERIFICATION: verified_by (Actor UUID) is Mandatory, and source_url MUST be omitted.
 * verification_timestamp: Mandatory across all authoritative source types, formatted strictly as UTC RFC 3339.
10. STATE MACHINE & LEGAL STATE TRANSITIONS
State transitions are strictly bounded. Administrative overrides or ingestion updates cannot execute illegal jumps.
| From State | To State | Permitted Condition / Authority |
|---|---|---|
| UNVERIFIED | VERIFIED | Permitted only after clearing all validation stages and receiving authorized verification evidence. |
| UNVERIFIED | CONFLICT | Permitted when normalization detects overlapping dimensions/scopes with contradictory data. |
| VERIFIED | CONFLICT | System-derived only. Triggered automatically by the validation pipeline upon detecting a conflicting active record. Manual assignment forbidden. |
| VERIFIED | EXPIRED | System-derived only (Runtime). Computed dynamically at runtime when T_{query} \ge effective\_until. Cannot be written as a persistent domain state. |
| INACTIVE | VERIFIED | Permitted subject to explicit administrative reactivation workflows and full re-validation. |
| CONFLICT | VERIFIED | Forbidden until the underlying conflict is fully resolved via versioning, expiration, or administrative supersession. |
11. CROSS-DOMAIN BOUNDARY ENFORCEMENT
Stage 14 enforces the strict architectural separation of domains:
 * Allergen Quarantine (KB-SPEC-006 vs KB-SPEC-007): Unstructured FAQ records containing explicit physiological EU-14 allergen safety claims (e.g., "This dish is completely peanut-free") MUST NOT be processed as conversational policy truth. They must be flagged, quarantined into the isolation status (QUARANTINED), and redirected to the KB-SPEC-006 verification pipeline. (General keyword queries, e.g., "Do you have peanut-free options?", are permitted as retrieval terms, but definitive safety claims in FAQ text require allergen matrix grounding).
 * Menu Isolation (KB-SPEC-005): Policies and FAQs cannot create, modify, or price menu items.
 * Opening Hours (KB-SPEC-004): Booking rules and policies cannot override venue operating hours.
12. ERROR TAXONOMY
| Code | Type | Description |
|---|---|---|
| VAL-STRUCT-001 | Structural | Invalid JSON, closed-property violation, or missing required field. |
| VAL-TYPE-001 | Type | Field type mismatch (e.g., Float provided for integer minor units). |
| VAL-ENUM-001 | Enum | Unknown or unmapped ENUM value provided. |
| VAL-NULL-001 | Nullability | Explicit null detected where omission is required. |
| VAL-TEMP-001 | Temporal | Invalid date range, half-open interval violation, or malformed RFC 3339 timestamp. |
| VAL-ID-001 | Identity | Deterministic identity hash mismatch (tampering or drift). |
| VAL-PROV-001 | Provenance | Missing mandatory lineage fields (e.g., missing source_url or verified_by). |
| VAL-CONFLICT-001 | Conflict | Overlapping records on the exact same operational dimension/scope with contradictory values. |
| VAL-TENANT-001 | Security | Cross-tenant record leakage or missing venue_id context. |
| VAL-BOUNDARY-001 | Boundary | Cross-domain transgression (e.g., allergen safety assertions inside unstructured FAQ). |
| VAL-MONETARY-001 | Monetary | Invalid minor-unit representation, floating-point usage, or currency collision. |
| VAL-STATE-001 | State Machine | Illegal state transition attempted (e.g., manual shift from CONFLICT to VERIFIED). |
13. PRODUCTION ACCEPTANCE CRITERIA
| AC ID | Category | Requirement | Verification Method | Expected Result | Pass/Fail | Severity | Blocking? |
|---|---|---|---|---|---|---|---|
| AC-01 | Structural | Payloads violating closed-property rules or containing explicit null are rejected instantly. | CI Schema Test | 100% rejection rate. | Required | Critical | Yes |
| AC-02 | Security | RLS ensures query isolation; venue_id is mandatory across all execution vectors. | RLS Pen-Test | Zero cross-tenant data access. | Required | Critical | Yes |
| AC-03 | Temporal | Interval validation enforces half-open logic [from, until); expired records are excluded. | Temporal Test Suite | Invalid intervals rejected. | Required | Critical | Yes |
| AC-04 | Conflict | Overlapping active records sharing identical dimensions/scopes automatically trigger CONFLICT. | Business Logic Suite | Status = CONFLICT. | Required | Critical | Yes |
| AC-05 | Provenance | Records submitted as VERIFIED without complete, source-dependent lineage fail validation. | Unit Test Suite | Payload rejected. | Required | High | Yes |
| AC-06 | ID Integrity | Re-hashing identity inputs via domain-specific canonicalization matches stored IDs 100%. | Identity Test Suite | Exact hash match. | Required | Critical | Yes |
| AC-07 | Boundary | FAQs containing explicit EU-14 allergen safety assertions are quarantined prior to RAG indexing. | Safety Pipeline Test | Assertions quarantined. | Required | Critical | Yes |
| AC-08 | State Machine | Illegal state transitions (e.g., manual CONFLICT \rightarrow VERIFIED) are blocked by application validators. | State Transition Test | Transition blocked. | Required | Critical | Yes |
| AC-09 | Monetary | Floating-point monetary inputs or invalid penalty invariant combinations are rejected. | Boundary Test Suite | Floats/invalid enums blocked. | Required | High | Yes |
| AC-10 | Audit | Every validation rejection, quarantine, or conflict generates an immutable audit record. | Audit Log Verification | 100% event capture. | Required | High | Yes |
14. VERSION HISTORY
| Version | Date | Description | Author | Approval Status |
|---|---|---|---|---|
| 1.1.2 | August 2026 | QA polish patch: Harmonized final-state consistency regarding CONFLICT as a persisted domain state versus EXPIRED as strictly runtime-derived. | Ramy Bella | 🔒 LOCKED / Approved for Implementation |
| 1.1.1 | August 2026 | Pre-sale QA patch: Populated all AC Blocking values, harmonized persisted state model to include CONFLICT, formalized QUARANTINED as a staging/isolation status, locked EXPIRED as strictly runtime-derived, and tightened Stage 12 conflict semantics. | Ramy Bella | Superseded |
| 1.0.0 | August 2026 | Initial validation specification. Established the 16-stage pipeline, layer boundaries, storage vs quarantine semantics, state machine transition matrix, and provenance gating. | Ramy Bella | Superseded |
15. FINAL NON-NEGOTIABLE PRINCIPLES
 * Validation is Mandatory: No record reaches canonical operational storage or active context without clearing the 16-stage pipeline.
 * Fail-Closed: Errors are never repaired with guesses; they result in rejection, quarantine, or demotion to CONFLICT.
 * Domain Precedence: Domain-specific canonicalization rules take absolute precedence over generic validation utilities.
 * Tenant Isolation: Every record must be bound to a verified venue_id.
 * No Nulls: Omission is the exclusive mechanism for representing missing or open-ended data.
 * Immutable Provenance: Verification status requires verifiable evidence tied to authorized source types.
 * Cross-Domain Integrity: Boundary violations (e.g., allergen assertions in FAQs) are automatically quarantined.
 * Deterministic Hashing: All IDs are recalculated and checked against persistent storage using strict domain-specific input strings.
 * System-Derived States: EXPIRED is strictly runtime-derived and cannot be manually assigned or persisted. CONFLICT is system-derived when triggered by validation, but may be persisted as a canonical domain state and cannot be manually assigned by administrators.
 * Audit Trail: Every validation event generates an immutable, append-only audit record.