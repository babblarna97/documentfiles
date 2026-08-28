PE-SPEC-17: Prompt Evaluation & Testing
1. DOCUMENT CONTROL
| Attribute | Value |
|---|---|
| Document Title | 17 Prompt Evaluation & Testing.md |
| Document ID | PE-SPEC-17 |
| Version | 1.0.1 |
| Status | DRAFT / Implementation Specification |
| Author | Ramy Bella |
| Classification | Confidential / Enterprise Proprietary |
| Target Audience | AI Evaluation Architects, QA Architects, Prompt Engineers, Red-Team Engineers, Reliability Architects |
| Parent Document | PE-SPEC-01 |
| Related Documents | PE-SPEC-04 through PE-SPEC-16, CE-SPEC-01 through CE-SPEC-12 |
| System | Restaurant AI System |
| Phase | Phase 4 — Prompt Engineering |
| Lifecycle Folder | LIFECYCLE |
| Last Updated | August 2026 |
2. EXECUTIVE PURPOSE
The probabilistic nature of Large Language Models introduces systemic risk if prompt changes, model version updates, or context schema modifications are deployed without mathematical and behavioral verification. The Prompt Evaluation & Testing Architecture (PE-SPEC-17) defines the deterministic evaluation framework used to verify that prompt artifacts perform correctly, safely, and consistently across releases.
This specification operates on a core architectural invariant:
PROMPT EVALUATION \neq PROMPT EXECUTION \neq BUSINESS AUTHORITY.
PE-SPEC-17 determines whether prompt artifacts and prompt executions meet defined quality, safety, structural, grounding, and behavioral requirements before and during controlled evaluation. It distinguishes clearly between structural validation (binary checks), behavioral evaluation (semantic adherence), and security/safety evaluation (adversarial resistance), providing the immutable evidence required for release gating.
3. PURPOSE AND SCOPE
Scope: What PE-SPEC-17 Controls
 * Evaluation Architecture: The orchestration of test suites against prompt releases.
 * Test Case & Dataset Models: The canonical structure of versioned golden datasets.
 * Reference Expectations: Exact, structural, and semantic evaluation matching rules.
 * Evaluation Methodologies: Deterministic validators vs. bounded LLM-as-a-judge pipelines.
 * Testing Categories: Regression, Adversarial (Red-Team), Safety, Security, Grounding, and Tool-use evaluation.
 * Evaluation Thresholds & Release Gates: The signals determining if a release candidate is viable.
 * Test Reproducibility: Ensuring evaluations yield consistent results given identical inputs.
 * Evaluation Provenance & Observability: Immutable auditing of evaluation runs.
Scope: What PE-SPEC-17 Explicitly Does NOT Control
 * Prompt compilation mechanics — Owned by PE-SPEC-04.
 * Context retrieval & grounding execution — Owned by PE-SPEC-05.
 * Blueprint assembly — Owned by PE-SPEC-06.
 * Variable hydration — Owned by PE-SPEC-07.
 * Version lifecycle & production activation — Owned by PE-SPEC-10.
 * Security & Data Boundary definition — Owned by PE-SPEC-11 & PE-SPEC-12.
 * Tool capability definitions & output contracts — Owned by PE-SPEC-13 & PE-SPEC-14.
 * Error recovery — Owned by PE-SPEC-15.
 * Safety policy ownership — Owned by PE-SPEC-16.
 * Business authority — Owned by Phase 3.
PE-SPEC-17 evaluates these systems; it does not replace them.
4. ARCHITECTURAL POSITION
PE-SPEC-17 sits alongside the prompt lifecycle pipeline, acting as the verification engine that produces release-gate signals.
[PROMPT ARTIFACT / RELEASE CANDIDATE]
                     ↓
[PE-SPEC-17: EVALUATION ORCHESTRATOR]
                     ↓
         [Structural Validation]
         [Deterministic Test Suite]
         [Behavioral / Semantic Evaluation]
         [Security + Safety + Grounding Eval]
         [Regression Comparison]
                     ↓
           [PASS / FAIL / BLOCKED]
                     ↓
[PE-SPEC-10: RELEASE GOVERNANCE / LIFECYCLE]

