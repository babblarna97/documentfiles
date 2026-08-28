KB-SPEC-003: Venue Metadata Schema

1. DOCUMENT CONTROL
| Attribute | Value |
|---|---|
| Document Title | Venue Metadata Schema |
| Document ID | KB-SPEC-003 |
| Version | 1.1.1 |
| Status | 🔒 LOCKED / Approved for Implementation |
| Author | Ramy Bella |
| Classification | Confidential / Enterprise Proprietary |
| Target Audience | Backend Engineers, Data Engineers, AI Engineers, Database Engineers, QA Engineers, Platform Architects |
| Parent Document | KB-SPEC-002 — Master JSON Schema |
| System | Restaurant AI System |
| Phase | Phase 2 — Knowledge Base Specification |
| Last Updated | August 2026 |


2. PURPOSE & SCOPE
Purpose
The Venue Metadata Schema defines the authoritative data contract for the venue domain within the Restaurant AI System's Master Knowledge Base. It establishes the structural representation, validation logic, normalization standards, provenance tracking, and security constraints governing restaurant and café identity data.
Venue identity is foundational to the multi-tenant AI platform. It provides the core context (name, business type, primary language, currency, and descriptive overview) that grounds every interaction handled by the AI chatbot. Without a deterministic and verified venue identity, downstream components cannot accurately answer basic inquiries, price items, or enforce tenant isolation.
Scope: What This Document Controls



The exact JSON contract for the venue object defined in KB-SPEC-002.

Field-level data types, constraints, omission rules, and validation standards.

Controlled vocabularies and normalization algorithms for venue types, currencies, and languages.

Rules governing identity stability, tenant binding, immutability, and change management.

Source lineage, provenance mapping, and explicit field-level verification states.

Explicit boundaries for RAG grounding and AI conversational usage.
Scope: What This Document Explicitly Does NOT Control

Menu structures, item pricing, or dietary matrices (governed by subsequent menu and allergen specifications).

Operating hours, temporal exceptions, or shift schedules (governed by operating hours specifications).

Booking rules, reservation windows, or seating capacities (governed by booking specifications).

Conversation flow logic, prompt engineering templates, or widget execution code (governed by Phase 3–8 specifications).


3. RELATIONSHIP TO KB-SPEC-002
KB-SPEC-002 establishes the top-level architectural blueprint and global object structure for the Master Knowledge Base. It declares that every valid KB document must contain a venue key containing core brand and identity properties.
This document (KB-SPEC-003) acts as the detailed domain specification for that specific key. It expands the high-level outline from KB-SPEC-002 into an exhaustive engineering contract.
In the event of any perceived ambiguity, KB-SPEC-002 remains the authoritative top-level container contract, while KB-SPEC-003 is the authoritative operational standard for the venue namespace.


4. VENUE DOMAIN ARCHITECTURE
The logical architecture of the venue domain separates legal corporate entity details from guest-facing brand presentation, operational classification, and localization settings.
"venue": {
"legal_name": "String (Optional)",
"display_name": "String (Mandatory)",
"venue_type": "ENUM (Mandatory)",
"description": "String (Optional)",
"primary_language": "ISO 639-1 String (Mandatory when verified)",
"currency": "ISO 4217 String (Mandatory when verified)"
},
"verification": {
"approval_status": "APPROVED",
"field_status": {
"primary_language": "UNKNOWN",
"legal_name": "VERIFIED_ABSENT"
}
}



Architectural Rationale

legal_name vs display_name: Separates corporate accountability (used for billing and legal disclaimers) from customer-facing branding (used in AI greetings and UI widgets).

venue_type: Establishes domain context for the AI engine without relying on unstructured text descriptions.

primary_language & currency: Dictates baseline formatting rules for monetary values and default linguistic parameters. Because these require strict ISO string formats, unverified states are handled by omitting the field from the venue block and logging the missing state in verification.field_status.

description: Supplies a verified, source-backed brand summary for context grounding, strictly quarantined from AI hallucinations.


5. FIELD-BY-FIELD DATA CONTRACT
5.1 Legal Name (legal_name)



