KB-SPEC-008: Booking Rules Schema
1. DOCUMENT CONTROL
| Attribute | Value |
|---|---|
| Document Title | Booking Rules Schema |
| Document ID | KB-SPEC-008 |
| Version | 1.1.0 |
| Status | 🔒 LOCKED / Approved for Implementation |
| Author | Ramy Bella |
| Classification | Confidential / Enterprise Proprietary |
| Target Audience | Backend Engineers, Booking Engine Engineers, AI/RAG Engineers, Database Engineers, QA Engineers, Security Engineers, Platform Architects |
| Parent Document | KB-SPEC-002 — Master JSON Schema |
| Related Documents | KB-SPEC-001, KB-SPEC-003, KB-SPEC-004, KB-SPEC-005, KB-SPEC-006, KB-SPEC-007 |
| System | Restaurant AI System |
| Phase | Phase 2 — Knowledge Base Specification |
| Last Updated | August 2026 |
2. PURPOSE & SCOPE
Purpose
The Booking Rules Schema defines the authoritative, machine-readable constraints governing reservation eligibility and execution within the Restaurant AI System.
Booking rules dictate strict mathematical and operational boundaries—such as minimum lead times, seating durations, party size limits, and deposit requirements. If these rules are hallucinated by an LLM or improperly evaluated by the Booking Engine, the system risks creating invalid reservations, overbooking the venue, violating table turnover math, or committing the restaurant to unfulfillable guarantees.
Scope: What This Document Controls
 * The canonical JSON structure for deterministic booking constraint records (booking_rules).
 * Machine-readable parameters for party sizes, advance booking windows, lead times, seating durations, walk-in eligibility, and deposit mandates.
 * Explicit deterministic resolution of scope-specific rules (e.g., rules for the Terrace vs. Full Venue).
 * The 14-step deterministic evaluation pipeline utilized by the downstream Booking Engine API.
 * Strict boundaries on what the conversational AI is permitted to say regarding booking logic.
Scope: What This Document Explicitly Does NOT Control
 * Real-Time Availability: This schema defines rules (e.g., "Max party size is 8"). It does NOT store or track real-time table inventory or time-slot availability.
 * Opening Hours: Temporal operations are strictly governed by KB-SPEC-004. Booking rules act as filters within those open hours.
 * Menu & Pricing: Governed by KB-SPEC-005.
 * Allergen Safety: Governed by KB-SPEC-006.
 * General Operational Policies: Cancellation penalties, dress codes, and pet policies are governed by KB-SPEC-007. KB-SPEC-008 references cancellation execution only when a deposit is actively required to secure the reservation.
3. NON-NEGOTIABLE PRINCIPLES
 * UNKNOWN IS NEVER AN ASSUMPTION: Missing data equates to UNKNOWN, never to "unlimited" or "allowed".
 * NO BOOKING INFERENCE: The AI must never infer booking constraints from general restaurant descriptions.
 * NO SILENT DEFAULTS: The system must not quietly fall back to industry averages.
 * NO CROSS-TENANT ACCESS: Every booking rule evaluation is strictly isolated to its parent venue_id.
 * NO LLM-BASED BOOKING DECISIONS: The Booking Engine computes eligibility deterministically. The LLM only explains the outcome.
 * NO INVENTED AVAILABILITY / CAPACITY: The AI must never promise a table or time slot.
 * FAIL CLOSED ON CONFLICT: Conflicting booking rules instantly demote to CONFLICT and block automated booking execution for that dimension.
 * TEMPORAL STRICTNESS: Expired rules are operationally dead.
 * OMISSION OVER NULL: Missing facts rely exclusively on omitted keys, never on null.
 * NO CROSS-DOMAIN OVERRIDES: Booking rules cannot override opening hours (KB-SPEC-004), allergen assertions (KB-SPEC-006), or general policy truth (KB-SPEC-007).
4. DOMAIN ARCHITECTURE
To align with KB-SPEC-002, the booking constraints are encapsulated within the "booking_rules" dictionary object, mapped as an array of structured records.
"booking_rules": {
  "records": []
}

Every record represents an atomic, scope-bound rule configuration that the Booking Engine queries before authorizing a transaction.
5. CONTROLLED ENUMS
To guarantee deterministic evaluation, booking rules rely entirely on strict enumerations. Free-form text is banned for rule evaluation fields.
5.1 rule_type
 * PARTY_SIZE_LIMITS: Constrains minimum and maximum guest counts.
 * ADVANCE_BOOKING_WINDOW: Constrains how early or late a booking can be made.
 * SEATING_DURATION: Defines table turnover time allowances.
 * WALK_IN_POLICY: Declares walk-in eligibility and conditional boundaries.
 * DEPOSIT_REQUIREMENT: Mandates upfront financial holds to secure a booking.
 * CHANNEL_RESTRICTION: Limits which platforms/methods can execute bookings.
