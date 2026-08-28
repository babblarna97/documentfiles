KB-SPEC-001: Knowledge Base Specification
Document Control
Attribute
Value
Document Title
Knowledge Base Specification
Document ID
KB-SPEC-001
Version
1.0.0
Status
Approved for Implementation
Author
Ramy Bella
Classification
Confidential / Enterprise Proprietary
Target Audience
Software Engineers, AI Architects, Data Engineers, QA, Product Managers
Last Updated
August 2026

Executive Summary & Strategic Intent
This specification defines the complete data architecture, structural contracts, lifecycle state machine, quality gates, and retrieval mechanics for the Master Knowledge Base (KB) powering the Restaurant AI System.
In a safety-critical hospitality environment where incorrect allergen information can lead to fatal outcomes and erroneous business policies degrade brand reputation, the Knowledge Base operates as the absolute Single Source of Truth (SSOT). No generative component within the Restaurant AI System is permitted to synthesize operational facts, menu items, pricing, or safety policies out of latent model knowledge or probabilistic assumptions.
1. Purpose & Strategic Intent
Purpose
The Knowledge Base provides a deterministic, structured, and auditable data foundation for the Restaurant AI System. It bridges the gap between raw, unstructured website content and the probabilistic context generation required by Large Language Models (LLMs).
The Single Source of Truth Guarantee
Every AI response generated across all guest-facing touchpoints (Chat Widget, Meta Channels, Web Interfaces) must be directly traceable to a verified entity stored within the Knowledge Base.



[ Unstructured Web Source ]
            │
            ▼
┌───────────────────────┐
│ Ingestion & Scraper   │
└───────────┬───────────┘
            │
            ▼
┌───────────────────────┐
│ Normalization Engine  │
└───────────┬───────────┘
            │
            ▼
┌───────────────────────┐
│ Schema Validation     │
└───────────┬───────────┘
            │
            ▼
┌───────────────────────┐
│ Human Verification    │ ───► [ REJECT / REVISE ]
└───────────┬───────────┘
            │ (Approved)
            ▼
┌────────────────────────────────────────────────────────┐
│             KNOWLEDGE BASE (SSOT ENGINE)               │
│  - Venue Metadata          - Menu & Price Matrix       │
│  - Temporal Exceptions     - Strict Allergen Grid      │
│  - Operational Policies    - Capacity & Booking Rules  │
└───────────┬────────────────────────────────────────────┘
            │
            ▼
┌───────────────────────┐
│ Hybrid Vector Index   │
└───────────┬───────────┘
            │
            ▼
┌───────────────────────┐
│ Grounded Retrieval    │ ───► Zero-Hallucination AI Context
└───────────────────────┘


