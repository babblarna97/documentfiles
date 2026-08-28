# TEST-SPEC-12: Schema Validation & Provenance Integrity

## 1. DOCUMENT CONTROL

| Attribute | Value |
|---|---|
| Document Title | 12 Schema Validation & Provenance Integrity.md |
| Document ID | TEST-SPEC-12 |
| Version | 1.0.1 |
| Status | APPROVED FOR IMPLEMENTATION |
| Author | Ramy Bella |
| Classification | Confidential / Enterprise Proprietary |
| Target Audience | Data Architects, Knowledge Base Engineers, RAG Specialists, QA Engineers |
| Parent Document | TEST-SPEC-01 |
| Related Documents | Phase 2 Specs (KB-SPEC), TEST-SPEC-01, TEST-SPEC-02, TEST-SPEC-13, TEST-SPEC-14 |
| System | Restaurant AI System |
| Phase | Phase 6 — Validation & Verification (V&V) |
| Lifecycle Folder | 04 Knowledge Base & RAG Verification |
| Last Updated | August 2026 |

---

## 2. EXECUTIVE PURPOSE

A RAG-driven Knowledge Base (Phase 2) serving restaurant menus, allergen facts, operating hours, and booking policies is only as reliable as its ingestion pipeline and provenance lineage. Ingesting corrupt JSON schemas, unverified document chunks, stale menu snapshots, or un-provenanced third-party text causes severe operational failures, such as quoting incorrect prices or missing life-threatening allergen warnings.

`TEST-SPEC-12` defines the **Schema Validation & Provenance Integrity Test Architecture**. It establishes automated verification of Knowledge Base ingestion schemas (`KBMenuIngestSchema@1.0.0`), document chunk provenance hashing (`ChunkProvenance@1.0.0`), cryptographic ingestion signatures, chunking boundary sanity, and freshness/TTL lifecycle enforcement. It guarantees that every knowledge fragment retrieved into the RAG prompt context is mathematically traceable to an authorized, valid source document.

### Core Testing Invariants:
* `UNPROVENANCED CHUNK IN RETRIEVAL \implies PROHIBITED RAG CONTEXT INJECTION`
* `SCHEMA VALIDATION FAILURE \implies IMMEDIATE INGESTION PIPELINE HALT`
* `EXPIRED KB SNAPSHOT (TTL EXCEEDED) \implies AUTOMATED STALE DATA REJECTION`
* `CRYPTOGRAPHIC PROVENANCE SIGNATURE MISMATCH \implies DATA CORRUPTION ALARM`
* `MENU/ALLERGEN DATA WITHOUT VERIFIABLE SOURCE METADATA \implies INGESTION REJECTED`

---

## 3. KNOWLEDGE BASE INGESTION SCHEMA & PROVENANCE CONTRACTS

All data entering the Phase 2 Knowledge Base MUST conform to strict JSON schemas and carry an immutable provenance record.

### 3.1. Ingestion Data Schema (`KBMenuIngestSchema@1.0.0`)

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "title": "KBMenuIngestSchema@1.0.0",
  "type": "object",
  "properties": {
    "tenant_id": { "type": "string" },
    "document_id": { "type": "string" },
    "schema_version": { "type": "string", "enum": ["1.0.0"] },
    "effective_from": { "type": "string", "format": "date-time" },
    "effective_until": { "type": "string", "format": "date-time" },
    "menu_categories": {
      "type": "array",
      "items": {
        "type": "object",
        "properties": {
          "category_name": { "type": "string" },
          "items": {
            "type": "array",
            "items": {
              "type": "object",
              "properties": {
                "item_id": { "type": "string" },
                "name": { "type": "string" },
                "price": { "type": "number", "minimum": 0.0 },
                "currency": { "type": "string", "enum": ["SEK", "EUR", "USD"] },
                "allergens": { "type": "array", "items": { "type": "string" } },
                "is_available": { "type": "boolean" }
              },
              "required": ["item_id", "name", "price", "currency", "allergens", "is_available"]
            }
          }
        },
        "required": ["category_name", "items"]
      }
    }
  },
  "required": ["tenant_id", "document_id", "schema_version", "effective_from", "effective_until", "menu_categories"]
}


