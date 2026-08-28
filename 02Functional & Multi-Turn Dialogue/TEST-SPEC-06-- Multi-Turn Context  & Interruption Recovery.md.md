# TEST-SPEC-06: Multi-Turn Context & Interruption Recovery Tests

## 1. DOCUMENT CONTROL

| Attribute | Value |
|---|---|
| Document Title | 06 Multi-Turn Context & Interruption Recovery Tests.md |
| Document ID | TEST-SPEC-06 |
| Version | 1.0.0 |
| Status | APPROVED FOR IMPLEMENTATION |
| Author | Ramy Bella |
| Classification | Confidential / Enterprise Proprietary |
| Target Audience | AI Engineers, Conversation Architects, QA Engineers, State Machine Engineers |
| Parent Document | TEST-SPEC-01 |
| Related Documents | Phase 3 Specs (CE-SPEC), TEST-SPEC-01, TEST-SPEC-04, TEST-SPEC-05, TEST-SPEC-07, TEST-SPEC-15 |
| System | Restaurant AI System |
| Phase | Phase 6 — Validation & Verification (V&V) |
| Lifecycle Folder | 02 Functional & Multi-Turn Dialogue |
| Last Updated | August 2026 |

---

## 2. EXECUTIVE PURPOSE

In multi-turn restaurant dialogues, users frequently change topics mid-flow (e.g., asking about parking while in the middle of entering booking details), use pronouns and relative references ("den där rätten", "samma tid nästa vecka"), or drop out of sessions temporarily. If the conversation engine loses context or mishandles interruptions, it causes severe conversational friction, corrupted slot assignments, or stranded active transactions.

`TEST-SPEC-06` defines the **Multi-Turn Context & Interruption Recovery Test Suite**. It establishes verification harnesses for the Phase 3 Conversation Stack, testing context persistence across turns, mid-flow topic interruptions, resumption protocols, anaphora/coreference resolution, context decay/TTL timeouts, and maximum re-prompt escalation triggers.

### Core Testing Invariants:
* `MID-FLOW TOPIC INTERRUPTION \implies PUSH STATE TO CONTEXT STACK; ANSWER INTERRUPTION; OFFER RESUMPTION`
* `ANAPHORA / PRONOUN AMBIGUITY \implies CLARIFY RATHER THAN BIND UNCERTAIN SLOT`
* `RE-PROMPT COUNTER EXCEEDED (N = 3) \implies MANDATORY FALLBACK OR ESCALATION DISPATCH`
* `SESSION TTL EXPIRED \implies SENSITIVE PII FLUSH; RESET TO IDLE STATE`
* `EXPLICIT CONTEXT RESET ("börja om") \implies ATOMIC CLEAR OF STACK AND SLOTS`

---

## 3. CONTEXT STACK & INTERRUPTION ARCHITECTURE

The test harness evaluates the Phase 3 Context Stack Manager during multi-turn interactions using a Last-In, First-Out (LIFO) state interruption stack.


[TURN 1: Active Flow]
  State: BOOKING_COLLECT_PARTY_SIZE ──► Slots: {date: "2026-08-28"}
         │
         ▼ [USER INTERRUPTS] "Har ni handikappanpassad entré?"
┌──────────────────────────────────────────────────────────────────┐
│ CONTEXT STACK ENGINE                                             │
│  1. Push current flow to stack: [BOOKING_COLLECT_PARTY_SIZE]      │
│  2. Route to interruption handler: HOURS_LOCATION_QUERY (Access) │
│  3. Generate Answer: "Ja, vi har ramp och handikappanpassad WC." │
│  4. Pop stack & prompt resumption: "Vill du fortsätta boka?"     │
└──────────────────────────────────────────────────────────────────┘
         │
         ▼ [USER ACCEPTS] "Ja, vi blir 4 personer"
  State Resumed: BOOKING_COLLECT_PARTY_SIZE ──► Slots: {date: "2026-08-28", party_size: 4}


