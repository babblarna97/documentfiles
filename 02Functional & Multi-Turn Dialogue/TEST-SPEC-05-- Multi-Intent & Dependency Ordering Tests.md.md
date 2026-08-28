# TEST-SPEC-05: Multi-Intent & Dependency Ordering Tests

## 1. DOCUMENT CONTROL

| Attribute | Value |
|---|---|
| Document Title | 05 Multi-Intent & Dependency Ordering Tests.md |
| Document ID | TEST-SPEC-05 |
| Version | 1.1.0 |
| Status | APPROVED FOR IMPLEMENTATION |
| Author | Ramy Bella |
| Classification | Confidential / Enterprise Proprietary |
| Target Audience | AI Engineers, Conversation Architects, QA Engineers, State Machine Engineers |
| Parent Document | TEST-SPEC-01 |
| Related Documents | Phase 3 Specs (CE-SPEC-09), TEST-SPEC-01, TEST-SPEC-03, TEST-SPEC-04, TEST-SPEC-06, TEST-SPEC-15 |
| System | Restaurant AI System |
| Phase | Phase 6 — Validation & Verification (V&V) |
| Lifecycle Folder | 02 Functional & Multi-Turn Dialogue |
| Last Updated | August 2026 |

---

## 2. EXECUTIVE PURPOSE

Real-world restaurant guests rarely speak in single, isolated commands. A typical user turn often contains compound intents (e.g., *"Har ni glutenfria alternativ och kan jag boka ett bord för 4 ikväll kl 19?"*). Executing these intents without strict dependency management causes state corruption or degraded user experience.

`TEST-SPEC-05` defines the **Multi-Intent Decomposition & Dependency Ordering Test Suite**. It inherits conversation engine semantics directly from `CE-SPEC-09`, establishing execution ordering rules, Directed Acyclic Graph (DAG) resolution algorithms, conditional vs. independent evaluation paths, and conflict resolution mechanisms. It guarantees that compound inputs are decomposed and dispatched in deterministic alignment with Phase 3 state machine specifications.

### Core Testing Invariants:
* `CE-SPEC-09 PARITY \implies CONDITIONAL SAFETY REQUIRES SEQUENTIAL DAG; INDEPENDENT INTENTS EXECUTE IN PARALLEL`
* `CONDITIONAL SAFETY FAILURE \implies SHORT-CIRCUIT DEPENDENT MUTATION NODES`
* `CONFLICTING INTENTS (e.g., CANCEL + MODIFY) \implies MANDATORY CLARIFICATION STEP`
* `SILENT TRUNCATION OF USER INTENTS \implies STRICTLY PROHIBITED (MUST REJECT & CLARIFY)`
* `INTENT OVERCROWDING (N > 3) \implies REJECT_AND_CLARIFY FALLBACK`

---

## 3. DEPENDENCY SEMANTICS: CONDITIONAL VS. INDEPENDENT EXECUTION

In strict compliance with `CE-SPEC-09`, the multi-intent engine distinguishes between **Conditional Dependencies** and **Independent Co-occurrences**.


[COMPOUND USER UTTERANCE]

  Scenario A: CONDITIONAL DEPENDENCY               Scenario B: INDEPENDENT CO-OCCURRENCE
  "Boka bord för 2 kl 19 OM ni har nötfritt."      "Har ni nötter i maten? Förresten, boka bord för 2."
                        │                                                 │
                        ▼                                                 ▼
             ┌─────────────────────┐                           ┌─────────────────────┐
             │ Node 1: ALLERGEN    │                           │ Node 1: ALLERGEN    │
             └──────────┬──────────┘                           │ Node 2: BOOKING     │
                        │ Safe?                                └──────────┬──────────┘
             ┌──────────┴──────────┐                                      │
             ▼                     ▼                                      ▼
     [Node 2: BOOKING]      [CANCEL BOOKING]                 [PARALLEL EXECUTION]
   (Executes if Safe)      (Short-Circuit)               (Safety response prioritized
                                                          in dialogue generation)