JSON Path: venue.legal_name

Data Type: String

Requirement: Optional

Nullable: No (null is forbidden for factual data)

Allowed Values: Any valid corporate legal name string.

Length Constraints: Minimum 1 character; Maximum 150 characters.

Format Requirements: Unicode NFC normalized; trimmed of leading/trailing whitespace.

Default Behavior: If no formal corporate entity name is discovered, the key is omitted entirely from the JSON object.

Missing-Data Behavior: The verification.field_status map records "legal_name": "UNKNOWN" (if pending) or "VERIFIED_ABSENT" (if confirmed non-existent). System falls back to display_name for conversational operations.

Validation Rules: Must not contain executable scripts, HTML tags, or markdown formatting.

AI Usage Implications: Used strictly for legal compliance notices or formal invoice/billing integrations; never injected into standard chat responses unless explicitly requested.

Security Implications: Sanitized against SQL injection and XSS payloads.

Example Valid Value: "Brasserie Gabriel Aktiebolag"
5.2 Display Name (display_name)

JSON Path: venue.display_name

Data Type: String

Requirement: Required

Nullable: No

Allowed Values: Public trade or brand name of the venue.

Length Constraints: Minimum 2 characters; Maximum 100 characters.

Format Requirements: Unicode NFC normalized; title case or official brand casing preserved.

Default Behavior: None (Mandatory field; ingestion fails if absent).

Missing-Data Behavior: Ingestion pipeline halts with a structural validation error.

Validation Rules: Must match verified source metadata; cannot consist solely of generic terms (e.g., "Restaurant").

AI Usage Implications: Primary brand anchor for the AI chatbot (e.g., "Welcome to Brasserie Gabriel, how may I assist you?").

Security Implications: Sanitized against prompt injection attempts disguised as venue names.

Example Valid Value: "Brasserie Gabriel"
5.3 Venue Type (venue_type)

JSON Path: venue.venue_type

Data Type: String (ENUM)

Requirement: Required

Nullable: No

Allowed Values: See Section 6 (Controlled ENUM).

Length Constraints: Maximum 30 characters.

Format Requirements: Uppercase snake_case.

Default Behavior: None (Mandatory field).

Missing-Data Behavior: Halts ingestion; defaults to OTHER only if explicitly forced by human override or deterministically mapped by the normalizer.

Validation Rules: Must strictly match one of the predefined system ENUM strings.

AI Usage Implications: Provides domain context to downstream conversational systems without directly determining tone or personality (tone is strictly controlled by Phase 1 Personality architecture).

Security Implications: Enum constraint prevents injection through unstructured classification strings.

Example Valid Value: "RESTAURANT"
5.4 Description (description)

JSON Path: venue.description

Data Type: String

Requirement: Optional

Nullable: No

Allowed Values: Factual overview of the venue's concept, atmosphere, or culinary style.

Length Constraints: Minimum 10 characters; Maximum 500 characters.

Format Requirements: Plain text string; no unescaped control characters.

Default Behavior: Omitted if no description is extracted.

Missing-Data Behavior: verification.field_status records UNKNOWN or VERIFIED_ABSENT. RAG context builder excludes the description chunk without failing ingestion.

Validation Rules: Must be directly traceable to verified website copy or admin input; synthetic AI generation is strictly prohibited during ingestion.

AI Usage Implications: Injected into RAG context to provide background flavor text for the AI.

Example Valid Value: "Classic French bistro serving traditional dishes with a modern Nordic twist in central Stockholm."
5.5 Primary Language (primary_language)

JSON Path: venue.primary_language

Data Type: String

Requirement: Required (When verified)

Nullable: No

Allowed Values: Valid ISO 639-1 two-letter language codes (e.g., "sv", "en", "de").

Length Constraints: Exactly 2 characters.

Format Requirements: Lowercase ASCII alphabetic string.

Default Behavior: If undetermined, the field is omitted from the venue object.

Missing-Data Behavior: The verification.field_status map records "primary_language": "UNKNOWN". Ingestion auto-publishing halts, requiring manual operator configuration or explicit extraction confirmation. It never silently defaults to English or any other language.

