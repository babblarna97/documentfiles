CE-SPEC-04: Recommendation Flow
1. DOCUMENT CONTROL
| Attribute | Value |
|---|---|
| Document Title | 04 Recommendation Flow.md |
| Document ID | CE-SPEC-04 |
| Version | 1.1.0 |
| Status | Draft / Implementation Specification |
| Author | Ramy Bella |
| Classification | Confidential / Enterprise Proprietary |
| Target Audience | AI Engineers, Conversation Designers, Backend Engineers, QA Engineers, Integration Engineers, Security/Privacy Engineers, Platform Architects |
| Parent Document | 01 AI Identity.md |
| Related Documents | CE-SPEC-01, CE-SPEC-02, CE-SPEC-03, CE-SPEC-07, CE-SPEC-09, CE-SPEC-10, KB-SPEC-005, KB-SPEC-006 |
| System | Restaurant AI System |
| Phase | Phase 3 — Conversation Engine |
| Last Updated | August 2026 |
2. PURPOSE & SCOPE
Purpose
The Recommendation Flow specifies the deterministic conversational state machine required to collect guest preferences, validate them against authoritative constraints, retrieve matching candidates, and present mathematically filtered food, beverage, or operational recommendations.
Scope: What This Document Controls
 * Recommendation intent detection and semantic classification.
 * Requirement and constraint collection (Hard vs. Soft vs. Safety).
 * Candidate retrieval, filtering, and deterministic ranking logic.
 * Conversational response generation boundaries (preventing fabricated praise).
 * Cross-flow routing and context preservation (e.g., recommendations mid-booking).
 * Fail-safe handling of insufficient data, missing candidates, and conflicting constraints.
Scope: What This Document Explicitly Does NOT Control
 * Medical Advice: The Assistant does not provide health guidance.
 * Allergen Truth: Owned exclusively by KB-SPEC-006.
 * Menu/Price Truth: Owned exclusively by KB-SPEC-005.
 * Booking/Cancellation Execution: Owned by CE-SPEC-01 and CE-SPEC-02.
 * Inventory Truth: The Assistant does not track real-time stock unless explicitly provided by an authorized integration API.
 * Restaurant Operational Policies: Owned by KB-SPEC-007.
 * Unsupported Assumptions: The Assistant does not infer flavor profiles, portion sizes, or popularity unless explicitly defined in the Knowledge Base.
3. RELATIONSHIP TO CORE PRINCIPLES
This flow operationalizes the core mandates from 01 AI Identity.md:
 * Zero Fabrication: The Assistant MUST NOT invent dishes, ingredients, prices, or availability.
 * Authoritative KB Grounding: Every recommended item MUST be a verified record retrieved from KB-SPEC-005.
 * Deterministic Behavior: Candidate filtering uses strict Boolean logic. Ranking cannot override filtering.
 * Safety-First Routing: Any mention of allergens or dietary hazards MUST suspend the recommendation and route to CE-SPEC-03.
 * Preference vs. Safety Distinction: Semantic intent determines if "no dairy" is a soft preference or a hard medical safety constraint.
 * Privacy/Data Minimization: Dietary data is collected only to execute the active search and is never stored persistently without authorized consent.
 * Context Preservation: Parent flows (e.g., Booking) are safely suspended and resumed.
 * Fail-Closed: If the KB is unreachable or no items match the hard constraints, the system MUST state that no recommendation can be made, rather than hallucinating an approximate fit.
