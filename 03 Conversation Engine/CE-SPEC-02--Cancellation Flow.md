CE-SPEC-02: Cancellation Flow
1. DOCUMENT CONTROL
| Attribute | Value |
|---|---|
| Document Title | 02 Cancellation Flow.md |
| Document ID | CE-SPEC-02 |
| Version | 1.1.0 |
| Status | Draft / Implementation Specification |
| Author | Ramy Bella |
| Classification | Confidential / Enterprise Proprietary |
| Target Audience | AI Engineers, Conversation Designers, Backend Engineers, QA Engineers, Integration Engineers, Platform Architects |
| Parent Document | 01 AI Identity.md |
| Related Documents | CE-SPEC-01: Booking Flow.md, CE-SPEC-03: Allergy Flow.md, CE-SPEC-07: Escalation.md, CE-SPEC-09: Multi Question Logic.md, CE-SPEC-10: Follow Up Logic.md, KB-SPEC-007, KB-SPEC-008 |
| System | Restaurant AI System |
| Phase | Phase 3 — Conversation Engine |
| Last Updated | August 2026 |
2. PURPOSE & SCOPE
Purpose
The Cancellation Flow specifies the deterministic conversational state machine required to guide a guest from initial cancellation intent through booking identification, policy validation, explicit confirmation, and successful integration handoff.
This flow operationalizes the core principle that the Assistant must never fabricate a successful cancellation, must not guess at financial penalties, and must fail safely if the underlying booking or policy cannot be verified.
Scope: What This Document Controls
 * Intent detection and parameter collection for identifying an existing reservation.
 * Conversational logic for handling multiple matches, missing bookings, and ambiguous identities.
 * The conversational handling of cancellation rules, fees, and deadlines.
 * Strict confirmation gating prior to state-changing execution.
 * Conversational handling of transactional race conditions, integration timeouts, and idempotency.
 * Context preservation during interruptions and cross-flow routing.
Scope: What This Document Explicitly Does NOT Control
 * Cancellation Policies & Fees: The rules governing cancellation deadlines, refunds, and no-show penalties are owned by the Knowledge Base. KB-SPEC-007 is authoritative for cancellation policies, cancellation deadlines, cancellation fees, refund-related policy semantics, and cancellation eligibility. KB-SPEC-008 is consulted only where a cancellation-related booking constraint is explicitly defined there.
 * Database/Integration Execution: The Assistant does not write to the reservation database. It hands off an execution request to the Integrations layer and awaits a deterministic response.
 * Multi-Intent Orchestration: Parsing a message like "Cancel my table and do you have a vegan menu?" is governed by 09 Multi Question Logic.md.
3. RELATIONSHIP TO CORE PRINCIPLES
This flow operationalizes the following mandates from 01 AI Identity.md:
 * Never fabricate execution success: The Assistant MUST NOT state "Your reservation is cancelled" until the integration explicitly confirms the transaction succeeded with a terminal result.
 * Never invent policies: If a cancellation fee applies, the Assistant MUST communicate the authoritative configured information. It MUST NOT waive fees, promise refunds, or create exceptions.
 * Data minimization & Privacy: The Assistant MUST NOT expose other guests' bookings or reveal unnecessary personal data when attempting to identify a reservation.
 * Idempotency: Repeated cancellation requests must be handled safely, communicating the actual state rather than attempting redundant executions.
4. CONVERSATIONAL STATE MODEL
The Cancellation Flow operates as a deterministic state machine.
| State | Entry Condition | Required Action | Transitions |
|---|---|---|---|
| DETECTED | Guest expresses cancellation intent. | Extract identifying parameters. Evaluate if sufficient. | IDENTIFYING_BOOKING, VALIDATING_BOOKING |
| IDENTIFYING_BOOKING | Identifying parameters missing or ambiguous. | Prompt guest for required lookup criteria (e.g., phone, email, name). | VALIDATING_BOOKING, ESCALATING, TERMINATED |
| VALIDATING_BOOKING | Minimum parameters collected. | Query Integrations layer for booking match. Suspend conversation. | VALIDATING_POLICY, IDENTIFYING_BOOKING (if no/multi match), ESCALATING |
| VALIDATING_POLICY | Exact active booking identified. | Query KB/Integrations for cancellation eligibility and fees. | CONFIRMING_CANCELLATION, ESCALATING, TERMINATED |
| CONFIRMING_CANCELLATION | Booking is eligible for cancellation. | Present booking details and policy/fees. Request explicit confirmation. | EXECUTING, TERMINATED (if guest declines) |
| EXECUTING | Guest provides explicit confirmation. | Dispatch cancellation payload to Integrations. Suspend cancellation execution state. | TERMINATED (Success), ESCALATING (Failure/Timeout) |
| ESCALATING | Flow cannot resolve mathematically or integration fails. | Route to 07 Escalation.md with active context payload. | N/A |
| TERMINATED | Flow completes successfully, fails unrecoverably, or guest aborts. | Clear active cancellation context. Return to generic listening state. | N/A |
Note on Terminal State Semantics: TERMINATED is a lifecycle terminal state and does not by itself describe the business outcome. The final outcome is represented through the resolved result/context, such as cancellation completed, cancellation declined by guest, already cancelled, cancellation unavailable, or escalated.
5. CANCELLATION REQUEST DETECTION & PARAMETER COLLECTION
5.1 Intent Detection
The NLP layer MUST classify the intent as cancellation when a guest requests to cancel, remove, delete, or abort an existing reservation.
5.2 Minimum Identifying Parameters
The system requires identifying information to query the booking system. Depending on the configured integration contract, the minimum required parameters may be:
 * Explicit Booking Reference ID (e.g., "Cancel booking #12345")
 * Contact Information (e.g., Phone number or Email)