Validation Rules: When present, must be a recognized code within the platform's supported localization registry.

AI Usage Implications: Dictates the default conversational language of the AI assistant when a guest initiates contact without explicit locale signals.
5.6 Currency (currency)

JSON Path: venue.currency

Data Type: String

Requirement: Required (When verified)

Nullable: No

Allowed Values: Valid ISO 4217 three-letter currency codes (e.g., "SEK", "EUR", "USD").

Length Constraints: Exactly 3 characters.

Format Requirements: Uppercase ASCII alphabetic string.

Default Behavior: Country/TLD may be used as a candidate signal during scraping, but the field is omitted and verification.field_status logs "currency": "UNVERIFIED" until supported by an authoritative source or explicit human verification.

Missing-Data Behavior: Halts auto-publishing; currency cannot be implicitly guessed or hardcoded.

Validation Rules: Must match ISO 4217 standard.

AI Usage Implications: Used when formatting menu prices in conversational responses.


6. VENUE TYPE ENUM
To ensure architectural uniformity across multi-tenant deployments, the system enforces a controlled vocabulary for venue_type.
Controlled ENUM Definition



RESTAURANT: Full-service dining establishment offering seated meals with dedicated table service.

CAFE: Casual establishment focusing on coffee, pastries, light lunches, and beverages.

BAR: Establishment focused primarily on alcoholic beverage service with optional light snacks.

BISTRO: Small, moderately priced European-style restaurant with a casual atmosphere.

BAKERY: Production and retail venue specializing in bread, pastries, and baked goods.

FOOD_TRUCK: Mobile culinary unit operating in semi-permanent or transient locations.

PUB: Social establishment licensed for alcoholic drinks, traditional pub fare, and social gatherings.

HOTEL_RESTAURANT: Dining facility operating within a hotel or hospitality complex.

OTHER: Fallback classification for specialized hospitality venues not covered by primary categories.
Normalization & Evaluation Rules

Case Sensitivity: Incoming strings from scrapers must be converted to uppercase and checked against the ENUM list.

Unknown Venue Types (Deterministic Rule): If an unmapped classification string is extracted (e.g., "Eatery & Lounge"), the normalization engine deterministically maps it to OTHER, appends a data quality warning log, and routes the record to the verification review queue without crashing ingestion.

Prohibition of Inference: The AI and ingestion pipelines are strictly forbidden from inferring a venue type based on visual branding, color schemes, or ambiguous menu items. Explicit textual confirmation from official source metadata is required.


7. IDENTITY & IMMUTABILITY RULES
Maintaining persistent entity identities across frequent website re-scrapes is critical for RAG cache stability and operational continuity.
Identifier Definitions



venue_id: The globally unique, immutable tenant key (e.g., se-sto-brasserie-01). Created once during initial tenant provisioning. Never changes.

record_id: A UUIDv4 representing a specific, point-in-time snapshot of the venue's ingested Knowledge Base document. Changes on every successful re-scrape or manual edit.

entity_id: A UUIDv4 assigned to sub-components within the venue record to track continuity.
Immutability Matrix
| Attribute | Mutable? | Rule / Behavior |
|---|---|---|
| venue_id | No | Immutable. Forms the absolute root of tenant isolation. |
| legal_name | Yes | May change via legal corporate restructuring; tracked via version increments. |
| display_name | Yes | May change during rebranding; does not alter venue_id or tenant scope. |
| venue_type | Yes | May change if business model evolves; requires re-verification. |
| primary_language | Yes | May be updated by admin; triggers conversational localization updates. |
| currency | Yes | Rarely changes; updating requires menu price contract re-validation. |
Re-Scrape Identity Matching
When the automated scraping pipeline executes a periodic refresh of an existing venue:

The scraper targets the registered source URL bound to the existing venue_id.

Extracted candidate data is processed by the normalization engine.

The system generates a new record_id while preserving the parent venue_id.

Changes to display_name or description are logged as schema revisions without disrupting active chat sessions or vector bindings.


