CE-SPEC-03: Allergy Flow
1. DOCUMENT CONTROL
| Attribute | Value |
|---|---|
| Document Title | 03 Allergy Flow.md |
| Document ID | CE-SPEC-03 |
| Version | 1.1.1 |
| Status | Draft / Implementation Specification |
| Author | Ramy Bella |
| Classification | Confidential / Enterprise Proprietary |
| Target Audience | AI Engineers, Conversation Designers, Backend Engineers, QA Engineers, Integration Engineers, Security/Privacy Engineers, Platform Architects |
| Parent Document | 01 AI Identity.md |
| Related Documents | CE-SPEC-01: Booking Flow.md, CE-SPEC-02: Cancellation Flow.md, CE-SPEC-07: Escalation.md, CE-SPEC-09: Multi Question Logic.md, CE-SPEC-10: Follow Up Logic.md, KB-SPEC-006: Allergen Matrix Schema |
| System | Restaurant AI System |
| Phase | Phase 3 — Conversation Engine |
| Last Updated | August 2026 |
2. PURPOSE & SCOPE
Purpose
The Allergy Flow specifies the deterministic conversational state machine required to handle allergy and food-safety-related interactions within the Restaurant AI System. It operationalizes the core principle that the Assistant must never fabricate allergen safety, must distinguish semantic safety relevance from dietary preference, and must strictly defer to the authoritative Knowledge Base (KB-SPEC-006) and integration contracts.
Scope: What This Document Controls
 * Allergy intent detection and semantic safety classification.
 * Conversational logic for requesting and communicating allergen information.
 * Processing of safety-sensitive menu and cross-contact questions.
 * Conversational handling of insufficient, conflicting, or missing safety data.
 * Safety confirmation boundaries and explicit limits on reassurance.
 * Cross-flow routing and context preservation (e.g., handling allergy disclosures mid-booking).
 * Integration handoff logic and data-minimization privacy rules for sensitive health data.
 * Escalation paths for safety-critical uncertainty and emergencies.
Scope: What This Document Explicitly Does NOT Control
 * Medical Diagnosis or Treatment: The Assistant provides zero medical advice.
 * Authoritative Ingredient/Allergen Databases: The actual truth regarding ingredients and EU-14 allergen states is exclusively owned by KB-SPEC-006.
 * Restaurant Kitchen Procedures: The Assistant does not define, control, or independently interpret physical kitchen operations.
 * Final Medical Safety Determination: The Assistant communicates documented facts; the guest makes the final medical decision.
 * Backend Database Execution: The Assistant does not write to the database.
3. RELATIONSHIP TO CORE PRINCIPLES
This flow directly operationalizes the mandates established in 01 AI Identity.md:
 * Never fabricate safety: The Assistant MUST NOT guarantee safety, claim an allergen is absent, or infer safety without explicitly matched authoritative support from the Knowledge Base.
 * Fail closed on uncertainty: If safety information is unavailable, conflicting, or ambiguous, the Assistant MUST escalate to human staff. It MUST NEVER convert uncertainty into reassurance.
 * Defer to authoritative data: The Assistant does not guess ingredient lists based on dish names (e.g., assuming a "Vegan Burger" is nut-free).
 * Preserve context: A safety disclosure during a booking flow must interrupt the flow safely, handle the safety requirement, and resume without losing booking parameters.
 * Protect guest privacy: Health data is strictly minimized and handled according to authorized integration privacy controls.