4. CONVERSATIONAL STATE MODEL
The Recommendation Flow operates as a deterministic state machine.
| State | Entry Condition | Required Action | Transitions |
|---|---|---|---|
| DETECTED | NLP flags potential recommendation, menu query, or preference intent. | Pass to semantic classifier. | ANALYZING_INTENT |
| ANALYZING_INTENT | Intent requires categorization. | Extract constraints, preferences, and safety markers. | SAFETY_ROUTING, COLLECTING_REQUIREMENTS, RETRIEVING_CANDIDATES |
| SAFETY_ROUTING | Semantic safety or allergy intent detected. | Suspend flow, preserve context, route to CE-SPEC-03. | RESUMING_FROM_SAFETY |
| COLLECTING_REQUIREMENTS | Constraints are too broad to provide a useful result. | Prompt guest for minimum necessary clarification (e.g., "Are you looking for food or drinks?"). | RETRIEVING_CANDIDATES, TERMINATED |
| RETRIEVING_CANDIDATES | Sufficient constraints exist. | Query authoritative KB (KB-SPEC-005) / Integrations. Suspend conversation. | FILTERING_AND_RANKING, ESCALATING (on error) |
| FILTERING_AND_RANKING | KB response received. | Apply strict filtering based on hard/safety constraints, then sort by soft preferences. | GENERATING_RESPONSE, NO_CANDIDATES_FOUND |
| GENERATING_RESPONSE | 1 or more candidates survive filtering. | Formulate deterministic response using authoritative facts. | PRESENTING_RECOMMENDATION |
| PRESENTING_RECOMMENDATION | Response formulated. | Deliver recommendation to guest. | TERMINATED, RESUMING_PARENT_FLOW |
| NO_CANDIDATES_FOUND | 0 candidates survive filtering. | Inform guest no exact match exists. Offer to relax soft constraints or escalate. | COLLECTING_REQUIREMENTS, TERMINATED, ESCALATING |
| ESCALATING | KB failure, conflicting rules, or explicit staff request. | Route to CE-SPEC-07. | N/A |
| TERMINATED | Flow completes or guest aborts. | Clear transient recommendation context. Return to generic listening. | N/A |
5. RECOMMENDATION INTENT DETECTION
The system MUST use semantic classification rather than keyword-only detection.
 * Explicit recommendation request: "What do you recommend for dinner?" \rightarrow Standard retrieval.
 * Preference-based request: "I love spicy food." \rightarrow Standard retrieval with spicy attribute filter.
 * Dietary restriction: "I am vegan." \rightarrow Apply strict dietary hard-constraint filter ONLY if the relevant dietary attribute exists and is verified in the authoritative KB-SPEC-005 data.
 * Allergy/safety request: "I am allergic to nuts, what can I eat?" \rightarrow Mandatory immediate route to CE-SPEC-03.
 * Budget constraint: "Show me wines under €50." \rightarrow Apply price hard-constraint filter.
 * Occasion/context request: "What's good for a birthday?" \rightarrow Search for contextual tags (e.g., celebratory, sharing) in KB.
 * Availability-dependent recommendation: "What's the special tonight?" \rightarrow Query real-time integration/KB status.
 * Generic menu question: "Do you serve steak?" \rightarrow Standard retrieval, no ranking required.
 * Multi-intent request: "Book a table for 2 and recommend a good red wine." \rightarrow Decompose via CE-SPEC-09, execute Booking first, then run Recommendation.
6. REQUIREMENT & CONSTRAINT COLLECTION
The system MUST NOT ask unnecessary questions. If sufficient information already exists in the active context, proceed to retrieval.
Constraint Hierarchy
 * Safety Constraints: Allergens, severe medical restrictions (Handled via CE-SPEC-03 then passed as filters).
 * Hard Constraints: Explicit exclusions ("no meat"), dietary requirements ("vegan"), budget limits ("under €20").
 * Operational Constraints: Time of day ("breakfast menu"), availability status.
 * Soft Preferences: "Spicy", "light", "sweet".
 * Contextual Preferences: "Sharing", "romantic".
If the user says "Recommend something," the Assistant MAY ask a single clarifying question (e.g., "Are you looking for a starter, main course, or drinks?") to narrow the scope, but MUST NOT engage in a protracted 20-questions sequence.
7. AUTHORITATIVE DATA VALIDATION
All factual recommendation inputs MUST originate from KB-SPEC-005 (Menu), KB-SPEC-006 (Allergens), or authorized Integrations.
The Assistant MUST NOT invent dishes, ingredients, allergens, prices, availability, preparation methods, dietary properties, restaurant policies, substitutions, portion sizes, or nutritional claims.
| KB State | Assistant Behavior |
|---|---|
| VERIFIED_INFORMATION_AVAILABLE | Extract data. Proceed to filtering. |
| INSUFFICIENT_INFORMATION | Do not recommend the item. If all items lack info, fail to NO_CANDIDATES_FOUND. |
| CONFLICTING_INFORMATION | Candidate-level conflict: Immediately discard candidate. Systemic KB conflict (preventing reliable recommendation): Transition to ESCALATING. |
| STALE_OR_UNVERIFIED_INFORMATION | Discard candidate from recommendation pool. |
| HUMAN_CONFIRMATION_REQUIRED | Discard candidate from automated recommendation pool. |
8. CANDIDATE RETRIEVAL & FILTERING
The system executes a deterministic recommendation pipeline:
User Intent \rightarrow Requirement Extraction \rightarrow Safety/Constraint Validation \rightarrow KB Retrieval \rightarrow Candidate Filtering \rightarrow Candidate Ranking \rightarrow Response Generation
Deterministic Filtering Rules
 * Elimination: A candidate MUST NOT survive filtering if it violates an explicit safety constraint or a hard user constraint.
 * No Override: Ranking logic MUST NEVER override a safety or hard constraint. If a user asks for "Your most popular vegan dish," and the most popular dish is not verified VEGAN in KB-SPEC-005, it MUST be eliminated.
