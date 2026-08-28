## 1. DOCUMENT CONTROL

| Attribute         | Value                                                                                        |
| ----------------- | -------------------------------------------------------------------------------------------- |
| Document Title    | 07 Escalation & Human-in-the-Loop Tests.md                                                   |
| Document ID       | TEST-SPEC-07                                                                                 |
| Version           | 1.0.0                                                                                        |
| Status            | APPROVED FOR IMPLEMENTATION                                                                  |
| Author            | Ramy Bella                                                                                   |
| Classification    | Confidential / Enterprise Proprietary                                                        |
| Target Audience   | AI Engineers, Conversation Architects, QA Leads, Customer Operations Engineers               |
| Parent Document   | TEST-SPEC-01                                                                                 |
| Related Documents | Phase 3 Specs (CE-SPEC), TEST-SPEC-01, TEST-SPEC-04, TEST-SPEC-06, TEST-SPEC-18, INT-SPEC-13 |
| System            | Restaurant AI System                                                                         |
| Phase             | Phase 6 — Validation & Verification (V&V)                                                    |
| Lifecycle Folder  | 02 Functional & Multi-Turn Dialogue                                                          |
| Last Updated      | August 2026                                                                                  |

---

## 2. EXECUTIVE PURPOSE

No conversational AI system can resolve 100% of complex human situations. When a user experiences severe frustration, requests an explicit human agent, or presents an edge-case transaction beyond bot capabilities, the system must perform a seamless, deterministic handover. Failing to escalate or dropping conversation context during a handover damages guest trust and creates severe operational risk.

`TEST-SPEC-07` defines the **Escalation & Human-in-the-Loop (HITL) Test Architecture**. It establishes verification protocols for explicit user escalation requests, sentiment-triggered handovers, automated 3-strike failure dispatches (`TEST-SPEC-06`), structured context bundle generation (`HandoverContext@1.0.0`), bot muting protocols, and off-hours/no-agent asynchronous fallback routing.

### Core Testing Invariants:
* `EXPLICIT HUMAN REQUEST \implies IMMEDIATE BOT MUTE & DISPATCH TO AGENT`
* `HIGH NEGATIVE SENTIMENT THRESHOLD \implies MANDATORY ESCALATION PROMPT`
* `HUMAN TAKEOVER ACTIVE \implies AI BOT GENERATION STRICTLY PROHIBITED`
* `INCOMPLETE HANDOVER BUNDLE \implies EMERGENCY AGENT NOTIFICATION TRIGGER`
* `NO ACTIVE HUMAN AGENTS AVAILABLE \implies ASYNCHRONOUS TICKET CREATION + SLA ACKNOWLEDGEMENT`

---

## 3. ESCALATION TAXONOMY & TRIGGER ARCHITECTURE

Escalation events are categorized into four distinct triggers, each following a deterministic routing path to human operations.


[INCOMING DIALOGUE TURN]
       │
       ├── Trigger 1: Explicit Request ("jag vill tala med en människa")
       ├── Trigger 2: Sentiment Threshold (Sentiment Score < -0.80)
       ├── Trigger 3: Systemic Fallback (3-Strike Failure from TEST-SPEC-06)
       └── Trigger 4: Explicit Intent (COMPLAINT_ESCALATE from TEST-SPEC-04)
       │
       ▼ [ESCALATION ENGINE DISPATCH]
┌─────────────────────────────────────────────────────────────────┐
│ 1. Lock Bot Output State (MUTE AI Generation)                    │
│ 2. Compile HandoverContext@1.0.0 (Transcript, Slots, Sentiment) │
│ 3. Check Real-Time Agent Availability (POS / Desk Integration)  │
└──────────────────────────────┬──────────────────────────────────┘
                               │
               ┌───────────────┴───────────────┐
               ▼                               ▼
     [AGENT AVAILABLE]                [NO AGENT / OFF-HOURS]
  Live Chat/Voice Transfer         Create Async Ticket + Send SLA
  & Push Handover Context          Acknowledgement to Guest