Strategic Objectives
Safety Isolation: Prevents model hallucinations concerning allergens, legal disclaimers, and dietary restrictions by enforcing strict context boundaries.
Consistency Across Channels: Guarantees that guests receive identical operational answers whether interacting via web chat, SMS, or embedded widgets.
Rapid Multi-Tenant Onboarding: Enables seamless ingestion of new hospitality venues within minutes through standardized data normalization contracts.
Auditability & Compliance: Provides full data lineage, showing exactly when a fact was scraped, normalized, validated, human-approved, and served.
2. Scope & Domain Boundaries
In-Scope Data Domains
The Knowledge Base strictly governs and stores data within the following operational domains:
Venue Identity & Metadata: Legal entity name, trade names, physical address, geo-coordinates, canonical contact info, social links, brand policies.
Operating & Kitchen Hours: Regular weekly schedules, kitchen last-order cutoffs, bar hours, shift patterns, and temporal holiday exceptions.
Menu Architecture: Categories, sub-categories, dish items, item variants, options, modifiers, pricing tiers, currency units, and availability flags.
Allergen & Dietary Safety Matrix: Mandatory 14 EU major allergen classifications, cross-contamination warnings, dietary suitability tags (e.g., Vegan, Halal, Kosher, Gluten-Free).
Booking & Capacity Rules: Maximum party sizes online, seating duration caps, reservation windows, cancellation terms, deposit requirements.
Knowledge Snippets & FAQs: Curated answers for parking, public transport, dress codes, pet policies, child facilities, private event inquiries.
Operational & Legal Policies: Terms of service, privacy disclaimers, service charge disclosures, payment methods accepted.
Temporary Notices: Short-lived operational overrides (e.g., unexpected closures, private buyouts, temporary kitchen outages).
Explicit Out-of-Scope Domains
The Knowledge Base must NEVER store or process:
Guest Personally Identifiable Information (PII): Guest names, phone numbers, payment details, or reservation histories (handled isolated in transient memory or secure booking endpoints).
Internal Payroll & HR Data: Staff wages, work schedules, internal operational costs.
Real-Time Dynamic POS Inventory: Per-second stock tracking (e.g., "3 steaks left in kitchen").
Probabilistic / Unverified Guarantees: Unsubstantiated claims about ingredients not explicitly confirmed by venue management.
3. Core Architectural Philosophy
The Knowledge Base is governed by ten enterprise architectural tenets:
Single Source of Truth (SSOT): No auxiliary database or external service may override a verified record inside the Knowledge Base during runtime retrieval.
Structured Over Unstructured Data: Raw unstructured text must be normalized into strict, typed schema entities before becoming accessible to downstream vectorization or retrieval layers.
Verified Facts Before AI Generation: Deterministic facts precede probabilistic generation. The AI Engine interprets facts; it never creates them.
Human-Approved Information Gate: No auto-scraped entity enters production status (KB_STATUS = ACTIVE) without passing an automated schema validation check AND an explicit human approval or automated high-confidence verification workflow.
Retrieval Before Reasoning: System prompts must consume extracted, highly relevant Knowledge Base chunks rather than relying on general model knowledge.
Immutable Source Records: Historic versions of the Knowledge Base are archived immutably to allow complete historical audit trails of past customer interactions.
Version Controlled Information: Every alteration to a menu item, opening hour, or policy increments the domain schema version and triggers re-vectorization.
Quality Before Quantity: A sparse, 100% accurate Knowledge Base is infinitely superior to a dense, partially hallucinated or conflicting dataset.
Consistent Schemas: Every venue, regardless of size or style, conforms to the exact same canonical entity contracts.
Extensibility Without Breaking Changes: New data domains must be introduced via backward-compatible schema extensions.
4. End-to-End System Architecture
The Knowledge Base operates within a pipeline architecture comprising seven distinct execution layers:



┌────────────────────────────────────────────────────────────────────────┐
│ LAYER 1: DATA INGESTION & EXTRACTION ENGINE                            │
│ Scrapes DOM, parses PDFs, processes manual admin UI inputs.            │
└───────────────────────────────────┬────────────────────────────────────┘
                                    │ Raw Extract
                                    ▼
┌────────────────────────────────────────────────────────────────────────┐
│ LAYER 2: NORMALIZATION & CLEANING ENGINE                               │
│ Converts raw strings into typed structures, standardizes currencies.    │
└───────────────────────────────────┬────────────────────────────────────┘
                                    │ Normalized Entities
                                    ▼
┌────────────────────────────────────────────────────────────────────────┐
│ LAYER 3: VALIDATION & CONTRACT ENFORCEMENT ENGINE                       │
│ Enforces Pydantic/JSON schemas, cross-checks logical rules.             │
└───────────────────────────────────┬────────────────────────────────────┘
                                    │ Validated Payload
                                    ▼
┌────────────────────────────────────────────────────────────────────────┐
│ LAYER 4: HUMAN APPROVAL & AUDIT GATE                                   │
│ Admin review, status mutation to APPROVED/ACTIVE.                      │
└───────────────────────────────────┬────────────────────────────────────┘
                                    │ Approved Master Record
                                    ▼
┌────────────────────────────────────────────────────────────────────────┐
│ LAYER 5: PERSISTENCE & VERSIONING ENGINE                               │
│ Commits immutable master records to PostgreSQL, updates Redis cache.   │
└───────────────────────────────────┬────────────────────────────────────┘
                                    │ Master Entities
                                    ▼
