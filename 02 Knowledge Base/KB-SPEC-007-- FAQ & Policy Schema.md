
KB-SPEC-007: FAQ & Policy Schema

1. DOCUMENT CONTROL
| Attribute | Value |
|---|---|
| Document Title | FAQ & Policy Schema |
| Document ID | KB-SPEC-007 |
| Version | 1.3.0 |
| Status | 🔒 LOCKED / Approved for Implementation |
| Author | Ramy Bella |
| Classification | Confidential / Enterprise Proprietary |
| Target Audience | Backend Engineers, Data Engineers, AI Engineers, RAG Engineers, Database Engineers, QA Engineers, Security Engineers, Platform Architects |
| Parent Document | KB-SPEC-002 — Master JSON Schema |
| Related Documents | KB-SPEC-001, KB-SPEC-003, KB-SPEC-004, KB-SPEC-005, KB-SPEC-006, MASD-DOC-007 |
| System | Restaurant AI System |
| Phase | Phase 2 — Knowledge Base Specification |
| Last Updated | August 2026 |


2. PURPOSE & SCOPE
Purpose
The FAQ & Policy Schema defines the authoritative data contract for all operational policies and frequently asked questions within the Restaurant AI System's Master Knowledge Base.
Restaurant policies (e.g., cancellations, reservations, pet access, dress codes) dictate the operational boundaries of the guest experience. If policies are hallucinated, inferred incorrectly, or presented out-of-date, it directly causes guest disputes, lost revenue, and reputational damage. This schema ensures that policy data is stored deterministically, retrieved securely, and presented accurately.
Scope: What This Document Controls



The canonical JSON structure for both structured operational policies and unstructured faq records.

Explicit deterministic hashing for immutable record identity.

The explicit state machine governing policy validity (VERIFIED, CONFLICT, EXPIRED, etc.).

Temporal validity mapping (half-open intervals) and overlap conflict resolution based on specific operational dimensions and scopes.

Multilingual representation separating policy logic from translated strings.

Strict boundaries on AI interpretation and presentation phrasing.
Scope: What This Document Explicitly Does NOT Control

Allergen & Dietary Safety: EU-14 allergen tracking and dietary safety assertions are strictly governed by KB-SPEC-006. Policies must never override safety data.

Opening Hours: Temporal venue operations are strictly governed by KB-SPEC-004.

Menu Data & Pricing: Food items and item-level pricing are strictly governed by KB-SPEC-005.


3. RELATIONSHIP TO PREVIOUS SPECIFICATIONS



KB-SPEC-002 (Master JSON): Resolves the "policies": {} and "faq": {} object namespaces defined in the master schema by specifying their internal structure (objects containing indexed records).

KB-SPEC-003 (Venue Metadata): Inherits venue_id for strict multi-tenant isolation. Note: venue_id is mandatory at the Master Knowledge Base document/root level and MUST be propagated into every persistence and RAG indexing record as immutable tenant metadata. It need not be duplicated inside the canonical policy/FAQ JSON record.

KB-SPEC-004 (Opening Hours): Maintains clear boundaries; "Kitchen closing" policies may be explained here textually for FAQs, but operational routing math belongs exclusively to KB-004.

KB-SPEC-006 (Allergen Schema): CRITICAL BOUNDARY. A policy stating "Vegan options available" or "Nut-free kitchen" MUST NOT mutate the EU-14 allergen matrix. Allergen claims must be processed strictly through the KB-SPEC-006 verification pipeline.


4. DOMAIN PHILOSOPHY / NON-NEGOTIABLE PRINCIPLES



UNKNOWN IS NOT AN ANSWER: If a policy cannot be verified, the AI must not invent an answer. Absence of a record evaluates deterministically to UNKNOWN.

NO POLICY INFERENCE: The AI must never infer restaurant policy from cuisine type, generic industry norms, or absence of a statement (e.g., silence on pets \ne pets prohibited).

EXPLICIT POLICY EVIDENCE: A policy may only be represented as VERIFIED when supported by an authoritative source or authorized human verification.

SOURCE CONFLICT = FAIL CLOSED: If authoritative sources conflict on the exact same dimension, the policy evaluates to CONFLICT. The AI must explicitly state its inability to provide a settled answer.

POLICY \ne ALLERGEN SAFETY: Policy assertions cannot create, modify, or override physiological safety data.

POLICY \ne MENU DATA: Policy assertions cannot implicitly create menu items or alter menu prices.