Boundary Rule: PE-SPEC-17 produces evaluation evidence and release-gate signals. PE-SPEC-17 MUST NOT directly activate or deploy production versions.
5. EVALUATION MODEL
Every evaluation execution against a prompt candidate MUST be defined by a canonical logical EvaluationCase.
{
  "evaluation_id": "eval_8891-abc",
  "evaluation_version": "1.0.0",
  "test_case_id": "tc_booking_missing_time_01",
  "category": "BEHAVIORAL",
  "input": "I need a table for 4 on Friday.",
  "authoritative_context": {"venue_id": "v_123", "intent": "BOOKING_CREATE"},
  "expected_behavior": "Prompt guest for target_time",
  "expected_output_contract": "CLARIFICATION_REQUEST",
  "expected_tool_behavior": "NONE",
  "safety_class": "NORMAL",
  "security_class": "STANDARD",
  "tenant_scope": "v_123",
  "session_scope": "MOCK_SESSION_01",
  "prompt_release_id": "restaurant_booking_release_2026_08_001",
  "required_invariants": ["No hallucinated booking ID", "additionalProperties: false"],
  "evaluation_method": "STRUCTURAL",
  "threshold": "100%",
  "severity": "HIGH",
  "provenance": "QA_AUTOMATION"
}

Note: The exact implementation format may vary (e.g., stored in a specialized eval database), but the logical contract is normative.
6. TEST CATEGORIES
The evaluation suite MUST cover multiple deterministic categories:
 * Structural Tests: Missing blocks, invalid slots, invalid templates, invalid routing, invalid output contracts, unresolved conditionals, version mismatches.
 * Grounding Tests: Missing fact handling, contradictory facts, wrong tenant facts, stale facts, unsupported claims, hallucination resistance.
 * Behavioral Tests: Correct response behavior, instruction adherence, multi-intent handling, clarification behavior, refusal behavior.
 * Tool Tests: Correct tool selection, no hallucinated tools, correct parameters, no unauthorized tools, safe handling of failed tool execution, no false success claims.
 * Security Tests: Direct/indirect prompt injection, role spoofing, secret extraction, tool privilege escalation, tenant isolation attacks.
 * Safety Tests: Harmful requests, medical safety, allergy safety, emergency framing, false certainty, unsafe instructions.
 * Regression Tests: Preservation of existing approved behavior, ensuring known bugs remain fixed, and verifying upgrades do not unexpectedly alter critical invariants.
7. TEST CASE MODEL & EXPECTATION LEVELS
A test case defines the input and the criteria for success. The expected output MUST NOT always require an exact literal response when multiple semantic responses are equally valid.
PE-SPEC-17 defines three explicit expectation levels:
 * EXACT: An exact structural or literal string match is required. (Used for JSON schema validation, specific emergency handoff phrasing, or IDs).
 * STRUCTURAL: Schema, type, and field correctness are required, but the underlying text values may vary stochastically.
 * SEMANTIC: Behavior must satisfy defined invariants without requiring identical wording. (e.g., "The model must decline the request politely," regardless of whether it says "I apologize" or "I am sorry").
8. DETERMINISTIC VS SEMANTIC EVALUATION
This is a critical architectural boundary in evaluation engineering.
Deterministic Tests:
These validators operate via code (AST parsing, JSON schema validation, regex) and MUST be used to validate: JSON schemas, types, enums, identifiers, versions, checksums, tenant/session bindings, prohibited fields, tool IDs, required fields, and state consistency.
Semantic Tests (LLM-as-a-Judge):
These validators use an auxiliary evaluator LLM to assess: helpfulness, instruction adherence, grounding accuracy, refusal tone, and safety framing.
LLM-as-a-Judge Constraints:
 * LLM-as-a-judge MUST NOT become the authoritative business validator.
 * Their role is evaluation-only; they CANNOT change business state or activate releases.
 * Their judgment MUST be bounded by explicit evaluation criteria/rubrics.
 * High-severity security/safety assertions (e.g., "Did the model output a secret?") SHOULD preferably rely on deterministic verification wherever technically possible, rather than relying solely on a secondary LLM to detect the secret.
9. GOLDEN DATASETS / EVALUATION DATASETS
Prompt evaluation relies on versioned, immutable evaluation datasets.
 * A dataset MUST contain: dataset_id, version, test_cases, provenance, expected_behavior, tenant_assumptions, safety/security_metadata, and change_history.
 * Production prompt releases MUST be evaluated against the explicitly pinned dataset version.
 * Prohibition: There is NO implicit "latest" evaluation dataset. The pairing of Prompt Release Candidate + Dataset Version must be explicit and recorded.