┌────────────────────────────────────────────────────────────────────────┐
│ LAYER 6: VECTORIZATION & HYBRID INDEXING PIPELINE                      │
│ Generates dense vector embeddings, constructs sparse keyword indices.  │
└───────────────────────────────────┬────────────────────────────────────┘
                                    │ Vector + Keyword Index
                                    ▼
┌────────────────────────────────────────────────────────────────────────┐
│ LAYER 7: GROUNDED RETRIEVAL & CONTEXT ASSEMBLY LAYER                   │
│ Hybrid search (dense + sparse), metadata filtering, context formatting. │
└────────────────────────────────────────────────────────────────────────┘


5. Knowledge Base Lifecycle & State Engine
Every Knowledge Base record traverses an explicit, deterministic finite-state machine (FSM).



   [ UNPROCESSED ]
          │
          ▼ (Scrape Complete)
     [ EXTRACTED ]
          │
          ▼ (Normalization Pass)
    [ NORMALIZED ]
          │
          ▼ (Schema Validation Pass)
     [ VALIDATED ] ───► (Validation Fails) ───► [ REJECTED ]
          │                                           │
          ▼ (Human / Confidence Gate)                 │ (Manual Fix)
      [ PENDING ] ────────────────────────────────────┘
          │
          ▼ (Human Approval)
      [ APPROVED ]
          │
          ▼ (Embedding Pipeline Complete)
       [ ACTIVE ]
          │
          ├─────────────────────────┐
          ▼ (Schema Update / Edit)  ▼ (Deactivation / Deprecation)
     [ STALE ]                  [ ARCHIVED ]


Lifecycle State Definitions
UNPROCESSED: Raw source URL identified; no extraction initiated.
EXTRACTED: Raw DOM nodes, HTML strings, or PDF documents extracted into raw staging storage.
NORMALIZED: Unstructured raw strings cleaned, stripped of control characters, and mapped into entity properties.
VALIDATED: Data payload verified against all structural schema rules and business constraints.
REJECTED: Payload failed validation or was explicitly turned down during human review. Requires remediation.
PENDING: Data passed automated validation; awaiting human operator review or automated auto-approval threshold.
APPROVED: Human operator or high-confidence rule verified data accuracy.
ACTIVE: Vector embeddings generated, hybrid index populated, live for production runtime context assembly.
STALE: Data timestamp exceeded freshness thresholds (e.g., >30 days without re-verification). Triggers re-audit alert.
ARCHIVED: Historical record retained for legal audit, replaced by newer version increment.
6. Source System Governance
Source Data Classification
Data ingested into the Knowledge Base originates from three distinct source classifications, each carrying a different initial trust rating:
Source Type
Examples
Trust Score
Processing Flow
Direct Venue Admin Input
Manual edits in Admin Dashboard
1.0 (Absolute)
Direct to VALIDATED -> APPROVED
Official Digital Presence
Scraped Venue Website, Managed API
0.85 (High)
Scrape -> Normalize -> Validate -> PENDING
Third-Party Aggregators
External Directories, Unofficial APIs
0.50 (Low)
Requires mandatory human sign-off