4. CONVERSATIONAL STATE MODEL
The Allergy Flow operates as a deterministic state machine designed to suspend parent flows, evaluate safety, and resolve deterministically.
| State | Entry Condition | Required Action | Transitions |
|---|---|---|---|
| DETECTED | NLP flags potential dietary, ingredient, or safety intent. | Pass utterance to Semantic Classification. | CLASSIFYING_SAFETY |
| CLASSIFYING_SAFETY | Intent requires categorization (Preference vs. Allergy). | Classify semantic safety relevance. | COLLECTING_ALLERGEN_CONTEXT, RESUMING_PARENT_FLOW (if mere preference without safety context) |
| COLLECTING_ALLERGEN_CONTEXT | Safety intent classified but context is incomplete. | Request minimum necessary clarifying parameters (e.g., specific allergen, specific dish). | VALIDATING_INFORMATION, ESCALATING |
| VALIDATING_INFORMATION | Safety context collected. | Query authoritative KB-SPEC-006 or integrations for matching safety data. Suspend conversation. | ASSESSING_RESPONSE, ESCALATING (if timeout/error) |
| ASSESSING_RESPONSE | KB response received. | Evaluate if the data definitively answers the safety question without violating safety boundaries. | SAFE_INFORMATION_AVAILABLE, UNCERTAINTY_REQUIRES_ESCALATION |
| SAFE_INFORMATION_AVAILABLE | Deterministic, verified safety data matches request. | Present authoritative data using strict fallback phrasing. Do not editorialize. | RESUMING_PARENT_FLOW, TERMINATED |
| UNCERTAINTY_REQUIRES_ESCALATION | Data is unknown, conflicting, insufficient, or staff confirmation is required. | Formulate escalation prompt, retain context, and route to human. | ESCALATING |
| ESCALATING | Safety cannot be verified or emergency detected. | Route to 07 Escalation.md with active safety context payload. | N/A |
| RESUMING_PARENT_FLOW | Safety interaction completed successfully without aborting parent intent. | Restore suspended parent context (e.g., CE-SPEC-01) and pass safety flags if required by integration. | N/A |
| TERMINATED | Standalone allergy flow completes. | Clear transient allergy context (unless required for active booking). Return to generic listening. | N/A |
5. ALLERGY INTENT DETECTION & SEMANTIC CLASSIFICATION
The system MUST NOT rely on simplistic keyword triggers (e.g., "peanut"). It MUST evaluate the semantic meaning of the utterance to determine routing.
5.1 Semantic Categories & Required Routing
| Category | Definition / Example | Routing Rule |
|---|---|---|
| A. Active Allergy Disclosure | "I am allergic to peanuts." / "My daughter has a severe shellfish allergy." | Route to CE-SPEC-03. Flag as safety-critical context. |
| B. Safety Question | "Are there peanuts in the satay?" / "Is this safe for celiacs?" | Route to CE-SPEC-03. Query KB-SPEC-006. |
| C. Cross-Contact Question | "Are the fries cooked in the same fryer as the fish?" | Route to CE-SPEC-03. Query KB. Escalate if unverified. |
| D. Dietary Preference | "Can you recommend something without dairy? I don't like milk." / "I don't eat pork." | Do NOT trigger Allergy Flow unless safety semantics are added. Handle via standard Menu Q&A. |
| E. Generic Food Statement | "My friend loves peanuts." | Do NOT trigger Allergy Flow. |
| F. Historical Safety Incident | "I got sick last time I ate there." | Route to CE-SPEC-07 (Escalation) for incident reporting. |
| G. Emergency Context | "I am having trouble breathing." / "Where is the epipen?" | Route immediately to Authorized Emergency Protocol. |
| H. Ambiguous Statement | "No nuts for me." | Treat as Active Allergy Disclosure (Fail Safe) and clarify intent. |
6. ALLERGEN CONTEXT COLLECTION
When an active allergy or safety question is detected but lacks the necessary context to query the Knowledge Base, the system MUST collect clarification using Data Minimization.
 * Required Parameters: Target Allergen(s), Target Dish(es) (if inquiring about a specific menu item).
 * Prohibited Actions: The Assistant MUST NOT pressure the guest to disclose medical history, severity levels, symptom types, or the identity of the affected guest unless the specific venue integration contract strictly requires it for a safety protocol.
 * Clarification Phrasing: If a guest says "Is it safe?", the Assistant MUST ask: "To ensure I check the correct information, which specific allergens do you need to avoid?"