10. GOLDEN / REFERENCE EXPECTATIONS
Reference expectations MUST be designed for durability across model updates.
 * GOOD Expectation: "Must not claim booking confirmed before runtime SUCCESS flag is present." (Verifiable structurally via output contract).
 * BAD Expectation: "Must respond exactly: 'Your table is confirmed.'" (Brittle; will break if the model adds "Great news!").
Exact natural-language matching MUST NOT be used unless exactness is truly required by an overarching specification (e.g., specific legal disclaimers defined in PE-SPEC-16).
11. EVALUATION METRICS
The orchestrator MUST aggregate measurable metrics, including but not limited to:
 * Pass rate (Overall)
 * Critical failure rate
 * Hallucination / Un-grounded rate
 * Tool-call validity rate
 * False-confirmation rate
 * Safety / Security violation rate
 * Output contract compliance rate
 * Regression rate
 * Tenant isolation pass rate
 * Latency bounds (where applicable)
Constraint: Aggregate pass rates NEVER override severity-based release gates. Critical security, safety, data-boundary, tenant, authorization, or equivalent critical failures are release-blocking regardless of aggregate pass rate. Release gating is determined by configured thresholds AND severity overrides. A high overall pass rate cannot mask a Critical failure.
12. SEVERITY MODEL
Failures during evaluation carry explicit severities that dictate release gate behavior:
 * CRITICAL: Security bypass, cross-tenant leak, PCI/Secret exposure, fabricated business success, unsafe medical certainty, unauthorized tool execution pathway.
 * HIGH: Output contract violation (malformed JSON), severe hallucination, instruction ignorance causing workflow failure.
 * MEDIUM: Tone issues, verbose responses, minor unhelpful clarifications.
 * LOW: Formatting deviations that do not break downstream parsers.
 * INFO: Latency warnings or metadata tracking.
Rule: A single CRITICAL evaluation failure MUST fail/block a release candidate.
13. REGRESSION TESTING
Prompt versions MUST be evaluated against previous approved baselines to detect regressions.
 * Comparison MUST include: Structural differences, Behavioral differences, Safety/Security differences, Tool behavior, Output contract adherence, and Grounding behavior.
 * Rule: Not all behavioral differences are regressions. A change in expected behavior MUST be evaluated; if it is intentional, it MUST be documented and linked to a version/change reason in PE-SPEC-10.
14. VERSION / RELEASE EVALUATION
Integrating with PE-SPEC-10 (Versioning):
A complete evaluation record for a release candidate MUST include:
 * prompt_release_id
 * Artifact versions (Templates, Routes, Variables)
 * Evaluation dataset version
 * Evaluation suite/runner version
 * Evaluation results & metrics
 * Checksum references
PE-SPEC-17 packages this evidence. PE-SPEC-10 consumes this evidence to execute lifecycle promotion.
15. RED-TEAM / ADVERSARIAL EVALUATION
The architecture MUST support structured adversarial evaluation (Red-Teaming) to test PE-SPEC-11 and PE-SPEC-12 defenses.
Coverage MUST include:
Direct/Indirect prompt injection, malicious guest notes, tool-result poisoning, role impersonation, system prompt extraction, secret extraction, cross-tenant/cross-session probing, unauthorized tool requests, safety bypass attempts, false authority claims, and version manipulation attempts.
Red-Team Test Structure:
Each test MUST define: attack_id, attack_category, payload, expected_defense (e.g., FAIL_CLOSED), severity, evaluation_result, evidence, and dataset_reference.
16. GROUNDING EVALUATION
Grounding evaluations test the prompt's adherence to PE-SPEC-05 (Context Injection).
Implementation-Grade Scenarios:
 * Fact exists \rightarrow Model uses it correctly.
 * Fact absent \rightarrow Model refuses/admits lack of information.
 * User claim contradicts authoritative fact \rightarrow Authoritative fact wins.
 * Wrong venue fact injected \rightarrow Must fail isolation.
 * Stale state injected \rightarrow Must not be treated as current.
 * Conflicting facts \rightarrow Ambiguity policy is respected (e.g., Clarification).
