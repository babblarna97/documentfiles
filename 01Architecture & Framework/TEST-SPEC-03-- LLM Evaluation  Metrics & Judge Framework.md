# TEST-SPEC-03: LLM Evaluation Metrics & Judge Framework

## 1. DOCUMENT CONTROL

| Attribute | Value |
|---|---|
| Document Title | 03 LLM Evaluation Metrics & Judge Framework.md |
| Document ID | TEST-SPEC-03 |
| Version | 1.0.1 |
| Status | APPROVED FOR IMPLEMENTATION |
| Author | Ramy Bella |
| Classification | Confidential / Enterprise Proprietary |
| Target Audience | AI Engineers, QA Architects, NLP Specialists, Security Leads, MLOps Engineers |
| Parent Document | TEST-SPEC-01 |
| Related Documents | Phase 1–5 Specifications, TEST-SPEC-01, TEST-SPEC-02, TEST-SPEC-12 through 17 |
| System | Restaurant AI System |
| Phase | Phase 6 — Validation & Verification (V&V) |
| Lifecycle Folder | 01 Architecture & Framework |
| Last Updated | August 2026 |

---

## 2. EXECUTIVE PURPOSE

Evaluating Generative AI outputs purely through subjective human inspection or naive keyword matching is mathematically unviable at scale. Conversely, relying on uncalibrated "LLM-as-a-Judge" setups creates circular self-enhancement bias, position-bias distortions, and false confidence.

`TEST-SPEC-03` defines the quantitative **LLM Evaluation & Metrics Framework**. It establishes the mathematical scoring formulas, multi-judge ensemble architecture, calibration standards, and automated evaluation pipeline governing all probabilistic assessments across Phase 6. It guarantees that probabilistic conversational outputs are measured against rigorous, reproducible, and mathematically sound SLAs prior to release.

### Core Evaluation Invariants:
* `GENERATOR MODEL == JUDGE MODEL FAMILY \implies INVALID EVALUATION`
* `ALLERGEN / SAFETY HALLUCINATION \implies ZERO TOLERANCE (Score = 0.00)`
* `INTER-JUDGE AGREEMENT KAPPA (\kappa) < 0.85 \implies EVALUATION INVALIDATION`
* `DETERMINISTIC ASSERTION FAILURE \implies SHORT-CIRCUIT EVALUATION PIPELINE`
* `PROBABILISTIC SLA VIOLATION \implies STAGING PROMOTION BLOCK`

---

## 3. EVALUATION METRICS SUITE & MATHEMATICAL FORMULAS

The framework combines deterministic metrics, embedding-based semantic distance, and structured probabilistic judge rubrics. Every evaluation turn generates a composite score vector.

### 3.1. Faithfulness & RAG Grounding Score ($S_{\text{faith}}$)
Measures the extent to which claims made in the AI response ($R$) are factually supported by the retrieved Knowledge Base context ($C$).

$$S_{\text{faith}} = \frac{\vert{}\text{Claims in } R \text{ verified by } C\vert{}}{\vert{}\text{Total Claims extracted from } R\vert{}}$$

* **Target Threshold:** $S_{\text{faith}} \ge 0.99$ ($1.00$ required for allergen/menu assertions).

### 3.2. Context Recall ($S_{\text{recall}}$)
Measures whether the Knowledge Base retrieval pipeline fetched all context ($C$) necessary to answer the ground truth user prompt ($Q$).

$$S_{\text{recall}} = \frac{\vert{}\text{Ground Truth Claims present in } C\vert{}}{\vert{}\text{Total Ground Truth Claims required by } Q\vert{}}$$

* **Target Threshold:** $S_{\text{recall}} \ge 0.95$.

### 3.3. Semantic Embedding Distance ($S_{\text{sem}}$)
Measures the cosine similarity between the vector representation of the generated output ($\mathbf{e}_{\text{response}}$) and the golden reference response ($\mathbf{e}_{\text{reference}}$).

$$S_{\text{sem}} = \cos(\mathbf{e}_{\text{response}}, \mathbf{e}_{\text{reference}}) = \frac{\mathbf{e}_{\text{response}} \cdot \mathbf{e}_{\text{reference}}}{\Vert{}\mathbf{e}_{\text{response}}\Vert{} \Vert{}\mathbf{e}_{\text{reference}}\Vert{}}$$

* **Target Threshold:** $S_{\text{sem}} \ge 0.92$.

### 3.4. Intent & Slot Extraction Micro-$F_1$ Score ($F_1^{\text{slots}}$)
Evaluates exact-match precision and recall for extracted structured slots (e.g., `reservation_time`, `party_size`, `allergen_list`).

$$F_1^{\text{slots}} = 2 \times \frac{\text{Precision} \times \text{Recall}}{\text{Precision} + \text{Recall}}$$

* **Target Threshold:** $F_1^{\text{slots}} \ge 0.98$.

### 3.5. Hallucination Rate ($H_{\text{rate}}$)
Quantifies ungrounded or contradictory statements present in the response text.

