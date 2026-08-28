# TEST-SPEC-13: Grounding, Hallucination & Citation Tests

## 1. DOCUMENT CONTROL

| Attribute | Value |
|---|---|
| Document Title | 13 Grounding, Hallucination & Citation Tests.md |
| Document ID | TEST-SPEC-13 |
| Version | 1.0.1 |
| Status | APPROVED FOR IMPLEMENTATION |
| Author | Ramy Bella |
| Classification | Confidential / Enterprise Proprietary |
| Target Audience | AI Engineers, RAG Architects, NLP Specialists, QA Engineers |
| Parent Document | TEST-SPEC-01 |
| Related Documents | Phase 2 Specs (KB-SPEC), TEST-SPEC-01, TEST-SPEC-03, TEST-SPEC-12, TEST-SPEC-14 |
| System | Restaurant AI System |
| Phase | Phase 6 — Validation & Verification (V&V) |
| Lifecycle Folder | 04 Knowledge Base & RAG Verification |
| Last Updated | August 2026 |

---

## 2. EXECUTIVE PURPOSE

Retrieval-Augmented Generation (RAG) pipelines are susceptible to generative hallucinations—producing claims that sound plausible but have no factual grounding in the retrieved Knowledge Base context. In a restaurant system, a hallucinated opening hour, fake dish price, or invented ingredient listing degrades customer trust, causes monetary discrepancies, and introduces operational friction.

`TEST-SPEC-13` defines the **Grounding, Hallucination & Citation Test Architecture**. Operationalizing the evaluation formulas defined in `TEST-SPEC-03`, it establishes automated testing harnesses to measure RAG Faithfulness ($S_{\text{faith}}$), Context Recall ($S_{\text{recall}}$), Citation Precision ($P_{\text{cite}}$), and "I Don't Know" (IDK) fallback compliance when questions exceed Knowledge Base boundaries. It guarantees that generated responses are mathematically grounded in provenanced chunks (`TEST-SPEC-12`) and accurately cited before reaching the user.

### Core Testing Invariants:
* `HALLUCINATION RATE > 0.00 ON MENU / PRICE / HOUR DATA \implies HARD BUILD BLOCK`
* `CLAIM WITHOUT EXPLICIT CITATION LINK \implies INVALID RAG RESPONSE`
* `QUERY OUTSIDE KB SCOPE \implies MANDATORY HONEST FALLBACK ("Jag saknar den informationen")`
* `CITATION POINTER MISMATCH \implies AUTOMATED RESPONSE REDACTION`
* `FAITHFULNESS SCORE S_faith < 0.99 \implies STAGING PROMOTION BLOCK`

---

## 3. GROUNDING & CITATION METRICS SUITE

This specification enforces three primary quantitative metrics evaluated continuously against synthetic and golden datasets.

### 3.1. RAG Faithfulness Score ($S_{\text{faith}}$)
Calculates the proportion of atomic claims in the generated response ($R$) that are directly supported by retrieved context chunks ($C$).

$$S_{\text{faith}} = \frac{\vert{}\text{Claims in } R \text{ verified by } C\vert{}}{\vert{}\text{Total Claims extracted from } R\vert{}}$$

* **Target Threshold:** $S_{\text{faith}} \ge 0.99$ ($1.00$ required for menu pricing and operating hours).

### 3.2. Citation Precision ($P_{\text{cite}}$)
Measures the accuracy of inline citation markers (e.g., `[Doc: doc_menu_summer#chk_8f92]`) in referencing the exact chunk that contains the supporting fact.

$$P_{\text{cite}} = \frac{\vert{}\text{Correctly Cited Claims in } R\vert{}}{\vert{}\text{Total Citations Provided in } R\vert{}}$$

* **Target Threshold:** $P_{\text{cite}} = 1.00$ ($0\%$ citation mismatch permitted).

### 3.3. Out-of-Scope Fallback Rate ($F_{\text{fallback}}$)
Evaluates whether the system correctly refuses to answer queries when the required information is absent from retrieved context ($C = \emptyset$ or relevance score $< 0.70$).

* **Target Behavior:** $100\%$ compliance with canonical fallback phrasing: *"Jag har tyvärr inte den informationen i min meny eller min informationstext."*

---

## 4. RAG CITATION CONTRACT & RESPONSE SCHEMA (`RAGResponsePayload@1.0.0`)