3.1. Dependency Classification Engine
Dependency Type
Linguistic Trigger Pattern
DAG Topological Structure
Execution Behavior
Conditional Dependency
"om ni har...", "if safe then...", "förutsatt att..."
Sequential Strict DAG (Node_1 \rightarrow Node_2)
Node_1 (Safety/Read) MUST resolve first. If Node_1 evaluates to UNSAFE, Node_2 (Mutation) is short-circuited and cancelled.
Independent Co-occurrence
"och...", "förresten...", "samt...", or unlinked phrases
Parallel Multi-Node DAG (Node_1 \parallel Node_2)
Both nodes process independently. Safety answers are prioritized in the response prompt, but booking slot collection continues normally.
Mutation Collision
"avboka... nej ändra...", "boka... och avboka..."
Conflicting Mutation Graph
Dispatches directly to CONFLICT_CLARIFICATION state machine node. Zero database mutations executed.

4. CONFLICT RESOLUTION MATRIX
When a user turn contains contradictory or mutually exclusive intents, the system MUST NOT execute conflicting operations.
Intent I_1
Intent I_2
Linkage Type
Resolution Action
Expected System Behavior
BOOKING_CANCEL
BOOKING_MODIFY
Mutually Exclusive
Block Execution
Trigger clarification prompt asking user to select cancellation or modification.
ALLERGEN_QUERY
BOOKING_CREATE
Conditional ("om nötfritt")
Sequential DAG
Evaluate ALLERGEN_QUERY. If unsafe, cancel BOOKING_CREATE and inform guest.
ALLERGEN_QUERY
BOOKING_CREATE
Independent ("har ni nötter? vill boka")
Parallel DAG
Process ALLERGEN_QUERY context AND execute BOOKING_CREATE. Prioritize allergen answer in prompt.
MENU_QUERY
COMPLAINT_ESCALATE
Context Divergence
Preempt Execution
Execute COMPLAINT_ESCALATE immediately; defer MENU_QUERY to human agent.

5. MULTI-INTENT TEST SCENARIOS & EXECUTION MATRIX
The test harness evaluates compound utterances against expected DAG topologies and execution outputs.
5.1. Multi-Intent Test Vector Suite
# Representative Test Harness Structure for Multi-Intent DAG Execution
@pytest.mark.rtm(req_id="NLU-SPEC-05-MULTI-001")
@pytest.mark.parametrize("utterance, expected_dag_type, expected_execution_plan", [
    (
        "Boka ett bord för 2 personer kl 19:00 om ni har glutenfria alternativ.",
        "CONDITIONAL_SEQUENTIAL",
        ["ALLERGEN_QUERY_EVAL", "CONDITIONAL_BOOKING_CREATE"]
    ),
    (
        "Finns det nötter i er pesto? Förresten, kan jag boka ett bord för 2 kl 20?",
        "INDEPENDENT_PARALLEL",
        ["ALLERGEN_QUERY_EXEC", "BOOKING_CREATE_EXEC"]
    ),
    (
        "Avboka min bokning ikväll, och ändra den till imorgon istället.",
        "CONFLICTING_MUTATION",
        ["TRIGGER_CONFLICT_CLARIFICATION"]
    ),
    (
        "Vad har ni för öppettider, finns det parkering, vad kostar biffen och boka bord för 10?",
        "INTENT_OVERCROWDING",
        ["REJECT_AND_CLARIFY_PROMPT"]
    )
])
def test_multi_intent_dag_execution(utterance, expected_dag_type, expected_execution_plan):
    plan = multi_intent_engine.parse_and_build_dag(utterance)
    assert plan.dag_type == expected_dag_type
    assert plan.get_execution_steps() == expected_execution_plan


6. SECURITY & THREAT MODEL
Adversaries may attempt to bypass state machines by embedding hidden mutations inside long compound queries or flooding the NLU parser with nested intent trees.
Threat
Attack Vector
Preventive Control
Severity
Intent Overcrowding (DoS)
User submits 10 nested requests in one sentence to force token inflation or semantic confusion.
Enforce hard cap (N_{\text{max}} = 3). Excess intents trigger REJECT_AND_CLARIFY. Silent truncation is strictly prohibited.
HIGH
Hidden Mutation Injection
Malicious booking cancellation hidden inside an innocent opening hours query.
All mutating intents require explicit slot validation and confirmation regardless of position in utterance.
CRITICAL
DAG Circular Dependency
Adversarial phrasing creates an infinite loop in the multi-intent dependency graph.
Topological sort with cycle detection; fails fast (ERR_TEST_05_03) if a graph cycle is detected.
HIGH

