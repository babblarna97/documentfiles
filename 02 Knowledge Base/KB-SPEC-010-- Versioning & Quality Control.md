KB-SPEC-010: Versioning & Quality Control
1. DOCUMENT CONTROL
| Attribute | Value |
|---|---|
| Document Title | Versioning & Quality Control |
| Document ID | KB-SPEC-010 |
| Version | 1.1.0 |
| Status | 🔒 LOCKED / Approved for Implementation |
| Author | Ramy Bella |
| Classification | Confidential / Enterprise Proprietary |
| Target Audience | Backend Engineers, Data Engineers, Database Engineers, Booking Engine Engineers, AI/RAG Engineers, QA Engineers, Security Engineers, Platform Architects |
| Parent Document | KB-SPEC-002 — Master JSON Schema |
| Related Documents | KB-SPEC-001, KB-SPEC-002, KB-SPEC-003, KB-SPEC-004, KB-SPEC-005, KB-SPEC-006, KB-SPEC-007, KB-SPEC-008, KB-SPEC-009 |
| System | Restaurant AI System |
| Phase | Phase 2 — Knowledge Base Specification |
| Last Updated | August 2026 |
2. PURPOSE & SCOPE
Purpose
The Versioning & Quality Control specification defines the authoritative lifecycle, versioning, and quality-control contract for all Knowledge Base (KB) records. It establishes how records evolve, supersede, correct, and expire over time without destroying historical truth or compromising safety-critical domains.
Scope: What This Document Controls
 * The orthogonal dimensional state model governing KB records (separating domain states, lifecycle states, runtime-derived states, and isolation states).
 * The deterministic versioning and integer revision mechanics.
 * The explicit decoupling of domain identity (item_id / policy_id) from version identity.
 * The multi-tier source fingerprinting, change materiality detection, and explicit revalidation framework.
 * The safety-critical versioning guardrails governing allergen and operational changes.
 * The append-only audit trail and concurrency-safe atomicity requirements.
Scope: What This Document Explicitly Does NOT Control
 * Domain Semantics: Does not redefine menu catalog schemas, allergen states (KB-SPEC-006), or opening-hours mathematics.
 * Validation Pipeline: The 16-stage pipeline remains governed strictly by KB-SPEC-009.
3. RELATIONSHIP TO PREVIOUS SPECIFICATIONS
 * KB-SPEC-001–KB-SPEC-009: KB-SPEC-010 is the orchestrator for the lifecycle of records defined in these specifications.
 * Precedence: Domain-specific identity rules established in upstream specifications (e.g., KB-SPEC-005 where an item name change forces a brand-new item_id) take absolute precedence over generic versioning identity models. KB-SPEC-010 never forces domain identity to remain static when an upstream specification dictates a new entity.
4. VERSIONING PHILOSOPHY / NON-NEGOTIABLE PRINCIPLES
 * HISTORICAL TRUTH IS IMMUTABLE: Previous versions of a record are never deleted or modified. The system maintains an append-only audit trail of the record’s evolution.
 * NO DESTRUCTIVE MUTATION: A correction to an erroneous record creates a new version; it does not "fix" the old record in place.
 * ORTHOGONAL DIMENSIONAL STATES: Record state is not a single flat enum. Domain states, lifecycle metadata, runtime-derived states, and isolation statuses are strictly separated.
 * DOMAIN IDENTITY PRECEDENCE: Upstream domain identity rules (such as safety-critical identity resets) override generic versioning continuity.
 * FAIL-CLOSED ON QUALITY DEGRADATION: If source changes or conflicts invalidate a record's freshness, it is automatically demoted and blocked from active RAG/Booking Engine retrieval.
 * CONCURRENCY SAFETY: Simultaneous administrative updates are strictly governed by optimistic versioning pre-conditions.
