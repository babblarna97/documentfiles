CE-SPEC-01: Booking Flow
1. DOCUMENT CONTROL
| Attribute | Value |
|---|---|
| Document Title | 01 Booking Flow.md |
| Document ID | CE-SPEC-01 |
| Version | 1.1.0 |
| Status | 🔒 LOCKED / Approved for Implementation |
| Author | Ramy Bella |
| Classification | Confidential / Enterprise Proprietary |
| Target Audience | AI Engineers, Conversation Designers, Backend Engineers, QA Engineers, Platform Architects |
| Parent Document | 01 AI Identity.md (Master AI System Definition) |
| Related Documents | KB-SPEC-004, KB-SPEC-008, 02 Cancellation Flow.md, 03 Allergy Flow.md, 07 Escalation.md, 09 Multi Question Logic.md |
| System | Restaurant AI System |
| Phase | Phase 3 — Conversation Engine |
| Last Updated | August 2026 |
2. PURPOSE & SCOPE
Purpose
The Booking Flow specifies the deterministic conversational state machine required to guide a guest from initial reservation intent to successful booking execution handoff. It operationalizes the core principle that the Assistant must prioritize accuracy and never fabricate availability, while maintaining a hospitable, context-aware experience.
Scope: What This Document Controls
 * Intent detection and trigger conditions for reservation requests.
 * State transitions for parameter collection.
 * The conversational handling of rule violations derived from KB-SPEC-008 (Booking Rules Schema).
 * The logic for offering alternative time slots when primary requests fail.
 * Context preservation during mid-flow guest corrections.
 * Handoff protocols and atomicity gates to the Integrations layer for actual execution.
Scope: What This Document Explicitly Does NOT Control
 * Booking Rules Definition: The rules governing party sizes and lead times are owned by KB-SPEC-008. This document solely defines how the Assistant converses about those rules.
 * Database Execution: The Assistant does not write to the reservation database. It hands off a structured payload to the Integrations layer.
 * Allergen Safety: Conversational handling of severe dietary restrictions during a booking is strictly routed to 03 Allergy Flow.md.
 * Multi-Intent Orchestration: Parsing a message like "Book a table and what are your vegan options?" is governed by 09 Multi Question Logic.md.
3. RELATIONSHIP TO CORE PRINCIPLES
This flow operationalizes the following mandates from 01 AI Identity:
 * Never fabricate information: The Assistant MUST NOT confirm a table without explicit ELIGIBLE status from the Booking Engine and a verified hold.
 * Context must be preserved: If a guest says "Actually, make it 4 people," the system MUST retain the previously collected Date and Time.
 * Escalation limits: Requests that mathematically violate KB-SPEC-008 (e.g., party size exceeds maximum) MUST fail safely or trigger 07 Escalation.md if authorized, rather than silently altering the request.
4. CONVERSATIONAL STATE MODEL
The Booking Flow operates as a stateful machine. The current conversational state dictates the Assistant's permissible actions and required validations.
4.1 State Definitions
| State | Entry Condition | Required Action |
|---|---|---|
| DETECTED | Guest expresses booking intent. | Extract any provided parameters from the utterance. Move to COLLECTING. |
| COLLECTING | 1 or more required parameters are missing. | Prompt the guest for the specific missing parameter(s). |
| VALIDATING_KB | All required parameters collected. | Query KB-SPEC-008 rules via backend. Suspend conversation until deterministic response returns. |
| VALIDATING_INV | KB Rules passed. | Query Integrations layer for real-time table inventory. |
| NEGOTIATING | Inventory check returns UNAVAILABLE. | Present pre-validated alternative times provided by the Integrations layer. |
| CONFIRMING | Inventory check returns AVAILABLE. | Present final details. Request explicit guest confirmation. Requires an execution-safe reservation hold/token from the Integrations layer to prevent race conditions. |
| HANDOFF | Guest confirms details. | Execute final availability re-check (if no hard token exists), pass locked payload to Integrations. Terminate flow. |
| ESCALATING | Flow cannot resolve mathematically or system times out. | Route to 07 Escalation.md with active context payload. |
| TERMINATED | Guest explicitly cancels intent or timeout limit is reached. | Clear active booking context. Return to generic listening state. |
5. PARAMETER COLLECTION LOGIC
5.1 Required Parameters
The minimum booking eligibility set consists of exactly three parameters. Additional parameters MAY become mandatory when required by the venue configuration, Booking Engine, or integration contract (e.g., requested_scope, booking_channel, contact_information).
The baseline minimum set is:
 * party_size: Integer > 0.
 * date: Resolved to a specific calendar date (YYYY-MM-DD) in the venue's canonical IANA timezone (KB-SPEC-004).
 * time: Resolved to a specific HH:MM in the venue's canonical IANA timezone (KB-SPEC-004).