9. RANKING LOGIC
If multiple candidates survive filtering, they are ranked deterministically based on structured inputs.
 * Deterministic Inputs:
   * Explicit user preference (e.g., ingredient match count).
   * Dietary requirement match.
   * Price preference (closest to target without exceeding).
   * Popularity/Signature status, ONLY if explicitly defined as a structured attribute in KB-SPEC-005 (e.g., is_signature: true).
 * Prohibited Behavior:
   * Never use hidden LLM weights or external internet knowledge as ranking facts.
   * If no ranking data (like popularity) is explicitly defined in the KB, the Assistant MUST NOT fabricate ranking justification. Items must be presented neutrally.
10. ALLERGY & SAFETY INTERACTION
The Recommendation Flow MUST integrate seamlessly with CE-SPEC-03: Allergy Flow.
 * Detection: If an allergy or medical safety requirement is detected, the flow MUST suspend and route to CE-SPEC-03.
 * Execution Boundary: The Recommendation Flow MUST NOT independently determine allergen safety. It receives validated safety constraints back from CE-SPEC-03.
 * Safety Primacy: Safety constraints ALWAYS override recommendation preferences.
 * No Assumption: The Assistant MUST NOT transform UNKNOWN safety states into "safe". It MUST NOT transform a FREE_FROM ingredient status into a cross-contact guarantee.
 * Context: The recommendation request context is preserved while the safety flow executes.
11. RESPONSE GENERATION
Recommendations MUST clearly distinguish between verified facts, user preferences, and recommendation rationale.
11.1 Allowed Output Construction
 * Fact: "The {{DISH_NAME}} is {{PRICE}}."
 * Rationale: "Since you asked for something spicy and vegan, I recommend the {{DISH_NAME}}."
 * KB-Backed Status: "It is one of our signature dishes." (Requires KB flag).
11.2 Prohibited Unsupported Language
The Assistant MUST NOT imply certainty beyond available evidence. Banned phrasing unless explicitly supported by authoritative data:
 * "Definitely"
 * "Guaranteed"
 * "100% safe"
 * "Best" (Unless supported by an authoritative most_popular ranking metric).
 * "Healthy" (Unless supported by a verified KB dietary claim).
12. INSUFFICIENT / CONFLICTING INFORMATION
The system MUST fail safely if required recommendation information is missing, conflicting, stale, unverified, or unavailable. The system MUST NOT fabricate an answer.
| Scenario | Deterministic Behavior |
|---|---|
| No items match hard constraints | State: NO_CANDIDATES_FOUND. "I don't have any items that match all those requirements. Would you like me to look for something else?" |
| Price/Attribute Unknown | Exclude item from constraint-based recommendation pool. |
| KB Unreachable / Timeout | Transition to ESCALATING. "I can't access the menu right now. Let me connect you with staff." |
| Conflicting Constraints | E.g., "Vegan steak." Explain conflict: "Our steaks are meat-based, but we have a vegan portobello dish. Which would you prefer?" |
13. CONTEXT PRESERVATION & CROSS-FLOW ROUTING
13.1 Interruption by Recommendation
If the guest is in CE-SPEC-01 (Booking Flow) and says, "Book for 2 at 7 PM. Also, do you have good seafood?":
 * CE-SPEC-09 (Multi Question Logic) routes the menu query to CE-SPEC-04.
 * CE-SPEC-04 processes the recommendation.
 * CE-SPEC-04 transitions to RESUMING_PARENT_FLOW.
 * CE-SPEC-01 resumes, maintaining party size and time context, and proceeds to confirmation.
   Rule: CE-SPEC-04 MUST NOT fabricate that a booking or cancellation has completed.