5. ORTHOGONAL STATE ARCHITECTURE
To prevent state-machine collapse, the system enforces a strict multi-dimensional state model. A record's operational posture is evaluated through four non-colliding dimensions:
 * Domain State (Owned by Domain Specs e.g., KB-SPEC-006/007/008):
   * Examples: VERIFIED, UNVERIFIED, CONFLICT, INACTIVE.
   * Governs the operational validity of the domain facts.
 * Lifecycle Metadata (Owned by KB-SPEC-010):
   * Values: CANDIDATE, VALIDATED, ACTIVE, SUPERSEDED, RETIRED.
   * Governs where the record sits in its temporal publishing lifecycle.
 * Runtime-Derived State (Owned by Runtime Engine e.g., KB-SPEC-004/007/008):
   * Values: ACTIVE_WINDOW, EXPIRED.
   * Computed dynamically based on UTC time and half-open intervals [effective_from, effective_until). EXPIRED is never persisted as a domain state.
 * Isolation Status (Owned by KB-SPEC-009):
   * Values: NONE, QUARANTINED.
   * Governs security and safety quarantine overrides.
6. KNOWLEDGE RECORD LIFECYCLE
The lifecycle metadata governing publishing evolution consists of:
 * CANDIDATE: Raw ingestion/entry stage.
 * VALIDATED: Cleared all KB-SPEC-009 validation stages.
 * ACTIVE: The single authorized operational version for a given operational dimension/scope and time range.
 * SUPERSEDED: Replaced by a newer ACTIVE version. Retained purely for auditability.
 * RETIRED: Permanently archived end-of-life status.
7. DOMAIN IDENTITY VS. VERSION IDENTITY
 * Domain Identity (item_id, policy_id, booking_rule_id): Governed by upstream specifications (KB-SPEC-005, KB-SPEC-007, KB-SPEC-008). If a domain specification explicitly mandates that a change creates a new entity (e.g., an item name change altering its hash in KB-SPEC-005), KB-SPEC-010 honors that rule as a brand-new logical entity rather than forcing version continuity.
 * Version Identity (Revision Number): Monotonically increasing integers (1, 2, 3...) representing structural or semantic updates to an existing stable domain identity.
8. MULTI-TIER SOURCE FINGERPRINTING & HASHING
To prevent unnecessary version inflation from trivial formatting, whitespace, or timestamp shifts, the system enforces three distinct hashes:
 * Raw Source Hash (SHA-256): Computed over the raw scraped HTML/DOM or document bytes. Used to detect if an external webpage has physically changed.
 * Normalized Payload Hash (SHA-256): Computed over the normalized string payload after applying domain extraction and whitespace collapsing. Used to detect semantic candidate shifts.
 * Canonical Domain Payload Hash (SHA-256): Computed over the strictly validated Pydantic model representation (excluding metadata like verification_timestamp). Used to determine actual Materiality.
Rule: A new record version is spawned only if the Canonical Domain Payload Hash changes or an explicit administrative mutation occurs. Whitespace or timestamp-only shifts are marked as NON_MATERIAL and do not increment the active version.
9. MATERIALITY & CHANGE CLASSIFICATION
Every change event is classified to dictate downstream operational workflows:
 * NON_MATERIAL: Formatting, typo corrections, or translation adjustments. Action: Updates record text without forcing re-validation or RAG re-indexing.
 * MATERIAL: Price adjustments, deadline changes, policy modifications. Action: Requires QC review and targeted RAG re-indexing.
 * SAFETY_CRITICAL: Any mutation affecting allergen safety matrices (KB-SPEC-006) or emergency operating parameters (KB-SPEC-004). Action: Mandatory human review, full safety gate re-evaluation, and immediate RAG purge of stale embeddings.
 * IDENTITY_CHANGING: Changes affecting core domain identity hashes. Action: Treated as a new logical entity with an automatic reset to UNVERIFIED.