3.1. Context Lifecycles & State Transitions
Context State
Trigger Event
Stack Behavior
Target Resolution
Active Primary Flow
User initiates linear flow (BOOKING_CREATE).
State active at stack depth D=0.
Collect required slots sequentially.
Side-Topic Interruption
User asks out-of-flow question (MENU_QUERY, HOURS_QUERY) mid-transaction.
Primary state pushed to stack (D=1). Interruption processed.
Answer question \rightarrow prompt user to resume or abandon primary flow.
Flow Replacement
User explicitly switches to a new primary flow ("nej förresten vill ställa in").
Flush active stack depth (D=0). Init new state.
Purge pending slots from abandoned flow.
Anaphora Resolution
User uses pronouns or relative references ("den", "samma bord").
Inspect previous N=3 turns in context buffer.
Bind entity to current active slot or request clarification if ambiguous.
Session Expiration
Inactivity timeout (T_{\text{idle}} > 15\text{ min}).
Flush entire context stack.
Reset to IDLE state; scrub temporary PII from session memory.

4. ANAPHORA & COREFERENCE RESOLUTION VERIFICATION
The system must correctly resolve relative references across multi-turn dialogue histories without binding incorrect entities.
4.1. Coreference Test Matrix
Test ID
Turn N-1 Context
Turn N Input
Target Slot to Bind
Expected Resolution Output
Assertion Rule
COREF-01
Discussing dish "Lammracks"
"Finns den som laktosfri?"
MENU_QUERY.dish_name
"Lammracks"
Exact Entity Bind
COREF-02
Booking for "Fredag 28 augusti"
"Boka samma tid nästa vecka istället"
BOOKING_CREATE.date
"2026-09-04"
Relative Date Math (+7\text{ days})
COREF-03
Mentions "Sven" & "Maria"
"Ändra hans stol till barnstol"
SEATING_PREFERENCE
Trigger Clarification
Ambiguous Pronoun \implies Clarify
COREF-04
Selected "Uteservering"
"Ska vi ta det inne om det regnar?"
SEATING_PREFERENCE
"INDOORS"
Contrastive Switch

5. RE-PROMPT COUNTER & FALLBACK ESCALATION SUITE
When user responses are invalid, low-confidence, or unparseable, the conversation engine MUST NOT enter an infinite clarification loop. The test suite verifies the 3-Strike Re-Prompt Escalation Protocol.
[INVALID / UNKNOWN USER INPUT]
       │
       ├── Strike 1 (Attempt 1) ──► Re-prompt with gentle clarification guidance.
       ├── Strike 2 (Attempt 2) ──► Re-prompt with explicit structured options/examples.
       └── Strike 3 (Attempt 3) ──► ESCALATE: Dispatch to Human Agent or Fallback Handler.


5.1. Re-Prompt Test Scenarios
# Representative Test Harness for 3-Strike Re-Prompt Escalation
@pytest.mark.rtm(req_id="CTX-SPEC-06-REPROMPT-001")
def test_three_strike_reprompt_escalation():
    session = dialogue_engine.start_session(tenant_id="tenant_demo")
    session.send_user_turn("Jag vill boka bord") # Enters BOOKING_CREATE
    
    # Strike 1: Garbage Input
    r1 = session.send_user_turn("asdfghjkl")
    assert r1.state == "BOOKING_COLLECT_PARTY_SIZE"
    assert r1.reprompt_count == 1
    assert "Hur många personer" in r1.bot_response # Gentle re-prompt

    # Strike 2: Off-topic / Unparseable Input
    r2 = session.send_user_turn("kanske måndag eller blått")
    assert r2.reprompt_count == 2
    assert "Ange ett antal med siffror, till exempel 2 eller 4" in r2.bot_response # Guided

    # Strike 3: Final Failure Trigger
    r3 = session.send_user_turn("fortfarande fel")
    assert r3.reprompt_count == 3
    assert r3.state == "ESCALATE_TO_HUMAN" # Escalation Dispatch Triggered


6. SECURITY & THREAT MODEL
Adversaries may attempt to corrupt session memory, bypass state checks via multi-turn topic switching, or extract context from other users via session poisoning.
Threat
Attack Vector
Preventive Control
Severity
Context Stack Exhaustion
User repeatedly triggers nested interruptions to force stack overflow in session memory.
Enforce maximum stack depth (D_{\text{max}} = 2). Deeper interruptions replace intermediate stack frames.
HIGH
Session Cross-Contamination
Attacker crafts requests to access state variables left behind by a previous user session.
Strict multi-tenant session isolation (INT-SPEC-13); mandatory state sanitization on session termination.
CRITICAL
State Injection via Interruption
Attacker interrupts booking flow to inject unauthorized admin flags into context memory.
Context stack preserves state variables in read-only immutable frames while handling side-topics.
CRITICAL