7. AUTHORITATIVE INFORMATION VALIDATION
The Conversation Engine queries KB-SPEC-006 or the Integrations layer. The Assistant MUST faithfully represent the returned state and MUST NOT make independent safety judgments.
| Authoritative KB State | Meaning | Permitted Conversational Action |
|---|---|---|
| VERIFIED_INFORMATION_AVAILABLE | Data exists and maps directly to the guest's query (e.g., FREE_FROM or CONTAINS). | Proceed to SAFE_INFORMATION_AVAILABLE. Communicate exact state. |
| INSUFFICIENT_INFORMATION | The allergen state is UNKNOWN or unmapped in the KB. | Transition to UNCERTAINTY_REQUIRES_ESCALATION. |
| CONFLICTING_INFORMATION | The KB reports a CONFLICT state for the allergen/item. | Transition to UNCERTAINTY_REQUIRES_ESCALATION. |
| STALE_OR_UNVERIFIED_INFORMATION | The record exists but lacks VERIFIED status. | Transition to UNCERTAINTY_REQUIRES_ESCALATION. |
| HUMAN_CONFIRMATION_REQUIRED | The venue configuration mandates a staff check for this specific allergen/dish. | Transition to UNCERTAINTY_REQUIRES_ESCALATION. |
8. SAFETY RESPONSE LOGIC
The Assistant's response MUST directly reflect the authoritative data without semantic embellishment.
Strict Output Rules:
 * Rule 1: The Assistant MUST NOT turn "no known allergen listed" into "100% safe."
 * Rule 2: The Assistant MUST NOT guarantee zero cross-contact unless KB-SPEC-006 or the venue's explicit policy explicitly provides a FREE_FROM_CROSS_CONTACT guarantee for that specific context.
 * Rule 3: The Assistant MUST NOT invent ingredient substitutions (e.g., "We can just take the nuts off").
Deterministic Phrasing Examples:
 * Explicitly Present (CONTAINS): "According to our menu data, the {{DISH_NAME}} contains {{ALLERGEN}}."
 * Explicitly Absent (FREE_FROM): "Based on our verified menu data, the {{DISH_NAME}} is prepared free from {{ALLERGEN}}."
 * Cross-Contact Risk (MAY_CONTAIN): "The venue states that the {{DISH_NAME}} may contain traces of {{ALLERGEN}} due to shared preparation areas."
 * Incomplete/Unknown (UNKNOWN): "I do not have verified information regarding {{ALLERGEN}} in the {{DISH_NAME}}. Let me connect you with a staff member to confirm."
9. CROSS-CONTACT / KITCHEN SAFETY BOUNDARY
This section enforces the critical distinction between ingredient lists and operational reality.
 * Ingredient Presence vs. Cross-Contact: "Peanuts are not listed as an ingredient" does NOT automatically mean "There is no peanut cross-contact." The Assistant MUST clearly separate these concepts.
 * FREE_FROM Semantics: A FREE_FROM ingredient status does NOT automatically mean "guaranteed safe from cross-contact." A FREE_FROM result may only be communicated within the exact scope returned by KB-SPEC-006. If the KB explicitly returns an authoritative FREE_FROM_CROSS_CONTACT state for the exact dish/allergen/context, that state may be communicated according to the KB. Otherwise, the Assistant MUST NOT convert ingredient absence into a cross-contact guarantee. Do NOT invent new KB states unless they are explicitly described as optional/contract-dependent.
 * Kitchen Procedures vs. Medical Guarantee: "The kitchen uses separate cutting boards" does NOT mean "This meal is guaranteed medically safe."
 * Guest Inquiries: If a guest explicitly asks: "Can you guarantee there will be no cross-contact?", the Assistant MUST evaluate the authoritative policy. If no explicit absolute guarantee is provided by the KB, the Assistant MUST state: "I cannot provide a medical guarantee against cross-contact. I will need to connect you with the kitchen staff to discuss their specific preparation controls."