### 3.2. Cryptographic Chunk Provenance Contract (ChunkProvenance@1.0.0)

Every text chunk stored in the vector database includes an immutable metadata block linking it back to the authoritative source document.

{
  "chunk_id": "chk_8f92a1b0",
  "tenant_id": "tenant_luxury_dining",
  "source_document_id": "doc_menu_summer_2026",
  "source_author_role": "MENU_MANAGER",
  "ingested_at": "2026-08-01T10:00:00Z",
  "content_sha256": "e3b0c44298fc1c149afbf4c8996fb92427ae41e4649b934ca495991b7852b855",
  "provenance_signature": "KMS_SIG_7f8a9b...",
  "ttl_expires_at": "2026-09-01T00:00:00Z"
}

### 3.3. Canonical Provenance Signature Payload

The provenance signature MUST cover the canonicalized provenance metadata and content digest, not the content hash alone.

The signed payload MUST include at minimum:

- `tenant_id`
- `chunk_id`
- `source_document_id`
- `content_sha256`
- `ingested_at`
- `ttl_expires_at`

All fields MUST be serialized using one deterministic canonicalization method before signing.

Any modification to the chunk content or any signed provenance field MUST invalidate signature verification and cause the chunk to be rejected.

4. PROVENANCE VERIFICATION PIPELINE
The RAG retrieval engine validates chunk provenance before injecting retrieved context into the LLM prompt template.
[RETRIEVED VECTOR CHUNKS]
           │
           ▼ [PROVENANCE INTEGRITY CHECKER]
┌─────────────────────────────────────────────────────────────────┐
│ 1. Validate `tenant_id` matches current IsolationContext       │
│ 2. Verify SHA-256 Digest (`content_sha256` == hash(chunk_text)) │
│ 3. Verify KMS Cryptographic Signature over the canonical provenance payload, including `tenant_id`, `chunk_id`, `source_document_id`, `content_sha256`, `ingested_at`, and `ttl_expires_at`.   │
│ 4. Evaluate TTL Expiration (`ttl_expires_at > current_time`)    │
└──────────────────────────────┬──────────────────────────────────┘
                               │
               ┌───────────────┴───────────────┐
               ▼                               ▼
      [ALL CHECKS PASSED]            [VERIFICATION FAILURE]
    Inject Chunk into RAG          Purge Chunk; Log `ERR_TEST_12_01`
    Prompt Template                Fall Back to Safe KB Query


5. AUTOMATED SCHEMA & PROVENANCE TEST HARNESS
The test suite executes automated ingestion validation and retrieval provenance audits.
# Representative Automated Provenance Integrity Test Harness
@pytest.mark.rtm(req_id="KB-SPEC-12-PROV-001")
def test_rag_chunk_provenance_signature_verification():
    # Setup: Retrieve chunks from Vector DB
    retrieved_chunks = vector_db.retrieve_context(
        tenant_id="tenant_demo",
        query="Vilka allergener finns i Erssons Fisksoppa?"
    )
    
    assert len(retrieved_chunks) > 0
    
    for chunk in retrieved_chunks:
        # Assert 1: Provenance metadata exists
        assert "provenance" in chunk.metadata
        prov = chunk.metadata["provenance"]
        
        # Assert 2: SHA-256 Hash matches text content exactly
        computed_hash = hashlib.sha256(chunk.page_content.encode('utf-8')).hexdigest()
        assert prov["content_sha256"] == computed_hash
        
         # Assert 3: Cryptographic signature over canonical payload is valid
        canonical_provenance = canonicalize({
            "tenant_id": prov["tenant_id"],
            "chunk_id": prov["chunk_id"],
            "source_document_id": prov["source_document_id"],
            "content_sha256": prov["content_sha256"],
            "ingested_at": prov["ingested_at"],
            "ttl_expires_at": prov["ttl_expires_at"],
        })

        assert kms_verifier.verify_signature(
            data=canonical_provenance,
            signature=prov["provenance_signature"]
        ) == True
        
        # Assert 4: Tamper verification - changing tenant_id MUST invalidate signature
        tampered_prov = dict(prov)
        tampered_prov["tenant_id"] = "tenant_attacker"

        tampered_payload = canonicalize({
            "tenant_id": tampered_prov["tenant_id"],
            "chunk_id": tampered_prov["chunk_id"],
            "source_document_id": tampered_prov["source_document_id"],
            "content_sha256": tampered_prov["content_sha256"],
            "ingested_at": tampered_prov["ingested_at"],
            "ttl_expires_at": tampered_prov["ttl_expires_at"],
        })

        assert kms_verifier.verify_signature(
            data=tampered_payload,
            signature=prov["provenance_signature"]
        ) == False