10. REVALIDATION TRIGGERS
The system enforces deterministic revalidation triggers under specific operational conditions:
| Trigger Event | Affected Domain | Required Action | Blocking Severity | RAG Re-Index? | Booking Engine Re-Eval? |
|---|---|---|---|---|---|
| Source Material Materially Changed | All Domains | Create CANDIDATE, route through KB-SPEC-009 pipeline. | High | Yes (Upon activation) | Yes (Upon activation) |
| Safety-Critical Field Changed | KB-SPEC-006 (Allergens) | Mandatory human review; full matrix re-verification. | Critical | Yes (Immediate purge) | No |
| Upstream Schema Changed | All Domains | Batch re-validate affected records against updated schema rules. | Critical | Yes | Yes |
| Domain Identity Changed | KB-SPEC-005 / KB-SPEC-007 | Treat as new entity; reset safety/verification to UNVERIFIED. | Critical | Yes | Yes |
| Source Conflict Detected | KB-SPEC-007 / KB-SPEC-008 | Demote state to CONFLICT; fail closed. | Critical | Yes (Removal) | Yes (Blocking) |
| Source Becomes Unavailable | All Domains | Preserve current ACTIVE truth; flag freshness warning; do not delete. | Medium | No | No |
11. QUALITY CONTROL (QC) FRAMEWORK
The system enforces a continuous Quality Control framework encompassing four pillars:
 * Source Freshness: Scraped sources maintain TTLs (e.g., 30 days for opening hours, 7 days for unverified candidates). Expired freshness triggers scraper re-polling.
 * Review Status: Tracks whether candidate data has cleared automated rules and is awaiting human sign-off.
 * Stale Detection: Background cron jobs flag records whose source fingerprints have diverged or whose verification timestamps exceed domain freshness thresholds.
 * Materiality Classification: Automatically categorizes incoming diffs to route them to the correct validation queue.
12. SAFETY-CRITICAL VERSIONING (KB-SPEC-006 GUARDRAILS)
To ensure human life and health are never compromised by legacy assumptions, safety-critical versioning enforces absolute isolation:
 * No Inheritance of Safety: A newly created record version MUST NEVER inherit FREE_FROM, CONTAINS, or MAY_CONTAIN states from its predecessor automatically.
 * Mandatory Re-Verification: Any modification to a menu item's underlying description, category, or structural identity instantly resets all 14 EU-14 allergen states to UNKNOWN and demotes verification_status to UNVERIFIED.
 * Zero Trust: An active safety claim requires explicit, fresh verification evidence. Version evolution cannot bridge uncertainty into safety.
13. CONCURRENCY & ATOMIC ACTIVATION
To prevent race conditions where two simultaneous requests attempt to publish two active versions concurrently:
 * Optimistic Concurrency Control (OCC): Every administrative write request MUST supply the current Revision Number. If the database revision has advanced, the transaction is rejected with a VAL-CONCURRENCY-001 error.
 * Atomic Activation Constraint: The transition of Version N+1 to ACTIVE and Version N to SUPERSEDED MUST occur within a single ACID database transaction, protected by a unique partial database index ensuring that only one record per [venue_id, logical_entity_id, operational_dimension, operational_scope] may hold lifecycle_state == ACTIVE at any given instant.
14. INVALID DATA & FAILURE CASES
| Failure Case | Cause | Detecting Layer | Required Action | Severity |
|---|---|---|---|---|
| Concurrent Write Collision | Revision mismatch during write. | Persistence Layer | Reject transaction; force client reload. | High |
| Active Version Collision | Two records marked ACTIVE for same scope. | Database Unique Index | Abort transaction; demote both to CONFLICT. | Critical |
| Safety-Critical Inheritance | New item version inherits old allergen state. | Application Service | Block activation; reset EU-14 to UNKNOWN. | Critical |
| Destructive Mutation | Direct UPDATE on immutable historical row. | Database RLS / Trigger | Reject operation; trigger security audit. | Critical |
| Silent Rollback | Deleting history to revert state. | Audit Service | Trigger security alert; lock venue tenant. | Critical |
15. COMPLETE EXAMPLE JSON (VERSIONED AUDIT & LIFECYCLE EVENT)
{
  "kb_lifecycle_event": {
    "event_id": "evt_9f8e7d6c-5b4a",
    "venue_id": "brasserie_sthlm",
    "logical_entity_id": "pol_7a3d2f91",
    "domain": "POLICIES",
    "change_classification": "MATERIAL",
    "transition": {
      "from_lifecycle": "ACTIVE",
      "to_lifecycle": "SUPERSEDED",
      "from_revision": 1,
      "to_revision": 2
    },
    "provenance_fingerprints": {
      "raw_source_hash": "e3b0c44298fc1c149afbf4c8996fb92427ae41e4649b934ca495991b7852b855",
      "canonical_payload_hash": "8c6976e5b5410415bde908bd4dee15dfb167a9c873fc4bb8a81f6f2ab448a918"
    },
    "actor": {
      "actor_id": "admin_uuid_42",
      "actor_type": "HUMAN_ADMIN"
    },
    "timestamp": "2026-08-12T16:30:00Z",
    "reason": "Updated cancellation deadline from 24 hours to 48 hours per venue management request."
  }
}