8. MULTI-TENANT SECURITY
The Restaurant AI System is architected to support thousands of independent restaurant tenants on a single shared platform infrastructure.
Isolation Enforcements



Database Scoping (Row-Level Security): Every database query interacting with venue metadata must include an explicit filter condition: WHERE venue_id = :current_tenant. PostgreSQL RLS policies enforce this at the storage engine level.

API Authorization: API endpoints serving venue configuration data require cryptographically signed JSON Web Tokens (JWT) containing verified tenant claims (venue_id).

RAG Filtering: Vector store queries must inject hard metadata constraints ({"venue_id": {"$eq": "{{VENUE_ID}}"}}) into every similarity search execution. Cross-tenant retrieval is architecturally impossible.

Admin Editing: Venue metadata updates performed via the Admin Dashboard are validated against the authenticated user's permission matrix.


> CRITICAL SECURITY DIRECTIVE (TENANT AUTHORIZATION):
Client requests (such as a web widget initiating a chat) must authenticate using a pre-provisioned, backend-issued Widget Installation Identifier. The server securely maps this identifier to the verified venue_id and establishes a secure server-side session.
Origin headers (Referer / Origin) provide supplementary defense-in-depth, but are NEVER the primary proof of tenant authorization or identity.



9. LANGUAGE & LOCALE



primary_language Definition: The core linguistic operational language of the venue staff and default interface.

ISO 639-1 Standard: Values must strictly conform to two-letter ISO 639-1 codes.

Relationship to Chatbot Capabilities: The venue's primary_language dictates the default fallback language of the AI assistant. However, advanced multi-tenant configurations permit the AI conversation engine to converse in guest-preferred languages if supported by the overarching platform runtime. This document controls only the venue's baseline primary language setting.


10. CURRENCY



ISO 4217 Standard: The currency field must contain an official three-letter alphabetic code.

Relationship to Menu Pricing: The venue currency establishes the monetary unit for all nested menu items stored as integer values in price_cents (e.g., currency: "SEK", price_cents: 16500 = 165.00 SEK).

Signal vs. Authority Resolution: Country codes (e.g., .se) or website domain TLDs may be used as candidate signals during initial scraping heuristics, but the currency attribute must remain omitted/UNVERIFIED in the schema until confirmed by an authoritative source (explicit currency text symbols/ISO strings on the page) or explicit human verification. The AI is strictly prohibited from guessing currencies.


11. DESCRIPTION FIELD
The description field contains source-backed factual information regarding the venue's culinary concept or physical atmosphere.



Source Requirement: Must be extracted directly from official website copy (e.g., "About Us" sections) or entered manually by authorized venue administrators.

Prohibition of Synthetic Marketing: The ingestion pipeline and AI modules are prohibited from generating unverified marketing copy, exaggerating quality claims, or inventing awards and historical milestones.

Guest-Facing Usage: The AI may use the verified description to establish conversational context (e.g., acknowledging that the venue is a cozy Italian trattoria), but must never state unverified opinions as objective facts.

Length Limit: Capped at 500 characters to prevent prompt bloat in RAG context injection windows.


12. DATA SOURCE & PROVENANCE
In alignment with KB-SPEC-001 and KB-SPEC-002, every venue metadata field must maintain traceable lineage. Granular provenance is established by ensuring that each factual attribute or the parent venue record maintains explicit field-to-source mapping.
"lineage": {
"source_type": "SCRAPED_WEBSITE",
"source_url": "https://brasseriegabriel.se",
"extraction_timestamp": "2026-08-12T02:14:00Z",
"extractor_version": "v1.4.2",
"manual_override": false,
"field_provenance": {
"display_name": { "source": "meta_title", "confidence": 0.99 },
"currency": { "source": "footer_pricing_text", "confidence": 0.95 }
}
}



Conflict Resolution Hierarchy
When conflicting venue metadata is discovered across disparate sources (e.g., website states one display name, while a booking aggregator states another), the system applies the following precedence hierarchy:

Manual Admin Override (Highest Priority): Direct edits committed via the Admin Dashboard by an authenticated operator.

Official Scraped Website: Primary domain target registered during onboarding.

Third-Party Integrations / Aggregators (Lowest Priority): External directory listings used strictly as secondary validation hints.


13. VERIFICATION MODEL
Venue metadata records operate under a strict verification state machine to ensure unverified data never compromises production accuracy. This is tracked globally, as well as on a per-field basis inside a verification.field_status map.
Controlled Verification States



UNKNOWN: The field has not been encountered during ingestion or extracted data was completely unparsable.

UNVERIFIED: Data has been successfully extracted and normalized into the schema, but has not yet undergone human review or high-confidence automated validation.

VERIFIED: Data has passed structural validation and been explicitly confirmed by automated high-confidence heuristics or a human administrator.

VERIFIED_ABSENT: Confirmed through extraction that a specific optional field (such as legal_name) does not exist for this venue.

NOT_APPLICABLE: The field is structurally irrelevant to this venue category.


> IMPORTANT RULE: A document-level APPROVED status in the overall lifecycle model does not imply that every individual sub-field is unconditionally verified; the AI must refer to the granular field_status map before asserting a fact.



14. NULLABILITY & MISSING DATA (STRICT)
To eliminate ambiguity, the venue domain strictly forbids null values for factual data.



Omitted Key: If data is missing, the JSON key is omitted from the venue object.

Explicit State Map: The absence of the key is explicitly explained in the verification.field_status map as UNKNOWN, VERIFIED_ABSENT, or NOT_APPLICABLE.

Empty String (""): Forbidden for text fields; treated as a structural validation failure.

null: Architecturally banned within the venue identity payload to prevent silent type-coercion errors downstream.


15. NORMALIZATION RULES
The normalization engine applies deterministic transformations to raw scraped candidate values before schema validation:



String Sanitization: Leading, trailing, and redundant internal whitespace characters are collapsed into single spaces. Control characters and zero-width spaces are stripped.

Unicode Normalization: All text strings are normalized to Unicode Normalization Form C (NFC).

Casing Standards: venue_type is normalized to uppercase snake_case; language codes to lowercase ISO 639-1; currency codes to uppercase ISO 4217.

URL Normalization: Primary website URLs are sanitized while preserving the source scheme (HTTP vs. HTTPS); normalization applies safely, preferring HTTPS when available without forcing breaking conversions on legacy sites.


16. CONFLICT RESOLUTION
When ingestion encounters conflicting attribute candidates:



The normalization engine tags the conflicting candidates with source provenance identifiers.

The conflict resolution protocol evaluates candidates against the Source Precedence Hierarchy (Section 12).

If the conflict involves core identity properties (display_name or currency) with competing high-confidence sources, the record is routed to the Human Verification Queue, preventing automated publication.


17. SCRAPER → VENUE MAPPING PIPELINE
The operational lifecycle transforming raw web data into validated venue metadata follows six sequential execution gates:
[ 1. RAW HTML DOM ]
│
▼ (Extraction Engine: DOM Selectors / Meta Tags)
[ 2. CANDIDATE EXTRACT ]
│
▼ (Normalization Engine: Unicode, Casing, Trimming)
[ 3. NORMALIZED CANDIDATE ]
│
▼ (Validation Layer: Pydantic Schema & Enum Check)
[ 4. VALIDATED PAYLOAD ]
│
▼ (Provenance & Lineage Injection)
[ 5. TRACEABLE RECORD ]
│
▼ (Human / Automated Confidence Gate)
[ 6. APPROVED VENUE METADATA ]


18. AI / RAG USAGE RULES
Permitted AI Usage



Referencing the verified display_name during conversational greetings and acknowledgments.

Grounding responses in the factual description and venue_type to establish appropriate contextual background.

Formatting financial figures using the authorized currency.

Operating in the correct linguistic mode dictated by primary_language.
Strict Prohibitions (What the AI Must NEVER Infer)

The AI must never infer cuisine types, price tiers, or culinary quality claims unless explicitly verified in the structured schema.

