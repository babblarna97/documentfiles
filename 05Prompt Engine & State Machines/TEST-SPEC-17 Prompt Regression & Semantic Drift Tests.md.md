## 1. DOCUMENT CONTROL

| Attribute | Value |
|---|---|
| Document Title | 17 Prompt Regression & Semantic Drift Tests.md |
| Document ID | TEST-SPEC-17 |
| Version | 1.0.0 |
| Status | APPROVED FOR IMPLEMENTATION |
| Author | Ramy Bella |
| Classification | Confidential / Enterprise Proprietary |
| Target Audience | AI Engineers, Prompt Architects, MLOps Engineers, QA Leads |
| Parent Document | TEST-SPEC-01 |
| Related Documents | Phase 4 Specs (PROMPT-COMPILER), TEST-SPEC-01, TEST-SPEC-02, TEST-SPEC-03, TEST-SPEC-16 |
| System | Restaurant AI System |
| Phase | Phase 6 — Validation & Verification (V&V) |
| Lifecycle Folder | 05 Prompt Engine & State Machines |
| Last Updated | August 2026 |

---

## 2. EXECUTIVE PURPOSE

In LLM-driven systems, changing a single word in a Phase 4 system prompt (e.g., changing "be helpful" to "be concise") can cause catastrophic, unintended behavioral shifts across the entire conversation space. This "semantic drift" can silently break persona guidelines, alter extraction accuracy, or change the tone from premium to abrupt. 

`TEST-SPEC-17` defines the **Prompt Regression & Semantic Drift Test Architecture**. Executed in the Stage 3 (E2E Contract & Evals) pipeline (`TEST-SPEC-02`), it establishes a quantitative baseline for prompt stability. It compares new prompt candidate outputs against a frozen "Golden Dataset" of established behaviors, measuring vector distance, intent shifts, and tone deviation. It guarantees that prompt updates improve targeted metrics without causing regressions in unrelated conversation flows.

### Core Testing Invariants:
* `INTENT EXTRACTION REGRESSION \implies SEV-1 BUILD BLOCK`
* `SEMANTIC DRIFT > 5% ON UNRELATED FLOWS \implies STAGING PROMOTION BLOCK`
* `NEW PROMPT CANDIDATE \implies MANDATORY A/B EVALUATION AGAINST GOLDEN DATASET`
* `TONE / PERSONA DEVIATION \implies AUTOMATED PIPELINE REJECTION`
* `EVALUATION MUST BE AUTOMATED VIA DECOUPLED JUDGE ENSEMBLE (TEST-SPEC-03)`

---

## 3. SEMANTIC DRIFT EVALUATION ARCHITECTURE

Every time a system prompt template (`TEST-SPEC-16`) is modified, the pipeline executes the new prompt ($P_{\text{new}}$) and the old prompt ($P_{\text{old}}$) against the Golden Dataset ($N=1,000$ baseline turns) and computes the regression delta ($\Delta$).