All RAG-generated response outputs MUST conform to a structured contract containing inline citation metadata and claim-level verification traces.

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "title": "RAGResponsePayload@1.0.0",
  "type": "object",
  "properties": {
    "response_text": { "type": "string" },
    "grounding_summary": {
      "type": "object",
      "properties": {
        "faithfulness_score": { "type": "number", "minimum": 0.0, "maximum": 1.0 },
        "hallucination_detected": { "type": "boolean" },
        "claims_total": { "type": "integer" },
        "claims_verified": { "type": "integer" }
      },
      "required": ["faithfulness_score", "hallucination_detected", "claims_total", "claims_verified"]
    },
    "citations": {
      "type": "array",
      "items": {
        "type": "object",
        "properties": {
          "citation_id": { "type": "string" },
          "claim_text": { "type": "string" },
          "source_chunk_id": { "type": "string" },
          "source_document_id": { "type": "string" }
        },
        "required": ["citation_id", "claim_text", "source_chunk_id", "source_document_id"]
      }
    }
  },
  "required": ["response_text", "grounding_summary", "citations"]
}


5. AUTOMATED RAG GROUNDING TEST HARNESS SUITE
The test suite evaluates candidate LLM outputs against provenanced context chunks using the cross-family judge evaluator framework (TEST-SPEC-03).
# Representative Automated RAG Grounding & Hallucination Test
@pytest.mark.rtm(req_id="KB-SPEC-13-GRD-001")
def test_rag_faithfulness_and_hallucination_prevention():
    # Setup: Query context containing specific pricing and availability
    retrieved_context = [
        Chunk(id="chk_01", text="Plankstek kostar 295 kr och serveras med bearnaisesås och potatismos."),
        Chunk(id="chk_02", text="Uteserveringen stänger kl 22:00 alla dagar.")
    ]
    
    user_query = "Vad kostar planksteken och när stänger uteserveringen?"
    
    # Act: Generate response via system pipeline
    rag_output = rag_engine.generate(query=user_query, context=retrieved_context)
    
    # Assert 1: Validate Schema
    assert validate_schema(rag_output.json(), "RAGResponsePayload@1.0.0")
    
    # Assert 2: Evaluator Judge Faithfulness Score (TEST-SPEC-03)
    eval_result = judge_evaluator.evaluate_faithfulness(
        response=rag_output.response_text,
        context=retrieved_context
    )
    
    assert eval_result.faithfulness_score >= 0.99
    assert eval_result.hallucination_detected == False
    assert len(rag_output.citations) == 2


6. SECURITY & THREAT MODEL
Threat
Attack Vector
Preventive Control
Severity
Hallucination Coercion
User repeatedly prompts: "Are you sure you don't serve caviar? Check again."
Strict system prompt rule: System MUST NOT invent facts when context is missing (C = \emptyset).
HIGH (SEV-1)
Fake Citation Injection
LLM outputs a plausible citation tag referencing a non-existent chunk ID.
Post-generation citation validator verifies every cited chunk_id exists in the retrieved set (P_{\text{cite}} = 1.0).
HIGH (SEV-1)
RAG Context Poisoning
Malicious text in retrieved document tricks LLM into outputting false pricing.
Provenance integrity verification (TEST-SPEC-12) prior to prompt context insertion.
CRITICAL (SEV-0)

7. FAILURE ARCHITECTURE & ERROR TAXONOMY
Error Code
Description
Canonical Class
Severity
ERR_TEST_13_01
Faithfulness score fell below threshold (S_{\text{faith}} < 0.99) on menu/price context.
PROBABILISTIC_DRIFT
HIGH (SEV-1)
ERR_TEST_13_02
Hallucination detected in generated response text (H_{\text{rate}} > 0.00).
PROBABILISTIC_DRIFT
HIGH (SEV-1)
ERR_TEST_13_03
Citation tag pointed to non-existent or un-retrieved context chunk ID (P_{\text{cite}} < 1.0).
CONTRACT_MISMATCH
HIGH (SEV-1)
ERR_TEST_13_04
System attempted to answer an out-of-scope query instead of triggering honest IDK fallback.
LOGIC_FAIL
MEDIUM (SEV-2)
ERR_TEST_13_05
RAG response payload failed JSON Schema validation (RAGResponsePayload@1.0.0).
VALIDATION
HIGH (SEV-1)

