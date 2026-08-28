CE-SPEC-05: Menu Questions Flow
1. DOCUMENT CONTROL
| Attribute | Value |
|---|---|
| Document Title | 05 Menu Questions Flow.md |
| Document ID | CE-SPEC-05 |
| Version | 1.0.0 |
| Status | 🔒 LOCKED / Approved for Implementation |
| Author | Ramy Bella |
| Classification | Confidential / Enterprise Proprietary |
| Target Audience | AI Engineers, Conversation Designers, Backend Engineers, QA Engineers, Integration Engineers, Platform Architects |
| Parent Document | 01 AI Identity.md |
| Related Documents | CE-SPEC-01, CE-SPEC-02, CE-SPEC-03, CE-SPEC-04, CE-SPEC-07, CE-SPEC-09, CE-SPEC-10, KB-SPEC-005, KB-SPEC-006 |
| System | Restaurant AI System |
| Phase | Phase 3 — Conversation Engine |
| Last Updated | August 2026 |
2. PURPOSE & SCOPE
Purpose
The Menu Questions Flow specifies the deterministic conversational state machine required to answer factual questions about the restaurant's menu. It operationalizes the core principle that the Assistant must retrieve exact, authoritative data from KB-SPEC-005 (Menu & Category Schema) and must never fabricate dishes, ingredients, prices, preparation methods, or operational capabilities.
Scope: What This Document Controls
 * Intent detection and semantic classification for factual menu inquiries.
 * State transitions for querying KB-SPEC-005 and retrieving item attributes.
 * Rules for handling missing, conflicting, or stale menu data.
 * Boundary rules distinguishing factual retrieval from recommendations, allergy handling, and multi-intent orchestration.
 * Real-time availability routing (when explicitly supported by an authorized integration).
Scope: What This Document Explicitly Does NOT Control
 * Allergy and Safety Truth: Governed exclusively by KB-SPEC-006 and managed conversationally by CE-SPEC-03.
 * Recommendations and Ranking: Governed exclusively by CE-SPEC-04.
 * Booking and Cancellation Execution: Governed by CE-SPEC-01 and CE-SPEC-02.
 * Restaurant Policies: Operational rules are governed by KB-SPEC-007.
 * Multi-Intent / Follow-Up Logic: Delegated to CE-SPEC-09 and CE-SPEC-10.
3. RELATIONSHIP TO CORE PRINCIPLES
This flow directly operationalizes the following mandates from the Master AI Identity:
 * Never fabricate facts: The Assistant MUST NOT invent menu items, assume ingredients based on culinary norms, or guess prices.
 * Authoritative Grounding: Responses MUST be explicitly supported by KB-SPEC-005 or an authorized real-time availability integration.
 * Safety Primacy: Any ambiguity between a factual ingredient question and an allergen/medical safety question MUST fail safely and route to CE-SPEC-03.
 * Context Preservation: Answering a menu question mid-booking MUST preserve the active booking state.
 * Fail Closed: If authoritative data is unavailable or conflicting, the Assistant MUST state its inability to answer and escalate to human staff.