3.1. Escalation Trigger Definitions
Trigger ID
Category
Activation Condition
Mandatory System Action
TRG-ESC-01
Explicit User Command
Utterance contains "människa", "person", "agent", "receptionist", "ring mig".
Mute AI bot immediately; dispatch live chat transfer or call routing.
TRG-ESC-02
High Negative Sentiment
Sentiment analyzer detects sustained extreme frustration (\text{Score} \le -0.80).
Offer proactive human escalation prompt without waiting for 3-strike limit.
TRG-ESC-03
3-Strike Failure
Re-prompt counter reaches N=3 unparseable inputs (TEST-SPEC-06).
Escalate automatically to prevent infinite clarification loops.
TRG-ESC-04
High-Value Complaint
NLU classifies primary intent as COMPLAINT_ESCALATE (TEST-SPEC-04).
Halt automated transaction; route to manager console with high priority tag.

4. CONTEXT HANDOVER BUNDLE SPECIFICATION (HandoverContext@1.0.0)
During any escalation event, the system MUST compile and transmit an immutable, structured context payload to the human agent interface before the handover completes.
{
  "$schema": "[https://json-schema.org/draft/2020-12/schema](https://json-schema.org/draft/2020-12/schema)",
  "title": "HandoverContext@1.0.0",
  "type": "object",
  "properties": {
    "handover_id": { "type": "string" },
    "tenant_id": { "type": "string" },
    "session_id": { "type": "string" },
    "trigger_type": { "type": "string", "enum": ["EXPLICIT_REQUEST", "SENTIMENT_THRESHOLD", "REPROMPT_LIMIT", "COMPLAINT_INTENT"] },
    "timestamp": { "type": "string", "format": "date-time" },
    "guest_summary": {
      "type": "object",
      "properties": {
        "phone_number": { "type": "string" },
        "name": { "type": "string" },
        "sentiment_score": { "type": "number" }
      }
    },
    "conversation_state": {
      "type": "object",
      "properties": {
        "active_intent": { "type": "string" },
        "collected_slots": { "type": "object" },
        "pending_slots": { "type": "array", "items": { "type": "string" } }
      }
    },
    "transcript_history": {
      "type": "array",
      "items": {
        "type": "object",
        "properties": {
          "speaker": { "type": "string", "enum": ["USER", "BOT", "SYSTEM"] },
          "text": { "type": "string" },
          "timestamp": { "type": "string" }
        }
      }
    }
  },
  "required": ["handover_id", "tenant_id", "session_id", "trigger_type", "timestamp", "conversation_state", "transcript_history"]
}


5. HANDOVER EXECUTION & FALLBACK MATRIX
The test suite evaluates system behavior under both active agent availability and off-hours/no-agent scenarios.
5.1. Handover Execution Matrix
Scenario ID
Agent Availability State
Primary Action
Secondary / Fallback Action
Assertion Rule
HO-EXEC-01
Live Agent Online
Push HandoverContext@1.0.0 \rightarrow Connect Live Session
Mute AI Bot; output handover greeting.
0\% AI bot turns after connection.
HO-EXEC-02
All Agents Busy
Queue session \rightarrow Announce estimated wait time
If wait time > 180\text{s}, offer async callback ticket.
Queue status emitted to telemetry.
HO-EXEC-03
Off-Hours / No Agents
Generate async support ticket (TICKET_CREATE)
Send SMS/Chat confirmation with SLA response window.
Ticket contains full HandoverContext.
HO-EXEC-04
Agent Timeout (T > 60\text{s})
Trigger emergency notification alert to manager dashboard
Fallback to SMS callback registration.
Log ERR_TEST_07_03.

6. SECURITY & THREAT MODEL
Adversaries may attempt to exploit escalation pathways to cause agent queue flooding (Denial of Service) or bypass bot validation steps.
Threat
Attack Vector
Preventive Control
Severity
Escalation Queue Flooding
Automated bot repeatedly triggers TRG-ESC-01 to exhaust human support capacity.
Rate limit escalations per IP/Session (Max 2 escalations per hour per tenant).
HIGH
Bot Un-Muting Exploit
User crafts prompts during live agent chat trying to re-activate AI bot responses.
Hard session lock (state == HUMAN_HANDOVER_ACTIVE) completely blocks AI generation pipeline.
CRITICAL
Context Payload PII Leak
Handover bundle sent to unauthorized third-party agent interface.
Handover payloads encrypted in transit (TLS 1.3) and restricted by tenant scope (INT-SPEC-13).
CRITICAL

