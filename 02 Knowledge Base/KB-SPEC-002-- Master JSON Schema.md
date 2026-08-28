
# KB-SPEC-002: Master JSON Schema

## DOCUMENT CONTROL
| Attribute | Value |
|---|---|
| Document Title | Master JSON Schema |
| Document ID | KB-SPEC-002 |
| Version | 1.1.1 |
| Status | 🔒 LOCKED / Approved for Implementation |
| Author | Ramy Bella |
| Classification | Confidential / Enterprise Proprietary |
| Target Audience | Backend Engineers, Data Engineers, AI Engineers, Database Engineers, QA Engineers, Platform Architects |
| Parent Document | KB-SPEC-001 — Knowledge Base Specification |
| System | Restaurant AI System |
| Phase | Phase 2 — Knowledge Base Specification |
| Last Updated | August 2026 |

---

## 1. PURPOSE & DATA CONTRACT OBJECTIVE
This specification defines the authoritative Master JSON Data Contract for the Restaurant AI System.
The Master JSON Schema is the canonical structural boundary between unstructured external information (raw website scraping, PDF menus, manual administrative inputs) and deterministic, verifiable internal knowledge consumed by the RAG AI Engine.
Every approved restaurant knowledge record MUST conform to this exact schema. Downstream systems must never assume fields or structures outside this contract. Upstream ingestion pipelines (Scrapers, Normalizers) are contractually bound to transform messy real-world reality into this strict shape before data is persisted or indexed.