10. EMERGENCY / ACUTE SAFETY CONTEXT
The Conversation Engine MUST NEVER attempt diagnosis or treatment. The Allergy Flow only detects and routes the emergency; it does not own the emergency response protocol.
10.1 Active Emergency Language
 * Trigger: Guest indicates active distress (e.g., "I am having trouble breathing after eating," "Where is the epipen?").
 * System Action:
   * Emergency detected.
   * Suspend normal flows.
   * Invoke authorized emergency protocol.
   * Attempt highest-priority escalation through CE-SPEC-07.
   * Preserve safety context.
 * Response: The Assistant MAY state that staff has been alerted ONLY after the authorized escalation/notification integration returns an explicit ESCALATION_ACCEPTED or NOTIFICATION_CONFIRMED result. If no such confirmation exists, the Assistant MUST NOT claim that staff has been alerted. Use a fallback such as: "If you are experiencing a medical emergency, please contact emergency services ({{LOCAL_EMERGENCY_NUMBER}}) immediately. I will escalate this to restaurant staff through the available emergency support channel."
 * Routing: Emergency handling is delegated to the authoritative emergency/escalation protocol defined by the platform's approved emergency handling configuration and CE-SPEC-07 where applicable.
10.2 Historical Incident
 * Trigger: "My husband had an allergic reaction last night."
 * System Action: Do not diagnose, apologize for medical outcomes, or accept liability.
 * Routing: Transition to ESCALATING for management/incident response.
11. CONTEXT PRESERVATION & CROSS-FLOW ROUTING
When a safety intent interrupts another flow, the system MUST preserve the active context.
11.1 Interruption of Booking (CE-SPEC-01)
 * Scenario: Guest says "Book for 2 tomorrow at 7. I have a shellfish allergy."
 * System Action:
   * Parse booking parameters (party_size=2, date=tomorrow, time=19:00). Push booking state to memory.
   * Transition to CE-SPEC-03 to acknowledge the shellfish allergy.
   * The Assistant may state "I have noted your shellfish allergy for the kitchen" ONLY IF:
     * The integration contract authorizes a structured safety field;
     * The safety information is written through the approved structured mechanism;
     * The integration returns a successful/accepted result.
       If the write fails, times out, or is unsupported: Do NOT claim the kitchen was notified. Preserve the booking context, escalate according to the safety rules, and do NOT place the allergy into generic reservation_notes. The Assistant MUST NOT imply that merely detecting or remembering an allergy means the kitchen has been notified, the allergy was successfully attached to the booking, or the booking is automatically safe. Only an authorized successful structured integration write may establish that the safety information was successfully transmitted.
   * Resume Logic: Restore all previously collected booking parameters. The booking state may be preserved, but availability MUST be revalidated where CE-SPEC-01 requires it. Do NOT assume inventory is still valid merely because it was valid before the allergy interruption. Re-run any required validation/inventory step according to CE-SPEC-01. If required booking parameters are missing or ambiguous after the safety interaction, return to the appropriate COLLECTING state. Never ask the guest to repeat information that remains valid in active context.
11.2 Interruption of Cancellation (CE-SPEC-02)
 * Scenario: Guest says "Cancel my reservation because I had an allergic reaction."
 * System Action:
   * Identify booking for cancellation. Suspend cancellation execution.
   * Detect Historical Safety Incident.
   * Route to ESCALATING (management response required). Identifying the cancellation reason does not itself authorize cancellation; cancellation execution still requires the normal CE-SPEC-02 confirmation/execution rules, and the safety incident context must not be silently discarded. Do not silently process standard cancellation without notifying staff of the safety context.
   * Resume Logic: If returning to the cancellation flow, preserve the identified booking context. Resume the prior cancellation state only if that state is still valid. NEVER automatically transition to EXECUTING after the allergy interaction. Explicit cancellation confirmation must still be required.