$$H_{\text{rate}} = 1.0 - S_{\text{faith}}$$

* **Target Threshold:** $H_{\text{rate}} = 0.00$ for safety/allergy data; $H_{\text{rate}} \le 0.02$ for general conversation.

---

## 4. LLM-AS-A-JUDGE ARCHITECTURE & CALIBRATION

To eliminate model bias, the evaluation engine enforces strict isolation between the Generator LLM (the candidate model being tested) and the Evaluator Judge LLM.


[CANDIDATE GENERATOR LLM (e.g., GPT-4o)]
                 │
                 ▼ Generates Response (R)
=====================================================
[TEST-SPEC-03 EVALUATION PIPELINE]
  ├─ Step 1: Deterministic Pre-Filter (JSON / Regex)
  ├─ Step 2: Vector Embedding Cosine Distance
  └─ Step 3: Decoupled Judge Ensemble
                 │
                 ├── Judge A: Non-Homologous Model (e.g., Claude 3.5 Sonnet)
                 └── Judge B: Cross-Family Audit Model
=====================================================
                 │
                 ▼ Structured JSON Evaluation
[CALIBRATION & AGREEMENT CHECK (\kappa >= 0.85)]
                 │
                 ▼
[AGGREGATED EVALUATION SCORE VECTOR]


4.1. Non-Homology & Cross-Family Isolation Rules
Homology Prohibition: A model family MUST NOT judge its own outputs (e.g., OpenAI models CANNOT judge OpenAI outputs).
Mandatory Pairing Matrix:
Candidate Generator = OpenAI GPT-4o \implies Evaluator Judge = Anthropic Claude 3.5 Sonnet (or calibrated open-weights ensemble).
Candidate Generator = Anthropic Claude 3.5 \implies Evaluator Judge = OpenAI GPT-4o.
4.2. Judge Bias Mitigation Protocols
Position-Bias Inversion: Multiple-choice or comparative evaluation prompts execute twice with swapped input order (A/B vs B/A). If scores diverge by > 5\%, the evaluation turn is flagged as inconsistent.
Verbosity-Normalization: Judge rubrics strictly penalize fluff. Evaluation prompts enforce length-normalized scoring rubrics to prevent verbose responses from receiving artificially high scores.
Structured JSON Output: Judges MUST respond strictly using a validated JSON Schema. Unstructured narrative feedback is rejected.
4.3. Judge Calibration & Inter-Annotator Agreement (\kappa)
Before a Judge model configuration is deployed to the CI/CD pipeline, it MUST be calibrated against a Human-Curated Calibration Dataset (N \ge 500 annotated turns).
Agreement Metric: Cohen’s Kappa (\kappa) or Fleiss’ Kappa (for multi-judge ensembles).
Calibration Threshold: \kappa \ge 0.85. If \kappa < 0.85, the judge system prompt and rubric weights MUST be recalibrated prior to pipeline execution.
5. EVALUATION EXECUTION PIPELINE & RUBRIC CONTRACT
The evaluation engine executes sequentially. Failure at a higher-priority deterministic stage instantly halts execution to save compute budget.
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "title": "LLMJudgeEvaluationOutput@1.0.0",
  "type": "object",
  "properties": {
    "turn_id": { "type": "string" },
    "evaluator_model": { "type": "string" },
    "deterministic_assertions_passed": { "type": "boolean" },
    "scores": { "type": "object",
      "properties": {
        "faithfulness": { "type": "number", "minimum": 0.0, "maximum": 1.0 },
        "context_recall": { "type": "number", "minimum": 0.0, "maximum": 1.0 },
        "semantic_distance": { "type": "number", "minimum": 0.0, "maximum": 1.0 },
        "hallucination_rate": { "type": "number", "minimum": 0.0, "maximum": 1.0 },
        "policy_compliance": { "type": "number", "minimum": 0.0, "maximum": 1.0 }
      },
      "required": ["faithfulness", "context_recall", "semantic_distance", "hallucination_rate", "policy_compliance"]
    },
    "reasoning_trace": { "type": "string" },
    "verdict": { "type": "string", "enum": ["PASS", "FAIL", "REQUIRES_HUMAN_REVIEW"] }
  },
  "required": ["turn_id", "evaluator_model", "deterministic_assertions_passed", "scores", "verdict"]
}


6. SECURITY & THREAT MODEL
Threat
Attack Vector
Preventive Control
Severity
Judge Manipulation Payload
Adversarial input in user prompt tricks the Evaluator Judge into awarding 1.0 score.
Input sanitization; user text is escaped within strict XML data blocks (<user_input>) in judge prompt.
CRITICAL
Circular Self-Enhancement
Generator and Judge use same base model family, ignoring subtle semantic hallucinations.
Strict cross-family non-homology pairing enforced at runtime by TEST-SPEC-03.
HIGH
Judge Model Behavior Drift
Provider updates judge API version, silently changing scoring calibration.
Pin exact judge model snapshots; re-verify against Human Calibration Dataset on every pipeline run.
HIGH
Evaluator Overfitting
Prompts engineered specifically to trick judge rubrics without improving quality.
Dual evaluation combining probabilistic judge scores with strict vector distance (S_{\text{sem}}).
HIGH