4. CONVERSATIONAL STATE MODEL
The Menu Questions Flow operates as a deterministic state machine to ensure safe and structured retrieval of menu facts.
| State | Entry Condition | Required Action | Transitions |
|---|---|---|---|
| DETECTED | NLP flags a question related to food, drink, or menu attributes. | Pass utterance to Semantic Classifier. | ANALYZING_INTENT |
| ANALYZING_INTENT | Intent requires categorization. | Classify the question type (A-P) and determine boundary ownership. | CLASSIFYING_QUESTION, ROUTING_TO_SAFETY (CE-03), ROUTING_TO_RECOMMENDATION (CE-04), ROUTING_TO_MULTI_INTENT (CE-09) |
| CLASSIFYING_QUESTION | Query confirmed as a factual menu question. | Map query to specific KB-SPEC-005 fields (e.g., price, ingredients). | COLLECTING_CONTEXT |
| COLLECTING_CONTEXT | Target item or category is ambiguous. | Prompt guest for minimum necessary clarification or resolve via CE-SPEC-10 (coreference). | RETRIEVING_MENU_DATA, ESCALATING, TERMINATED |
| RETRIEVING_MENU_DATA | Unambiguous target identified. | Query KB-SPEC-005 or authorized Integrations API. Suspend conversation. | VALIDATING_MENU_DATA, ESCALATING (on error/timeout) |
| VALIDATING_MENU_DATA | KB response received. | Check state (VERIFIED_INFORMATION_AVAILABLE, etc.). | GENERATING_RESPONSE, ESCALATING |
| GENERATING_RESPONSE | Verified data available. | Formulate deterministic response strictly bound to KB facts. | PRESENTING_INFORMATION |
| PRESENTING_INFORMATION | Response formulated. | Deliver factual response to the guest. | RESUMING_PARENT_FLOW, TERMINATED |
| RESUMING_PARENT_FLOW | Menu question answered during an active parent flow. | Restore suspended parent context (e.g., CE-SPEC-01). | N/A |
| TERMINATED | Standalone menu query completes or guest aborts. | Clear transient context. Return to generic listening. | N/A |
5. MENU QUESTION INTENT DETECTION
The system MUST use semantic classification to distinguish factual menu inquiries from adjacent intents.
 * Factual Menu Query (CE-SPEC-05): "Do you serve steak?", "How much is the ribeye?", "What is in the Caesar salad?"
 * Recommendation Request (CE-SPEC-04): "What do you recommend?", "What's your best dessert?", "What should I order?" \rightarrow Mandatory route to CE-SPEC-04.
 * Safety/Allergy Question (CE-SPEC-03): "Does the pasta contain nuts?", "Is this safe for my dairy allergy?" \rightarrow Mandatory route to CE-SPEC-03.
 * Multi-Intent Request (CE-SPEC-09): "Book a table for 4 and tell me what desserts you have." \rightarrow Mandatory route to CE-SPEC-09 for orchestration.
6. QUESTION TYPE CLASSIFICATION
Factual menu queries are deterministically categorized to map to specific KB-SPEC-005 fields.
| Category | Semantic Meaning | Authoritative Source | Allowed Action / Routing |
|---|---|---|---|
| A. Menu existence | "Do you have steak?" | KB-SPEC-005 Items | Boolean search across item names/tags. |
| B. Menu category | "What desserts do you have?" | KB-SPEC-005 Categories | Return list of active items in category. |
| C. Ingredient/content | "What's in the burger?" | KB-SPEC-005 Ingredients | Return documented ingredients. (If allergy intent \rightarrow CE-SPEC-03). |
| D. Price | "How much is the ribeye?" | KB-SPEC-005 Pricing | Return exact price and currency. |
| E. Portion/size | "How big is the pizza?" | KB-SPEC-005 Sizes | Return defined sizes/portions. |
| F. Included components | "What comes with the steak?" | KB-SPEC-005 Inclusions | Return linked sides/components. |
| G. Prep/cooking method | "Is the fish fried?" | KB-SPEC-005 Preparation | Return documented prep method. |
| H. Customization | "Can I get dressing on side?" | KB-SPEC-005 Modifications | Check supported_customizations. Escalate if absent. |
| I. Dietary attribute | "Do you have vegan options?" | KB-SPEC-005 Dietary Tags | Filter and return matching items. |
| J. Availability | "Do you have the special?" | Integration Contract | Query authorized inventory integration. |
| K. Menu structure | "What's in the set menu?" | KB-SPEC-005 Composites | Return hierarchical item structure. |
| L. Recommendation | "What's your best dish?" | N/A | Route to CE-SPEC-04. |
| M. Allergy/safety | "Does this have peanuts?" | KB-SPEC-006 | Route to CE-SPEC-03. |
| N. Operational/policy | "Can I bring my own cake?" | KB-SPEC-007 | Route to KB-SPEC-007 policy handling. |
| O. Ambiguous question | "What about the other one?" | N/A | Route to CE-SPEC-10 to resolve coreference. |
| P. Multi-intent | "How much is it & is it vegan?" | N/A | Route to CE-SPEC-09. |
7. AUTHORITATIVE DATA VALIDATION
All responses MUST be validated against the deterministic state returned by the Knowledge Base or authorized integration.
| KB State | Conversational Action |
|---|---|
| VERIFIED_INFORMATION_AVAILABLE | Proceed to GENERATING_RESPONSE. Formulate exact factual statement. |
| INSUFFICIENT_INFORMATION | Transition to ESCALATING. "I don't have verified information about that aspect of the menu. Let me connect you with staff." |
| CONFLICTING_INFORMATION | Transition to ESCALATING. "I'm seeing conflicting information about that item. Let me connect you with staff." |
| STALE_OR_UNVERIFIED_INFORMATION | Transition to ESCALATING. Do not present unverified data to the guest. |
| HUMAN_CONFIRMATION_REQUIRED | Transition to ESCALATING. "That requires confirmation from the kitchen staff. Connecting you now." |
8. MENU FACT RETRIEVAL
 * The Assistant MUST query the exact parameters mapped in KB-SPEC-005 (e.g., price_minor_units, currency, ingredients_array, dietary_tags).
 * The Assistant MUST NOT execute broad web searches, query external culinary databases, or use base LLM knowledge to supplement missing menu data.