TENANT ISOLATION: Every record is strictly scoped to its inherited venue_id.

DETERMINISTIC RETRIEVAL: Policy retrieval relies on explicit structured fields and metadata, not uncontrolled LLM interpretation.

AUDITABLE OVERRIDE: Human verification and administrative edits must retain exact provenance (actor identity, timestamp).

NO FABRICATION: The AI is mathematically constrained to answering strictly using the retrieved canonical fields.


5. DOMAIN ARCHITECTURE
To strictly align with the KB-SPEC-002 definition of "policies": {} and "faq": {} as JSON objects, the domain is partitioned into two complementary dictionary objects mapped by unique record IDs:



policies (Structured): Machine-readable rules mapped by policy_id. Contains explicit operational_dimension and operational_scope fields along with defined ENUMs, integers, and booleans intended to drive both AI answers and downstream booking/operational logic deterministically.

faq (Unstructured): Free-form Question & Answer pairs mapped by faq_id. Covers edge cases or highly specific venue quirks that do not fit into rigid operational schemas.
Both partitions share the identical state machine, temporal validity, and provenance logic.


6. POLICY / FAQ CATEGORY DEFINITION
Every record must be classified under a strictly controlled ENUM to ensure deterministic RAG filtering and UI categorization.
Controlled Category ENUMs



RESERVATION: Rules regarding booking, walk-ins, party sizes, and lead times.

CANCELLATION: Deadlines, no-show fees, and modification rules.

PAYMENT: Accepted cards, cash policies, and split-bill rules.

ACCESS_AND_PARKING: Wheelchair accessibility, elevators, parking availability.

FACILITIES: Wi-Fi, coat check, restrooms, highchairs.

ATMOSPHERE: Dress codes, volume levels, child policies.

PETS: Rules regarding dogs, service animals, and permitted zones.

EVENTS_AND_GROUPS: Private dining, minimum spend, large group policies.

EXTERNAL_GOODS: Corkage fees, cakeage fees, outside food policies.

GENERAL_FAQ: Any venue-specific question not covered by the operational categories.


7. POLICY STATE MODEL
Records must possess exactly one of the following deterministic states:
| State | Semantic Definition | AI Interaction Rule |
|---|---|---|
| VERIFIED | Explicitly supported by an authoritative source or human admin. | Retrieve and present as authoritative current policy. |
| UNVERIFIED | Candidate data extracted via scraping, pending human/heuristic approval. | Excluded from guest-facing RAG. |
| CONFLICT | Two or more authoritative sources provide mutually exclusive values for the exact same operational dimension and scope. This is a system-derived state and MUST NOT be manually assigned by an administrator. | Excluded from declarative RAG. AI must respond: "Sources conflict; please contact the venue." |
| EXPIRED | A deterministic temporal/runtime state derived from the current UTC time passing the effective_until timestamp. Historical records remain preserved for auditability. Superseding a policy must never delete historical records. This is a system-derived state and MUST NOT be manually assigned by an administrator. | Excluded from current RAG context. |
| INACTIVE | Manually disabled by a venue administrator. | Excluded from all RAG context. |
Note: The distinction between "the restaurant has no policy" and "the system has no verified information" is explicit. If the venue has no dress code, a VERIFIED policy exists stating dress_code: NONE. If no record exists, the system evaluates to UNKNOWN.


8. MASTER JSON STRUCTURE
8.1 The policies Object (Structured)
"policies": {
"records": [
{
"policy_id": "pol_7a3d2f91",
"category": "CANCELLATION",
"operational_dimension": "cancellation_penalty",
"operational_scope": "FULL_VENUE",
"state": "VERIFIED",
"structured_value": {
"cancellation_deadline_hours": 24,
"penalty_type": "FLAT_FEE",
"penalty_amount_minor_units": 50000
},
"translations": [
{
"language": "en",
"text": "Cancellations must be made at least 24 hours in advance to avoid a 500 SEK fee."
}
],
"effective_from": "2026-01-01T00:00:00Z",
"lineage": {
"source_type": "OFFICIAL_BOOKING_PLATFORM",
"source_url": "https://booking.com/brasserie",
"verification_timestamp": "2026-08-10T12:00:00Z"
}
}
]
}