Conflict Resolution Hierarchy
When conflicting facts are detected across sources (e.g., website states opening at 17:00, but a PDF menu says 18:00), the ingestion engine applies the following deterministic hierarchy:
Manual Admin Override (Highest Priority)
Official Structured Venue Website Data
Scraped PDF / Document Asset
Third-Party External Source (Lowest Priority)
7. Ingestion & Scraping Architecture
Ingestion Contract
The Ingestion Engine must treat every web scrape target as untrusted, highly dynamic external data.
Processing Requirements
DOM Deconstruction: Scrapers must extract raw textual content while preserving semantic structural hierarchies (<h1>, <h2>, <table>, <dl>, <ul>).
Noise Elimination: Script tags, tracking pixels, cookie consent banners, site navigation footers, and non-operational inline markup must be stripped prior to normalization.
Asset Parsing: PDF menus must be ingested via layout-aware OCR engines to retain column relationships, dish-price pairings, and subtext allergen notes.
Rate Limiting & Polling Governance: Scrapers must respect robots.txt and employ adaptive polling to avoid imposing denial-of-service conditions on venue web hosts.
8. Cleaning & Normalization Engine
The Normalization Engine converts messy, real-world scraped text strings into uniform canonical types.
String Sanitization Standard
Unicode normalization (NFC standard).
Removal of zero-width spaces, unprintable control characters, and redundant whitespace.
HTML entity decoding (e.g., converting & to &, € to €).
Type Conversions
Currencies: All price representations must be converted into ISO 4217 code (SEK, EUR, USD) plus integer amounts in the smallest currency unit (e.g., 16500 cents/öre for 165.00).
Timestamps & Schedules: All times standardized to ISO 8601 strings in local venue time zones (e.g., 17:00:00+02:00).
Booleans: Textual indicators (e.g., "yes", "ja", "allowed", "welcome") mapped strictly to binary true/false.
Telephone Numbers: Standardized to E.164 international format (e.g., +4681234567).
9. Validation Framework & Contracts
No record is permitted to persist into the Master Knowledge Base without passing four distinct validation boundaries.



┌────────────────────────────────────────────────────────────────────────┐
│ BOUNDARY 1: STRUCTURAL SCHEMA VALIDATION                               │
│ Checks type signatures, required fields, nullability, array bounds.    │
└───────────────────────────────────┬────────────────────────────────────┘
                                    │ Pass
                                    ▼
┌────────────────────────────────────────────────────────────────────────┐
│ BOUNDARY 2: BUSINESS LOGIC & TIME VALIDATION                           │
│ Validates opening < closing times, price > 0, party size min < max.    │
└───────────────────────────────────┬────────────────────────────────────┘
                                    │ Pass
                                    ▼
┌────────────────────────────────────────────────────────────────────────┐
│ BOUNDARY 3: SAFETY & ALLERGEN DISCLOSURE VALIDATION                    │
│ Enforces mandatory allergen presence flags per dish.                   │
└───────────────────────────────────┬────────────────────────────────────┘
                                    │ Pass
                                    ▼
┌────────────────────────────────────────────────────────────────────────┐
│ BOUNDARY 4: CROSS-DOMAIN CONSISTENCY VALIDATION                        │
│ Verifies kitchen closing time <= venue closing time.                   │
└────────────────────────────────────────────────────────────────────────┘


10. Human Approval & Review Workflow
The Human Approval workflow protects the live production environment from automated ingestion errors.
Review Triggers
A Knowledge Base record is routed to the Human Review Queue when:
Automated data confidence score ({{DATA_CONFIDENCE}}) falls below 0.90.
Major allergen changes or dish additions are detected on a menu.
Operating hours change by more than 2 hours compared to the previous version.
Structural validation succeeds, but semantic ambiguity is flagged by the parser.
Operator Actions
Within the Admin UI, authorized human operators may execute four actions:
Approve: Mutates {{APPROVAL_STATUS}} to APPROVED, releasing the record to vectorization.
Edit & Approve: Manually corrects values directly in the normalized editor and approves.
Reject: Rejects record, returning it to REJECTED status with an attached reason log.
Re-Scrape: Triggers an immediate re-extraction pipeline with heightened selector verbosity.
11. Version Control & Lineage Architecture
To guarantee full enterprise auditability, the Knowledge Base implements an immutable append-only versioning scheme.
Semantic Schema Versioning
All KB domain models carry a semantic version string (X.Y.Z):
Major (X): Structural schema breaking changes (e.g., domain field deprecation).
Minor (Y): Non-breaking schema additions (e.g., adding a new dietary tag).
Patch (Z): Data content updates (e.g., updating dish price from 165 to 175).
Provenance Tracking
Every entity record maintains mandatory audit metadata:
Source URI or ingestion vector identifier.
Ingestion timestamp ({{SCRAPE_TIMESTAMP}}).
Operator ID of approving authority ({{APPROVED_BY}}).
Cryptographic hash (SHA-256) of raw ingested source data to detect upstream changes.
12. Metadata Management System
Venue Metadata establishes the operational identity of the venue.
Required Properties
Unique system venue key ({{VENUE_ID}}).
Legal name and display branding.
Physical street address, city, postal code, country code, and GPS coordinates.
Primary language code (ISO 639-1) and secondary supported languages.
Canonical phone, email, and web domain URLs.
Operational classifications (e.g., Fine Dining, Casual Café, Bar).
13. Menu Catalog Structure & Taxonomy
Menus are represented as hierarchical, typed structures.