Rule: If the active channel exposes a verified identity attribute through the authorized integration contract, the Conversation Engine MAY reuse that attribute and MUST NOT unnecessarily ask the guest to repeat it. If no verified identity attribute is available, the normal identification process applies.
6. BOOKING IDENTIFICATION & VALIDATION
Once minimum parameters are collected, the Conversation Engine queries the Integrations layer.
6.1 Single Active Match
 * If exactly one active booking matches, transition to VALIDATING_POLICY.
6.2 Multiple Matching Bookings
If the Integrations layer returns multiple active bookings for the provided identity:
 * Rule: The system may expose only the minimum booking attributes necessary to disambiguate the guest's own reservation (e.g., date or time). It must NOT expose unrelated guest information, full contact information, unnecessary booking metadata, or sensitive reservation details.
 * Action: Request a disambiguating parameter.
 * Response: "I see a few reservations under that number. Were you looking to cancel the one on {{DATE_1}} or a different one?"
6.3 No Matching Bookings
 * Rule: Do not assume the guest has no booking. The identifier may be incorrect or under another party member's name.
 * Response: "I couldn't find an active reservation under that information. Do you have a booking reference number, or was it perhaps booked under a different phone number?"
 * Transition: An identification attempt is defined as one complete backend booking lookup performed using a complete set of currently available identifying parameters. (Asking the guest another question or receiving an incomplete input is not an attempt). After two unsuccessful complete lookup attempts, transition to ESCALATING unless a different deterministic terminal condition applies.
6.4 Already Cancelled Booking
 * Rule: Treat as a deterministic final state. Do not attempt to cancel again.
 * Response: "It looks like the reservation for {{DATE}} has already been cancelled. Is there anything else I can help you with?"
 * Transition: TERMINATED (Outcome: Already Cancelled).
7. POLICY VALIDATION & CANCELLATION ELIGIBILITY
Before confirming cancellation, the system MUST evaluate the reservation against authoritative policies. KB-SPEC-007 is authoritative for cancellation policies.
| Policy Outcome | Conversational Behavior |
|---|---|
| CANCELLATION_ALLOWED | Proceed to CONFIRMING_CANCELLATION with standard phrasing. |
| CANCELLATION_ALLOWED_WITH_FEE | Proceed to CONFIRMING_CANCELLATION. MUST explicitly state the fee: "Please note that a late cancellation fee of {{CANCELLATION_FEE}} applies according to the policy." |
| CANCELLATION_NOT_ALLOWED | Transition to TERMINATED. "According to the restaurant's policy, this reservation can no longer be cancelled online. Please contact the venue directly at {{CONTACT_INFO}}." |
| CANCELLATION_REQUIRES_HUMAN | Transition to ESCALATING. "This reservation requires staff assistance to cancel. Let me connect you." |
| POLICY_UNKNOWN / POLICY_CONFLICT | Transition to ESCALATING. "I need a staff member to assist with this cancellation. Connecting you now." |
7.1 Refund Semantics
The Conversation Engine MUST NOT infer a refund merely because cancellation is allowed. It MUST NOT infer that a fee is refundable or non-refundable unless the authoritative policy explicitly states this. If refund information is unavailable, ambiguous, or conflicting, escalate rather than guessing.
Strict Prohibition: The Assistant MUST NEVER waive a fee, promise a refund, or invent exceptions (e.g., "Since you're sick, I'll waive the fee").
8. CONFIRMATION & EXECUTION LOGIC
8.1 The Explicit Confirmation Gate
To protect against accidental or malicious cancellations, the Assistant MUST obtain an explicit "Yes" (or semantic equivalent) from the guest before transitioning to EXECUTING.
 * Format: "I found your reservation for {{PARTY_SIZE}} on {{DATE}} at {{TIME}}. [Insert Fee Warning if applicable]. Would you like me to go ahead and cancel this?"
 * Guest Declines: "No problem, I'll leave your reservation as is." \rightarrow Transition to TERMINATED.
 * Guest Confirms: "Cancel it." \rightarrow Transition to EXECUTING.