5.2 operational_scope
 * FULL_VENUE, INDOOR_DINING, OUTDOOR_TERRACE, BAR_AREA, PRIVATE_DINING.
5.3 state
 * VERIFIED: Approved for booking engine execution.
 * UNVERIFIED: Pending approval; excluded from execution.
 * CONFLICT: Contradictory rules detected; blocks execution.
 * EXPIRED: System-derived temporal expiration.
 * INACTIVE: Manually disabled by admin.
5.4 walk_in_behavior
 * ALLOWED: Walk-ins accepted subject to physical capacity.
 * NOT_ALLOWED: Strictly reservation-only.
 * RESTRICTED: Allowed only under specific structured conditions.
6. BOOKING RULE TYPES & STRUCTURED VALUES
Each rule_type requires a specific structured_value payload. Mixing fields across rule types renders the payload structurally invalid.
6.1 PARTY_SIZE_LIMITS
Governs the mathematical boundaries of a single reservation.
 * Fields:
   * min_party_size (Integer, Optional)
   * max_party_size (Integer, Optional)
 * Invariants: If both are present, max_party_size \ge min_party_size.
6.2 ADVANCE_BOOKING_WINDOW
Governs temporal lead times and future horizons. Timezone context is strictly inherited from the venue's canonical timezone (KB-SPEC-004).
 * Fields:
   * min_lead_time_minutes (Integer, Optional): Minimum time duration between booking creation timestamp and reservation start timestamp.
   * max_advance_days (Integer, Optional): Maximum calendar days into the future a booking is allowed.
   * same_day_cutoff_time (String HH:MM, Optional): Venue-local wall-clock time after which same-day bookings are blocked. Evaluated strictly against the venue's IANA timezone (KB-SPEC-004).
6.3 SEATING_DURATION
Governs table turnover allocations.
 * Fields:
   * base_duration_minutes (Integer, Required): Default duration applied when no party size tier matches.
   * tiered_durations (Array of Objects, Optional): Overrides base duration based on party size.
     * min_party_size (Integer, Required)
     * max_party_size (Integer, Optional)
     * duration_minutes (Integer, Required)
 * Invariants (Tier Completeness): Tier ranges must be completely contiguous and non-overlapping. If a party size falls outside explicitly defined tiers, it MUST fall back to base_duration_minutes.
6.4 WALK_IN_POLICY
 * Fields:
   * walk_in_behavior (ENUM, Required): ALLOWED, NOT_ALLOWED, RESTRICTED.
   * restrictions (Array of Strings, Conditional): Required if walk_in_behavior is RESTRICTED (e.g., ["BAR_SEATING_ONLY", "MAX_PARTY_SIZE_4"]). Must be omitted otherwise.
6.5 DEPOSIT_REQUIREMENT
Governs upfront holds required to secure the booking.
 * Fields:
   * deposit_type (ENUM, Required): FLAT_FEE, PER_PERSON_FEE, PERCENTAGE, NONE, FULL_RESERVATION_VALUE.
   * deposit_amount_minor_units (Integer, Conditional): Required for FLAT_FEE and PER_PERSON_FEE. Must be omitted otherwise.
   * deposit_percentage_basis_points (Integer, Conditional): Required for PERCENTAGE (e.g., 5000 = 50.00%). Must be omitted otherwise.
 * Monetary Inheritance: Currency is strictly inherited from venue.currency (KB-SPEC-003). Local currency declarations within booking rules are forbidden.
6.6 CHANNEL_RESTRICTION
 * Fields:
   * allowed_channels (Array of ENUMs, Optional): DIRECT_WEBSITE, PHONE, IN_PERSON, THIRD_PARTY_API.
 * Absence Semantics: If allowed_channels is omitted or empty, the rule evaluates to UNKNOWN (meaning channel availability is unverified, rather than defaulting to "all channels allowed").
7. DETERMINISTIC IDENTITY & TEMPORAL VERSIONING
To guarantee identity stability and prevent unbound array bloat, booking_rule_id is generated cryptographically. A venue may only have ONE active rule per rule_type per operational_scope at any given instant.
rule_identity_input = canonical_utf8(venue_id + "|" + rule_type + "|" + operational_scope)
booking_rule_id = "br_" + SHA256(rule_identity_input)