8.2 The faq Object (Unstructured)
"faq": {
"records": [
{
"faq_id": "faq_c8d9e230",
"category": "GENERAL_FAQ",
"state": "VERIFIED",
"translations": [
{
"language": "en",
"question": "Do you sell gift cards?",
"answer": "Yes, physical gift cards can be purchased at the bar during opening hours."
}
],
"effective_from": "2026-06-01T00:00:00Z",
"lineage": {
"source_type": "HUMAN_VERIFICATION",
"verified_by": "admin_uuid_42",
"verification_timestamp": "2026-08-11T09:00:00Z"
}
}
]
}

9. STRUCTURED POLICY DEFINITIONS
Where applicable, policies MUST utilize machine-readable structured_value schemas alongside explicit operational_dimension and operational_scope indicators to support algorithmic downstream booking engines.
9.1 Operational Scope ENUM
operational_scope strictly bounds the physical or logical area a policy applies to. This MUST be populated from a controlled ENUM to ensure reliable conflict detection and prevent scope evasion (e.g., distinguishing "TERRACE" from "OUTDOOR").



ENUMs: FULL_VENUE, INDOOR_DINING, OUTDOOR_TERRACE, BAR_AREA, PRIVATE_DINING.
9.2 Cancellation Invariants

operational_dimension: "cancellation_penalty"

cancellation_deadline_hours (Integer)

penalty_type (ENUM)
Strict Validation Invariants for Cancellation Penalties:

NONE: penalty_amount_minor_units MUST be omitted; penalty_percentage_basis_points MUST be omitted.

FLAT_FEE: penalty_amount_minor_units MUST be present; penalty_percentage_basis_points MUST be omitted.

PER_PERSON_FEE: penalty_amount_minor_units MUST be present; penalty_percentage_basis_points MUST be omitted.

PERCENTAGE: penalty_percentage_basis_points MUST be present (e.g., 5000 = 50.00%); penalty_amount_minor_units MUST be omitted.

FULL_RESERVATION_VALUE: Both penalty_amount_minor_units and penalty_percentage_basis_points MUST be omitted.
9.3 Pets & Service Animals

operational_dimension: "pet_access"

pet_access (ENUM: ALLOWED_EVERYWHERE, TERRACE_ONLY, NOT_ALLOWED)

service_animal_access (ENUM: ALLOWED, RESTRICTED, UNKNOWN) (Note: Separating these ensures that restricting standard pets to the terrace does not conflict with legal service animal access rules).
9.4 Atmosphere

operational_dimension: "atmosphere"

dress_code (ENUM: NONE, CASUAL, SMART_CASUAL, FORMAL, JACKET_REQUIRED)

children_allowed (Boolean)
If a source provides natural language that cannot map deterministically to these fields, the Normalization pipeline must drop the structured_value, retain the translations text, and flag the record for human review.


10. POLICY IDENTITY & VERSIONING
To guarantee deterministic hashing across rescrapes and ensure safe multi-tenant schema isolation, identifiers are generated cryptographically.
10.1 Deterministic Hashing (policy_id)
policy_identity_input = canonical_utf8(venue_id + "|" + category + "|" + operational_dimension + "|" + operational_scope)
policy_id = "pol_" + SHA256(policy_identity_input)



10.2 Deterministic Hashing (faq_id)
faq_identity_input = canonical_utf8(venue_id + "|" + normalized_question_text)
faq_id = "faq_" + SHA256(faq_identity_input)

(Where normalized_question_text is processed strictly via: Unicode NFKC normalization \rightarrow Unicode casefold \rightarrow trim leading/trailing whitespace \rightarrow removal of all punctuation \rightarrow collapse of internal whitespace to a single space \rightarrow UTF-8 encoding).
10.3 Versioning & Translation Semantics

Translations Separation: Multilingual support is handled within the translations array of a single policy ID. The system MUST NOT create separate policy records merely because a policy is translated. Translations share the same structured logic and operational truth.

Versioning: A policy version MUST retain the same policy_id when category, operational_dimension, operational_scope, and venue_id remain unchanged. Temporal versioning is represented by effective_from and effective_until dates, not by generating a new identity hash. If a structured policy fundamentally changes (e.g., cancellation window moves from 24h to 48h), the existing policy's effective_until is populated with the current timestamp, and a new historical record state is effectively created by the effective_from of the overriding data. Historical records remain preserved for auditability. Superseding a policy must never delete historical records.


11. SOURCE & PROVENANCE
Every authoritative policy must have traceable provenance.
Accepted Source ENUMs



OFFICIAL_WEBSITE

OFFICIAL_POLICY_PAGE

OFFICIAL_BOOKING_PLATFORM (A booking platform is authoritative only when it is explicitly registered/recognized as an official venue-controlled or venue-authorized source. Third-party aggregator listings must never automatically qualify as authoritative.)