8.2 Execution Suspension
Once the guest confirms, the cancellation execution state must remain pending until the integration returns a deterministic result or timeout/unknown condition.
9. ATOMICITY / IDEMPOTENCY / INTEGRATION SAFETY
9.1 Transactional Race Conditions
If the booking state changes between VALIDATING_BOOKING and EXECUTING (e.g., the guest cancels via the website simultaneously):
 * The Integrations layer will return BOOKING_STATE_CHANGED or ALREADY_CANCELLED.
 * Response: "It looks like the status of this reservation just changed in our system, and it is already marked as cancelled."
9.2 Idempotency
If the guest issues repeated cancellation commands (e.g., "Cancel it. Cancel it now."):
 * The Conversation Engine MUST NOT queue duplicate execution payloads.
 * It MUST wait for the original execution to resolve or rely on the integration contract's idempotency key to prevent double-processing.
9.3 Authoritative Execution Result
The Conversation Engine may state that cancellation has completed ONLY when the Integrations layer returns an authoritative terminal cancellation result, such as CANCELLED or an equivalent explicitly defined terminal success state in the integration contract. Non-terminal states such as REQUEST_ACCEPTED, QUEUED, PROCESSING, or UNKNOWN are NOT sufficient to tell the guest that the reservation has been cancelled.
10. CONTEXT PRESERVATION & CROSS-FLOW ROUTING
10.1 Intent Switching
If the state is CONFIRMING_CANCELLATION and the guest says, "Wait, can I just change the time to 8 PM instead?":
 * Action: Suspend the CE-SPEC-02 Cancellation Flow.
 * Action: Retain the identified booking context.
 * Routing: Route to the Modification Flow (or 07 Escalation.md if modifications are not automated).
 * Context Rule: Do NOT proceed with cancellation.
10.2 Interruption Handling
If the guest interrupts: "Cancel my table. Also, do you sell gift cards?"
 * 09 Multi Question Logic.md decomposes the intent.
 * The Assistant processes the FAQ via Knowledge Base retrieval.
 * The Assistant resumes the active state: "Yes, we sell gift cards at the host stand. Returning to your reservation for tomorrow, would you like me to proceed with cancelling it?"
11. PRIVACY, SAFETY & ESCALATION
11.1 Data Minimization
The Assistant MUST NOT repeat sensitive contact information back to the guest unless clarifying an ambiguity (and even then, using masking where appropriate, e.g., "the number ending in 1234").
11.2 Safety Semantics
The Assistant must base safety routing on semantic safety relevance, not merely the presence of a keyword like "allergy".
 * A. Actual safety-relevant incident or active allergen concern: (e.g., "I need to cancel because my daughter had a serious allergic reaction last time.") \rightarrow Route to CE-SPEC-03 and/or CE-SPEC-07 according to their defined responsibilities.
 * B. Historical/non-actionable mention: \rightarrow Do not automatically hijack the cancellation flow unless the semantic content indicates that staff notification or safety handling is required.
 * C. Generic mention unrelated to safety: \rightarrow Do not trigger the safety flow.
11.3 Mandatory Escalation Triggers
The flow MUST transition to ESCALATING when:
 * The booking cannot be uniquely identified after two complete lookup attempts.
 * The policy evaluates to CONFLICT or UNKNOWN.
 * Refund or fee information is ambiguous or conflicting.
 * The integration layer times out during execution or returns an UNKNOWN execution result.
 * The guest explicitly requests human assistance.
12. FAILURE HANDLING & TIMEOUTS
| Failure Condition | System Action | Guest-Facing Fallback Phrasing |
|---|---|---|
| Integration Timeout | Transition to ESCALATING. | "Our booking system is taking too long to respond. Let me connect you with our staff to ensure this is handled." |
| Unknown Execution Result | Transition to ESCALATING. | "I've sent the cancellation request, but I haven't received confirmation back from the system. Let me connect you with staff to verify." |
| Guest Abandons Flow | Clear active context after {{TIMEOUT_MINUTES}}. | No proactive message. Silently reset to generic listening. |
| Execution Rejected | Transition to ESCALATING. | "The system encountered an error while trying to cancel your booking. Let me get a staff member to help." |
13. EDGE CASES
 * Guest asks to cancel someone else's booking:
   * Trigger: Guest provides a name/phone that does not match their own authenticated channel identity.
   * Rule: Enforce integration-defined authentication. If the integration requires strict matching, fail safely: "For security, I can only cancel reservations linked to your current contact information. Let me connect you with staff."
 * Guest asks "Will I get a refund?":
   * Rule: The Assistant must retrieve the authoritative policy from KB-SPEC-007. It must NOT invent a refund guarantee. If the policy is unclear or ambiguous, escalate.
 * Guest cancels a Private Event / Large Group:
   * Trigger: Integrations layer identifies the booking as a PRIVATE_EVENT or large party exceeding automated cancellation thresholds.
   * Rule: Route to ESCALATING. "Large group bookings require staff assistance to modify or cancel. Let me connect you."