```text
[PROMPT TEMPLATE COMMIT: v1.2.0]
           │
           ▼
[REGRESSION EVALUATION ENGINE]
  ├─ Executes $P_{\text{new}}$ over Golden Dataset (1,000 turns)
  ├─ Executes $P_{\text{old}}$ over Golden Dataset (1,000 turns)
           │
           ▼
[METRIC DELTA COMPUTATION ($\Delta$)]
┌─────────────────────────────────────────────────────────────────┐
│ 1. Intent Extraction Precision ($\Delta F_1 \ge 0$)             │
│ 2. Semantic Embedding Distance ($\cos(P_{\text{new}}, P_{\text{old}}) \ge 0.95$)│
│ 3. Tone / Persona Alignment Score (via `TEST-SPEC-03` Judge)    │
└──────────────────────────────┬──────────────────────────────────┘
                               │
               ┌───────────────┴───────────────┐
               ▼                               ▼
    [NO NEGATIVE DRIFT DETECTED]      [REGRESSION / DRIFT DETECTED]
      Promote to Stage 4 (Shadow)      Block Build; Log `ERR_TEST_17_01`


4. REGRESSION VERDICT CONTRACT (SemanticDriftReport@1.0.0)
The drift evaluation MUST output a strict, auditable JSON report detailing the regression deltas.
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "title": "SemanticDriftReport@1.0.0",
  "type": "object",
  "properties": {
    "evaluation_id": { "type": "string" },
    "prompt_version_base": { "type": "string" },
    "prompt_version_candidate": { "type": "string" },
    "dataset_size": { "type": "integer", "minimum": 100 },
    "metrics_delta": {
      "type": "object",
      "properties": {
        "intent_f1_shift": { "type": "number" },
        "semantic_cosine_similarity": { "type": "number", "minimum": 0.0, "maximum": 1.0 },
        "tone_alignment_shift": { "type": "number" }
      },
      "required": ["intent_f1_shift", "semantic_cosine_similarity", "tone_alignment_shift"]
    },
    "verdict": { 
      "type": "string",
      "enum": ["PASS_NO_REGRESSION", "FAIL_INTENT_REGRESSION", "FAIL_SEMANTIC_DRIFT", "FAIL_TONE_DEVIATION"]
    },
    "evaluated_at": { "type": "string", "format": "date-time" }
  },
  "required": ["evaluation_id", "prompt_version_base", "prompt_version_candidate", "dataset_size", "metrics_delta", "verdict", "evaluated_at"]
}


5. AUTOMATED PROMPT REGRESSION TEST HARNESS
The regression suite leverages the LLM Judge (TEST-SPEC-03) to ensure semantic stability.
# Representative Automated Prompt Regression Test Harness
@pytest.mark.rtm(req_id="PRM-SPEC-17-REG-001")
def test_prompt_update_semantic_regression(golden_dataset):
    # Act: Evaluate new prompt vs base prompt
    drift_report = regression_engine.evaluate_prompt_candidate(
        base_prompt_id="booking_v1.1.0",
        candidate_prompt_id="booking_v1.2.0",
        dataset=golden_dataset
    )
    
    # Assert 0: Report schema validation
    assert validate_schema(drift_report.json(), "SemanticDriftReport@1.0.0")
    
    # Assert 1: Intent Extraction must not degrade (Delta >= 0)
    assert drift_report.metrics_delta.intent_f1_shift >= 0.0, "ERR_TEST_17_01: Intent extraction degraded"
    
    # Assert 2: Semantic Similarity must remain high for non-targeted flows
    assert drift_report.metrics_delta.semantic_cosine_similarity >= 0.95, "ERR_TEST_17_02: Semantic drift threshold exceeded"
    
    # Assert 3: Verdict must be PASS
    assert drift_report.verdict == "PASS_NO_REGRESSION"


6. SECURITY & THREAT MODEL
Threat
Attack Vector
Preventive Control
Severity
Silent Intent Degradation
A prompt tweak to "be more polite" causes the model to miss implicit party_size entities.
Hard block if Intent F1 drops by even 0.1\% on the Golden Dataset (\Delta F_1 < 0).
HIGH (SEV-1)
Persona Hijacking via Update
Rogue developer commits a prompt change that alters the bot to act like a pirate.
TEST-SPEC-03 Judge evaluates Tone Alignment; deviation blocks pipeline.
HIGH (SEV-1)
Dataset Overfitting
Prompts are engineered specifically to pass the Golden Dataset but fail in reality.
Golden dataset is split (80% visible, 20% blind holdout) for regression evaluation.
MEDIUM (SEV-2)

7. FAILURE ARCHITECTURE & ERROR TAXONOMY
Error Code
Description
Canonical Class
Severity
ERR_TEST_17_01
Intent extraction F_1 score degraded compared to the base prompt baseline.
PROBABILISTIC_DRIFT
HIGH (SEV-1)
ERR_TEST_17_02
Semantic vector drift exceeded 5% limit on unrelated conversational flows.
PROBABILISTIC_DRIFT
HIGH (SEV-1)
ERR_TEST_17_03
Tone and persona alignment judge detected a shift in brand voice.
CONTRACT_MISMATCH
HIGH (SEV-1)
ERR_TEST_17_04
SemanticDriftReport failed JSON schema validation (SemanticDriftReport@1.0.0).
VALIDATION
HIGH (SEV-1)

8. PRODUCTION ACCEPTANCE CRITERIA
AC ID
Category
Requirement
Verification Method
Pass/Fail
AC-REG-01
Zero Intent Regression
100\% of prompt updates must maintain or improve Intent F_1 scores against the Golden Dataset.
Regression Metric Audit
REQUIRED
AC-REG-02
Semantic Stability
Global cosine similarity between P_{\text{old}} and P_{\text{new}} outputs must remain \ge 0.95.
Vector Distance Audit
REQUIRED
AC-REG-03
Schema Compliance
100\% of drift reports validate against SemanticDriftReport@1.0.0, bounded by exact verdict enums.
Schema Inspector
REQUIRED
AC-REG-04
Blind Dataset Pass
The prompt candidate must pass regression metrics on the 20% blind holdout dataset to prevent overfitting.
Holdout Harness Test
REQUIRED

9. INTEGRATION AUTHORITY MATRIX
Component
Owns
Consumes
Produces
TEST-SPEC-17
Prompt Regression Matrix, Semantic Drift Measurement, Golden Dataset Execution
Compiled Prompts (TEST-SPEC-16)
SemanticDriftReport, Regression Signals
TEST-SPEC-03
LLM Evaluation Formulas & Judge Framework
Prompt Outputs
Tone & Similarity Metric Scores
TEST-SPEC-16
Dynamic Prompt Assembly
Raw Templates
Prompt Candidates
TEST-SPEC-02
CI/CD Stage 3 Evals Gate Enforcement
Regression Signals (this document)
Staging Promotion Blocks

10. FINAL NON-NEGOTIABLE PRINCIPLES
NO PROMPT UPDATE MAY DEGRADE INTENT EXTRACTION OR ENTITY RECOGNITION METRICS.
ALL PROMPT CANDIDATES MUST BE EVALUATED AGAINST AN IMMUTABLE GOLDEN DATASET BEFORE PROMOTION.
SEMANTIC DRIFT ON UNRELATED CONVERSATION FLOWS MUST BE BOUNDED WITHIN A 5% THRESHOLD.
REGRESSION VERDICTS MUST BE LOGGED AS STRICTLY TYPED, SCHEMA-VALIDATED REPORTS.
PROMPT TONE AND PERSONA ALIGNMENT MUST BE CONTINUOUSLY VERIFIED BY DECOUPLED JUDGE MODELS.
11. VERSION HISTORY & ARCHITECTURAL VERDICT
Version
Date
Description
Author
Status
1.0.0
August 2026
Initial specification of Prompt Regression & Semantic Drift Tests (TEST-SPEC-17).
Ramy Bella
APPROVED FOR IMPLEMENTATION

ARCHITECTURAL VERDICT:
APPROVED FOR IMPLEMENTATION TEST-SPEC-17 establishes the prompt regression pipeline, SemanticDriftReport@1.0.0 contract, semantic distance boundaries, and Stage 3 pipeline gates for Phase 6. Ready for implementation