7. FAILURE ARCHITECTURE & ERROR TAXONOMY
Error Code
Description
Canonical Class
Severity
ERR_TEST_06_01
Interruption state manager failed to resume original flow after side-topic resolution.
STATE_CORRUPTION
HIGH
ERR_TEST_06_02
Incorrect anaphora binding performed on ambiguous pronoun without clarification.
PROBABILISTIC_DRIFT
MEDIUM
ERR_TEST_06_03
Re-prompt counter failed to escalate after 3 consecutive unparseable user turns.
LOGIC_FAIL
HIGH
ERR_TEST_06_04
Context stack depth limit (D_{\text{max}} = 2) exceeded without frame truncation.
INTERNAL
MEDIUM
ERR_TEST_06_05
Session TTL expiration failed to scrub temporary PII from active session storage.
SECURITY_BOUNDARY
CRITICAL

8. PRODUCTION ACCEPTANCE CRITERIA
AC ID
Category
Requirement
Verification Method
Pass/Fail
AC-CTX-01
Interruption Resumption
100\% of interrupted booking flows offer resumption after answering side-topic queries.
Multi-Turn Flow Suite
REQUIRED
AC-CTX-02
Coreference Accuracy
Anaphora resolution achieves \ge 98\% accuracy; 100\% of ambiguous pronouns trigger clarification.
Anaphora Test Suite
REQUIRED
AC-CTX-03
Escalation Enforcement
100\% of sessions reaching 3 consecutive invalid inputs transition to fallback/human escalation.
3-Strike Test Harness
REQUIRED
AC-CTX-04
Stack Boundary
Context stack depth never exceeds D_{\text{max}} = 2; excess frames are gracefully popped/truncated.
Boundary Test Harness
REQUIRED
AC-CTX-05
Memory Scrubbing
Session timeouts (T > 15\text{ min}) or explicit resets flush 100\% of temporary user PII slots.
Memory Leak Inspector
REQUIRED

9. INTEGRATION AUTHORITY MATRIX
Component
Owns
Consumes
Produces
TEST-SPEC-06
Multi-Turn Flow Verification, Context Stack Tests, Anaphora Resolution, Re-Prompt Escalation
Parsed Slots (TEST-SPEC-04), Execution DAGs (TEST-SPEC-05)
Multi-Turn Quality Metrics, Context Resumption Signals
Phase 3 (CE-SPEC)
Authoritative Conversation State Machine & Stack Architecture
Raw User Turns, Context Signals
State Transitions, Prompt Context
TEST-SPEC-04
Single Intent & Slot Extraction
Individual User Turns
Extracted Entities
TEST-SPEC-07
Human Escalation Protocols
Escalation Trigger Signals (ERR_TEST_06_03)
Agent Handover Context

10. FINAL NON-NEGOTIABLE PRINCIPLES
MID-FLOW INTERRUPTIONS MUST PRESERVE PRIMARY TRANSACTION STATE AND OFFER RESUMPTION AFTER RESOLUTION.
AMBIGUOUS PRONOUNS OR COREFERENCES MUST ALWAYS TRIGGER CLARIFICATION RATHER THAN BINDING WRONG SLOTS.
THREE CONSECUTIVE UNPARSEABLE TURNS MUST MANDATORILY ESCALATE TO HUMAN AGENT OR FALLBACK HANDLER.
SESSION TIMEOUTS OR EXPLICIT RESETS MUST ATOMICALLY FLUSH ALL TEMPORARY CONTEXT AND USER PII.
THE CONTEXT INTERRUPTION STACK DEPTH MUST NOT EXCEED TWO FRAMES UNDER ANY OPERATIONAL SCENARIO.
11. VERSION HISTORY & ARCHITECTURAL VERDICT
Version
Date
Description
Author
Status
1.0.0
August 2026
Initial specification of Multi-Turn Context & Interruption Recovery Tests (TEST-SPEC-06).
Ramy Bella
APPROVED FOR IMPLEMENTATION

ARCHITECTURAL VERDICT:
APPROVED FOR IMPLEMENTATION TEST-SPEC-06 establishes the multi-turn context persistence suite, interruption stack LIFO validation, coreference resolution test matrix, and 3-strike escalation harness for Phase 6. Ready for implementation.