MENU CATALOG
  └── CATEGORY (e.g., "Starters / Förrätter")
        ├── Display Order
        ├── Availability Window
        └── ITEM (e.g., "Steak Tartare / Råbiff")
              ├── Description
              ├── Base Price & Currency
              ├── Variant Groups (e.g., "Half / Full Portion")
              ├── Addons / Modifiers
              ├── Dietary Flags (e.g., Vegan, Gluten-Free)
              └── Allergen Explicit Matrix


Constraints & Rules
Every dish must belong to at least one parent category.
Prices must be explicitly attached to items or variants; category-level ambiguous pricing is forbidden.
Off-menu items or seasonal specials must carry explicit start and end temporal dates.
14. Allergen & Dietary Safety Matrix
The Allergen & Dietary Safety Matrix is the most safety-critical component of the Knowledge Base specification, directly enforcing MASD-DOC-007.
The 14 EU Major Allergens Standards
The Knowledge Base explicitly models the 14 major food allergens mandated by European Regulation (EU) No 1169/2011:
Cereals containing gluten
Crustaceans
Eggs
Fish
Peanuts
Soybeans
Milk / Lactose
Nuts (Almond, Hazelnut, Walnut, Cashew, etc.)
Celery
Mustard
Sesame seeds
Sulphur dioxide / Sulphites
Lupin
Molluscs
Allergen Verification States
For every item on a menu, each of the 14 allergens must be assigned an explicit status:
CONTAINS: Allergen is explicitly present as an ingredient.
MAY_CONTAIN: Risk of cross-contamination flagged by kitchen or supplier.
FREE_FROM: Explicitly confirmed by venue management to be absent and safe.
UNVERIFIED: Presence or absence has not been verified by human staff.
CRITICAL RULE: If an allergen status is UNVERIFIED, the system must treat the allergen as unknown and trigger a safety deferral to human staff. The AI Engine is strictly forbidden from guessing.
15. Operating Hours & Temporal Exception Engine
Operating hours require complex temporal reasoning engines due to night shifts, holiday shifts, and kitchen Last Calls.
Canonical Weekly Schedule
A standard schedule defines Monday through Sunday ranges with explicit opening, kitchen closing, and venue closing times.
Midnight Boundary Rules
For venues operating past midnight (e.g., opening Friday 18:00, closing Saturday 02:00):
The session must be tied to the operational business day (Friday).
Time ranges spanning past 00:00 must be marked with explicit day-overflow indicators to prevent mathematical comparison errors in retrieval logic.
Holiday & Exception Overrides
Temporary exceptions (e.g., Christmas Eve closure, Midsummer special hours) override standard weekly schedules for specific calendar dates (YYYY-MM-DD).
16. Knowledge Snippets & FAQ Architecture
Unstructured operational information (parking, dress code, child seats) is organized as deterministic Knowledge Snippets.
Structural Requirements
Question / Intent Vector: Canonical representation of user query intent.
Direct Answer Payload: Concise, verified text string suitable for direct injection into context.
Domain Tagging: Categorized under PARKING, ACCESSIBILITY, PETS, CHILDREN, PAYMENTS, or DRESS_CODE.
Validity Period: Optional expiration date for temporary answers.
17. Booking Policies & Capacity Logic
To allow the AI Engine to manage table bookings accurately, operational parameters must be stored deterministically in the KB.
Policy Parameters
Minimum and maximum online party size caps (e.g., min 1, max 8 guests).
Standard table allocation duration (e.g., 120 minutes per booking).
Reservation window cutoffs (e.g., bookings close 30 minutes prior to shift start).
Deposit policies for large groups (e.g., mandatory deposit for parties over 6).
Cancellation window terms (e.g., free cancellation up to 24 hours prior).
18. Operational Policies & Legal Boundaries
Operational policies protect the business legally and set clear expectations for guests.
Policy Domains
Payment Methods: List of accepted credit cards, digital wallets, cash policies (e.g., "Cashless Venue").
Service Charge & Tipping: Disclosures regarding included service charges or gratuity policies.
Child & Stroller Policy: Space allocations for strollers, high chair availability.
Pet Policy: Dogs permitted indoors, outdoor terrace only, or assistance animals only.
19. Temporary Notices & Override Protocol
When unexpected events occur (e.g., water pipe burst, terrace closed due to storm), the venue requires an immediate mechanism to inform guests.
Priority Level Definitions
INFO: Minor announcement (e.g., "New summer menu live today").
WARNING: Partial service impact (e.g., "Terrace seating closed today due to rain").
CRITICAL: Complete operational shutdown (e.g., "Venue closed today for private event").
Override Execution Logic
Active CRITICAL temporary notices immediately supersede standard operating hours and booking policies across the entire system.
20. Enterprise Runtime Variable System
The Knowledge Base management system relies on a standardized, typed variable architecture. These variables are populated during ingestion, validation, vectorization, and runtime retrieval:
Variable Name
Type
Description
Example Value
{{VENUE_ID}}
String
Unique venue system key
se-sto-brasserie-01
{{SCHEMA_VERSION}}
String
Semantic version of JSON schema
1.0.0
{{DATA_SOURCE}}
Enum
Origin classification of record
SCRAPED_WEBSITE
{{LAST_VERIFIED}}
ISO8601
Timestamp of last human verification
2026-08-01T10:00:00Z
{{VALIDATION_STATUS}}
Enum
Outcome of structural/business validation
PASSED
{{MENU_VERSION}}
String
Current active menu iteration key
v2026.8.1
{{OPENING_HOURS_VERSION}}
String
Current active schedule version
v2026.1
{{FAQ_VERSION}}
String
Current FAQ content iteration
v1.4
{{SCRAPE_TIMESTAMP}}
ISO8601
UTC execution time of scraper run
2026-08-02T04:12:00Z
{{DATA_CONFIDENCE}}
Float
Confidence score of auto-parser (0.00-1.00)
0.96
{{EMBEDDING_STATUS}}
Enum
Status of vector indexing process
COMPLETED
{{VECTOR_ID}}
UUID
Primary reference key in vector store
f81d4fae-7dec-11d0-a765-00a0c91e6bf6
{{LANGUAGE}}
ISO639-1
Primary language code of record content
sv
{{APPROVAL_STATUS}}
Enum
Human approval state
APPROVED
{{KB_STATUS}}
Enum
Master operational state of KB record
ACTIVE