13.2 Interruption of Recommendation
If the guest is receiving a recommendation and states a safety concern ("Actually, does that have dairy?"):
 * Transition to SAFETY_ROUTING.
 * CE-SPEC-03 assumes control to evaluate the safety claim.
 * Safety intent takes absolute priority over ordinary recommendation intent.
14. MULTI-INTENT & MULTI-CONSTRAINT LOGIC
Scenario: "Recommend something vegetarian under €25 that is safe for my nut allergy."
The system processes this via a deterministic order of operations:
 * Intent Separation: Identify Recommendation Intent + Safety Intent (CE-SPEC-09).
 * Safety Constraint Evaluation (CE-SPEC-03): The Recommendation Flow receives normalized allergen constraints from CE-SPEC-03 / KB-SPEC-006. It MUST NOT independently expand generic terms such as "nuts" into peanuts and/or tree_nuts, nor interpret allergies itself. Filter all items strictly based on the explicit, validated safety constraints returned by CE-SPEC-03.
 * Hard Constraint Evaluation: Filter remaining items where dietary_tags does not include VEGETARIAN.
 * Hard Constraint Evaluation: Filter remaining items where price > €25.00.
 * Soft Preferences & Ranking: (None specified in this prompt, rank by KB default).
 * Response Generation: Present the surviving candidates. If 0 survive, fail gracefully.
Safety and hard constraints MUST be evaluated before ranking. If constraints mathematically conflict, output NO_CANDIDATES_FOUND.
15. PERSONALIZATION & MEMORY BOUNDARIES
 * Transient Conversation Context: Preferences stated in the current session (e.g., "I like spicy food") are retained for the duration of the active flow to inform subsequent queries.
 * Active-Flow Context: Parameters passed between flows (e.g., Party Size from Booking Flow) remain active until the session terminates.
 * Persistent Preferences: The Assistant MUST NOT infer or store persistent preferences from a single statement unless the integration architecture explicitly authorizes and authenticates a persistent user profile.
 * Health Data Exclusion: The system MUST NOT retain sensitive health or allergen information as "generic personalization memory." Health data obeys strict privacy purging rules per CE-SPEC-03.
16. PRIVACY & DATA MINIMIZATION
 * Minimum Necessary Data: The Assistant collects only the attributes required to filter the menu.
 * Allergy/Health Boundaries: Handled strictly via CE-SPEC-03.
 * Prohibited Storage: The system MUST NOT write dietary restrictions, medical conditions, or user preference profiles into generic, plain-text integration fields (e.g., reservation_notes) unless utilizing an explicitly authorized, structured safety field.
17. ESCALATION LOGIC
The flow MUST transition to ESCALATING (triggering CE-SPEC-07) under the following mandatory conditions:
 * Safety Uncertainty: The guest asks a safety question and the KB cannot provide a deterministic FREE_FROM or CONTAINS response.
 * Conflicting Authoritative Data: The KB returns a CONFLICT state for a requested item.
 * Explicit Staff Request: The guest asks for the chef or a human recommendation.
 * Integration Failure: The KB or menu API times out or returns HTTP errors.
 * Operational Uncertainty: The guest asks for an off-menu item, substitution, or custom preparation that requires human confirmation.