17. TOOL EVALUATION
Evaluating PE-SPEC-13 and PE-SPEC-14 compliance:
Tests MUST evaluate: Tool availability vs. authorization, correct tool selection, parameter validity against strict schemas, missing required fields, unauthorized tool rejection, fabricated/hallucinated tools, duplicate state-changing proposals, false success claims, and appropriate error classification.
18. SAFETY EVALUATION
Evaluating PE-SPEC-16 compliance:
Tests MUST evaluate: Harmful request refusal (tone and success), medical non-diagnosis boundaries, allergy uncertainty framing, absence of absolute safety guarantees, emergency handoff framing, lack of fabricated emergency action, correct safety policy selection, and safety override resistance.
19. SECURITY EVALUATION
Evaluating PE-SPEC-11 and PE-SPEC-12 compliance:
Tests MUST evaluate: Prompt injection resistance, indirect injection resistance via RAG, secret non-disclosure, role spoofing resistance, tenant/session isolation, unauthorized tool proposals, checksum tampering, version substitution attempts, and data exfiltration attempts.
20. HUMAN EVALUATION
While deterministic and LLM-as-a-judge automation drives CI/CD, human evaluation remains necessary for nuanced semantic quality.
 * Humans MAY evaluate: Tone, hospitality, clarity, naturalness, edge-case quality, and ambiguous semantic boundaries.
 * Humans MUST NOT override deterministic security, tenant, authorization, or structural validators. (A human cannot "pass" a prompt that fails a structural PCI check).
 * Human evaluations MUST be attributable, versioned, and auditable within the evaluation record.
21. EVALUATION REPRODUCIBILITY
Given identical:
 * Prompt Release ID
 * Evaluation Dataset Version
 * Test Configuration
 * Authoritative Context
 * Evaluator Version
 * Model/Configuration (where relevant)
The evaluation run MUST be reproducible within the defined stochastic tolerance of the underlying LLM.
Architectural Distinction:
The architecture distinguishes between deterministic structural evaluation (which must match exactly) and probabilistic semantic evaluation. Any stochastic tolerance used for semantic evaluation MUST itself be explicitly configured, versioned, and recorded as part of the evaluation configuration. The system does not claim that LLM outputs are mathematically identical across repeated runs; rather, it ensures that behavior remains within the explicitly configured stochastic tolerance. Exact model versions (e.g., gpt-4-0613) must be recorded to ensure reproducibility of semantic evaluations.
22. EVALUATION RESULT MODEL
The output of an evaluation run MUST be encapsulated in a canonical EvaluationResult object.
{
  "evaluation_run_id": "run_20260813_001",
  "evaluation_case_id": "eval_8891-abc",
  "prompt_release_id": "restaurant_booking_release_2026_08_001",
  "dataset_version": "ds_eval_v2.1",
  "evaluator_version": "eval_engine_v1.4",
  "model_version": "gpt-4-0613",
  "result": "PASS",
  "severity": "NONE",
  "metrics": {"latency_ms": 412, "tokens_used": 150},
  "failure_codes": [],
  "evidence_reference": "s3://eval-evidence/run_20260813_001/eval_8891-abc.json",
  "timestamp": "2026-08-13T12:10:00Z",
  "provenance": "CI_CD_PIPELINE",
  "tenant_scope": "GLOBAL"
}

Immutability: Results MUST be immutable after finalization. If an evaluation was flawed, a new evaluation run is generated. Historical evidence is NEVER silently mutated.
23. RELEASE GATES
Deterministic release-gate behavior dictates promotion viability.
 * PASS: All deterministic and semantic tests meet thresholds. Zero CRITICAL or HIGH failures.
 * PASS_WITH_REVIEW: MEDIUM/LOW failures exist, requiring human sign-off.
 * FAIL: Deterministic thresholds missed, or HIGH severity failures present.
 * BLOCKED: Any CRITICAL security, safety, tenant, or authorization failure.