### The Knowledge Maturation Pipeline
```text
[ Raw External Data ] (HTML, PDF, Chaos)
         │
         ▼
[ Normalization Engine ] (Mapping to Master Schema Structure)
         │
         ▼
[ Validation Layer ] (Pydantic / Type & Business Rule Checks)
         │
         ▼
[ Human Verification Queue ] (Operator Review & Approval)
         │
         ▼
[ Master Knowledge Base ] (Persisted SSOT Record - ACTIVE)
         │
         ▼
[ RAG & AI Engine ] (Grounding Context for Guest Inquiries)

## 2. SCHEMA DESIGN PRINCIPLES

### Principle 1: Deterministic Structure & Strong Typing

- **Purpose:** Eliminate ambiguous data interpretation by downstream services.
- **Principle Statement:** Every schema field must possess an explicit, non-negotiable data type.
- **Reasoning:** Polymorphic types or shifting shapes introduce runtime exceptions and unstable AI parsing.
- **Acceptance Criteria:** 100% rejection rate for payloads failing type validation.

### Principle 2: Multi-Tenant Tenant Isolation

- **Purpose:** Guarantee strict separation of data across independent hospitality venues.
- **Principle Statement:** Every entity payload must be bound to a definitive `venue_id`.
- **Reasoning:** Prevents cross-tenant data leaks in a shared multi-tenant database or vector store.
- **Acceptance Criteria:** Ingestion fails automatically if `venue_id` is missing or invalid.

### Principle 3: Non-Ambiguous Nullability & Missing Data Discipline

- **Purpose:** Prevent AI hallucinations caused by confusing missing data with confirmed absence.
- **Principle Statement:** The schema forbids treating null, missing fields, and empty states as interchangeable. Explicit states must be used: `UNKNOWN`, `NOT_APPLICABLE`, `VERIFIED_ABSENT`, or actual structured values.
- **Reasoning:** In safety-critical contexts (allergens, venue constraints), not knowing a fact is fundamentally different from knowing it is false.

### Principle 4: Granular Entity-Level Lineage & Provenance

- **Purpose:** Allow absolute auditability down to the individual factual item level.
- **Principle Statement:** Lineage metadata is not limited to the global document; granular fields or sub-entities must carry origin provenance where applicable.
- **Reasoning:** When an AI answers a query about a specific menu item price or allergen, the system must trace that exact attribute back to its source URL or admin edit event.

### Principle 5: Backward Compatibility & Extensibility

- **Purpose:** Support platform feature evolution without breaking deployed instances.
- **Principle Statement:** Schema additions must follow semantic versioning rules, ensuring older components safely ignore unknown extension fields.

## 3. MASTER OBJECT ARCHITECTURE

The top-level JSON object is logically partitioned into specialized domains to separate identity, physical reality, product offerings, operational rules, safety matrices, and system metadata.

{
  "schema_metadata": {
    "schema_version": "1.1.1",
    "tenant": {
      "venue_id": "se-sto-brasserie-01"
    }
  },
  "venue": {},
  "contact": {},
  "location": {},
  "operating_hours": {},
  "menu": {},
  "allergens": {},
  "faq": {},
  "policies": {},
  "booking": {},
  "services": {},
  "lineage": {},
  "verification": {},
  "quality": {},
  "lifecycle": {},
  "retrieval_hints": {}
}

### Domain Responsibilities

- `**schema_metadata**`: Controls versioning and tenant binding (`venue_id`).
- `**venue**`, `**contact**`, `**location**`: Defines the physical brand and location.
- `**operating_hours**`: Manages regular schedules, shifts, and date overrides.
- `**menu**`: Canonical catalog of product offerings and pricing.
- `**allergens**`: Explicit safety-critical dietary classification matrix.
- `**faq**`, `**policies**`, `**booking**`, `**services**`: Rules, answers, and operational capabilities.
- `**lineage**`, `**verification**`, `**quality**`, `**lifecycle**`: Trust, state machines, and data health metadata.
- `**retrieval_hints**`: Search tags and embedding state hints (decoupled from internal database implementation).

## 4. TOP-LEVEL SCHEMA CONTRACT

|Field Name|JSON Type|Requirement|Nullable|Description|
|---|---|---|---|---|
|`schema_metadata`|Object|Required|No|Version contract & tenant scoping (`venue_id`).|
|`venue`|Object|Required|No|Core brand identity and operational type.|
|`contact`|Object|Required|No|Validated communication channels.|
|`location`|Object|Required|No|Physical address coordinates.|
|`operating_hours`|Object|Required|No|Standard shifts, kitchen cutoffs, and exceptions.|
|`menu`|Object|Optional|No|Structured categories and items.|
|`allergens`|Object|Required|No|Centralized safety matrix for EU 14 allergens.|
|`faq`|Array|Optional|No|Curated operational question-answer pairs.|
|`policies`|Object|Optional|No|Rules regarding pets, children, payments.|
|`booking`|Object|Required|No|Reservation limits and rules.|
|`services`|Object|Optional|No|Capabilities (takeaway, outdoor seating).|
|`lineage`|Object|Required|No|Source URL and extraction tracing.|
|`verification`|Object|Required|No|Human or automated approval tracking.|
|`quality`|Object|Required|No|Heuristic completeness and confidence scoring.|
|`lifecycle`|Object|Required|No|FSM state (e.g., `ACTIVE`, `STALE`).|
|`retrieval_hints`|Object|System-Gen|No|Semantic search tags and indexing status.|

## 5. IDENTIFIER & VERSIONING MODEL

- `**venue_id**`: Immutable tenant string (`{country}-{city}-{slug}`).
- `**schema_version**`: Semantic contract version (`1.1.1`).
- `**record_id**`: UUIDv4 representing a specific ingestion snapshot.
- `**entity_id**`: UUIDv4 assigned to sub-elements (menus, dishes, FAQs) to allow stable patching during re-scrapes.

## 6. VENUE DOMAIN

"venue": {
  "legal_name": "Brasserie Gabriel AB",
  "display_name": "Brasserie Gabriel",
  "venue_type": "RESTAURANT",
  "description": "Classic French bistro in central Stockholm.",
  "primary_language": "sv",
  "currency": "SEK"
}

## 7. CONTACT DOMAIN

"contact": {
  "primary_phone": "+4681234567",
  "booking_email": "boka@brasserie.se",
  "primary_website": "[https://brasserie.se](https://brasserie.se)",
  "social_links": ["[https://instagram.com/brasseriegabriel](https://instagram.com/brasseriegabriel)"]
}

### Strict Key Omission & Array Rules:

1. **Explicit `null` is FORBIDDEN:** The keys inside the contact object MUST NEVER contain `null`.
2. **Missing String Fields:** If `primary_phone` or `booking_email` is missing, unverified, or non-existent, the corresponding key MUST be omitted from the JSON payload entirely.
3. **Traceability:** Missing keys MUST be explicitly tracked in the top-level `verification.field_status` map as `"UNKNOWN"` or `"VERIFIED_ABSENT"`.
4. **List Fields:** Array properties (such as `social_links`) MUST default to an empty list `[]` if no links exist, never `null`.

## 8. LOCATION DOMAIN

"location": {
  "street_address": "Storgatan 12",
  "city": "Stockholm",
  "postal_code": "11435",
  "country_code": "SE",
  "coordinates": {
    "lat": 59.3328,
    "lng": 18.0645
  }
}

## 9. OPERATING HOURS DOMAIN (SUPPORTING OVER-MIDNIGHT)

To support venues open past midnight (e.g., closing at 01:00), the schema introduces a `next_day` boolean flag.

"operating_hours": {
  "timezone": "Europe/Stockholm",
  "regular_schedule": [
    {
      "day_of_week": 5,
      "shifts": [
        { 
          "open": "17:00", 
          "close": "01:00", 
          "next_day": true, 
          "type": "DINNER",
          "kitchen_close": "23:30"
        }
      ]
    }
  ],
  "exceptions": []
}

- **Rule:** If `next_day: true`, the closing time belongs to the calendar day immediately following `day_of_week`.

## 10. MENU DOMAIN

"menu": {
  "catalogs": [
    {
      "catalog_id": "uuid-cat-1",
      "name": "Evening Menu",
      "categories": [
        {
          "category_id": "uuid-cat-sub-1",
          "name": "Mains",
          "items": [
            {
              "item_id": "uuid-item-1",
              "name": "Entrecôte",
              "description": "Served with fries and café de Paris butter.",
              "price_cents": 32000,
              "is_available": true,
              "lineage": {
                "source_url": "[https://brasserie.se/menu](https://brasserie.se/menu)",
                "extracted_at": "2026-08-11T10:00:00Z"
              }
            }
          ]
        }
      ]
    }
  ]
}

## 11. ALLERGEN & DIETARY DATA CONTRACT (SAFETY-CRITICAL)

The allergen block is elevated to a top-level domain to ensure it is never treated as an implicit sub-property of menus.

"allergens": {
  "verification_status": "VERIFIED",
  "matrix": [
    {
      "item_id": "uuid-item-1",
      "eu_14_allergens": {
        "cereals_gluten": "FREE_FROM",
        "milk": "CONTAINS",
        "peanuts": "VERIFIED_ABSENT",
        "celery": "UNKNOWN"
      }
    }
  ]
}

### Allowed Allergen States

- `**CONTAINS**`: Explicitly present.
- `**MAY_CONTAIN**`: Cross-contamination risk.
- `**FREE_FROM**`: Explicitly confirmed absent.
- `**VERIFIED_ABSENT**`: Inspected and verified not in recipe.
- `**UNKNOWN**`: Data missing or unverified (Triggers human fallback).

## 12. FAQ DOMAIN

"faq": [
  {
    "faq_id": "uuid-faq-1",
    "category": "PARKING",
    "question": "Where can I park?",
    "answer": "Public parking garage located 100 meters down the street.",
    "language": "sv"
  }
]

## 13. POLICY DOMAIN

"policies": {
  "pets": "TERRACE_ONLY",
  "children": "WELCOME",
  "dress_code": "SMART_CASUAL",
  "cash_accepted": false
}

## 14. BOOKING DOMAIN

"booking": {
  "is_booking_enabled": true,
  "rules": {
    "min_party_size": 1,
    "max_party_size_online": 8,
    "slot_duration_minutes": 120,
    "booking_window_days": 30
  }
}

## 15. SERVICES & CAPABILITIES DOMAIN

"services": {
  "dine_in": true,
  "takeaway": true,
  "delivery": false,
  "outdoor_seating": true,
  "private_events": true
}

## 16. SOURCE & DATA LINEAGE CONTRACT

"lineage": {
  "source_type": "SCRAPED_WEBSITE",
  "source_url": "[https://brasserie.se](https://brasserie.se)",
  "scrape_timestamp": "2026-08-11T12:00:00Z",
  "extractor_version": "v2.1.0"
}

## 17. VERIFICATION CONTRACT

"verification": {
  "approval_status": "APPROVED",
  "verified_by": "human_admin_id_442",
  "verified_at": "2026-08-11T12:30:00Z"
}

## 18. DATA QUALITY CONTRACT

`confidence_score` is defined strictly as a machine-generated heuristic confidence metric (0.00 to 1.00), separate from human verification.

"quality": {
  "confidence_score": 0.94,
  "completeness_score": 0.88,
  "warnings": []
}

## 19. LIFECYCLE & VERIFICATION STATE SYNCHRONIZATION

To eliminate runtime query collisions in RAG retrieval contexts and maintain clear separation of concerns, the system enforces a strict dual-state model:

1. **`verification.approval_status` (Review Gate):**
    - Governs human and automated verification evaluation.
    - Controlled ENUM: `["PENDING", "APPROVED", "REJECTED"]`.
2. **`lifecycle.state` (Runtime Searchability & Index State):**
    - Governs physical database and vector index availability.
    - Controlled ENUM: `["RAW", "NORMALIZED", "VALIDATED", "PENDING_REVIEW", "ACTIVE", "STALE", "ARCHIVED"]`.

### Deterministic State Transition Sequence:

- **`VALIDATED` → `PENDING_REVIEW`**: Payload passes structural/business rules; awaits review.
- **`PENDING_REVIEW` → `APPROVED` (Review Gate)**: Human operator or automated high-confidence check approves the payload (`verification.approval_status` mutates to `"APPROVED"`).
- **`APPROVED` → `ACTIVE` (Lifecycle Mutation)**: Approval triggers the Vectorization & Hybrid Indexing Pipeline. Upon successful vector insertion and cache hydration, `lifecycle.state` mutates atomically to `"ACTIVE"`.

> **BINDING RETRIEVAL RULE:** Downstream RAG Retrieval and Vector Search engines MUST filter strictly on `lifecycle.state == 'ACTIVE'`. `verification.approval_status` MUST NOT be used as a query filter in runtime vector search.

## 20. RETRIEVAL HINTS CONTRACT (DECOUPLED)

Keeps search hints semantic without tying the contract to physical database table IDs (such as vector database specific IDs).

"retrieval_hints": {
  "search_tags": ["french", "bistro", "stockholm", "meat"],
  "index_status": "INDEXED"
}

## 21. MULTI-TENANT ISOLATION CONTRACT

Every record is bound to `schema_metadata.tenant.venue_id`, enforced by database row-level security and mandatory filter clauses during RAG context retrieval.

## 22. NULLABILITY & MISSING DATA POLICY (STRICT)

To solve the ambiguity of null, the schema enforces distinct semantics:

- **`UNKNOWN` / Omitted Field**: Data has not been verified or scraped.
- `**NOT_APPLICABLE**`: Field does not apply to this venue type.
- `**VERIFIED_ABSENT**`: Explicitly confirmed not to exist.
- **Actual `null`**: Strictly FORBIDDEN across all factual payload domains.

## 23. EXTENSIBILITY & FUTURE COMPATIBILITY

Parsers are configured to ignore unknown fields (`extra = 'ignore'`) to allow future schema evolution without breaking active deployments.

## 24. VALIDATION & TYPE ENFORCEMENT

Enforced via Pydantic data models matching this contract during the Normalization and Validation pipeline phases.

## 25. EXAMPLE MASTER JSON DOCUMENT

{
  "schema_metadata": {
    "schema_version": "1.1.1",
    "tenant": {
      "venue_id": "se-sto-cafe-01"
    }
  },
  "venue": {
    "display_name": "Café Fictional",
    "venue_type": "CAFE",
    "primary_language": "sv",
    "currency": "SEK"
  },
  "contact": {
    "primary_phone": "+4689876543",
    "primary_website": "[https://cafefictional.se](https://cafefictional.se)"
  },
  "location": {
    "street_address": "Kaffegatan 4",
    "city": "Stockholm",
    "postal_code": "11355",
    "country_code": "SE"
  },
  "operating_hours": {
    "timezone": "Europe/Stockholm",
    "regular_schedule": [
      {
        "day_of_week": 1,
        "shifts": [{ "open": "08:00", "close": "17:00", "next_day": false, "type": "ALL_DAY" }]
      }
    ],
    "exceptions": []
  },
  "menu": {
    "catalogs": [
      {
        "catalog_id": "cat-cafe-1",
        "name": "Standard Menu",
        "categories": [
          {
            "category_id": "sub-cafe-1",
            "name": "Bakery",
            "items": [
              {
                "item_id": "item-cafe-1",
                "name": "Kanelbulle",
                "price_cents": 4500,
                "is_available": true
              }
            ]
          }
        ]
      }
    ]
  },
  "allergens": {
    "verification_status": "VERIFIED",
    "matrix": [
      {
        "item_id": "item-cafe-1",
        "eu_14_allergens": {
          "cereals_gluten": "CONTAINS",
          "milk": "CONTAINS",
          "eggs": "CONTAINS",
          "peanuts": "VERIFIED_ABSENT"
        }
      }
    ]
  },
  "faq": [],
  "policies": {
    "pets": "WELCOME",
    "children": "WELCOME"
  },
  "booking": {
    "is_booking_enabled": false,
    "rules": {
      "min_party_size": 0,
      "max_party_size_online": 0,
      "slot_duration_minutes": 0
    }
  },
  "services": {
    "dine_in": true,
    "takeaway": true,
    "delivery": false
  },
  "lineage": {
    "source_type": "SCRAPED_WEBSITE",
    "scrape_timestamp": "2026-08-11T14:00:00Z"
  },
  "verification": {
    "approval_status": "APPROVED",
    "verified_by": "system_auto"
  },
  "quality": {
    "confidence_score": 0.98
  },
  "lifecycle": {
    "state": "ACTIVE"
  },
  "retrieval_hints": {
    "search_tags": ["bakery", "coffee", "fika"]
  }
}

## 26. INVALID JSON EXAMPLES & FAILURE CASES

|Failure Case|Why It Is Invalid|Detecting Layer|Expected Behavior|
|---|---|---|---|
|Missing `venue_id`|Breaks multi-tenant isolation.|Schema Validator|Immediate payload rejection.|
|Allergen marked missing instead of `UNKNOWN`|Violates safety nullability rules.|Semantic Validator|Fails validation; blocks `APPROVED` state.|
|Close time earlier than open time (without `next_day: true`)|Temporal logic corruption.|Business Rule Validator|Rejects shift definition.|
|Explicit `null` in contact string fields|Violates Strict Key Omission rule.|Schema Validator|Rejects payload.|

## 27. SCHEMA EVOLUTION & MIGRATION

Semantic versioning (1.x to 2.x) dictates breaking changes, while minor updates (1.1) allow safe property extensions.

## 28. RELATIONSHIP TO OTHER MASTER DOCUMENTS

Governs the normalized output expected by downstream components defined in Phase 3 and beyond.

## 29. ENGINEERING IMPLEMENTATION REQUIREMENTS

Engineers must build Pydantic models matching this specification exactly before writing scraping or RAG ingestion logic.

## 30. PRODUCTION ACCEPTANCE CRITERIA

|ID|Category|Requirement|Verification Method|Pass/Fail|Severity|
|---|---|---|---|---|---|
|AC-01|Structural|All active KB documents conform to Schema 1.1.1.|CI Test Suite|Required|Critical|
|AC-02|Security|Every tenant record includes valid `venue_id`.|DB/App Test|Required|Critical|
|AC-03|Safety|Allergens enforce strict ENUM states without silent defaults.|Unit Test|Required|Critical|
|AC-04|Temporal|Over-midnight shifts correctly evaluate with `next_day: true`.|Logic Test|Required|High|

## 31. VERSION HISTORY

|Version|Date|Description|Author|Approval Status|
|---|---|---|---|---|
|1.0.0|August 2026|Initial master schema release.|Ramy Bella|Superseded|
|1.1.0|August 2026|Revised Master JSON Schema addressing nullability, over-midnight hours, and granular lineage.|Ramy Bella|Superseded|
|1.1.1|August 2026|Corrected Strict Key Omission in contact domain and synchronized dual-FSM lifecycle states (`APPROVED` -> `ACTIVE`).|Ramy Bella|🔒 LOCKED / Approved for Implementation|

## 32. FINAL DESIGN PRINCIPLES

- **The Schema is the Contract:** Absolute compliance required.
- **Fail Closed:** Invalid data never reaches production RAG index.
- **Safety First:** Unknown data is never assumed safe.