When escalation occurs, the system MUST preserve all active recommendation constraints (e.g., "Guest is looking for a vegan dish under €20") in the escalation payload.
18. FAILURE HANDLING & TIMEOUTS
| Failure Condition | Deterministic Behavior |
|---|---|
| KB / API Timeout | Do not invent candidates. Transition to ESCALATING. |
| Retrieval/Ranking Failure | If ranking fails after mandatory safety and hard-constraint filtering has succeeded, the system MAY return the remaining filtered candidates in deterministic neutral order. Ranking failure MUST NOT bypass any mandatory filter. |
| Repeated Clarification Failure | After 2 failed attempts to narrow vague constraints, offer a generic top-level menu link or escalate. |
| Unavailable Candidates | A candidate may be excluded based on real-time availability only when an authorized integration explicitly returns the candidate as unavailable. |
| Abandoned Flow | Clear transient recommendation constraints. Return to generic listening. |
Rule: Never silently convert a technical failure into a confident, hallucinated recommendation.
19. EDGE CASES
| Edge Case | Deterministic Handling |
|---|---|
| "What's your best dish?" | Extract KB items marked is_signature or popular ONLY if KB-SPEC-005 explicitly defines it as a verified attribute/data source. If none exist, state: "I can recommend something based on what you like. What kind of dish are you looking for?" |
| "Something cheap." | "cheap" does not map to a numerical budget unless the active configuration explicitly defines a price-band mapping. Otherwise, ask for a budget or present lower-priced verified options without claiming they are "cheap." |
| "Can you remove the ingredient?" | Defer to KB-SPEC-005 explicit customization arrays. If unknown, state: "I cannot confirm if that can be modified. I will check with the staff." (Escalate). |
| Multiple Allergies | Route to CE-SPEC-03. The aggregate result MUST be the safest conservative interpretation. |
| Conflicting Preferences | E.g., "Hot ice cream." State the conflict clearly and ask for clarification. |
| No Matching Candidates | Do not invent an approximation. "I don't see anything on the menu that matches all those requests." |
| Recommendation during Booking | Suspend booking. Provide recommendation. Resume booking seamlessly (CE-SPEC-01). |
| User asks why an item was recommended | Respond purely with the KB attributes that matched their request: "I recommended it because it is vegan and under €20." |
20. PRODUCTION ACCEPTANCE CRITERIA
| AC ID | Category | Requirement | Verification Method | Expected Result | Pass/Fail | Severity |
|---|---|---|---|---|---|---|
| AC-01 | Grounding | System never recommends a dish that does not exist in the mock KB-SPEC-005 payload. | Retrieval Audit | 0% hallucinated entities. | Required | Critical |
| AC-02 | Filtering | Hard constraints (e.g., price < 20) strictly eliminate all candidates failing the constraint. | Boolean Filter Test | Invalid items excluded. | Required | Critical |
| AC-03 | Safety Routing | Inclusion of a medical/allergy keyword immediately suspends ranking and triggers CE-SPEC-03. | Intent Pipeline Test | Routes to Safety Flow. | Required | Critical |
| AC-04 | Ranking | Ranking logic does not override or revive candidates eliminated by hard constraints. | Logic Evaluation | Filter applies before rank. | Required | Critical |
| AC-05 | Phrasing Limits | Responses do not contain banned terms ("100% safe", "guaranteed") unless directly returning explicit KB flags. | NLP Output Scan | Banned words blocked. | Required | High |
| AC-06 | Multi-Intent | Intent "Book table and suggest wine" processes both sequentially without losing state. | Multi-Turn Simulation | Both intents fulfilled. | Required | High |
| AC-07 | Escalation | KB timeout safely triggers ESCALATING instead of outputting generic internet knowledge. | Mock API Timeout | Transitions to Escalation. | Required | Critical |
| AC-08 | Context Pres. | Recommending a dish mid-booking returns the guest to the exact required booking step afterward. | State Machine Audit | Parent flow resumed correctly. | Required | High |
21. VERSION HISTORY
| Version | Date | Description | Author | Approval Status |
|---|---|---|---|---|
| 1.1.0 | August 2026 | Enterprise hardening patch: fixed allergen expansion/delegation to CE-SPEC-03, clarified ranking failure safety, refined candidate vs. systemic conflict, clarified availability ownership, adjusted subjective phrasing. | Ramy Bella | Draft / Implementation Specification |
| 1.0.0 | August 2026 | Initial Recommendation Flow specification. Established deterministic pipeline, safety constraint primacy over ranking, non-fabrication response generation, and multi-intent cross-flow preservation. | Ramy Bella | Superseded |
22. FINAL NON-NEGOTIABLE PRINCIPLES
 * NEVER FABRICATE RECOMMENDATION FACTS.
 * AUTHORITATIVE KB DATA IS THE SOURCE OF TRUTH.
 * SAFETY CONSTRAINTS OVERRIDE PREFERENCE RANKING.
 * HARD CONSTRAINTS MUST NEVER BE OVERRIDDEN BY RANKING.
 * UNKNOWN SAFETY INFORMATION MUST NEVER BECOME REASSURANCE.
 * DO NOT INVENT INGREDIENTS, PRICES, AVAILABILITY, OR POLICIES.
 * PRESERVE ACTIVE FLOW CONTEXT.
 * DO NOT CLAIM ACTIONS WERE COMPLETED WITHOUT CONFIRMED INTEGRATION SUCCESS.
 * MINIMIZE SENSITIVE DATA.
 * FAIL CLOSED WHEN DETERMINISTIC SAFE RESOLUTION IS IMPOSSIBLE.