5.2 Extraction & Disambiguation Rules
 * Explicit Extraction: If the guest says, "Table for 2 tomorrow at 7 PM", the NLP layer MUST extract all required parameters immediately and bypass the COLLECTING state prompts.
 * Relative Date Resolution: Terms like "tomorrow" or "next Friday" MUST be resolved mathematically against the venue's canonical IANA timezone current time, not UTC or the user's browser time.
 * Ambiguous Time Resolution: "7" MUST be disambiguated. The system MUST NOT assume 19:00 vs 07:00 without checking KB-SPEC-004 (Opening Hours). If both are valid open hours, the Assistant MUST ask for clarification: "Did you mean 7:00 AM or 7:00 PM?"
5.3 Omission of Unnecessary Prompts
The Assistant MUST NOT ask a question if the parameter is already populated in the active state context.
6. VALIDATION ROUTING & KB-SPEC-008 INTEGRATION
Once the minimum eligibility set is collected, the Conversation Engine MUST halt guest prompting and query the backend validator.
6.1 Handling KB-SPEC-008 Rejections
If the backend returns an INELIGIBLE or CONFLICT state based on Booking Rules, the Assistant MUST execute the following deterministic responses:
| KB-SPEC-008 Violation | Assistant Response Strategy |
|---|---|
| MAX_PARTY_SIZE_EXCEEDED | "For groups larger than {{MAX_PARTY}}, please contact us directly at {{CONTACT_INFO}}." (Transitions to ESCALATING) |
| MIN_LEAD_TIME_VIOLATION | "We require at least {{LEAD_TIME_HOURS}} hours notice for reservations. Can I check availability for a later time?" |
| OUTSIDE_OPENING_HOURS | "We are closed at {{REQUESTED_TIME}}. Our hours on {{REQUESTED_DATE}} are {{OPENING_HOURS}}. What time would you prefer?" |
| CONFLICT / UNKNOWN | "I'm currently unable to verify our booking limits for that request. Let me connect you with our staff." (Transitions to ESCALATING) |
7. INVENTORY NEGOTIATION & ATOMICITY
If KB constraints pass, the system checks real-time inventory via the Integrations layer.
7.1 Alternative Offer Constraints
If the requested time is unavailable, the Integrations layer will return an array of alternative times (e.g., ["18:30", "19:30"]).
 * Rule 1 (Pre-Validation): The Integrations layer MUST guarantee that any alternative returned is already evaluated as AVAILABLE + KB-RULE-ELIGIBLE + OPEN-HOURS-VALID. The Conversation Engine MUST NOT be forced to re-validate alternative suggestions against KB-SPEC-008.
 * Rule 2 (No Fabrication): The Assistant MUST present the alternatives exactly as provided. If the Integrations layer returns an empty array, the Assistant MUST state: "I'm sorry, but we have no tables available on {{date}} around that time."
 * Rule 3 (Conciseness): If offering alternatives, the phrasing MUST be concise: "We don't have {{time}} available, but I can offer you {{alt_1}} or {{alt_2}}. Do either of those work?"
7.2 Freshness & Atomicity Gate (Crucial)
A race condition exists between the system discovering an AVAILABLE slot and the guest confirming it.
 * The Conversation Engine MUST demand an execution-safe reservation context (e.g., an inventory hold, quote, or time-locked token) from the Integrations layer prior to entering the CONFIRMING state.
 * If the Integration layer does not support holds, the Conversation Engine MUST trigger a final, silent availability re-check in the HANDOFF state immediately before committing the transaction. If the slot has been taken, it MUST revert to NEGOTIATING.
8. CONTEXT PRESERVATION & MODIFICATION
Guests frequently alter parameters mid-flow. The Conversation Engine MUST handle state modifications gracefully.
8.1 Modification Handling
If the state is CONFIRMING or NEGOTIATING and the guest says, "Actually, make it for 4 people":
 * Update party_size context to 4.
 * Retain date and time in the active booking context.
 * Revert state to VALIDATING_KB.
 * Re-run the entire validation and inventory pipeline.
8.2 Intent Switching
If the state is COLLECTING and the guest asks, "Do you have vegan options?":
 * Suspend the 01 Booking Flow.md state.
 * Route the query to 05 Menu Questions.md.
 * Upon completion of the menu query, the Follow Up Logic (10 Follow Up Logic.md) MUST resume the Booking Flow: "Yes, we have several vegan options. Now, what time would you like your table for 2 on Friday?"
9. CROSS-FLOW DEPENDENCIES & SAFETY
9.1 Semantic Allergen Detection During Booking
The Conversation Engine MUST react to safety-relevant semantic intent, not mere keywords.
 * Detection: "I am allergic to peanuts" \rightarrow Safety Intent. "My friend loves peanuts" \rightarrow Generic Intent.
 * Routing: If a safety-relevant semantic intent is detected during parameter collection or confirmation, the system MUST immediately trigger 03 Allergy Flow.md to handle safety acknowledgment. The Booking Flow state is preserved during this excursion.
9.2 Strict Privacy Rules for Allergen Handoff
Allergy information MUST NOT be persisted to plain-text reservation notes by default.
 * If the venue requires allergen data attached to the booking, the Integrations layer MUST use explicitly supported structured safety fields, containing only the minimum necessary information.
 * This transmission MUST comply with authorized integration handling and applicable privacy/consent controls as defined by Core Intelligence.