11.3 Interruption of Other Parent Flows
 * Restore the suspended context according to that flow's own state machine.
 * Never fabricate a completed parent-flow action.
12. PRIVACY & DATA MINIMIZATION
Allergy disclosures are safety-relevant and potentially sensitive health data.
 * Data Minimization: The Assistant MUST collect only what is necessary to answer the menu query or fulfill the booking integration requirements. The Assistant MUST NOT pressure the guest to disclose medical history, severity levels, symptom types, or the identity of the affected guest unless the specific venue integration contract strictly requires it for a safety protocol.
 * Integration Handoff: The Assistant MUST NOT place sensitive allergy information into generic, plain-text fields (e.g., reservation_notes) by default, as this risks exposing health data to unauthorized staff or external systems.
 * Authorized Handling: The system MUST use explicitly supported structured safety fields (e.g., dietary_requirements_array) if supported by the integration contract. If the integration contract does not support secure structured safety fields, the Assistant MUST escalate to staff rather than committing unsafe/unprotected data writes.
13. ESCALATION LOGIC
The Allergy Flow MUST fail closed. The following conditions trigger mandatory transition to ESCALATING:
 * Authoritative Information Unavailable: KB-SPEC-006 returns UNKNOWN or lacks data for the requested dish/allergen.
 * Conflicting Information: KB-SPEC-006 returns CONFLICT.
 * Cross-Contact Unknown: The guest explicitly requests a cross-contact or severe medical safety determination, and the KB lacks an explicit operational guarantee.
 * Staff Confirmation Required: The venue configuration demands human oversight for the specified allergen.
 * Active Emergency / Historical Incident: As defined in Section 10.
 * Guest Request: The guest explicitly asks to speak to the kitchen or staff.
 * Integration Failure: The backend query to the KB or Integrations layer fails.
When escalating, the payload MUST explicitly flag SAFETY_SENSITIVE = TRUE to prioritize human routing.
14. FAILURE HANDLING & TIMEOUTS
| Failure Condition | System Action | Guest-Facing Fallback Phrasing |
|---|---|---|
| Knowledge Base Timeout | Transition to ESCALATING. | "I am currently unable to access our allergen database. Let me connect you with a staff member to ensure your safety." |
| Repeated Clarification Failure | After 2 attempts to clarify the allergen, transition to ESCALATING. | "I want to be absolutely sure we get this right. Let me connect you with staff." |
| Guest Abandons Flow | Clear transient allergy context after {{TIMEOUT_MINUTES}} (unless attached to an active booking state). | No proactive message. |
| Escalation Failure | System cannot reach human staff during a safety inquiry. | "I cannot reach a staff member right now, and I cannot guarantee the safety of the menu items without their confirmation. Please call the restaurant directly at {{CONTACT_INFO}} before ordering." |
15. EDGE CASES
| Edge Case | Conversational Rule / Action |
|---|---|
| 15.1 "Is this allergen-free?" | Query KB for FREE_FROM state. If UNKNOWN, state: "I cannot confirm it is completely allergen-free. Escalating to staff." |
| 15.2 "Can you just remove the allergen?" | Assistant MUST NOT invent recipe modifications. Escalate to staff to verify if cross-contact risk allows removal. |
| 15.3 "I am mildly allergic." | Guest-reported severity MUST NOT weaken, bypass, or downgrade the authoritative safety validation pipeline. "Mild allergy" and "severe allergy" must both trigger the same deterministic safety validation. Severity MUST NOT be used by the Assistant to provide weaker reassurance. The Assistant MUST NOT make medical judgments based on severity. |
| 15.4 "I have a severe allergy." | Guest-reported severity MUST NOT weaken, bypass, or downgrade the authoritative safety validation pipeline. "Mild allergy" and "severe allergy" must both trigger the same deterministic safety validation. Severity MUST NOT be used by the Assistant to provide weaker reassurance. The Assistant MUST NOT make medical judgments based on severity. |
| 15.5 "The website says it is safe." | If the Assistant's authoritative KB differs from the user's claim, trust the KB. If KB = CONFLICT or UNKNOWN, escalate. |
| 15.6 "The waiter told me it was safe." | Do not override KB. Escalate to staff to resolve discrepancy. |
| 15.7 "Can you guarantee no cross-contact?" | See Section 9. Decline to provide a medical guarantee unless explicitly authorized by KB configuration. Escalate. |
| 15.8 Disclosed after booking confirmation. | Append to booking via Integration structured safety fields. If integration fails, escalate to staff immediately. |
| 15.9 Disclosed during cancellation. | Route to CE-SPEC-07 if stated as the reason for cancellation (Incident tracking). |
| 15.10 Mentioned as historical event. | Do not trigger menu evaluation; route to Escalation/Incident handling. |
| 15.11 Multiple allergens queried. | Evaluate every requested allergen independently. The aggregate result MUST be the safest conservative interpretation. Never allow one verified FREE_FROM result to override another allergen's UNKNOWN, CONFLICT, CONTAINS, or MAY_CONTAIN result.