8. PRODUCTION ACCEPTANCE CRITERIA
AC ID
Category
Requirement
Verification Method
Pass/Fail
AC-GRD-01
Zero Hallucination
0\% hallucination rate (H_{\text{rate}} = 0.00) across 1,000 baseline menu and pricing query turns.
Judge Evaluation Suite
REQUIRED
AC-GRD-02
Faithfulness SLA
S_{\text{faith}} \ge 0.99 verified across all non-safety Knowledge Base conversational turns.
Faithfulness Harness
REQUIRED
AC-GRD-03
Citation Precision
100\% of generated inline citations map to valid, retrieved chunk_id references (P_{\text{cite}} = 1.0).
Citation Auditor
REQUIRED
AC-GRD-04
Out-of-Scope Fallback
100\% of queries lacking context match (C = \emptyset) trigger canonical "I Don't Know" fallbacks.
Out-of-Scope Suite
REQUIRED
AC-GRD-05
Schema Enforcement
100\% of RAG execution outputs validate against RAGResponsePayload@1.0.0.
Schema Inspector
REQUIRED

9. INTEGRATION AUTHORITY MATRIX
Component
Owns
Consumes
Produces
TEST-SPEC-13
RAG Faithfulness Suite, Citation Precision Verifier, Out-of-Scope Fallback Tests
Provenanced Chunks (TEST-SPEC-12), System Prompts
Grounding Metrics, Citation Validation Verdicts
TEST-SPEC-03
LLM Evaluation Formulas (S_{\text{faith}}, H_{\text{rate}}) & Cross-Family Judge Engine
RAG Generation Outputs
Quantitative Quality Scores
TEST-SPEC-12
Schema Validation & Provenance Integrity
Raw Documents
Provenanced Vector Chunks
TEST-SPEC-14
Deterministic Allergen Guardrail Tests
Grounded Responses
Food Safety Interception Verdicts

10. FINAL NON-NEGOTIABLE PRINCIPLES
NO GENERATED RESPONSE MAY CONTAIN CLAIMS UNGROUNDED IN RETRIEVED KNOWLEDGE BASE CONTEXT.
EVERY FACTUAL CLAIM REGARDING MENU ITEMS, PRICES, OR HOURS MUST BE ACCURATELY CITED.
WHEN RETRIEVED CONTEXT IS ABSENT OR INSUFFICIENT, THE SYSTEM MUST HONESTLY STATE ITS LACK OF INFORMATION.
CITATION POINTERS MUST BE DETERMINISTICALLY VALIDATED AGAINST RETRIEVED CHUNK IDS PRIOR TO USER TRANSMISSION.
HALLUCINATION RATES ON MENU PRICING, OPERATING HOURS, AND INGREDIENTS MUST REMAIN STRICTLY AT ZERO (0.00) AS ENFORCED BY THE STAGING BUILD GATE (SECTION 3) AND CONTINUOUS PRODUCTION MONITORING; THIS IS A GATED ENGINEERING THRESHOLD, NOT A STATISTICAL CLAIM ABOUT ANY INDIVIDUAL GENERATION.
11. VERSION HISTORY & ARCHITECTURAL VERDICT
Version
Date
Description
Author
Status
1.0.0
August 2026
Initial specification of Grounding, Hallucination & Citation Tests (TEST-SPEC-13).
Ramy Bella
APPROVED FOR IMPLEMENTATION
1.0.1
August 2026
Review fix pass. Fixed a malformed $schema URL in RAGResponsePayload@1.0.0 (markdown link syntax had been baked into the JSON string, making it invalid JSON as written). Reworded Final Non-Negotiable Principle #5 to name its enforcement mechanism (staging build gate + continuous monitoring) rather than reading as an absolute statistical guarantee about a probabilistic system. No change to metrics, thresholds, schema fields, or acceptance criteria.
Ramy Bella
APPROVED FOR IMPLEMENTATION

ARCHITECTURAL VERDICT:
APPROVED FOR IMPLEMENTATION TEST-SPEC-13 establishes the RAG grounding metrics (S_{\text{faith}} \ge 0.99), citation precision verification (P_{\text{cite}} = 1.0), out-of-scope fallback compliance, and schema enforcement for Phase 6. Ready for implementation 