Versioning Semantics
 * A booking rule version MUST retain the same booking_rule_id when venue_id, rule_type, and operational_scope remain unchanged.
 * Temporal versioning is represented entirely by updating effective_from and effective_until dates, never by generating a new identity hash. Superseding a rule requires capping the old record's effective_until timestamp and spawning a new historical state entry.
8. TEMPORAL VALIDITY
Booking rules change over time.
 * effective_from: UTC RFC 3339 Timestamp (Mandatory).
 * effective_until: UTC RFC 3339 Timestamp (Optional). If omitted, the rule is open-ended.
 * Half-Open Intervals: Temporal intervals are exclusively half-open: [effective_from, effective_until). If effective_until exactly equals T_{query}, the rule is already expired.
 * Historical Preservation: Expired booking rules are retained for auditability. They are mathematically excluded from current evaluation.
9. RULE PRECEDENCE & SCOPE RESOLUTION
When the Booking Engine evaluates eligibility, it must resolve the specific operational_scope of the requested reservation.
Precedence Hierarchy
 * Specific Scope Match: If the reservation requests OUTDOOR_TERRACE, rules scoped to OUTDOOR_TERRACE evaluate first.
 * Fallback to Broad Scope: If no rule for the specific scope exists, the evaluation falls back to the FULL_VENUE rule of the same rule_type.
 * No Cross-Scope Contamination: An OUTDOOR_TERRACE rule MUST NOT influence an INDOOR_DINING evaluation.
10. DETERMINISTIC CONFLICT DETECTION
A conflict is triggered ONLY when two records possess:
 * Overlapping [effective_from, effective_until) active temporal windows.
 * The exact same rule_type and operational_scope.
 * Contradictory structured_value data.
Fail-Closed Resolution:
If conflict is detected, ALL conflicting records are demoted to CONFLICT state. The Booking Engine treats the dimension as UNKNOWN and halts automated booking execution to prevent unauthorized reservations.
11. UNKNOWN VS INELIGIBLE (CRITICAL SEMANTICS)
The system enforces a strict mathematical distinction between missing data and restrictions.
 * UNKNOWN: The record is missing or UNVERIFIED. The Booking Engine MUST NOT assume capacity is unlimited or channels are open.
 * INELIGIBLE: The data is VERIFIED, and the guest's request explicitly violates the mathematical boundary (e.g., party size exceeds max_party_size).