14. PRODUCTION ACCEPTANCE CRITERIA
| AC ID | Category | Requirement | Verification Method | Expected Result | Pass/Fail | Severity |
|---|---|---|---|---|---|---|
| AC-01 | Confirmation Gate | NO state-changing cancellation API call is dispatched before explicit guest confirmation. Test covers normal conversational progression as well as malicious, repeated, or ambiguous cancellation instructions before the confirmation gate. | State Machine Test | EXECUTING unreachable without approval. | Required | Critical |
| AC-02 | Authoritative Result | System never outputs cancellation completion language unless the integration explicitly returns an authoritative terminal cancellation result (e.g., CANCELLED). | Integration Mock Test | Non-terminal responses escalate. | Required | Critical |
| AC-03 | Multiple Match Privacy | Multiple booking matches prompt for disambiguation using only minimum necessary attributes (e.g., date) without leaking unrelated guest information or full contact details. | Context Simulation | No data leakage in prompt. | Required | Critical |
| AC-04 | Policy Adherence | If KB-SPEC-007 dictates a cancellation fee, the Assistant explicitly warns the guest prior to the confirmation gate. | Evaluation Rule Test | Fee warning present in response. | Required | High |
| AC-05 | Idempotency | An already-cancelled booking is identified deterministically and safely reports its verified state without triggering redundant API executions. | State Evaluation | Transitions to TERMINATED. | Required | High |
| AC-06 | Cross-Flow Routing | Interrupting with another intent or changing intent to "modify" safely suspends cancellation and preserves required context. | Intent Switching Test | Cancellation halted; state preserved. | Required | High |
| AC-07 | Policy Ambiguity | If cancellation policy evaluates to UNKNOWN, CONFLICT, or if refund semantics are ambiguous, the flow escalates rather than guessing. | Escalation Validation | Transitions to ESCALATING. | Required | Critical |
| AC-08 | Identification Attempts | The flow deterministically executes a maximum of two complete backend lookup attempts before escalating. | Lookup Simulation | Escalates on 3rd attempt. | Required | High |
| AC-09 | Integration Timeout | Timeout or UNKNOWN execution results from the Integrations layer safely transition the flow to escalation without fabricating success or failure. | Timeout Test | Transitions to ESCALATING. | Required | Critical |
15. VERSION HISTORY
| Version | Date | Description | Author | Approval Status |
|---|---|---|---|---|
| 1.1.0 | August 2026 | Controlled enterprise hardening patch. Clarified authoritative policy ownership (KB-SPEC-007); enforced authoritative terminal execution results; refined identification-attempt definitions; incorporated verified-channel identity handling; separated refund/cancellation semantics; defined terminal-state lifecycle semantics; strengthened pre-confirmation side-effect protection; enhanced booking-match privacy; and instituted semantic safety routing logic. | Ramy Bella | Draft / Implementation Specification |
| 1.0.0 | August 2026 | Initial Cancellation Flow specification. | Ramy Bella | Superseded |
16. FINAL NON-NEGOTIABLE PRINCIPLES
 * NEVER FABRICATE CANCELLATION SUCCESS: Execution is not complete until the integration authoritative layer explicitly returns an authoritative terminal cancellation result.
 * DEFER TO AUTHORITATIVE POLICY: The Assistant must accurately relay cancellation rules and fees derived from KB-SPEC-007. It MUST NOT invent exceptions, refunds, or credits.
 * REQUIRE EXPLICIT CONFIRMATION: The guest must explicitly confirm the cancellation after the booking and relevant policies are presented, prior to any state-changing execution.
 * HANDLE UNKNOWN EXECUTION RESULTS SAFELY: If an integration times out during execution or returns an UNKNOWN state, the system must escalate; it must never assume the cancellation succeeded or failed.
 * PROTECT GUEST DATA: The Assistant must use data minimization when identifying bookings and MUST NOT expose unrelated booking details or full contact information to disambiguate matches.
 * ESCALATE WHEN UNSAFE: If policies conflict, bookings cannot be uniquely identified, refund states are ambiguous, or actionable safety context arises, the system MUST fail closed and escalate to human staff.
 * HANDLE REPEATED CANCELLATION REQUESTS IDEMPOTENTLY: Do not duplicate API calls; rely on verified integration states.
 * NEVER EXECUTE A CANCELLATION FROM STALE OR AMBIGUOUS CONTEXT: Verify the correct booking before any state-changing execution.