9. RESPONSE GENERATION
Responses MUST be grounded strictly in the returned KB fields. The Assistant MUST NOT extrapolate or embellish.
Approved Phrasing Examples
 * Price verified: "The Ribeye is {{PRICE}} {{CURRENCY}}."
 * Ingredients verified: "The menu lists the ingredients as: {{INGREDIENTS_LIST}}."
 * Preparation verified: "The Salmon is listed as {{PREPARATION_METHOD}}."
Banned Phrasing
The Assistant MUST NEVER use the following speculative qualifiers:
 * "I think..."
 * "Usually..."
 * "Probably..."
 * "It should..."
 * "I believe..."
 * "Most restaurants..."
 * "That dish normally comes with..."
If the fact is verified, state it as a fact. If the fact is unverified or missing, state that it cannot be verified and offer to escalate.
10. UNKNOWN / MISSING / CONFLICTING DATA
The Assistant MUST fail closed when data is missing or conflicting.
 * Missing Item: "I don't see {{REQUESTED_ITEM}} on the current menu. Would you like me to list our available {{CATEGORY}}?"
 * Missing Attribute (e.g., unknown prep method): "I don't have the specific preparation method listed for the {{DISH_NAME}}. Would you like me to connect you with staff to confirm?"
 * Conflicting Data: "I'm receiving conflicting information regarding that dish's details. Let me connect you with a staff member to be absolutely sure."
11. PRICE QUESTIONS
Pricing must be deterministic and transparent.
 * No Currency Inference: The system MUST state the currency exactly as defined in KB-SPEC-003/KB-SPEC-005. It MUST NEVER assume "dollars" or "euros" if the symbol is missing, nor convert currencies on the fly.
 * No Invented Economics: The Assistant MUST NEVER invent taxes, service charges, fees, or dynamic discounts.
 * Missing Price: If a price field is missing or unverified, the Assistant MUST NOT fabricate. Response: "The price for that item is not currently listed. Let me connect you with staff to verify."
12. INGREDIENT QUESTIONS
The system MUST distinguish between a factual inquiry about recipe components and a medical safety inquiry.
 * Semantic Classifier Rule:
   * "What ingredients are listed for the burger?" \rightarrow Purely factual list query. CE-SPEC-05 queries KB-SPEC-005 and returns the ingredients_array.
   * "Does the burger contain peanuts?" \rightarrow Direct allergen/safety check. Triggers CE-SPEC-03 / KB-SPEC-006.
 * Ambiguity: If the system is uncertain whether a question is factual or safety-related (e.g., "Is there dairy in this?"), the system MUST default to the fail-safe option and route to CE-SPEC-03 (Allergy Flow). The architecture must prioritize safety whenever there is genuine safety ambiguity.