16. CONFLICTING_INFORMATION / STALE_OR_UNVERIFIED_INFORMATION / HUMAN_CONFIRMATION_REQUIRED \rightarrow ESCALATING.
17. INSUFFICIENT_INFORMATION / UNKNOWN \rightarrow ESCALATING.
18. MAY_CONTAIN (cross-contact risk) \rightarrow Must NOT be represented as safe. If the guest's request requires safety clearance, escalate.
19. CONTAINS \rightarrow The item does not satisfy a request to avoid that allergen.
20. Verified FREE_FROM \rightarrow May be communicated only for the exact verified scope represented by KB-SPEC-006.

If any requested allergen produces UNKNOWN, CONFLICT, insufficient verification, or unresolved cross-contact risk where safety clearance is required, the aggregate flow MUST fail closed and escalate. |
| 15.12 Uncovered ingredient query. | If the ingredient is not an EU-14 allergen and not mapped in the KB, state: "I do not have verified information on {{INGREDIENT}}. Connecting to staff." |
| 15.13 Conflicting KB records. | Fail closed. Transition to ESCALATING. |
| 15.14 Recommendation requested but insufficient info. | Do not recommend any item unless verified FREE_FROM for the specified allergen. |
| 15.15 Blaming another restaurant. | Do not comment on liability. Maintain polite neutrality. |
| 15.16 Active emergency language. | Trigger emergency protocol (Section 10.1). |
| 15.17 Multi-intent message. | Decompose via CE-SPEC-09. Handle safety intent FIRST before addressing other intents. |
16. PRODUCTION ACCEPTANCE CRITERIA
| AC ID | Category | Requirement | Verification Method | Expected Result | Pass/Fail | Severity |
|---|---|---|---|---|---|---|
| AC-01 | Semantic Detection | NLP correctly differentiates between "I don't like milk" (Preference) and "I am allergic to milk" (Safety). | Semantic Classification Test | Only the latter triggers CE-SPEC-03. | Required | Critical |
| AC-02 | No Fabrication | System responds with exact KB-SPEC-006 data mappings and never outputs "100% safe" for an UNKNOWN state. | Response Generation Test | Response strictly uses fallback templates. | Required | Critical |
| AC-03 | Cross-Contact | Guest asking for a cross-contact guarantee receives an escalation/refusal unless explicit KB authority exists. FREE_FROM ingredient status cannot be transformed into a cross-contact guarantee. | Edge Case Simulation | Escalates to human staff. | Required | Critical |
| AC-04 | Fail Closed | If KB times out or returns CONFLICT, the Assistant refuses to answer the safety question and escalates. | Mock Timeout Test | Transitions to ESCALATING. | Required | Critical |
| AC-05 | Privacy & Structured Write | Allergen context collected during booking is mapped strictly to structured safety fields, not generic text notes. Assistant cannot claim the kitchen was notified unless the authorized structured write succeeds. | Integration Payload Audit | Valid payload routing. | Required | High |
| AC-06 | Context Preservation | Booking parameters survive the allergy interruption and the booking flow revalidates as required. Cancellation never resumes directly into EXECUTING. | Multi-turn Simulation | Parameters preserved; flows re-validated safely. | Required | Critical |
| AC-07 | Emergency Handling | "I can't breathe" suspends normal flows and invokes authorized emergency protocol. Assistant cannot claim staff was alerted unless the notification/escalation integration confirms it. | Intent Routing Test | Triggers emergency protocol safely and verifies notification confirmation before claiming alert success. | Required | Critical |
| AC-08 | Multi-Allergen | Querying multiple allergens evaluates each independently. One unresolved/unsafe allergen prevents a falsely reassuring aggregate answer, resulting in escalation. | Multi-Entity Logic Test | Safest conservative interpretation; transitions to ESCALATING. | Required | High |
17. VERSION HISTORY
| Version | Date | Description | Author | Approval Status |
|---|---|---|---|---|
| 1.1.1 | August 2026 | Precision and consistency hardening patch: clarified authoritative emergency protocol ownership and notification confirmation, protected structured safety-write semantics, refined deterministic multi-allergen aggregation, enforced strict ingredient vs. cross-contact boundaries, specified deterministic parent-flow resumption revalidation, and corrected minor typos. No architectural redesign. | Ramy Bella | Draft / Implementation Specification |
| 1.1.0 | August 2026 | Controlled enterprise hardening patch. Implemented explicit emergency protocol ownership; authoritative notification confirmation for staff alerts; protected structured safety-write semantics; deterministic multi-allergen aggregation; strict ingredient vs cross-contact boundary; deterministic parent-flow resumption; severity-neutral safety routing; and minor typo correction. | Ramy Bella | Draft / Implementation Specification |
| 1.0.0 | August 2026 | Initial Allergy Flow specification. | Ramy Bella | Superseded |
18. FINAL NON-NEGOTIABLE PRINCIPLES
 * NEVER FABRICATE ALLERGEN SAFETY: The Assistant must never claim an allergen is absent, or infer safety, without explicitly matched authoritative support from the Knowledge Base.
 * DEFER TO AUTHORITATIVE ALLERGEN INFORMATION: All safety assertions are bound mathematically to the state responses of KB-SPEC-006.
 * NEVER GUARANTEE ZERO CROSS-CONTACT WITHOUT AUTHORITATIVE SUPPORT: Distinguish strictly between "not an ingredient" and "safe from cross-contact."
 * FAIL CLOSED WHEN SAFETY INFORMATION IS UNKNOWN OR CONFLICTING: Ambiguity, missing data, or KB conflicts mandate an immediate escalation. Uncertainty must never be converted into reassurance.
 * DISTINGUISH ALLERGY FROM DIETARY PREFERENCE: Use semantic intent, not just keywords, to route safety-critical interactions appropriately.
 * PRESERVE ACTIVE FLOW CONTEXT: Safety interactions must seamlessly suspend and resume parent flows (like bookings) without losing guest data.
 * PROTECT SENSITIVE SAFETY INFORMATION: Minimize collected health data and transmit it only via authorized, structured privacy controls. Do not dump medical data into plain-text notes.
 * ESCALATE WHEN DETERMINISTIC SAFE RESOLUTION IS IMPOSSIBLE: Do not guess. Do not loop. Handoff to human staff.
 * NEVER PROVIDE MEDICAL DIAGNOSIS OR TREATMENT: The Assistant provides documented menu facts; the guest is responsible for their medical decisions. Emergency detection triggers the authorized emergency/escalation protocol; staff notification may only be represented as completed when the integration explicitly confirms it, and standardized emergency phrasing must be used. Do not imply that detection itself guarantees successful staff notification.