6. SECURITY & THREAT MODEL
Threat
Attack Vector
Preventive Control
Severity
Un-Provenanced Context Injection
Malicious actor inserts fake menu item or allergen statement directly into vector DB.
Provenance integrity checker rejects any vector result lacking a valid KMS cryptographic signature.
CRITICAL (SEV-0)
Stale Data Hallucination
System quotes outdated winter menu prices in summer due to unpurged vector chunks.
Mandatory ttl_expires_at metadata filter enforced on all vector search queries.
HIGH (SEV-1)
Corrupted Ingestion Payload
Ingestion pipeline accepts malformed JSON with missing allergen arrays, defaulting to "no allergens".
Strict JSON Schema validation (KBMenuIngestSchema@1.0.0); missing required fields fail build.
CRITICAL (SEV-0)
Cross-Tenant Data Tampering
Attacker modifies source document ID to associate Tenant B's menu with Tenant A.
Tenant ID, chunk identity, source identity, content digest, ingestion timestamp, and TTL metadata are included in the canonical KMS-signed provenance payload; any tampering invalidates signature verification.
CRITICAL (SEV-0)

7. FAILURE ARCHITECTURE & ERROR TAXONOMY
Error Code
Description
Canonical Class
Severity
ERR_TEST_12_01
Retrieved vector chunk missing valid KMS provenance signature or SHA-256 hash mismatch.
DATA_CORRUPTION
CRITICAL (SEV-0)
ERR_TEST_12_02
Knowledge Base ingestion payload failed JSON Schema validation (KBMenuIngestSchema@1.0.0).
VALIDATION
HIGH (SEV-1)
ERR_TEST_12_03
Stale or expired Knowledge Base chunk retrieved into active RAG context (TTL Exceeded).
STALE_DATA
HIGH (SEV-1)
ERR_TEST_12_04
Un-provenanced text chunk injected into LLM prompt template without source metadata.
SECURITY_BOUNDARY
CRITICAL (SEV-0)
ERR_TEST_12_05
Menu item ingested without mandatory allergens array declaration.
SAFETY_VIOLATION
CRITICAL (SEV-0)

8. PRODUCTION ACCEPTANCE CRITERIA
AC ID
Category
Requirement
Verification Method
Pass/Fail
AC-KBS-01
100\% Provenance Pass
100\% of retrieved vector chunks contain valid, verifiable KMS signatures and SHA-256 hashes.
Vector Provenance Audit
REQUIRED
AC-KBS-02
Schema Enforcement
100\% of ingested menu datasets validate against KBMenuIngestSchema@1.0.0; 0 schema errors permitted.
Ingestion Pipeline Test
REQUIRED
AC-KBS-03
Zero Stale Context
0 expired chunks (T_{\text{now}} > T_{\text{expires}}) injected into RAG prompt templates during testing.
TTL Boundary Test Suite
REQUIRED
AC-KBS-04
Allergen Completeness
100\% of ingested food items explicitly declare an allergens array (even if empty []).
Allergen Schema Audit
REQUIRED
AC-KBS-05
Un-provenanced Block
Any chunk lacking provenance metadata is rejected at the RAG proxy boundary with ERR_TEST_12_04.
Fault Injection Harness
REQUIRED
| AC-KBS-06 | Provenance Tamper | Any modification to provenance metadata (tenant, TTL, source) MUST invalidate the signature and block RAG injection. | Canonical Tamper Test | REQUIRED |