13. PORTION / SIZE / INCLUSION QUESTIONS
 * Sizes: If KB-SPEC-005 defines distinct sizes (e.g., Small, Large) with associated price points, the Assistant MUST list the available options. The Assistant MUST NOT invent dimensions (e.g., "The large is 16 inches") unless explicitly defined in the KB.
 * Inclusions: If asked "Is the sauce included?", the Assistant MUST check the explicit inclusions array in the KB. It MUST NOT assume inclusion based on standard restaurant norms.
14. PREPARATION / COOKING METHOD QUESTIONS
 * If KB-SPEC-005 defines a preparation method (e.g., "Wood-fired", "Pan-seared"), state it.
 * If undefined, the Assistant MUST NOT guess. Response: "The exact cooking method isn't listed in my menu data. Let me check with the kitchen for you." (Escalate).
15. CUSTOMIZATION QUESTIONS
Customization requests (e.g., "Can I order this dish without the sauce?") must distinguish documented customization from staff-dependent modification.
 * Explicit Support: If KB-SPEC-005 explicitly lists the requested modification in the supported customizations array, the Assistant affirms it.
 * Unsupported/Unknown: If the modification is not explicitly supported by the KB, the Assistant MUST NOT promise that the kitchen can or will accommodate the request.
 * Fallback Phrasing: "I can't confirm that modification from the menu data. I'll need to check with the staff." (Transition to ESCALATING).
16. AVAILABILITY QUESTIONS
CE-SPEC-05 strictly distinguishes between an item existing in the static KB and being currently available for ordering.
 * Static Menu: A static KB entry does NOT automatically mean the item is currently available.
 * Integration Authority: The Assistant MUST state real-time availability ONLY when an authorized real-time POS/Inventory integration explicitly provides current availability status.
 * Missing Integration: If no real-time integration exists or if the status is unknown, the Assistant MUST state: "The {{DISH_NAME}} is on our menu, but I cannot confirm real-time kitchen availability. Would you like me to connect you with staff to check?"
17. DIETARY VS ALLERGY BOUNDARY
 * Dietary Tag (KB-SPEC-005): "Do you have vegetarian options?" The Assistant queries KB-SPEC-005 for items containing the VEGETARIAN tag. (Factual menu query; if it becomes a recommendation request, route to CE-SPEC-04).
 * Allergy/Safety (KB-SPEC-006): "Is this safe for my nut allergy?" The Assistant MUST route to CE-SPEC-03.
 * Cross-Contamination: The Assistant MUST NEVER use a dietary tag (like VEGAN) to guarantee the absence of cross-contact with allergens (like milk).
18. RECOMMENDATION VS MENU-FACT BOUNDARY
The system maintains a strict architectural boundary between factual retrieval and subjective ranking.
 * CE-SPEC-05 Ownership (Factual Retrieval):
   * "Do you have steak?"
   * "How much is the ribeye?"
   * "Which steak costs the least?" \rightarrow Comparative Factual Retrieval. CE-SPEC-05 performs a deterministic numerical sort of verified prices.
 * CE-SPEC-04 Ownership (Recommendation/Ranking):
   * "What steak do you recommend?"
   * "What should I order?"
   * "Which steak is the best?"
 * KB-Backed Attributes: If the guest asks "Which dish is marked as popular?", CE-SPEC-05 may retrieve items explicitly tagged with a popular boolean in KB-SPEC-005. However, asking the Assistant to subjectively choose the most popular item routes to CE-SPEC-04.
19. MULTI-INTENT & CROSS-FLOW ROUTING
 * Menu Question mid-Booking (CE-SPEC-01): Guest asks "Do you have steak?" while selecting a time. The system suspends the booking flow, retrieves the menu fact via CE-SPEC-05, delivers the answer, and resumes CE-SPEC-01 without losing booking parameters.
 * Menu Question mid-Cancellation (CE-SPEC-02): Guest asks a menu question before confirming cancellation. Suspend, answer factually, and resume without losing cancellation context.
 * Multi-Intent (CE-SPEC-09): "How much is the steak and what do you recommend with it?" \rightarrow Decomposed by CE-SPEC-09. CE-SPEC-05 handles the factual price retrieval; CE-SPEC-04 handles the recommendation.
 * Safety Priority: "Do you have anything vegetarian and is it safe for my nut allergy?" Safety intent takes absolute priority. CE-SPEC-03 owns the safety evaluation.
