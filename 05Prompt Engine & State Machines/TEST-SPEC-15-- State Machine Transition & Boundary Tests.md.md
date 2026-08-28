# TEST-SPEC-15: State Machine Transition & Boundary Tests

## 1. DOCUMENT CONTROL

| Attribute | Value |
|---|---|
| Document Title | 15 State Machine Transition & Boundary Tests.md |
| Document ID | TEST-SPEC-15 |
| Version | 1.0.0 |
| Status | APPROVED FOR IMPLEMENTATION |
| Author | Ramy Bella |
| Classification | Confidential / Enterprise Proprietary |
| Target Audience | State Machine Engineers, Conversation Architects, AI Engineers, QA Leads |
| Parent Document | TEST-SPEC-01 |
| Related Documents | Phase 3 Specs (CE-SPEC), TEST-SPEC-01, TEST-SPEC-02, TEST-SPEC-04, TEST-SPEC-05, TEST-SPEC-06 |
| System | Restaurant AI System |
| Phase | Phase 6 — Validation & Verification (V&V) |
| Lifecycle Folder | 05 Prompt Engine & State Machines |
| Last Updated | August 2026 |

---

## 2. EXECUTIVE PURPOSE

The Phase 3 Conversation Engine relies on a Deterministic Finite State Machine (FSM) to manage dialogue progression, booking reservations, escalation flows, and session state persistence. While NLU and LLM outputs are probabilistic, state machine transitions MUST remain 100% mathematically deterministic. Allowing illegal state jumps, unhandled state deadlocks, or state context corruption bypasses business rules and creates severe operational defects (e.g., confirming a booking without party size or skipping deposit collection).

`TEST-SPEC-15` defines the **State Machine Transition & Boundary Test Architecture**. Executed within Stage 1 (Unit) and Stage 3 (E2E) verification pipelines (`TEST-SPEC-02`), it establishes automated testing harnesses for valid state transition vectors, illegal transition rejections, boundary state conditions, context frame persistence, and state recovery from deadlocks. It guarantees that dialogue state transitions strictly adhere to Phase 3 state machine specifications (`CE-SPEC`).

### Core Testing Invariants:
* `STATE MACHINE TRANSITIONS = 100% DETERMINISTIC (0% PROBABILISTIC TOLERANCE)`
* `ILLEGAL STATE TRANSITION ATTEMPT \implies HARD REJECTION & FALLBACK TO LAST VALID STATE`
* `UNRESOLVED / UNKNOWN STATE \implies ATOMIC RESET TO IDLE WITH AUDIT ALERT`
* `MISSING REQUIRED STATE SLOTS \implies PROHIBITED TRANSITION TO MUTATION STATE`
* `STATE CONTEXT SCHEMA MISMATCH \implies IMMEDIATE STATE CORRUPTION ERROR (SEV-1)`

---

## 3. STATE TRANSITION MATRIX & BOUNDARY CONDITIONS

The conversation state machine enforces strict allowed transition vectors between core states.

```text
               ┌─────────────────────────────────────────────────┐
               │                   IDLE                          │
               └───────────────────────┬─────────────────────────┘
                                       │ Event: INTENT(BOOKING_CREATE)
                                       ▼
               ┌─────────────────────────────────────────────────┐
               │        BOOKING_COLLECT_PARTY_SIZE               │
               └───────────────────────┬─────────────────────────┘
                                       │ Event: SLOT_VALID(party_size)
                                       ▼
               ┌─────────────────────────────────────────────────┐
               │        BOOKING_COLLECT_DATE_TIME                │
               └───────────────────────┬─────────────────────────┘
                                       │ Event: SLOT_VALID(date, time)
                                       ▼
               ┌─────────────────────────────────────────────────┐
               │        BOOKING_CONFIRMATION_PENDING             │
               └───────────────────────┬─────────────────────────┘
                                       │ Event: USER_CONFIRM
                                       ▼
               ┌─────────────────────────────────────────────────┐
               │        BOOKING_EXECUTED (Terminal)              │
               └─────────────────────────────────────────────────┘
```

This diagram shows the core happy-path only. Section 3.1 is the canonical, complete transition set — including the deposit, correction, cancellation, and escalation branches — and governs in any case of conflict with this diagram.