PDF_DOCUMENT

HUMAN_VERIFICATION
Provenance Rules

If the source is algorithmic extraction (OFFICIAL_WEBSITE), source_url is Mandatory.

If the source is HUMAN_VERIFICATION, source_url MUST be omitted, but verified_by (Actor ID) is Mandatory.

Unverifiable provenance strictly prohibits the record from entering the VERIFIED state.


12. VERIFICATION MODEL & CONFLICT RESOLUTION
Deterministic Conflict Detection
The validation pipeline continually monitors the Knowledge Base for conflicts. A conflict is triggered ONLY when two or more records possess:



Overlapping [effective_from, effective_until) active temporal windows.

The exact same operational_dimension and operational_scope.

Contradictory structured_value assignments.
FAQ Conflict Boundaries: Free-form FAQ records MUST NOT be treated as autonomous structured policy truth when they cannot be deterministically mapped to an explicit operational dimension. Conflict detection for FAQ records must not depend on LLM semantic guessing. Only deterministically mapped structured policy dimensions participate in automatic dimensional conflict detection.
Fail-Closed Resolution
If a genuine scope conflict is detected (e.g., Website says dogs allowed on terrace, Booking platform says no pets on terrace):

The system MUST NOT use an LLM or array order to guess the correct policy.

The state of ALL conflicting records is immediately downgraded to CONFLICT.

CONFLICT records are mathematically excluded from active RAG retrieval.

The AI gracefully fails closed on guest inquiries regarding that specific dimension.


13. POLICY EFFECTIVE DATES (TEMPORAL VALIDITY)
Policies change over time. The schema strictly enforces temporal validity to prevent hallucinating expired rules.



effective_from: UTC RFC 3339 Timestamp (Mandatory). The exact second the policy becomes enforceable.

effective_until: UTC RFC 3339 Timestamp (Optional). If the policy is open-ended and has no defined expiration, this key MUST be omitted.

Half-Open Intervals: Temporal intervals are exclusively half-open: [effective_from, effective_until). If effective_until exactly equals T_{query}, the record is already expired.

Runtime Evaluation: Downstream RAG engines MUST inject a filter: effective_from <= UTC_NOW AND (effective_until IS OMITTED OR effective_until > UTC_NOW). EXPIRED is a deterministic temporal/runtime state derived from effective_until being passed.

Expired Exclusion: The AI MUST NOT answer with an expired policy as if it were current, and MUST NOT extrapolate historical data to predict current states.


14. NULLABILITY & MISSING DATA
To eliminate ambiguity, KB-SPEC-007 enforces the strict omission of keys for missing factual data. null is forbidden.



Missing Data: If no valid, active record exists for a specific policy category/dimension, the system evaluates that concept as UNKNOWN.

Explicit Absence: The schema forbids using null as a value inside structured_value to mean "not allowed." (e.g., penalty_type must be explicitly NONE, and penalty_amount_minor_units completely omitted).

Open-Ended Dates: effective_until must be omitted rather than assigned null.

Silent Defaults Banned: The absence of a pet policy does NOT mean pets are forbidden, nor does it mean they are allowed. It means UNKNOWN.


15. NORMALIZATION PIPELINE
Unstructured source text must be deterministically mapped.



Valid Extraction: "Please cancel at least 24 hours before your reservation" \rightarrow cancellation_deadline_hours: 24.

Prohibited Inference (Silence): "Pets are not mentioned on the website" \rightarrow MUST NOT become pet_access: NOT_ALLOWED. The category remains UNKNOWN.

Prohibited Inference (Generalization): "Most fine dining requires a jacket" \rightarrow MUST NOT populate the venue's policy. Candidate data must explicitly reference the specific venue.


16. AI / RAG SAFETY CONTRACT
The AI may only answer policy questions using authoritative retrieved records.
Retrieval Boundaries
RAG Queries MUST enforce strict metadata filtering prior to context injection:



state == VERIFIED

venue_id == CURRENT_CONTEXT_VENUE

effective_from <= NOW AND (effective_until IS OMITTED OR effective_until > NOW).
Fallback Phrasing Mandates
If no retrieved chunks satisfy the query, the AI MUST use the following fallback patterns:

For UNKNOWN: "I don't have verified information about the restaurant's [policy]. Please contact the venue directly."

For CONFLICT: "The available policy sources conflict, so I cannot reliably confirm the current rule for [policy]."
AI Prohibitions
The AI is strictly PROHIBITED from:

Inventing exceptions (e.g., "Pets aren't allowed, but a small dog in a bag is probably fine.").

Treating historical/expired policy as current policy.

Making legal claims about restaurant obligations.

Combining multiple unrelated records to hallucinate a new policy.


17. MULTI-TENANT SECURITY



Tenant Isolation: Every policy/FAQ record is strictly scoped to the inherited venue_id. venue_id MUST be propagated into every persistence and RAG indexing record as immutable tenant metadata.

PostgreSQL RLS: Row-Level Security policies natively prevent cross-tenant record retrieval.

Editing Authorization: Administrative overrides require an authenticated JWT bearing claims matching the target venue_id.

Auditability: Modifications do not mutate existing records; they cap effective_until and spawn new records, maintaining an immutable cryptographic audit trail of who changed the policy and when.


18. INVALID DATA & FAILURE CASES
| Failure Case | Why Invalid | Detecting Layer | Expected Behavior | Severity |
|---|---|---|---|---|
| Allergen Info in FAQ | Violates KB-SPEC-006 boundary. | Semantic Validator / Normalizer | Strip allergen data; log security warning. | Critical |
| null for Open-Ended Dates | Violates omission nullability rule. | Structural Validator | Reject payload / Normalizer strips null. | High |
| Overlapping Conflicting Scope | Creates non-deterministic operational truth on the same dimension. | Business Logic Validator | Demote both records to CONFLICT. | Critical |
| VERIFIED without Lineage | Breaks explicit evidence principle. | Structural Validator | Reject payload / force to UNVERIFIED. | High |
| Penalty Type Mismatch | FLAT_FEE used but basis_points defined instead of minor_units. | Semantic Validator | Reject payload. | High |
| Invalid State ENUM | Unrecognized state (LIKELY_TRUE). | Structural Validator | Reject payload. | High |


19. COMPLETE EXAMPLE JSON
{
"policies": {
"records": [
{
"policy_id": "pol_9a1b2c3d",
"category": "PETS",
"operational_dimension": "pet_access",
"operational_scope": "FULL_VENUE",
"state": "VERIFIED",
"structured_value": {
"pet_access": "TERRACE_ONLY",
"service_animal_access": "ALLOWED"
},
"translations": [
{
"language": "en",
"text": "Dogs are warmly welcome on our outdoor terrace, but only service animals are permitted inside the dining room."
},
{
"language": "sv",
"text": "Hundar är varmt välkomna på vår uteservering, men endast ledarhundar är tillåtna inne i matsalen."
}
],
"effective_from": "2025-01-01T00:00:00Z",
"lineage": {
"source_type": "OFFICIAL_WEBSITE",
"source_url": "https://brasserie.se/about",
"verification_timestamp": "2026-08-10T14:30:00Z"
}
}
]
},
"faq": {
"records": [
{
"faq_id": "faq_c8d9e230",
"category": "GENERAL_FAQ",
"state": "VERIFIED",
"translations": [
{
"language": "en",
"question": "Can I bring my own birthday cake?",
"answer": "Yes, but we charge a cakeage fee of 50 SEK per person."
}
],
"effective_from": "2026-05-01T00:00:00Z",
"lineage": {
"source_type": "HUMAN_VERIFICATION",
"verified_by": "admin_user_001",
"verification_timestamp": "2026-08-11T09:15:00Z"
}
}
]
}
}


20. INVALID EXAMPLES
Example A: Allergen Claim in FAQ
{
"faq_id": "faq_invalid_01",
"category": "GENERAL_FAQ",
"state": "VERIFIED",
"translations": [
{
"language": "en",
"question": "Do you have gluten-free options?",
"answer": "Yes, our kitchen is completely gluten-free."
}
]
}



Why it fails: Assertions of FREE_FROM allergen status are strictly quarantined to KB-SPEC-006. Processing this in the FAQ domain bypasses the EU-14 safety matrix, violating architectural boundaries. (Note: A general question like "Do you have a peanut-free menu?" is permitted as a keyword query, but answering it with a definitive safety claim via unstructured FAQ text is prohibited).
Example B: Invalid Penalty Constraints
{
"policy_id": "pol_invalid_02",
"category": "CANCELLATION",
"operational_dimension": "cancellation_penalty",
"operational_scope": "FULL_VENUE",
"state": "VERIFIED",
"structured_value": {
"cancellation_deadline_hours": 24,
"penalty_type": "PERCENTAGE",
"penalty_amount_minor_units": 5000
}
}