12. BOOKING DECISION MODEL (14-STEP PIPELINE)
The Booking Engine must execute the following deterministic pipeline before confirming an automated reservation:
 * Identify Context: Load CURRENT_CONTEXT_VENUE.
 * Enforce Tenant Isolation: Ensure the query maps strictly to the target venue_id via RLS constraints, propagating venue_id into all indexing and execution records.
 * Retrieve Active Rules: Fetch all records where state == VERIFIED.
 * Apply Temporal Filters: Discard records where T_{query} falls outside the half-open interval [effective_from, effective_until).
 * Filter by Scope & Precedence: Match records corresponding to the requested reservation area, falling back to FULL_VENUE.
 * Evaluate Conflict Check: If any targeted rule is in CONFLICT, abort and return FAIL_CLOSED_CONFLICT.
 * Evaluate Channels: If CHANNEL_RESTRICTION exists, reject if the current channel is unlisted. If omitted (UNKNOWN), defer to manual verification.
 * Evaluate Party Size: Check PARTY_SIZE_LIMITS. Reject if out of bounds.
 * Evaluate Advance Window: Check ADVANCE_BOOKING_WINDOW (calculating lead time from booking creation to reservation start, and validating same_day_cutoff_time against the venue's IANA timezone). Reject if out of bounds.
 * Determine Duration: Extract SEATING_DURATION, evaluating party size against defined tiers or falling back to base_duration_minutes.
 * Evaluate Deposit: Check DEPOSIT_REQUIREMENT (inheriting currency from venue.currency) to append required transaction holds.
 * Cross-Check Operating Hours (Strict): Validate the entire computed time block ([reservation_start, reservation_start + duration)) against KB-SPEC-004 (Opening Hours). Reject if any portion falls outside venue or kitchen operating hours.
 * Result Compilation: If all applicable verified rules pass, return ELIGIBLE (meaning rules permit the reservation, NOT that a physical table is guaranteed free) along with required deposit/duration metadata.
 * Missing Data Fallback: If critical dimensions are UNKNOWN, route to manual venue confirmation. Never invent availability.
13. PROVENANCE & AUDITABILITY
Every authoritative booking rule must possess traceable lineage.
 * Lineage Object: Requires source_type (OFFICIAL_WEBSITE, OFFICIAL_BOOKING_PLATFORM, HUMAN_VERIFICATION). Note: OFFICIAL_BOOKING_PLATFORM is authoritative only when explicitly registered as an official venue-controlled or venue-authorized source.
 * Source URL: Required for algorithmic extractions.
 * Verified By: Actor UUID required for human overrides.
 * Audit Immutability: Administrative changes MUST NOT mutate historical truth destructively. Updates cap effective_until on the old record and spawn a new record.
14. NULLABILITY & NORMALIZATION
 * Omission Over Null: Missing fields must be omitted. null is architecturally banned within structured_value.
 * Zero vs Omission: An omitted min_lead_time_minutes means UNKNOWN. A value of 0 means exactly zero minutes (instant booking allowed).
 * Normalization Pipeline: Scraped text (e.g., "Max 8 people") is mapped to max_party_size: 8. Ambiguous text MUST NOT automatically become an integer limit. It remains unstructured UNVERIFIED data requiring manual review.
15. AI / RAG SAFETY CONTRACT
Permitted AI Behavior
The AI may explain the deterministic rules to theater guests:
 * "The maximum party size for online bookings is 8 guests."
 * "You need to book at least 2 hours in advance."
Forbidden AI Behavior
 * Executing Decisions: The AI MUST NEVER state "Your booking is confirmed." Booking execution belongs solely to the transaction API.
 * Inventing Exceptions: The AI MUST NEVER say "The limit is 8, but we can probably squeeze in 9."
 * Inferring Availability: The AI MUST NEVER infer real-time table availability from static booking rules. ELIGIBLE strictly means rules permit it, not that a table is physically free.
16. MULTI-TENANT SECURITY
 * Venue Scoping: venue_id MUST be propagated into every persistence, RAG indexing, cache key, and evaluation pipeline operation.
 * PostgreSQL RLS: Row-Level Security policies natively prevent cross-tenant rule retrieval.
 * Authentication: API evaluation queries and administrative edits require authenticated authorization strictly matching the target venue_id.
17. INVALID DATA & FAILURE CASES
| Failure Case | Why Invalid | Detecting Layer | Expected Behavior | Severity |
|---|---|---|---|---|
| Missing Required Field | E.g., base_duration_minutes omitted from SEATING_DURATION. | Structural Validator | Reject payload instantly. | Critical |
| Logic Bounds Violation | min_party_size: 10, max_party_size: 4. | Business Logic Validator | Reject payload. | High |
| Invalid ENUM | deposit_type: "CASH_ONLY". | Semantic Validator | Reject payload. | High |
| Float Value | deposit_amount_minor_units: 50.50. | Structural Validator | Reject (requires integers). | Critical |
| null for Basis Points | Contains forbidden null value. | Structural Validator | Reject payload (omission required). | High |
| UNVERIFIED Rule in Pipeline | Unverified rule polled for booking execution. | API Evaluation Layer | Rule mathematically excluded from step 3. | High |
| Overlapping Tier Durations | Non-contiguous party size tiers. | Business Logic Validator | Demote record to CONFLICT. | Critical |
18. COMPLETE VALID JSON EXAMPLES
Example: Advanced Party Size & Duration
{
  "booking_rules": {
    "records": [
      {
        "booking_rule_id": "br_3b9f1a2c",
        "rule_type": "SEATING_DURATION",
        "operational_scope": "FULL_VENUE",
        "state": "VERIFIED",
        "structured_value": {
          "base_duration_minutes": 120,
          "tiered_durations": [
            { "min_party_size": 1, "max_party_size": 2, "duration_minutes": 90 },
            { "min_party_size": 3, "max_party_size": 6, "duration_minutes": 120 },
            { "min_party_size": 7, "duration_minutes": 150 }
          ]
        },
        "effective_from": "2025-01-01T00:00:00Z",
        "lineage": {
          "source_type": "HUMAN_VERIFICATION",
          "verified_by": "admin_user_007",
          "verification_timestamp": "2026-08-10T14:00:00Z"
        }
      },
      {
        "booking_rule_id": "br_8d4e9c1b",
        "rule_type": "ADVANCE_BOOKING_WINDOW",
        "operational_scope": "FULL_VENUE",
        "state": "VERIFIED",
        "structured_value": {
          "min_lead_time_minutes": 120,
          "max_advance_days": 90,
          "same_day_cutoff_time": "20:00"
        },
        "effective_from": "2025-01-01T00:00:00Z",
        "lineage": {
          "source_type": "OFFICIAL_BOOKING_PLATFORM",
          "source_url": "https://booking.com/brasserie",
          "verification_timestamp": "2026-08-10T14:00:00Z"
        }
      }
    ]
  }
}

19. INVALID EXAMPLES
Example A: Deposit Nullability & Float Violation
{
  "booking_rule_id": "br_invalid_01",
  "rule_type": "DEPOSIT_REQUIREMENT",
  "operational_scope": "FULL_VENUE",
  "state": "VERIFIED",
  "structured_value": {
    "deposit_type": "FLAT_FEE",
    "deposit_amount_minor_units": 500.00,
    "deposit_percentage_basis_points": null
  }
}

 * Why it fails: deposit_amount_minor_units contains a float instead of an integer. deposit_percentage_basis_points contains a forbidden null (it must be entirely omitted when type is FLAT_FEE).
20. PRODUCTION ACCEPTANCE CRITERIA
| AC ID | Category | Requirement | Verification Method | Expected Result | Pass/Fail | Severity |
|---|---|---|---|---|---|---|
| AC-01 | Structural | Structural validation enforces strict ENUM matching, integer constraints, and omission-over-null logic for all payload fields. | CI Schema Test | Invalid payloads rejected. | Required | Critical |
| AC-02 | Identity | booking_rule_id guarantees exactly one rule ID per rule_type + operational_scope combination. | Hash Collision Test | Identical IDs for identical scopes. | Required | Critical |
| AC-03 | Conflict | The Booking Engine gracefully fails closed (status CONFLICT) if database corruption forces overlapping identical rules. | Evaluation Simulation | Returns CONFLICT. | Required | Critical |
| AC-04 | Inference | Evaluation of missing rules correctly resolves to UNKNOWN and does not auto-populate default constraints. | Engine Unit Test | Returns UNKNOWN matrix. | Required | High |
| AC-05 | Precedence | PRIVATE_DINING scoped rules successfully supersede FULL_VENUE rules when the evaluation context explicitly requests private dining. | Scope Resolution Test | Specific rule evaluated. | Required | High |
| AC-06 | Temporal | Expired records (effective_until <= T_{query}) are ignored by the 14-step evaluation pipeline. | Temporal Simulation | Historical records skipped. | Required | Critical |
| AC-07 | Monetary | Monetary payloads use exact integer minor units; floats trigger instant ingestion rejection. | Boundary Test | Floats rejected. | Required | High |
21. DOWNSTREAM DEPENDENCIES
 * Knowledge Base Validation (KB-SPEC-009): Implements the Pydantic models enforcing temporal half-open logic, structural invariants, and deterministic ID hashing.
 * Booking Engine API (Modul 5): The primary consumer. Relies absolutely on the 14-step decision model to authorize or reject transaction requests.
 * AI Engine & RAG (Modul 3): Retrieves these rules strictly for factual guest Q&A, adhering to the AI prohibitions against inferring or executing bookings.
22. VERSION HISTORY
| Version | Date | Description | Author | Approval Status |
|---|---|---|---|---|
| 1.1.0 | August 2026 | Patch revision: Clarified temporal versioning over immutable IDs, enforced strict lineage gating for booking platforms, added explicit tier completeness invariants for seating durations, clarified ELIGIBLE semantics, and defined strict window evaluation against venue timezones. | Ramy Bella | 🔒 LOCKED / Approved for Implementation |
| 1.0.0 | August 2026 | Initial Booking Rules Schema. | Ramy Bella | Superseded |
23. FINAL NON-NEGOTIABLE PRINCIPLES
 * UNKNOWN IS NOT ELIGIBLE: Missing data means the system lacks information, not that restrictions are waived.
 * NO BOOKING INFERENCE: The AI and Booking Engine must execute strictly on explicit parameters. Silence is not permission.
 * FAIL CLOSED ON CONFLICT: If parameters contradict, automated booking evaluation halts immediately.
 * NO LLM BOOKING AUTHORITY: The LLM interprets language; the deterministic backend API interprets booking logic.
 * OMISSION OVER NULL: Missing facts rely exclusively on omitted keys, never on null.
 * IMMUTABLE AUDIT HISTORY: Changes to execution rules spawn new temporal versions via effective dates. History is never destroyed.
 * NO CROSS-DOMAIN OVERRIDES: Booking constraints cannot force the venue open outside of KB-SPEC-004 operating hours, nor alter KB-SPEC-007 policies.