3.1. Canonical State Transition Enforcement Table
Current State (S_t)
Incoming Event (E)
Allowed Next State (S_{t+1})
Prohibited Illegal Transitions (Examples)
IDLE
INTENT(BOOKING_CREATE)
BOOKING_COLLECT_PARTY_SIZE
Direct to BOOKING_EXECUTED (Skips slots).
BOOKING_COLLECT_PARTY_SIZE
SLOT_VALID(party_size), deposit policy not triggered
BOOKING_COLLECT_DATE_TIME
Jump to BOOKING_CANCELLED (Unless INTENT(BOOKING_CANCEL) — see cancellation row below).
BOOKING_COLLECT_PARTY_SIZE
SLOT_VALID(party_size), deposit policy triggered
BOOKING_COLLECT_DEPOSIT
Skipping deposit collection and proceeding directly to BOOKING_COLLECT_DATE_TIME once policy is triggered.
BOOKING_COLLECT_DEPOSIT
DEPOSIT_CONFIRMED
BOOKING_COLLECT_DATE_TIME
Proceeding without a confirmed deposit payment.
BOOKING_COLLECT_DATE_TIME
SLOT_VALID(date, time)
BOOKING_CONFIRMATION_PENDING
Jump to BOOKING_EXECUTED without confirmation.
BOOKING_CONFIRMATION_PENDING
USER_CONFIRM
BOOKING_EXECUTED
Return to BOOKING_COLLECT_PARTY_SIZE without a SLOT_EDIT_REQUEST event (see correction row below).
BOOKING_CONFIRMATION_PENDING
SLOT_EDIT_REQUEST(party_size)
BOOKING_COLLECT_PARTY_SIZE
Discarding previously collected date/time or deposit slots on re-entry; they remain populated and are only overwritten by a new SLOT_VALID event.
BOOKING_CONFIRMATION_PENDING
SLOT_EDIT_REQUEST(date_time)
BOOKING_COLLECT_DATE_TIME
Same as above — prior slots are preserved, not cleared.
BOOKING_COLLECT_PARTY_SIZE, BOOKING_COLLECT_DEPOSIT, BOOKING_COLLECT_DATE_TIME, BOOKING_CONFIRMATION_PENDING
INTENT(BOOKING_CANCEL)
BOOKING_CANCELLED (Terminal)
Invoking after BOOKING_EXECUTED — a completed booking requires the dedicated cancellation/refund flow, not this FSM's cancel event.
BOOKING_EXECUTED
SESSION_CLOSE
IDLE
Re-trigger execution dispatch (INT-SPEC-15).
ANY_STATE (pre-execution)
TRG-ESC-01, TRG-ESC-02, TRG-ESC-03, or INTENT(COMPLAINT_ESCALATE) (escalation triggers owned by TEST-SPEC-07)
ESCALATE_TO_HUMAN
Cannot be blocked by pending booking state; any of the four TEST-SPEC-07 trigger types must reach this state, not INTENT(COMPLAINT_ESCALATE) alone.

**Terminology note:** `ESCALATE_TO_HUMAN` is this FSM's canonical name for the muted, human-handed-off state. `TEST-SPEC-07` refers to the same state as "HUMAN_HANDOVER_ACTIVE" when describing its mute-lock behavior; the two terms name one state, owned here.

**Deposit policy note:** the party-size threshold and deposit amount that trigger `BOOKING_COLLECT_DEPOSIT` are restaurant-configured business rules owned by `CE-SPEC` (Phase 3), not by this test specification. This transition exists to guarantee that whatever threshold is configured, the deposit step cannot be skipped once triggered — the specific threshold value is out of scope here and must not be assumed from any single test example.

4. STATE CONTEXT CONTRACT (StateContextPayload@1.0.0)
State context persisted across dialogue turns MUST conform strictly to a validated schema. Modifying state properties outside defined contracts triggers instant rejection.
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "title": "StateContextPayload@1.0.0",
  "type": "object",
  "properties": {
    "session_id": { "type": "string" },
    "tenant_id": { "type": "string" },
    "current_state": { "type": "string", "enum": ["IDLE", "BOOKING_COLLECT_PARTY_SIZE", "BOOKING_COLLECT_DEPOSIT", "BOOKING_COLLECT_DATE_TIME", "BOOKING_CONFIRMATION_PENDING", "BOOKING_EXECUTED", "BOOKING_CANCELLED", "ESCALATE_TO_HUMAN"] },
    "previous_state": { "type": "string", "enum": ["IDLE", "BOOKING_COLLECT_PARTY_SIZE", "BOOKING_COLLECT_DEPOSIT", "BOOKING_COLLECT_DATE_TIME", "BOOKING_CONFIRMATION_PENDING", "BOOKING_EXECUTED", "BOOKING_CANCELLED", "ESCALATE_TO_HUMAN"] },
    "state_depth": { "type": "integer", "minimum": 0, "maximum": 5 },
    "collected_slots": {
      "type": "object",
      "properties": {
        "party_size": { "type": ["integer", "null"] },
        "deposit_confirmed": { "type": ["boolean", "null"] },
        "reservation_date": { "type": ["string", "null"], "format": "date" },
        "reservation_time": { "type": ["string", "null"], "format": "time" }
      }
    },
    "reprompt_counter": { "type": "integer", "minimum": 0, "maximum": 3 },
    "updated_at": { "type": "string", "format": "date-time" }
  },
  "required": ["session_id", "tenant_id", "current_state", "state_depth", "collected_slots", "reprompt_counter", "updated_at"]
}