21. Vectorization & Embedding Strategy
To support high-precision retrieval during AI context generation, Knowledge Base entities are embedded into dense vector space.
Chunking Principles
Semantic Unit Chunking: Documents must never be chunked by arbitrary token lengths. Individual menu items, FAQ snippets, or policy statements form discrete atomic chunk boundaries.
Context Inflation: Every vector chunk carries explicit parent metadata pre-pended to text before embedding generation (e.g., "Venue: Brasserie Gabriel | Category: Starters | Dish: Steak Tartare...").
Embedding Models & Vector Dimensions
Primary Embedding Model: Dense vector representations with minimum 1536 dimensions.
Metric Space: Cosine similarity or Inner Product distance matching vector store configuration.
22. Retrieval Architecture & Hybrid Search
Runtime context assembly relies on a Hybrid Search Paradigm combining dense semantic similarity with sparse keyword matching (BM25) and strict metadata pre-filtering.



Guest User Query: "Do you serve gluten-free pasta on Friday nights?"
                                 │
                                 ▼
┌────────────────────────────────────────────────────────────────────────┐
│ METADATA PRE-FILTERING LAYER                                           │
│ Filters vector index strictly by {{VENUE_ID}} == 'se-sto-brasserie-01'│
│ and {{KB_STATUS}} == 'ACTIVE'                                          │
└───────────────────────────────────┬────────────────────────────────────┘
                                    │ Filtered Space
                                    ▼