7. FAILURE ARCHITECTURE & ERROR TAXONOMY
Error Code
Description
Canonical Class
Severity
ERR_TEST_05_01
Conditional dependency violation (Mutation executed before required safety check resolved).
SAFETY_VIOLATION
CRITICAL
ERR_TEST_05_02
Contradictory intents executed without triggering a conflict resolution state.
STATE_CORRUPTION
HIGH
ERR_TEST_05_03
Circular dependency detected during multi-intent DAG graph construction.
VALIDATION
HIGH
ERR_TEST_05_04
Intent overcrowding threshold exceeded (N_{\text{intents}} > 3). Triggered REJECT_AND_CLARIFY.
RATE_LIMIT
MEDIUM
ERR_TEST_05_05
Silent truncation of user intent detected during multi-intent decomposition.
CONTRACT_MISMATCH
CRITICAL

8. PRODUCTION ACCEPTANCE CRITERIA
AC ID
Category
Requirement
Verification Method
Pass/Fail
AC-DEP-01
CE-SPEC-09 Parity
100\% of conditional safety utterances enforce sequential DAG resolution prior to booking mutation.
DAG Execution Audit
REQUIRED
AC-DEP-02
Independent Execution
100\% of unlinked safety + booking queries execute in parallel without blocking valid bookings.
Parallel Execution Test
REQUIRED
AC-DEP-03
Conflict Detection
100\% of mutually exclusive intent combinations (Cancel + Modify) trigger clarification flows.
Conflict Scenario Suite
REQUIRED
AC-DEP-04
Zero Silent Truncation
No user intent is ever silently dropped. Inputs with > 3 intents trigger REJECT_AND_CLARIFY.
Boundary Test Vector
REQUIRED
AC-DEP-05
Cycle Prevention
Topological DAG parser guarantees zero infinite loops under adversarial circular intent combinations.
Cycle Test Suite
REQUIRED

9. INTEGRATION AUTHORITY MATRIX
Component
Owns
Consumes
Produces
TEST-SPEC-05
Multi-Intent DAG Engine Tests, Dependency Semantics (CE-SPEC-09), Conflict Scenarios
Parsed NLU Tokens (TEST-SPEC-04), User Inputs
Ordered Execution Graphs, Parallel DAG Signals
Phase 3 (CE-SPEC-09)
Authoritative Multi-Intent & Safety Dependency Semantics
System Prompts, State Context
Conversation Routing Rules
TEST-SPEC-04
Single Intent & Slot Extraction
User Utterances
Raw Unordered Sub-Intents
TEST-SPEC-06
Multi-Turn Context & Recovery
Execution Graphs (TEST-SPEC-05)
Dialogue State Transitions

10. FINAL NON-NEGOTIABLE PRINCIPLES
CONDITIONAL SAFETY DEPENDENCIES MUST RESOLVE BEFORE MUTATION EXECUTION; INDEPENDENT INTENTS MUST PROCESS IN PARALLEL PER CE-SPEC-09.
SILENT TRUNCATION OF USER INTENTS IS STRICTLY PROHIBITED; OVERCROWDED INPUTS (> 3 INTENTS) MUST REJECT AND CLARIFY.
MUTUALLY EXCLUSIVE INTENTS IN A SINGLE TURN MUST BE BLOCKED AND ROUTED TO A CLARIFICATION FLOW.
IF A CONDITIONAL DEPENDENCY NODE FAILS, ALL DEPENDENT DOWNSTREAM MUTATIONS MUST BE SHORT-CIRCUITED.
MULTI-INTENT DAG CONSTRUCTORS MUST USE TOPOLOGICAL SORTING WITH MANDATORY CYCLE DETECTION.
11. VERSION HISTORY & ARCHITECTURAL VERDICT
Version
Date
Description
Author
Status
1.0.0
August 2026
Initial specification of Multi-Intent & Dependency Ordering Tests (TEST-SPEC-05).
Ramy Bella
SUPERSEDED
1.1.0
August 2026
Hardened specification aligning dependency semantics with CE-SPEC-09 and replacing silent truncation with explicit clarification.
Ramy Bella
APPROVED FOR IMPLEMENTATION

ARCHITECTURAL VERDICT:
APPROVED FOR IMPLEMENTATION TEST-SPEC-05 v1.1.0 establishes the multi-intent DAG decomposition rules, CE-SPEC-09 dependency alignment (conditional vs. independent), conflict resolution matrix, and zero-truncation clarification harness for Phase 6. Ready for implementation.