5. AUTOMATED STATE MACHINE TEST HARNESS SUITE
The state machine test suite executes exhaustive transition permutation testing against the Phase 3 FSM engine.
# Representative Automated State Machine Transition Test Harness
@pytest.mark.rtm(req_id="CE-SPEC-15-FSM-001")
def test_prohibited_state_jump_rejection():
    fsm = ConversationStateMachine(tenant_id="tenant_demo", initial_state="IDLE")
    
    # Act 1: Transition IDLE -> COLLECT_PARTY_SIZE (Valid)
    fsm.process_event(Event(name="INTENT_BOOKING_CREATE"))
    assert fsm.current_state == "BOOKING_COLLECT_PARTY_SIZE"
    
    # Act 2: Attempt illegal direct jump to BOOKING_EXECUTED without slots
    with pytest.raises(IllegalStateTransitionException) as exc_info:
        fsm.process_event(Event(name="EXECUTE_BOOKING_MUTATION"))
        
    # Assertions: Verify rejection, state preservation, and error code
    assert exc_info.value.error_code == "ERR_TEST_15_01"
    assert fsm.current_state == "BOOKING_COLLECT_PARTY_SIZE" # State held safely
    assert fsm.last_valid_state == "BOOKING_COLLECT_PARTY_SIZE"


6. SECURITY & THREAT MODEL
Adversaries or buggy NLU models may attempt to manipulate state variables to bypass business rules or cause deadlock conditions.
Threat
Attack Vector
Preventive Control
Severity
State Jump Injection
User prompt crafts text trying to force state = BOOKING_EXECUTED in context payload.
State machine state is maintained in immutable server-side session memory; un-editable via user prompt.
CRITICAL (SEV-0)
State Deadlock Lockup
NLU emits unknown event, trapping user in un-resumable state frame.
State machine fallback handler resets trapped session to IDLE or triggers escalation if unresolved.
HIGH (SEV-1)
Context Memory Corruption
Malformed slot types (e.g., string passed for integer party_size) passed to FSM.
Strict JSON Schema validation (StateContextPayload@1.0.0) on every state transition write.
HIGH (SEV-1)

7. FAILURE ARCHITECTURE & ERROR TAXONOMY
Error Code
Description
Canonical Class
Severity
ERR_TEST_15_01
Prohibited or invalid state transition attempted; transition rejected.
STATE_CORRUPTION
HIGH (SEV-1)
ERR_TEST_15_02
Transition to mutation state attempted without mandatory required slots populated.
VALIDATION
HIGH (SEV-1)
ERR_TEST_15_03
State machine trapped in unknown or deadlocked state; reset to IDLE triggered.
LOGIC_FAIL
HIGH (SEV-1)
ERR_TEST_15_04
State context payload failed JSON Schema validation (StateContextPayload@1.0.0).
CONTRACT_MISMATCH
HIGH (SEV-1)
ERR_TEST_15_05
Attempt to mutate immutable server-side state via client input payload detected.
SECURITY_VIOLATION
CRITICAL (SEV-0)
ERR_TEST_15_06
BOOKING_CANCEL invoked against a state that does not permit cancellation (e.g., after BOOKING_EXECUTED).
VALIDATION
MEDIUM (SEV-2)

8. PRODUCTION ACCEPTANCE CRITERIA
AC ID
Category
Requirement
Verification Method
Pass/Fail
AC-FSM-01
100% Transition Validity
100\% of executed dialogue turns follow strictly valid transition vectors defined in Section 3.
FSM Trace Audit
REQUIRED
AC-FSM-02
Zero Illegal Jumps
100\% of attempted illegal state jumps are rejected with state context preserved.
Illegal Transition Suite
REQUIRED
AC-FSM-03
Schema Compliance
100\% of persisted state payloads validate against StateContextPayload@1.0.0, including state name enum membership.
Schema Inspector
REQUIRED
AC-FSM-04
Deadlock Recovery
100\% of trapped deadlock conditions automatically recover to IDLE or escalate within < 100\text{ ms}.
Fault Injection Harness
REQUIRED
AC-FSM-05
Slot Enforcement
Mutation states (BOOKING_EXECUTED) cannot be entered without 100\% of required slots present.
Slot Completeness Test
REQUIRED
AC-FSM-06
Correction Integrity
100\% of SLOT_EDIT_REQUEST transitions preserve previously collected slots outside the one being edited.
Correction Path Test
REQUIRED