9.3 Emergency or Escalation Triggers
If the Integrations API times out, or the guest expresses frustration:
 * Transition immediately to ESCALATING (triggering 07 Escalation.md).
 * Pass all collected parameters (date, time, party_size) into the escalation payload so the human agent does not need to ask the guest to repeat themselves.
10. FAILURE HANDLING & TIMEOUTS
| Failure Condition | System Action | Guest-Facing Fallback Phrasing |
|---|---|---|
| API Timeout (Integrations) | Transition to ESCALATING. | "Our reservation system is taking too long to respond. Let me connect you with a staff member to finalize your booking." |
| Guest Abandons Flow | If no reply in {{TIMEOUT_MINUTES}}, purge active booking context, transition to TERMINATED. | No proactive message. Silently reset to generic listening. Subsequent requests trigger a fresh DETECTED intent. |
| Ambiguous Date Input | E.g., "Book a table for the 12th" (month unknown). | "Just to be sure, did you mean {{CURRENT_MONTH}} 12th or {{NEXT_MONTH}} 12th?" |
| Unsupported Booking Type | E.g., "Can I book the entire restaurant?" | "For full restaurant buyouts, we need to handle that personally. Let me get you our events team's contact info." |
11. EDGE CASES
11.1 The "Squeeze Us In" Edge Case
 * Guest Input: "I know you are full at 7, but can you just squeeze 2 people in at the bar?"
 * Engine Rule: The Assistant MUST NOT override the Booking Engine's UNAVAILABLE status.
 * Engine Rule: The Assistant MUST NOT promise walk-in availability unless KB-SPEC-008 explicitly marks walk_in_behavior: ALLOWED.
 * Response: "I'm sorry, but I can only book available tables in our system. I can't guarantee a spot at the bar, but you are welcome to try walking in." (Conditioned on WALK_IN_POLICY).
11.2 Historical Date Requests
 * Guest Input: Requesting a date in the past.
 * Engine Rule: NLP extraction must evaluate requested_date < CURRENT_DATE in the venue's canonical timezone.
 * Response: "It looks like that date has already passed. What future date were you looking for?"
12. PRODUCTION ACCEPTANCE CRITERIA
| AC ID | Category | Requirement | Verification Method | Expected Result | Pass/Fail | Severity |
|---|---|---|---|---|---|---|
| AC-01 | Parameter Collection | Flow does not advance to VALIDATING_KB until minimum eligibility parameters are collected. | State Machine Unit Test | Flow remains in COLLECTING. | Required | Critical |
| AC-02 | Context Preservation | Changing party size at the CONFIRMING stage retains the original date and time. | Multi-turn Simulation | Payload retains date/time; updates party. | Required | Critical |
| AC-03 | Safety Handoff | Semantic allergy intent mid-flow triggers 03 Allergy Flow.md without losing booking context. | Integration Test | Flow switches, acknowledges, resumes. | Required | Critical |
| AC-04 | Privacy | Allergen data is not dumped into plain-text reservation notes. | Data Pipeline Audit | Payload uses structured safety fields only. | Required | Critical |
| AC-05 | Atomicity Gate | Confirmation handoff executes a token hold or real-time re-check to prevent double-booking. | Race Condition Test | Double booking fails safely. | Required | Critical |
| AC-06 | Rule Adherence | A request violating max_party_size from KB-SPEC-008 generates a deterministic escalation route. | End-to-End Test | Escalation message served. | Required | High |
| AC-07 | Timeout Semantics | Flow abandonment triggers TERMINATED and completely purges the active booking context. | Context Expiration Test | Old context discarded on return. | Required | High |
13. VERSION HISTORY
| Version | Date | Description | Author | Approval Status |
|---|---|---|---|---|
| 1.1.0 | August 2026 | Enterprise safety patch. Introduced execution freshness/atomicity gate, expanded parameter eligibility, established semantic (non-keyword) allergy triggers with strict privacy requirements, defined specific timezone resolution, and clarified ESCALATING / TERMINATED timeout bounds. | Ramy Bella | 🔒 LOCKED / Approved for Implementation |
| 1.0.0 | August 2026 | Initial Booking Flow specification. | Ramy Bella | Superseded |
14. FINAL NON-NEGOTIABLE PRINCIPLES
 * NEVER FABRICATE AVAILABILITY: If the backend does not explicitly return AVAILABLE, the Assistant cannot confirm the booking.
 * DEFER TO KB-SPEC-008: The Conversation Engine does not decide what is mathematically permissible; it only routes the guest through the UX of those decisions.
 * TRANSACTIONAL ATOMICITY: The Assistant MUST utilize a hold token or final re-check to prevent race conditions during confirmation.
 * RESPECT THE ALLERGY & PRIVACY BOUNDARY: Semantic dietary hazard intents must trigger strict safety logic (03 Allergy Flow.md) and must never be carelessly written to generic plain-text reservation notes.
 * STATE IS SACRED: Never ask the guest to repeat a parameter they have already provided in the current active booking context unless clarifying an ambiguity.
 * FAIL SAFELY: If the flow cannot resolve, it must escalate (ESCALATING) rather than looping infinitely or guessing.