7. FAILURE ARCHITECTURE & ERROR TAXONOMY
Error Code
Description
Canonical Class
Severity
ERR_TEST_03_01
Evaluator Judge calibration agreement score below threshold (\kappa < 0.85).
VALIDATION
CRITICAL
ERR_TEST_03_02
Allergen or Safety Hallucination detected (H_{\text{rate}} > 0.00 on safety context).
SAFETY_VIOLATION
CRITICAL
ERR_TEST_03_03
Candidate Generator and Evaluator Judge model homology conflict detected.
CONFIG_ERROR
HIGH
ERR_TEST_03_04
Judge evaluation output failed JSON Schema validation (LLMJudgeEvaluationOutput@1.0.0).
CONTRACT_MISMATCH
HIGH
ERR_TEST_03_05
Position-bias inversion check failed (> 5\% variance between swap runs).
PROBABILISTIC_DRIFT
MEDIUM

8. PRODUCTION ACCEPTANCE CRITERIA
AC ID
Category
Requirement
Verification Method
Pass/Fail
AC-EVL-01
Homology Isolation
Evaluator judge model shares zero model family architecture with generator model.
Runtime Pipeline Audit
REQUIRED
AC-EVL-02
Zero Allergen Hallucination
S_{\text{faith}} = 1.00 (H_{\text{rate}} = 0.00) enforced across all food safety/allergy test cases.
Deterministic & Judge Audit
REQUIRED
AC-EVL-03
Judge Calibration
Judge configuration exhibits \kappa \ge 0.85 agreement with human calibration dataset.
Calibration Report Audit
REQUIRED
AC-EVL-04
Schema Enforcement
100\% of judge execution outputs validate against LLMJudgeEvaluationOutput@1.0.0.
Schema Inspector
REQUIRED
AC-EVL-05
Short-Circuit Evaluation
Deterministic assertion failures bypass judge execution, returning immediate FAIL verdict.
Fast-Fail Execution Log
REQUIRED

9. INTEGRATION AUTHORITY MATRIX
Component
Owns
Consumes
Produces
TEST-SPEC-03
Metric Formulas, Judge Rubrics, Calibration Engine, JSON Eval Schemas
Golden Datasets (TEST-SPEC-01/20), Candidate Responses
Metric Vector Scores, LLM Eval Reports
TEST-SPEC-01
Overall V&V Strategy & Tolerances
SLA Threshold Requirements
System Strategy Alignment
TEST-SPEC-02
CI/CD Stage Gate Enforcement
Metric Vector Scores (TEST-SPEC-03)
Signed Evidence Bundles / Build Blocks
TEST-SPEC-13
RAG Grounding Test Scenarios
RAG Context & Responses
Grounding Test Outputs

10. FINAL NON-NEGOTIABLE PRINCIPLES
NO GENERATOR LLM SHALL BE EVALUATED BY A JUDGE FROM THE SAME MODEL FAMILY OR VENDOR ARCHITECTURE.
ALLERGEN, SAFETY, AND LEGAL CONSTRAINTS CARRY ZERO TOLERANCE FOR HALLUCINATIONS (H_{\text{rate}} = 0.00).
DETERMINISTIC ASSERTIONS MUST ALWAYS PRE-FILTER AND SHORT-CIRCUIT PROBABILISTIC LLM JUDGE EXECUTION.
EVALUATOR JUDGES MUST BE CONTINUOUSLY CALIBRATED AGAINST HUMAN GROUND TRUTH DATASETS (\kappa \ge 0.85).
ALL JUDGE EVALUATIONS MUST PRODUCE CRYPTOGRAPHICALLY VALIDATED, STRUCTURED JSON SCHEMAS.
11. VERSION HISTORY & ARCHITECTURAL VERDICT
Version
Date
Description
Author
Status
1.0.0
August 2026
Initial specification of LLM Evaluation Metrics & Judge Framework (TEST-SPEC-03).
Ramy Bella
SUPERSEDED
1.0.1
August 2026
Review fix pass. Fixed a malformed $schema URI in LLMJudgeEvaluationOutput@1.0.0 (markdown link syntax had been baked into the JSON string, making it invalid JSON as written) — the same defect class already fixed in TEST-SPEC-12 and TEST-SPEC-16. No change to metric formulas, judge calibration rules, or acceptance criteria.
Ramy Bella
APPROVED FOR IMPLEMENTATION

ARCHITECTURAL VERDICT:
APPROVED FOR IMPLEMENTATION TEST-SPEC-03 v1.0.1 establishes the mathematical evaluation metrics, cross-family judge isolation, bias mitigation protocols, and calibrated evaluation pipeline for Phase 6. Ready for implementation.