Rule: Critical security/safety failures MUST produce FAIL or BLOCKED regardless of aggregate evaluation scores. There is NO automatic production activation; PE-SPEC-10 requires an explicit promotion trigger.
24. FAILURE ARCHITECTURE
Deterministic PE-SPEC-17 error codes generated when the evaluation pipeline itself fails:
| Failure ID | Condition | System Response | Severity |
|---|---|---|---|
| ERR_EVAL_01 | Invalid or missing evaluation dataset. | Abort Evaluation | Critical |
| ERR_EVAL_02 | Evaluator engine version mismatch. | Abort Evaluation | High |
| ERR_EVAL_03 | Incomplete evaluation run (timeout/crash). | Mark RUN INVALID | High |
| ERR_EVAL_04 | Critical Security Regression detected. | Gate = BLOCKED | Critical |
| ERR_EVAL_05 | Critical Safety Regression detected. | Gate = BLOCKED | Critical |
| ERR_EVAL_06 | Critical Grounding/Hallucination failure. | Gate = FAIL | High |
| ERR_EVAL_07 | Non-reproducible / Stale baseline used. | Mark RUN INVALID | Medium |
| ERR_EVAL_08 | Checksum mismatch on prompt candidate. | Abort Evaluation | Critical |
| ERR_EVAL_09 | Invalid or missing evaluation evidence. | Gate = BLOCKED | High |
All critical evaluation integrity failures MUST fail closed at the evaluation/release-gate level.
25. OBSERVABILITY / AUDIT
Audit records of evaluations MUST be securely retained.
Required Audit Fields:
evaluation_run_id, test_case_id, prompt_release_id, dataset_version, evaluator_version, model_version, result, severity, failure_codes, metric_summary, evidence_reference, timestamp, correlation_id.
Strict Prohibition: PE-SPEC-17 MUST NOT materialize, persist, expose, or unnecessarily reproduce PCI, secrets, or other prohibited sensitive data in evaluation evidence, logs, artifacts, or reports. The system MUST use references or identifiers instead of raw sensitive values wherever possible. (Malicious payloads injected during red-teaming should be referenced by test-case ID rather than dumped in plaintext into logging streams).
26. PRODUCTION ACCEPTANCE CRITERIA
| AC ID | Category | Requirement | Verification Method | Expected Result | Pass/Fail | Severity |
|---|---|---|---|---|---|---|
| AC-EVAL-01 | Schema Validation | Evaluator deterministically fails outputs missing required fields or having wrong types. | Structural Mock | FAIL | Required | Critical |
| AC-EVAL-02 | Grounding | Evaluator fails outputs that invent facts absent from authoritative_context. | Hallucination Mock | FAIL | Required | High |
| AC-EVAL-03 | False Confirm | Evaluator blocks releases where the model fabricates a BUSINESS_CONFIRMATION without runtime success. | Logic Mock | BLOCKED | Required | Critical |
| AC-EVAL-04 | Tool Correctness | Evaluator detects and fails undeclared or unauthorized tool proposals. | Schema Mock | BLOCKED | Required | Critical |
| AC-EVAL-05 | Safety Refusal | Evaluator verifies the model correctly refuses harmful prompts using PE-SPEC-16 policies. | Red-Team Eval | PASS | Required | Critical |
| AC-EVAL-06 | Medical Boundary | Evaluator detects and fails absolute allergy guarantees or medical diagnoses. | Safety Mock | BLOCKED | Required | Critical |
| AC-EVAL-07 | Prompt Injection | Evaluator tests indirect injection via RAG; verifies structural fence holds. | Security Eval | PASS | Required | Critical |
| AC-EVAL-08 | Tenant Isolation | Evaluator fails responses that blend data from disparate tenant_scopes. | Privacy Mock | BLOCKED | Required | Critical |
| AC-EVAL-09 | Data Leakage | Evaluator verifies PCI/Secrets are structurally absent from output logs/results, and that raw PCI/secrets are prevented from being materialized or reproduced in any evaluation evidence or artifact. | Secret Scan | PASS | Required | Critical |
| AC-EVAL-10 | Regression | Evaluator accurately flags deviations from pinned baseline expectations. | Version Compare | Regression Flagged | Required | High |
| AC-EVAL-11 | Version Pinning | Evaluations fail if run against an unpinned ("latest") dataset. | Setup Mock | ERR_EVAL_01 | Required | Critical |
| AC-EVAL-12 | Reproducibility | Repeating the exact eval run parameters yields the exact same metric output within the explicitly configured stochastic tolerance. | Duplication Test | Matching Metrics | Required | High |
| AC-EVAL-13 | Release Blocking | A single CRITICAL failure forces a BLOCKED release gate, regardless of overall pass rate. | Math Mock | Gate = BLOCKED | Required | Critical |
| AC-EVAL-14 | No Auto-Release | A 100% PASS eval run produces evidence but does not automatically mutate production state. | Pipeline Auth Test | State unchanged | Required | Critical |
| AC-EVAL-15 | Immutability | Evaluation results cannot be altered or overwritten once finalized. | DB Mutation Test | Mutate Rejected | Required | Critical |
27. INTEGRATION CONTRACTS
PE-SPEC-17 strictly defines evaluation criteria for the surrounding architecture:
 * PE-SPEC-04 / 06 (Assembly & Compilation): Evaluates if the blueprint generates correct payloads without compilation errors.
 * PE-SPEC-05 (Context): Evaluates the prompt's grounding accuracy against injected RAG facts.
 * PE-SPEC-07 (Variables): Evaluates if slotted data is correctly interpreted and protected.
 * PE-SPEC-08 / 09 (Templates & Routing): Evaluates behavioral outcomes of specific template graphs.
 * PE-SPEC-10 (Versioning): Provides the release candidate to PE-17; consumes the PASS/BLOCKED evidence for lifecycle promotion.
 * PE-SPEC-11 / 12 (Security & Data): Defines the critical invariants PE-17 must adversarial-test against.
 * PE-SPEC-13 / 14 (Tools & Outputs): Defines the exact JSON schemas PE-17 uses for structural validation tests.
 * PE-SPEC-15 (Error Handling): Evaluates if the system correctly recovers or fails-closed under simulated stress.
 * PE-SPEC-16 (Safety): Evaluates adherence to medical, allergy, and harm-prevention policies.
 * Phase 3 / Runtime: Provides the simulated business state for the evaluation context.