Why it fails: The PERCENTAGE penalty type mandates the presence of penalty_percentage_basis_points and strictly forbids penalty_amount_minor_units to prevent mathematical booking-engine errors.


21. PRODUCTION ACCEPTANCE CRITERIA
| AC ID | Category | Requirement | Verification Method | Expected Result | Pass/Fail | Severity |
|---|---|---|---|---|---|---|
| AC-01 | Structural | Payloads with invalid states, explicit null for omitted fields, or unknown categories are structurally rejected. | CI Schema Test | 100% rejection rate. | Required | High |
| AC-02 | Security | Cross-tenant database/RAG queries strictly enforce venue_id isolation. | RLS Pen-Test | Zero data leakage. | Required | Critical |
| AC-03 | Temporal Logic | Queries evaluated at T_{query} exclude records where effective_until <= T_{query}. | Integration Test | Expired records ignored. | Required | Critical |
| AC-04 | Conflict | Two VERIFIED policies targeting the exact same operational_dimension and operational_scope with overlapping dates automatically demote to CONFLICT. | Business Logic Test | Status updates to CONFLICT. | Required | Critical |
| AC-05 | Inference | AI refuses to answer questions about categories lacking a VERIFIED active record, utilizing strict fallback phrases. | Prompt Evaluation | Defers to UNKNOWN phrasing. | Required | High |
| AC-06 | Identity Hash | policy_id and faq_id generation guarantees identical output for identical canonicalized inputs across the deterministic fields, regardless of array order or timestamp. | Unit Test | Identical SHA-256 hash. | Required | High |
| AC-07 | Boundary | FAQs containing explicit EU-14 allergen safety assertions (e.g., "This kitchen is peanut-free") are flagged and quarantined prior to RAG indexing. | Semantic Pipeline Test | Assertion quarantined. | Required | Critical |


22. DOWNSTREAM DEPENDENCIES



Knowledge Base Validation (KB-SPEC-009): Implements the Pydantic models enforcing temporal half-open non-overlap, exact deterministic hashing, omission nullability rules, and cancellation penalty invariants.

AI Engine & RAG (Modul 3): Consumes this schema to dynamically answer guest queries regarding operational limits, enforcing the metadata date filters.

Booking Engine (Modul 5): Reaches into the policies array to evaluate structured_value constraints (e.g., cancellation deadlines) prior to confirming reservations.


23. VERSION HISTORY
| Version | Date | Description | Author | Approval Status |
|---|---|---|---|---|
| 1.3.0 | August 2026 | Added explicit operational_scope ENUM, clarified system-derived states, defined rigorous NFKC ID canonicalization, and cemented venue_id propagation requirements. | Ramy Bella | 🔒 LOCKED / Approved for Implementation |
| 1.2.0 | August 2026 | Implemented specific operational dimensions and scopes, exact deterministic hashing canonicalization, precise percentage penalty invariants, half-open temporal intervals, and refined pet policy dimensioning. | Ramy Bella | Superseded |
| 1.1.1 | August 2026 | Patch revision: Clarified EXPIRED lifecycle semantics, restricted FAQ conflict boundaries to deterministic structured mapping, and defined explicit authority constraints for OFFICIAL_BOOKING_PLATFORM. | Ramy Bella | Superseded |
| 1.1.0 | August 2026 | Structural alignment patch. Adapted to Master JSON object format ({ "records": [] }), eliminated null date defaults, refined operational conflict logic, and clarified allergen assertion quarantine. | Ramy Bella | Superseded |
| 1.0.0 | August 2026 | Initial FAQ & Policy Schema. | Ramy Bella | Superseded |


24. FINAL NON-NEGOTIABLE PRINCIPLES



UNKNOWN IS NOT AN ANSWER: Absence of a policy record means the policy is UNKNOWN. It does not mean it is implicitly allowed or prohibited.

No Policy Inference: The AI must never invent, assume, or interpolate operational rules based on industry norms or silence.

Fail-Closed on Dimensional Conflict: Contradictory authoritative sources on the exact same operational_dimension and operational_scope immediately demote the record to CONFLICT, removing it from declarative AI context.

Allergen Quarantine: Policies and FAQs MUST NEVER make, modify, or override physiological allergen safety assertions.

Temporal Strictness: Expired policies are operationally dead. They must never be presented to a guest as current rules.

Omission Over Null: Missing facts and open-ended dates rely on key omission, never null.