9. INTEGRATION AUTHORITY MATRIX
Component
Owns
Consumes
Produces
TEST-SPEC-12
Schema Validation Suite, Provenance Verification Engine, TTL Audit Harness
KB Ingestion Payloads, Retrieved Vector Chunks
Ingestion Pass/Fail Verdicts, Provenance Signals
Phase 2 (KB-SPEC)
Authoritative Knowledge Base Architecture & Vector Store Indexing
Raw Documents, Menu Files
Vector Embeddings & Metadata
TEST-SPEC-13
RAG Grounding, Hallucination & Citation Tests
Verified Chunks (TEST-SPEC-12)
Grounding Quality Scores
TEST-SPEC-14
Deterministic Allergen Guardrail Tests
Validated Menu Schemas
Food Safety Pass/Fail Verdicts

10. FINAL NON-NEGOTIABLE PRINCIPLES
NO KNOWLEDGE BASE CHUNK MAY ENTER RAG PROMPT CONTEXT WITHOUT A VALID CRYPTOGRAPHIC PROVENANCE SIGNATURE.
ALL INGESTED MENU AND POLICY DATA MUST STRICTLY VALIDATE AGAINST AUTHORITATIVE JSON SCHEMAS.
EXPIRED OR STALE KNOWLEDGE CHUNKS MUST BE AUTOMATICALLY PURGED AT THE RETRIEVAL BOUNDARY.
ALL MENU ITEMS MUST EXPLICITLY DECLARE AN ALLERGEN ARRAY; MISSING ALLERGEN FIELDS CONSTITUTES A SEV-0 HALT.
PROVENANCE HASHES MUST MATCH RAW CHUNK TEXT WITH 100% BITWISE EXACTNESS; ANY DISCREPANCY TRIGGERS ALARMS.

PROVENANCE SIGNATURES MUST COVER THE COMPLETE CANONICAL PROVENANCE PAYLOAD; MODIFICATION OF ANY SIGNED FIELD MUST INVALIDATE VERIFICATION AND BLOCK RAG INJECTION.

11. VERSION HISTORY & ARCHITECTURAL VERDICT
Version
Date
Description
Author
Status
1.0.0
August 2026
Initial specification of Schema Validation & Provenance Integrity (TEST-SPEC-12).
Ramy Bella
APPROVED FOR IMPLEMENTATION
1.0.1
August 2026
Review fix pass. Fixed a malformed $schema URL in KBMenuIngestSchema@1.0.0 (markdown link syntax had been baked into the JSON string, making it invalid JSON as written). Removed a literal duplicate of Section 3.2 (identical heading and JSON block had appeared twice). Fixed a duplicated step-number typo ("3. 3.") in the Provenance Verification Pipeline diagram. Numbered the previously-unnumbered Canonical Provenance Signature Payload subsection as 3.3 for consistency. Removed a stray UI-widget artifact (`<ElicitationsGroup>`/`<Elicitation>` tags) that had leaked into the end of the file from document generation tooling. Reformatted one Core Testing Invariant to match the implies-arrow convention used elsewhere. No change to schema fields, provenance model, TTL rules, or acceptance criteria.
Ramy Bella
APPROVED FOR IMPLEMENTATION

ARCHITECTURAL VERDICT:
APPROVED FOR IMPLEMENTATION TEST-SPEC-12 establishes the JSON Schema ingestion contracts (KBMenuIngestSchema@1.0.0), KMS chunk provenance signatures (ChunkProvenance@1.0.0), TTL freshness rules, and RAG verification harness for Phase 6. Ready for implementation.