28. VERSION HISTORY
| Version | Date | Description | Author | Approval Status |
|---|---|---|---|---|
| 1.0.0 | August 2026 | Initial Prompt Evaluation & Testing specification. Established deterministic evaluation architecture, golden dataset pinning, separation of structural vs semantic evaluation, red-team coverage, and strict release-gating based on critical failure thresholds. | Ramy Bella | DRAFT / Implementation Specification |
| 1.0.1 | August 2026 | Targeted consistency fixes: explicitly stated that aggregate pass rates cannot override severity-based release gates; enforced sequential section numbering; mandated explicit versioned configuration for semantic stochastic tolerance; hardened prohibitions against materializing PCI/secrets in evaluation evidence; updated AC-EVAL-09. | Ramy Bella | DRAFT / Implementation Specification |
29. FINAL NON-NEGOTIABLE PRINCIPLES
 * PE-SPEC-17 EVALUATES; IT DOES NOT EXECUTE BUSINESS LOGIC.
 * SECURITY AND SAFETY CRITICAL FAILURES CANNOT BE AVERAGED AWAY.
 * DETERMINISTIC VALIDATORS REMAIN AUTHORITATIVE FOR STRUCTURAL INVARIANTS.
 * LLM-AS-A-JUDGE IS EVALUATION SUPPORT, NOT BUSINESS AUTHORITY.
 * EVALUATION DATASETS MUST BE VERSIONED AND PINNED.
 * REGRESSION TESTING MUST BE RELEASE-AWARE.
 * CRITICAL SECURITY, SAFETY, DATA, AND AUTHORIZATION REGRESSIONS MUST BLOCK RELEASES.
 * EVALUATION RESULTS MUST BE IMMUTABLE AND AUDITABLE.
 * PE-SPEC-17 MUST NOT MODIFY PRODUCTION STATE.
 * PE-SPEC-10 REMAINS THE RELEASE/LIFECYCLE AUTHORITY.
 * PHASE 3 REMAINS THE BUSINESS AUTHORITY.
ARCHITECTURAL VERDICT
APPROVED — DRAFT / IMPLEMENTATION SPECIFICATION
Implementation Readiness Statement:
READY FOR ENTERPRISE IMPLEMENTATION. PE-SPEC-17 establishes a rigorous, deterministic, and fail-closed quality assurance architecture. By strictly delineating structural validation from semantic LLM-as-a-judge evaluation, and by enforcing that a single critical security, safety, or data-boundary failure mathematically blocks release promotion, this specification ensures that all prompt artifacts entering production are comprehensively vetted, fully reproducible, and structurally safe. It cleanly hands off final promotion authority to PE-SPEC-10, preserving perfect separation of concerns.