16. PRODUCTION ACCEPTANCE CRITERIA
| AC ID | Category | Requirement | Verification Method | Expected Result | Pass/Fail | Severity | Blocking? |
|---|---|---|---|---|---|---|---|
| AC-01 | Immutability | Historical records are never mutated or deleted; changes always generate new revisions. | Audit History Test | 0% historical data loss. | Required | Critical | Yes |
| AC-02 | Concurrency | Simultaneous updates with stale revision numbers are rejected via OCC. | Race Condition Test | Transaction rejected. | Required | High | Yes |
| AC-03 | Activation | Unique database indices guarantee exactly one ACTIVE record per logical scope. | Unique Index Test | Duplicate active blocked. | Required | Critical | Yes |
| AC-04 | Safety Isolation | Safety-critical domain changes (KB-SPEC-006) reset safety states to UNKNOWN and demand re-verification. | Safety Gate Test | Reset to UNKNOWN enforced. | Required | Critical | Yes |
| AC-05 | Revalidation | Material source changes trigger automated revalidation workflows without silent application. | Ingestion Test | Candidate created for review. | Required | High | Yes |
| AC-06 | RAG Gating | Superseded or retired records are completely excluded from active vector indexing. | RAG Index Test | Only ACTIVE retrieved. | Required | Critical | Yes |
| AC-07 | Fingerprinting | Canonical payload hashes correctly suppress non-material whitespace/formatting diffs. | Diff Test Suite | Non-material suppressed. | Required | High | Yes |
| AC-08 | Audit Trail | Every lifecycle event, correction, and rollback emits an immutable, append-only audit record. | Audit Log Test | 100% event capture. | Required | High | Yes |
17. DOWNSTREAM DEPENDENCIES
 * Validation Engine (KB-SPEC-009): Provides the foundational validation pipeline that candidate versions must pass before entering the lifecycle verification queue.
 * RAG / Indexing Layer: Consumes ACTIVE and VERIFIED state flags to maintain vector index freshness and purge stale embeddings.
 * Admin Dashboard: Exposes optimistic concurrency controls, version diff views, and rollback authorization workflows to authorized operators.
18. VERSION HISTORY
| Version | Date | Description | Author | Approval Status |
|---|---|---|---|---|
| 1.0.0 | August 2026 | Initial enterprise Versioning & Quality Control specification. Established orthogonal state architecture, multi-tier hashing, materiality classification, revalidation triggers, safety-critical isolation, and concurrency-safe atomic activation. | Ramy Bella | 🔒 LOCKED / Approved for Implementation |
19. FINAL NON-NEGOTIABLE PRINCIPLES
 * HISTORICAL TRUTH IS IMMUTABLE: Previous versions of a record are never deleted or modified. The system maintains an append-only audit trail.
 * NO DESTRUCTIVE MUTATION: Corrections and rollbacks create new revisions. History is strictly additive.
 * ORTHOGONAL STATES: Domain states, lifecycle metadata, runtime-derived states, and isolation statuses are strictly separated and non-colliding.
 * SAFETY-CRITICAL ZERO TRUST: Allergen safety changes (KB-SPEC-006) never inherit previous safety assumptions; they demand fresh verification evidence.
 * SINGLE ACTIVE GUARANTEE: Atomic database transactions and unique partial indices ensure exactly one ACTIVE version exists per logical scope.
 * MATERIALITY GATING: Trivial whitespace or formatting shifts do not trigger disruptive version churn.
 * RAG RETRIEVAL GATING: Superseded and retired records are purged from active retrieval contexts.
 * CONCURRENCY SAFETY: Optimistic version pre-conditions prevent race conditions during administrative edits.
 * AUDIT TRAIL APPEND-ONLY: Every lifecycle mutation emits an immutable audit event.
 * DOMAIN PRECEDENCE: Upstream specification identity rules override generic versioning continuity models.