┌────────────────────────────────────────────────────────────────────────┐
│ HYBRID SEARCH EXECUTION                                                │
│ ├── Dense Vector Search (Cosine Similarity for "gluten-free pasta")   │
│ └── Sparse BM25 Keyword Search (Exact matches for "gluten", "pasta")  │
└───────────────────────────────────┬────────────────────────────────────┘
                                    │ Score Normalization
                                    ▼
┌────────────────────────────────────────────────────────────────────────┐
│ RE-RANKING & CONFIDENCE THRESHOLD GATE                                 │
│ Relevance Score >= 0.82?                                               │
│  ├── YES: Inject into System Context                                  │
│  └── NO:  Drop Chunk / Trigger Fallback Protocol                       │
└────────────────────────────────────────────────────────────────────────┘


Zero-Hallucination Threshold Gate
If the hybrid retrieval score for a query falls below the minimum confidence threshold (0.82), the retrieval engine returns an empty context payload. This triggers the AI Engine's safety fallback ("I do not have verified information regarding that query...").
23. Data Quality & Integrity Framework
The Knowledge Base enforces strict enterprise quality metrics across ten dimensions:
Quality Metric
Definition
Threshold / Target
Completeness
% of required non-null fields populated
100%
Schema Compliance
% of entities passing Pydantic validation
100%
Allergen Rigor
% of dishes with explicitly verified allergens
100% (No unverified in ACTIVE state)
Freshness
Maximum days since last verification pass
<= 30 Days
Duplicate Ratio
Presence of identical menu dish names
0%
Temporal Coherence
Opening time earlier than closing time
100% Logical Pass
E.164 Compliance
Canonical phone format match rate
100%
ISO 4217 Match
Valid currency code match rate
100%
Vector Alignment
Index count matches active DB entity count
100% Parity
Audit Traceability
% of records with valid origin provenance
100%

24. Security, Scoping & Tenant Isolation
In a multi-tenant enterprise architecture serving hundreds of independent hospitality venues, strict cross-tenant isolation is enforced at every layer of the Knowledge Base.
Scoping Rules
Database Level: Row-Level Security (RLS) policies in PostgreSQL enforce mandatory filtering on {{VENUE_ID}} for every read/write execution.
Vector Store Level: Vector queries must include hard metadata match clauses (venue_id = X). Cross-venue vector queries are forbidden at the infrastructure level.
Cache Key Namespacing: Redis keys must follow explicit tenant scoping: kb:{venue_id}:{domain}:{entity_id}.
25. Scalability & Storage Optimization
To handle thousands of queries per second across large venue networks, storage is optimized across three performance tiers:
Hot Storage Tier (In-Memory Redis Cache): Stores active venue metadata, current day opening hours, and active temporary notices. Response latency: < 5ms.
Warm Storage Tier (PostgreSQL + pgvector): Stores full normalized JSON records, menu catalogs, and vector embeddings. Query latency: < 50ms.
Cold Storage Tier (Object Storage Archive): Holds historical KB version dumps, raw scraped HTML DOM snapshots, and audit log files.
26. Health Monitoring & Observability
The Knowledge Base system emits real-time telemetry metrics to track data health:
Core Observability Metrics
kb_ingestion_success_rate: Ratio of successful vs failed scrape-to-validation runs.
kb_validation_failure_count: Counter tracking schema validation rejections categorized by field and error type.
kb_stale_record_count: Gauge tracking entities exceeding the 30-day re-verification threshold.
kb_retrieval_miss_rate: Frequency of user queries returning zero context chunks from the vector store.
27. Maintenance, Archival & Purging Protocol
Automated Cron Audits
Nightly Freshness Sweep: Scans all active KB records. Sets {{KB_STATUS}} = STALE for records exceeding 30 days without review.
Weekly Integrity Check: Re-runs vector alignment routines to ensure vector index records match PostgreSQL primary records 1:1.
Data Retention & Archival
Active venue data remains in warm storage indefinitely.
Superceded KB versions are archived to cold storage after 90 days of inactivity.
Completely deleted venues have their data hard-purged after a 30-day soft-delete grace period to ensure compliance with privacy regulations.
28. System Dependencies & Downstream Interfaces
The Knowledge Base interacts directly with upstream ingestion tools and downstream execution modules:



UPSTREAM MODULES                                     DOWNSTREAM CONSUMERS
┌────────────────────┐                             ┌────────────────────┐
│ Module 1: Scraper  │ ─── Raw Data Payload ─────► │                    │
└────────────────────┘                             │                    │
┌────────────────────┐                             │                    │
│ Module 7: Admin UI │ ─── Manual Inputs ────────► │   KNOWLEDGE BASE   │
└────────────────────┘                             │   (KB-SPEC-001)    │
                                                   │                    │
                                                   │ ── Structured ───► │ Module 3: AI Engine
                                                   │    Context Chunks  │ (RAG Prompt Construction)
                                                   └────────────────────┘


29. Relationship to MASD Documents 001–010
This specification implements data architecture requirements established in Phase 1:
Inherits Safety Directives from MASD-DOC-007 (Safety & Fallback): Enforces strict, zero-hallucination allergen constraints and mandatory human deferral logic.
Inherits Data Scoping from MASD-DOC-002 (System Architecture): Enforces multi-tenant venue_id isolation across PostgreSQL and vector storage.
Supports Execution Pipeline in MASD-DOC-003 (AI Engine): Delivers pre-filtered, verified context chunks required for grounded prompt construction.
30. Production Acceptance Criteria & Launch Gate
Before any Knowledge Base implementation (KB-SPEC-002 through KB-SPEC-010) is declared production-ready for commercial deployment, it must pass 100% of the following verification tests:
ID
Category
Launch Gate Acceptance Criterion
Verification Method
Pass/Fail
AC-01
Safety
Zero unverified allergen flags exist in ACTIVE KB records.
Automated DB Audit Query
Mandatory
AC-02
Validation
100% of ingested menu items pass Pydantic schema validation without errors.
Continuous Integration Test Suite
Mandatory
AC-03
Isolation
Zero cross-tenant data leakage detected under simulated load tests across multi-venue DB queries.
Penetration & Security Audit
Mandatory
AC-04
Latency
Cold-start hybrid vector retrieval returns context chunks in < 100ms (p95).
Performance Benchmark
Mandatory
AC-05
Lineage
100% of stored entities possess valid, auditable {{DATA_SOURCE}} and {{SCRAPE_TIMESTAMP}} attributes.
Lineage Inspection Script
Mandatory
AC-06
Accuracy
Temporal exception engine correctly overrides standard weekly hours for holiday test dates.
Automated Logic Test Suite
Mandatory
AC-07
State Gate
Unapproved/Pending records are completely invisible to the live AI retrieval layer.
End-to-End Context Retrieval Test
Mandatory
AC-08
Freshness
System automatically flags records exceeding 30 days as STALE.
Cron Task Execution Verification
Mandatory
AC-09
Parity
Vector index chunk count matches PostgreSQL active entity count with 100% precision.
Index Audit Verification
Mandatory
AC-10
Human Gate
Scraped web data with confidence score < 0.90 successfully blocks auto-publishing and routes to Admin Queue.
Human-in-the-Loop Integration Test
Mandatory

Document Version History
Version
Date
Description of Changes
Author
Approved By
1.0.0
August 2026
Initial release of Master Knowledge Base Specification (Phase 2 Master Architecture).
Ramy Bella
Approved for Execution