9. INTEGRATION AUTHORITY MATRIX
Component
Owns
Consumes
Produces
TEST-SPEC-15
FSM Transition Verification, Illegal State Jump Tests, Schema Compliance Suite
FSM Events (TEST-SPEC-04), State Payload Contracts
State Machine Quality Verdicts, State Error Signals
Phase 3 (CE-SPEC)
Authoritative State Machine Architecture & FSM Transition Rules
NLU Events, Context Signals
State Transitions, Active Dialogue States
TEST-SPEC-06
Multi-Turn Context & Interruption Recovery
Active Dialogue States
Interruption Stack Signals
TEST-SPEC-07
Escalation & Human-in-the-Loop Tests
ESCALATE_TO_HUMAN State Definition
Escalation Trigger Test Targets
TEST-SPEC-11
Adversarial & Red Teaming Harness
BOOKING_COLLECT_DEPOSIT State Definition
Multi-Turn Coercion Test Targets
TEST-SPEC-18
E2E Adapter & Gateway Pipeline
Validated State Transitions
POS Integration Payloads

10. FINAL NON-NEGOTIABLE PRINCIPLES
ALL CONVERSATION STATE TRANSITIONS MUST BE 100% DETERMINISTIC AND MATHEMATIONALLY VERIFIED.
ILLEGAL STATE TRANSITIONS MUST BE REJECTED IMMEDIATELY WITH DIALOGUE CONTEXT SAFELY PRESERVED.
NO MUTATION STATE MAY BE ENTERED WITHOUT COMPLETE, TYPE-VALIDATED REQUIRED SLOT DATA.
STATE MACHINE CONTEXT MUST BE MAINTAINED IMMUTABLY ON THE SERVER SIDE; PROMPT MANIPULATION IS PROHIBITED.
DEADLOCKED OR TRACTIONLESS STATE FRAMES MUST AUTOMATICALLY RECOVER TO IDLE OR ESCALATE.
GUEST-INITIATED CORRECTIONS TO PREVIOUSLY COLLECTED SLOTS AND BOOKING CANCELLATIONS MUST BE REPRESENTED AS EXPLICIT, LEGAL TRANSITIONS — NOT HANDLED BY RESTARTING THE FSM FROM IDLE.
11. VERSION HISTORY & ARCHITECTURAL VERDICT
Version
Date
Description
Author
Status
1.0.0
August 2026
Initial specification of State Machine Transition & Boundary Tests (TEST-SPEC-15).
Ramy Bella
APPROVED FOR IMPLEMENTATION
1.0.1
August 2026
Review fix pass. Fixed the malformed $schema URL. Added an enum constraint on current_state/previous_state so schema validation actually catches an invalid or unknown state name, not just a wrong type. Added format constraints to reservation_date/reservation_time. Added BOOKING_COLLECT_DEPOSIT as an explicit, policy-gated branch (threshold owned by CE-SPEC, not fabricated here) to close the gap TEST-SPEC-11 depended on silently. Added BOOKING_CANCEL/BOOKING_CANCELLED as an explicit universal pre-execution transition, and SLOT_EDIT_REQUEST as an explicit correction transition — both were referenced as parenthetical exceptions in v1.0.0's table but never actually defined. Broadened the ANY_STATE escalation row to cover all four TEST-SPEC-07 trigger types instead of INTENT(COMPLAINT_ESCALATE) alone, and added a terminology note reconciling ESCALATE_TO_HUMAN (canonical, owned here) with TEST-SPEC-07's "HUMAN_HANDOVER_ACTIVE" phrasing for the same state. Fixed a spacing typo in Section 6. Added TEST-SPEC-07 and TEST-SPEC-11 to the Integration Authority Matrix as consumers of states this file now owns. No change to the core happy-path sequence, error taxonomy numbering, or acceptance-criteria methodology.
Ramy Bella
APPROVED FOR IMPLEMENTATION

ARCHITECTURAL VERDICT:
APPROVED FOR IMPLEMENTATION TEST-SPEC-15 establishes the FSM transition validation matrix, illegal jump rejection harness, StateContextPayload@1.0.0 schema contract, and deadlock recovery rules for Phase 6. Ready for implementation