The AI must never extrapolate ownership, brand associations, or historical awards from ambiguous website text.

The AI must never confirm operating status or real-time capacity based solely on venue metadata.


19. SECURITY & DATA INTEGRITY RULES



Injection Defense: Scraped strings destined for display_name and description are stripped of HTML tags, JavaScript payloads, and markdown instruction injections (e.g., # Instructions: Ignore safety guidelines).

Data as Data: Downstream AI prompt templates must strictly delineate venue metadata within bounded XML or JSON delimiters, ensuring LLMs treat the metadata purely as inert data rather than executable directives.

Access Control: Tenant isolation rules ensure no administrative user or API client can read or mutate venue metadata belonging to another venue_id.


20. VERSIONING & CHANGE MANAGEMENT



Minor Updates: Changes to display_name, description, or contact attributes increment the internal record revision (record_id) without altering the tenant's venue_id.

Major Breaking Changes: Structural modifications to the venue schema namespace require a major schema version increment (1.x to 2.0).

RAG Re-Indexing: Any update to display_name or description automatically triggers an incremental re-vectorization event to update cached embedding chunks in the search index.


21. VALIDATION RULES (PYDANTIC SPECIFICATION BASIS)
Engineering teams must implement validation models enforcing the following rules:



Structural: venue object must be present; mandatory fields (display_name, venue_type) must be present and non-empty. Keys omitting values must be absent from the dictionary entirely.

Semantic: If present, primary_language must match ISO 639-1 regex ^[a-z]{2}$; currency must match ISO 4217 regex ^[A-Z]{3}$.

Business Logic: venue_type must be an exact match to the controlled ENUM vocabulary.

Security: String lengths must adhere to strict bounds; content must be free of unauthorized script signatures.


22. INVALID DATA / FAILURE CASES
| Failure Case | Why Invalid | Detecting Layer | Expected Behavior | Severity |
|---|---|---|---|---|
| Missing display_name | Mandatory identity anchor is absent. | Structural Validator | Reject payload instantly; log validation error. | Critical |
| Unsupported venue_type ("SUSHI_BAR") | Violates controlled ENUM vocabulary. | Semantic Validator | Normalize to OTHER, log warning, route to review queue. | High |
| null assigned to description | Banned nullability semantics. | Schema Validator | Reject payload; require omitting the key and logging UNKNOWN. | High |
| Invalid Currency Code ("kr") | Violates ISO 4217 standard format. | Schema Validator | Reject payload; require ISO 3-letter code. | High |
| Script Injection in Description | Contains malicious JavaScript payload. | Security Sanitizer | Strip payload or reject; log security incident. | Critical |
| Origin Header Spoofing | Origin bypasses Widget ID verification. | API Authorization Layer | Terminate request with HTTP 403 Forbidden. | Critical |


23. EXAMPLE VALID VENUE OBJECT
{
"venue": {
"display_name": "Källaren Gastropub",
"venue_type": "PUB",
"description": "Traditional basement gastropub in central Uppsala offering local craft beers and classic pub fare.",
"primary_language": "sv",
"currency": "SEK"
},
"verification": {
"approval_status": "APPROVED",
"field_status": {
"legal_name": "VERIFIED_ABSENT"
}
},
"lineage": {
"source_type": "SCRAPED_WEBSITE",
"source_url": "https://kallaren.se",
"extraction_timestamp": "2026-08-12T02:14:00Z"
}
}



(Notice how legal_name is correctly omitted from the venue object and logged as VERIFIED_ABSENT in the verification map).
24. EXAMPLE INVALID VENUE OBJECTS
Example A: Forbidden Null Usage & Invalid ISO Formats
{
"venue": {
"legal_name": null,
"display_name": "Bad Data Cafe",
"venue_type": "HIPSTER_COFFEE_SHOP",
"primary_language": "english",
"currency": "swedish_krona"
}
}

Failure Reason: legal_name contains forbidden null; venue_type is not in the controlled ENUM; primary_language is not a 2-letter ISO code; currency is not a 3-letter ISO code. All trigger immediate rejection.


25. PRODUCTION ACCEPTANCE CRITERIA
| Acceptance ID | Category | Requirement | Verification Method | Expected Result | Pass/Fail | Severity |
|---|---|---|---|---|---|---|
| AC-01 | Structural | Every valid venue record forbids null values; missing data is handled via key omission + verification mapping. | Automated CI Schema Test | 100% compliance; zero null coercion errors. | Required | Critical |
| AC-02 | Identity | venue_id remains immutable across multiple re-scrape cycles and display name modifications. | Integration Test Suite | venue_id hash matches across updates. | Required | Critical |
| AC-03 | Security | Cross-tenant database and vector retrieval queries strictly enforce venue_id scoping via server-side session, not just Origin headers. | Penetration & RLS Audit | Zero cross-tenant data leakage under load. | Required | Critical |
| AC-04 | Normalization | Incoming raw scraping candidate strings are deterministically normalized to Unicode NFC, trimmed, and cased correctly. | Unit Test Suite | Output matches expected normalized string. | Required | High |
| AC-05 | Vocabulary | venue_type strictly enforces the 9-item controlled ENUM; unmapped types default to OTHER with a warning log. | Enum Validation Test | Invalid types normalized to OTHER safely. | Required | High |
| AC-06 | Localization | When present, primary_language validates against ISO 639-1; currency validates against ISO 4217. | Regex Validation Test | Invalid codes blocked at normalization layer. | Required | High |
| AC-07 | Security | Malicious HTML, scripts, and prompt injection attempts in descriptions are sanitized or blocked. | Security Test Suite | Sanitized plain text output; no execution. | Required | Critical |
| AC-08 | Provenance | Every approved venue record maintains traceable lineage (source_url, timestamp, field-level mapping). | Lineage Inspection Script | 100% of active records possess valid lineage. | Required | High |
| AC-09 | AI Grounding | The AI chatbot utilizes verified venue metadata without hallucinating unverified attributes. | Conversational Evaluation Suite | Zero unverified venue claims in test dialogues. | Required | Critical |


26. RELATIONSHIP TO FUTURE DOCUMENTS



Scraping Pipeline (Modul 1): Utilizes this specification to structure extracted web candidate data into the normalized venue JSON object.

Knowledge Base Validation (KB-SPEC-009): Implements the Pydantic structural and semantic validation rules defined herein.

AI Engine & RAG (Modul 3): Consumes approved venue metadata to ground prompt context and operational behavior.

Admin Dashboard (Modul 7): Renders venue metadata fields for authorized operators to review, edit, and verify.


27. VERSION HISTORY
| Version | Date | Description | Author | Approval Status |
|---|---|---|---|---|
| 1.1.1 | August 2026 | Micro-revision (Locked). Solved nullability conflict, finalized field-level UNKNOWN representation, moved origin header to secondary defense, decoupled venue_type from AI tone. | Ramy Bella | 🔒 LOCKED / Approved for Implementation |


28. FINAL NON-NEGOTIABLE PRINCIPLES



Venue Identity Must Be Deterministic: Vague, ambiguous, or polymorphic identity structures are strictly prohibited.

venue_id is Immutable: Tenant identity boundaries can never be altered via routine updates or rebranding.

Missing Data is Never Interpreted as False: Absence of optional attributes requires explicit state tracking, never silent assumption.

Unverified Data Cannot Silently Become Verified: Automated extractions must pass validation and verification gates before entering operational status.

Scraped Text is Data, Not Instructions: Downstream systems must treat ingested venue strings strictly as inert data to prevent prompt injection.

Tenant Isolation is Absolute: Server-side sessions and mandatory metadata filtering guarantee complete data segregation; Origin headers are supplementary.

AI Responses Must Remain Grounded: The AI chatbot is restricted to verified venue metadata and prohibited from inventing brand characteristics.

No Field May Bypass Validation: Every record, whether scraped or manually entered, must clear the exact same normalization and validation pipeline.

No Downstream System May Invent Venue Metadata: All operational components must consume the Master Knowledge Base as the single source of truth.