7. FAILURE ARCHITECTURE & ERROR TAXONOMY
Error Code
Description
Canonical Class
Severity
ERR_TEST_07_01
AI bot failed to mute generation pipeline after escalation trigger activation.
SECURITY_BOUNDARY
CRITICAL
ERR_TEST_07_02
HandoverContext@1.0.0 payload failed schema validation or missing required slots.
CONTRACT_MISMATCH
HIGH
ERR_TEST_07_03
Live agent handover connection timed out (T > 60\text{s}) without fallback execution.
TIMEOUT
HIGH
ERR_TEST_07_04
Off-hours ticket creation failed to dispatch async notification SLA to guest.
PROVIDER_UNAVAILABLE
MEDIUM
ERR_TEST_07_05
Rate limit exceeded for automated escalation requests from single session.
RATE_LIMIT
MEDIUM

8. PRODUCTION ACCEPTANCE CRITERIA
AC ID
Category
Requirement
Verification Method
Pass/Fail
AC-HITL-01
Instant Mute
100\% of escalation triggers instantly lock AI generation; zero bot responses emitted post-trigger.
Handover Test Harness
REQUIRED
AC-HITL-02
Bundle Integrity
100\% of escalation events deliver a valid, schema-compliant HandoverContext@1.0.0 payload.
Schema Audit Inspector
REQUIRED
AC-HITL-03
Off-Hours Fallback
100\% of off-hours escalations successfully generate async tickets and return SLA messages to guests.
Async Fallback Harness
REQUIRED
AC-HITL-04
Explicit Request
Utterances containing explicit human requests trigger escalation within < 500\text{ ms}.
Latency Benchmark
REQUIRED
AC-HITL-05
Session Lock
Un-muting exploits fail 100\% of the time while human agent handover state is active.
Security Injection Suite
REQUIRED

9. INTEGRATION AUTHORITY MATRIX
Component
Owns
Consumes
Produces
TEST-SPEC-07
Escalation Harness, HITL State Verification, Handover Context Schema, Fallback Matrix
Escalation Triggers (TEST-SPEC-04, 06), User Inputs
Handover Bundles, Agent Dispatch Signals
Phase 3 (CE-SPEC)
Authoritative Escalation & Mute State Machine Rules
Escalation Signals
Conversation State Locks
TEST-SPEC-06
3-Strike Re-Prompt Escalation Trigger
User Turns
Escalation Signal (TRG-ESC-03)
INT-SPEC-18
Telemetry & Audit Logs
Escalation Events
Audit Logs for Human Handovers

10. FINAL NON-NEGOTIABLE PRINCIPLES
UPON ACTIVATION OF ANY ESCALATION TRIGGER, THE AI BOT MUST MUTE IMMEDIATELY AND CEASE RESPONSE GENERATION.
ALL ESCALATION EVENTS MUST PRODUCE A CRYPTOGRAPHICALLY VALIDATED, SCHEMA-COMPLIANT HANDOVER BUNDLE.
IF NO HUMAN AGENTS ARE AVAILABLE, THE SYSTEM MUST GENERATE AN ASYNCHRONOUS SUPPORT TICKET WITH EXPLICIT SLA TIMELINES.
HUMAN HANDOVER STATES MUST REMAIN LOCKED AGAINST PROMPT INJECTION ATTEMPTS DESIGNED TO RE-ACTIVATE BOT OUTPUTS.
EXPLICIT REQUESTS FOR HUMAN ASSISTANCE MUST BE HONORED IMMEDIATELY WITHOUT BINDING FURTHER SLOTS OR GUESSING.
11. VERSION HISTORY & ARCHITECTURAL VERDICT
Version
Date
Description
Author
Status
1.0.0
August 2026
Initial specification of Escalation & Human-in-the-Loop Tests (TEST-SPEC-07).
Ramy Bella
APPROVED FOR IMPLEMENTATION

ARCHITECTURAL VERDICT:
APPROVED FOR IMPLEMENTATION TEST-SPEC-07 establishes the HITL escalation triggers, context handover bundle schema (HandoverContext@1.0.0), bot muting protocols, and off-hours fallback matrix for Phase 6. Ready for implementation.