20. CONTEXT PRESERVATION
CE-SPEC-10 (Follow Up Logic) provides coreference resolution for CE-SPEC-05.
 * Example: Guest asks "How much is the ribeye?" Assistant replies "€34." Guest asks "What sides come with it?"
 * Action: CE-SPEC-10 resolves "it" to the Ribeye ID. CE-SPEC-05 executes the query for inclusions using the preserved item context. The Assistant MUST NOT force the guest to repeat the item name that is still valid.
21. PRIVACY & DATA MINIMIZATION
 * The Assistant MUST NOT log dietary preferences or menu inquiries to persistent guest profiles unless explicitly authorized by a recognized consent/profile integration.
 * Menu queries must remain transient conversation context and should not trigger generic user-tracking updates.
22. ESCALATION LOGIC
Mandatory transition to ESCALATING (triggering CE-SPEC-07) occurs under the following conditions:
 * KB-SPEC-005 or Integrations API timeout.
 * Conflicting authoritative menu data returned by the KB.
 * Missing critical information (e.g., price is unknown but requested).
 * Unsupported or staff-only customization requests.
 * Operational questions outside CE-SPEC-05 (e.g., policies).
 * Explicit guest request for a human.
 * Unresolved conversational ambiguity after defined clarification attempts.
Rule: The Assistant MUST NOT over-escalate simple factual questions if authoritative data exists and matches the query.
23. FAILURE HANDLING & TIMEOUTS
| Failure Condition | Deterministic Behavior | Guest-Facing Fallback Phrasing |
|---|---|---|
| KB Timeout | Transition to ESCALATING. | "I'm having trouble accessing the menu system right now. Let me connect you with staff." |
| Integration Failure (Availability) | Handle gracefully without fabricating status. | "I can see that item on the menu, but I can't currently verify if it's in stock right now. Let me connect you to check." |
| Data Not Found | Return negative response neutrally. | "I do not see that item listed on our current menu." |
| Abandoned Flow | Clear transient menu context after {{TIMEOUT_MINUTES}}. | No proactive message. Silently reset. |
24. EDGE CASES
| Edge Case | Deterministic Handling |
|---|---|
| "Do you have Coke?" | Query KB for exact brand matches. If the KB lists "Pepsi" but no "Coke", the Assistant must not say "Yes" and must state the documented item: "We do not carry Coke, but our menu lists Pepsi." |
| Off-menu requests | If asked for a dish not in the KB (e.g., "Can you make a custom omelet?"), escalate to staff. Do not promise off-menu flexibility. |
| "What's in the dessert?" | Require clarification if multiple desserts exist, or list the categories. |
| Vague Modifier | "Does the burger come with stuff?" Check included_components. If populated, list them. If empty, state: "The menu does not list any included sides for the burger." |
| "Is it big?" | Do not invent subjective sizing. If weight/size is listed (e.g., "250g"), return the fact. Otherwise, state: "The menu does not specify the portion size. I can connect you with staff to confirm." |
25. PRODUCTION ACCEPTANCE CRITERIA
| AC ID | Category | Requirement | Verification Method | Expected Result | Pass/Fail | Severity |
|---|---|---|---|---|---|---|
| AC-01 | Grounding | Responses contain only items, prices, and ingredients exactly matching the KB-SPEC-005 payload. | Response Validation | 0% hallucinated menu facts. | Required | Critical |
| AC-02 | No Hallucinated Items | Queries for non-existent items deterministically return a "not found" response rather than an approximation. | Negative Query Test | Correct negative response. | Required | Critical |
| AC-03 | Price Accuracy | Price queries return the exact price value and canonical currency from the authoritative payload without modification or currency conversion | Numeric Boundary Test | Exact string match. | Required | Critical |
| AC-04 | Ingredient Accuracy | Factual ingredient queries strictly recite the ingredients_array. | Attribute Test | Exact array match. | Required | High |
| AC-05 | Recommendation Routing | Subjective queries ("What's best?") bypass CE-SPEC-05 and route directly to CE-SPEC-04. | Intent Classification | Routes to CE-SPEC-04. | Required | High |
| AC-06 | Allergy Routing | Queries combining ingredients and safety/medical terms trigger immediate routing to CE-SPEC-03. | Semantic Safety Test | Routes to CE-SPEC-03. | Required | Critical |
| AC-07 | Availability Boundary | Real-time availability is only stated when an authorized integration payload explicitly confirms it. | Integration Mock Test | Safe fallback if no API exists. | Required | Critical |
| AC-08 | Customization Boundary | Unsupported custom modification requests are refused/escalated rather than promised. | Edge Case Test | Transitions to ESCALATING. | Required | High |
| AC-09 | Multi-Intent Routing | Booking parameters + Menu query are orchestrated without dropping either intent. | Multi-Intent Simulation | Both intents executed. | Required | High |
| AC-10 | Context Preservation | Interjecting a menu question into an active Booking Flow preserves all booking parameters upon resumption. | State Machine Audit | Parameters preserved. | Required | Critical |
| AC-11 | KB Timeout | A backend timeout safely escalates without generating LLM-invented culinary knowledge. | Timeout Simulation | Transitions to ESCALATING. | Required | Critical |
| AC-12 | Conflicting Data | KB returning a CONFLICT state for an item forces an immediate escalation. | Data Integrity Test | Transitions to ESCALATING. | Required | Critical |
| AC-13 | Unknown Data | Querying a valid item's missing attribute (e.g., prep method) yields a safe fallback response. | Missing Data Test | Safe fallback phrasing. | Required | High |
| AC-14 | Follow-Up Reference | "How much is it?" accurately resolves to the item discussed in the previous conversational turn. | Coreference Test | Correct price returned. | Required | High |
| AC-15 | No Assumptions | System blocks banned phrasing ("usually", "probably", "I think") from all menu responses. | Output Scan | 100% block rate. | Required | Critical |
| AC-16 | Escalation Correctness | Unresolved ambiguity after defined clarification attempts successfully triggers human handoff. | Clarification Loop Test | Transitions to ESCALATING. | Required | High |
26. VERSION HISTORY
| Version | Date | Description | Author | Approval Status |
|---|---|---|---|---|
| 1.0.0 | August 2026 | Initial Menu Questions Flow specification. Established semantic intent classification, strict KB grounding, fail-closed data handling, cross-flow context preservation, and explicit boundaries against recommendation and allergy flows. | Ramy Bella | 🔒 LOCKED / Approved for Implementation |
27. FINAL NON-NEGOTIABLE PRINCIPLES
 * NEVER INVENT MENU FACTS: Every item, ingredient, price, and attribute must be deterministically retrieved from KB-SPEC-005.
 * SAFETY QUESTIONS TRIGGER CE-SPEC-03: Any ambiguity between a factual ingredient question and a medical/allergen question must fail safely and route to the Allergy Flow.
 * RECOMMENDATIONS TRIGGER CE-SPEC-04: Subjective ranking and "best" queries bypass factual menu retrieval and route to the Recommendation Flow.
 * NO ASSUMPTIONS ON MISSING DATA: If KB-SPEC-005 lacks data for an attribute, the Assistant must state the information is unverified and escalate.
 * REAL-TIME AVAILABILITY REQUIRES INTEGRATION: A static KB entry does not equal real-time kitchen availability.
 * NO FABRICATED CUSTOMIZATIONS: Only explicit, KB-supported modifications may be affirmed.
 * PRESERVE ACTIVE FLOW CONTEXT: Menu questions must seamlessly suspend and resume parent flows (like bookings) without dropping guest data.
 * BANNED SPECULATION: The Assistant must never use "probably," "usually," or "I think" when answering menu questions.
 * ESCALATE ON CONFLICT: Conflicting or unreachable KB data mandates immediate human handoff.
