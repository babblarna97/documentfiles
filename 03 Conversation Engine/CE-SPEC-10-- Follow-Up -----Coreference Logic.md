CE-SPEC-10: Follow-Up / Coreference Logic

1. DOCUMENT CONTROL

| Attribute | Value |
|---|---|
| Document Title | 10 Follow-Up / Coreference Logic.md |
| Document ID | CE-SPEC-10 |
| Version | 1.0.1 |
| Status | 🔒 LOCKED / Approved for Implementation |
| Author | Ramy Bella |
| Classification | Confidential / Enterprise Proprietary |
| Target Audience | AI Engineers, Conversation Designers, Backend Engineers, QA Engineers, Integration Engineers, Security/Privacy Engineers, Platform Architects |
| Parent Document | 01 AI Identity.md |
| Related Documents | CE-SPEC-01 through CE-SPEC-09, CE-SPEC-12, KB-SPEC-004, KB-SPEC-005, KB-SPEC-007 |
| System | Restaurant AI System |
| Phase | Phase 3 — Conversation Engine |
| Last Updated | August 2026 |

2. PURPOSE & SCOPE

Purpose

The Follow-Up / Coreference Logic (CE-SPEC-10) specifies the deterministic architecture for resolving conversational references such as "it", "that one", "tomorrow", "the cheaper one", "cancel that", or "book it" into explicitly valid entities, intent records, slot values, temporal values, or other authorized conversational context.

Its purpose is to preserve continuity across turns without forcing guests to repeat information unnecessarily while enforcing the core architectural principle:

FOLLOW-UP MUST RESOLVE CONTEXT, NOT INVENT CONTEXT.

CE-SPEC-10 is a reference-resolution and context-binding layer. It does not execute business operations.

Scope: What This Document Controls

* Detection and classification of referential markers in guest utterances.
* Resolution of pronouns, demonstratives, definite references, comparative references, ordinal references, selections, action continuations, and temporal references.
* Deterministic context candidate generation.
* Context authority hierarchy.
* Entity, intent, slot, and temporal resolution.
* Ambiguity handling and clarification limits.
* Context invalidation, expiration, and authorization boundaries.
* Coordination with CE-SPEC-09 for multi-intent messages.
* Handoff of normalized entity_id, intent_id, slot values, or other minimum necessary resolved context to the authoritative owning flow.
* Duplicate-action prevention through intent/entity continuity.

Scope: What This Document Explicitly Does NOT Control

* Business Logic Execution: CE-SPEC-10 does not execute bookings, cancellations, recommendations, menu queries, or allergy evaluations.
* Multi-Intent Orchestration: Owned by CE-SPEC-09.
* Human Escalation Orchestration: Owned by CE-SPEC-07.
* Emergency Handling: Owned by CE-SPEC-12.
* Allergy/Safety Truth: Owned by CE-SPEC-03 and KB-SPEC-006.
* Menu Truth: Owned by CE-SPEC-05 and KB-SPEC-005.
* Booking/Cancellation Execution: Owned by CE-SPEC-01 and CE-SPEC-02.

3. RELATIONSHIP TO CORE PRINCIPLES

CE-SPEC-10 strictly operationalizes the foundational principles from 01 AI Identity.md:

* FOLLOW-UP MUST RESOLVE CONTEXT, NOT INVENT CONTEXT: The system MUST only map a reference to explicitly valid, authorized context.
* ZERO FABRICATION: The system MUST NOT guess between multiple plausible candidates.
* DETERMINISTIC RESOLUTION: Resolution occurs only when the candidate set and target slot are sufficiently constrained by authoritative context.
* STRICT OWNERSHIP BOUNDARY: CE-SPEC-10 resolves references and returns normalized context; the authoritative owning specification executes the business operation.
* MINIMUM NECESSARY PROPAGATION: Reference resolution MUST NOT hydrate downstream payloads with unrelated conversational data.
* TEMPORAL FAIL-CLOSED: Temporal references require a valid venue timezone and deterministic configuration.
* STATE CONTINUITY: Existing active intent/entity records MUST be reused where the guest is continuing an existing action.
* SECURITY & TENANT ISOLATION: A follow-up reference MUST never resolve to another guest's data, another venue, private staff context, hidden system state, or unauthorized information.

4. FOLLOW-UP / COREFERENCE MODEL

The coreference architecture treats conversational context as a structured, authorization-scoped, TTL-bound collection of valid entities, intent records, slot values, and temporal anchors.

4.1 Context Registration

When an authorized Conversation Engine specification processes an entity or intent, it MAY register the minimum necessary reference metadata into the Active Context Store.

Examples:

* Menu entity → entity_id
* Booking intent → intent_id
* Active booking → booking_id
* Requested date → normalized date value
* Requested time → normalized time value
* Allergen → normalized allergen identifier where authorized

The system MUST NOT store arbitrary conversation content merely for the purpose of future coreference.

4.2 Reference Detection

The NLP layer identifies incomplete references such as:

* "it"
* "that"
* "the other one"
* "the first one"
* "the cheaper one"
* "tomorrow"
* "same time"
* "cancel that"
* "book it"

4.3 Candidate Generation

Candidate generation MUST use only valid context available to the current authenticated guest session and venue scope.

The system MUST NOT expand the candidate set using:

* unrelated historical conversations,
* another guest's data,
* another venue's data,
* hidden system context,
* external search,
* unsupported LLM assumptions.

4.4 Resolution

If exactly one valid candidate and one valid target slot are deterministically established, CE-SPEC-10 resolves the reference.

If multiple valid candidates or target slots remain, CE-SPEC-10 MUST transition to AMBIGUOUS.

If no valid candidate exists, CE-SPEC-10 MUST transition to CONTEXT_MISSING or CONTEXT_EXPIRED, depending on the cause.

5. REFERENCE CLASSIFICATION

The system MUST deterministically classify follow-up references into the following categories.

| Reference Type           | Semantic Meaning / Example               | Resolution Strategy                                                                                                           |
| ------------------------ | ---------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------- |
| Pronoun                  | "it", "that", "this", "they"             | Resolve against authorized candidate entities in context.                                                                     |
| Definite / Demonstrative | "the steak", "that booking"              | Match against the relevant entity type in valid context.                                                                      |
| Ordinal / Selection      | "the first one", "the second option"     | Use the structured option array explicitly presented in the applicable context.                                               |
| Comparative              | "the cheaper one", "the earlier time"    | Apply deterministic comparison to the explicitly bounded candidate set.                                                       |
| Temporal                 | "tomorrow", "same time"                  | Resolve using the venue's canonical timezone and active temporal context.                                                     |
| Attribute Follow-Up      | "how much is it?", "what comes with it?" | Resolve entity, then hand off the requested attribute to the owning flow.                                                     |
| Action Continuation      | "book it", "cancel that", "change it"    | Resolve to an existing entity or intent where possible, then hand off to the authoritative owner.                             |
| Slot Modification        | "change it to 8"                         | Resolve the target entity AND determine the target slot against the active flow schema; ambiguity MUST trigger clarification. |

6. CONTEXT SOURCES AND AUTHORITY

CE-SPEC-10 MUST evaluate context in the following deterministic hierarchy.

6.1 Authority Hierarchy

1. Current User Utterance
   * Explicit parameters stated in the current message.
   * Example: "Book the Ribeye and make it rare."

2. Immediately Previous Turn (n-1)
   * Entities or structured options explicitly presented or requested in the preceding conversational exchange.

3. Active Flow Context
   * Parameters and intent records held by the currently active owner, such as CE-SPEC-01 booking context.

4. Recent Valid Context (n-2 through configured historical depth)
   * Structured recent context that remains within configured TTL and authorization scope.

5. No Valid Context
   * If no valid candidate survives, transition to CONTEXT_MISSING.

6.2 Context Validity Requirements

A candidate is valid only when:

* It belongs to the current authenticated guest/session.
* It belongs to the current venue_id.
* It has not expired.
* Its source flow has not invalidated it.
* It remains semantically compatible with the requested reference.
* The target slot is still valid.
* Any required authorization remains active.

7. REFERENCE VS. DATA PROPAGATION

CE-SPEC-10 MUST strictly separate reference resolution from payload hydration.

7.1 Canonical Resolution Output

Where possible, CE-SPEC-10 SHOULD return identifiers rather than fully hydrated objects:

{
  "resolved_entity_id": "menu-item-123",
  "resolved_intent_id": null,
  "resolved_slot_updates": {},
  "resolution_type": "ENTITY",
  "resolution_state": "RESOLVED",
  "source_context": "N-1"
}

7.2 Minimum Necessary Propagation

Downstream flows MUST receive only the fields required to execute their own logic.

Example:

For:

"Does it contain peanuts?"

CE-SPEC-10 SHOULD return:

{
  "resolved_entity_id": "menu-item-123",
  "requested_allergen": "PEANUTS"
}

It MUST NOT attach:

* unrelated booking details,
* unrelated guest profile data,
* payment details,
* unrelated allergy history,
* private contact information,
* conversation transcript,
* another guest's context.

7.3 Hydration Boundary

The authoritative owner MAY retrieve additional authorized entity fields using the resolved identifier.

CE-SPEC-10 MUST NOT proactively hydrate unrelated payload data merely because it exists in context.

8. CE-SPEC-09 ↔ CE-SPEC-10 COORDINATION CONTRACT

CE-SPEC-10 and CE-SPEC-09 operate through a deterministic interleaved resolution model.

Neither component may guess.

Phase 1 — Global Reference Resolution

CE-SPEC-10 MAY resolve references that are deterministically identifiable from global/session-level context before intent decomposition.

Examples:

"Cancel that."

→ CE-SPEC-10 resolves the active booking to booking_id=123.

"Book it."

→ CE-SPEC-10 resolves the existing pending booking intent if exactly one valid intent exists.

Global resolution MUST NOT be forced when the reference depends on clause boundaries or multi-intent structure.

Phase 2 — Intent Decomposition

CE-SPEC-09 decomposes the utterance into discrete intent units and establishes:

* intent boundaries,
* intent ownership,
* dependencies,
* candidate scope,
* execution mode.

Phase 3 — Intent-Bound Reference Resolution

References whose interpretation depends on an isolated intent unit MUST be resolved by CE-SPEC-10 inside that unit.

Example:

"Book the cheaper one and ask whether it contains peanuts."

CE-SPEC-09 may first establish:

* Intent A: Booking
* Intent B: Allergy/Safety inquiry

CE-SPEC-10 then resolves "the cheaper one" using only the candidate set relevant to the booking intent.

Phase 4 — Failure Coordination

If a reference remains ambiguous:

* The affected intent unit transitions to PENDING_CLARIFICATION or equivalent owner state.
* Independent intent units MAY continue when CE-SPEC-09 determines that they are safely independent.
* CE-SPEC-10 MUST NOT force a global failure when only one intent unit is ambiguous.

Fundamental Contract:

Resolve as early as deterministically possible, but never resolve earlier than the information required for deterministic resolution.

9. ENTITY / SLOT RESOLUTION

When a reference requires mapping to a specific entity, intent, or parameter:

9.1 Single Valid Candidate

If exactly one valid candidate and one target interpretation exist:

* Resolve the reference.
* Return the minimum necessary identifier/value.
* Hand off to the authoritative owner.

9.2 Multiple Entity Candidates

If more than one valid entity remains:

* Transition to AMBIGUOUS.
* Ask a minimal clarification question.
* Do not select the "most recent" candidate merely because it is most recent unless the architecture explicitly defines that rule for the scenario.

Example:

Assistant: "We have Ribeye and Filet."

Guest: "How much is it?"

Correct response:

"Do you mean the Ribeye or the Filet?"

9.3 Slot Resolution

Entity resolution and slot resolution are separate operations.

For:

"Can I change it to 8?"

The system MUST:

1. Resolve what "it" refers to.
2. Inspect the active flow schema.
3. Determine which slot 8 could legally modify.
4. Verify that only one interpretation survives.

Examples:

* Active booking has confirmed time 20:00 and missing party_size → 8 MAY resolve to party_size=8.
* Active booking has confirmed party size and missing time, and the venue accepts numeric hour input → 8 MAY resolve to a time only if the time semantics are unambiguous and authorized.
* Both party size and time remain valid interpretations → AMBIGUOUS.

The system MUST NOT return an OR-condition such as:

party_size=8 OR time=20:00

as a resolved business value.

10. TEMPORAL / DATE / TIME FOLLOW-UPS

Temporal coreference MUST be deterministic and venue-scoped.

10.1 Canonical Timezone

"Today", "tomorrow", "tonight", and relative temporal calculations MUST be resolved against the venue's configured canonical IANA timezone.

The system MUST NOT silently fall back to:

* UTC,
* server timezone,
* user device timezone,
* model-local time.

If the venue timezone is missing, null, invalid, or unavailable:

* Temporal resolution MUST fail closed.
* Transition to CONTEXT_MISSING or the owning flow's configuration/escalation state.
* Do not fabricate the local date.

10.2 Relative Temporal References

Examples:

* "tomorrow"
* "same time"
* "earlier"
* "later"

Resolution rules:

1. Use an explicitly configured venue rule where one exists.
2. Use valid active-flow temporal context where applicable.
3. Ask for clarification if the semantic offset is not deterministically defined.

The system MUST NOT invent an arbitrary offset.

Example:

"Can we do later?"

If no explicit configuration defines "later":

"How much later would you like?"

10.3 Temporal Validation

Resolved temporal values MUST be passed to the relevant authoritative owner for validation.

Examples:

* Booking time → CE-SPEC-01.
* Opening-hours-related temporal fact → KB-SPEC-004 / owning flow.

CE-SPEC-10 does not independently authorize the booking or claim availability.

11. COMPARATIVE / ORDINAL / SELECTION REFERENCES

Comparative or selection references require a deterministic candidate set.

Examples:

* "the cheaper one"
* "the first one"
* "the second option"
* "the earlier time"

Requirements

The candidate set MUST be:

* explicitly known,
* authorized,
* semantically relevant,
* current,
* structurally comparable where applicable.

Comparative Resolution

"The cheaper one" requires:

* valid price fields,
* a defined candidate array,
* comparable currency/value semantics.

If prices are:

* missing,
* stale,
* conflicting,
* equal where a unique selection is required,
* or otherwise incomparable,

CE-SPEC-10 MUST NOT guess.

It MUST transition to AMBIGUOUS or CONTEXT_MISSING as appropriate.

External Expansion Prohibition

CE-SPEC-10 MUST NOT silently expand the candidate set through:

* web search,
* general LLM knowledge,
* unrelated KB records,
* external culinary knowledge.

12. FOLLOW-UP / COREFERENCE STATE MACHINE

The flow operates as a deterministic preprocessing and resolution layer.

| State | Entry Condition | Required Action | Transitions |
|---|---|---|---|
| IDLE | Awaiting input. | None. | RECEIVING_INPUT |
| RECEIVING_INPUT | Guest message arrives. | Detect whether reference resolution is required. | ANALYZING_REFERENCE, or bypass to owning/orchestration flow |
| ANALYZING_REFERENCE | Reference detected. | Classify reference type. | RESOLVING_ENTITY, RESOLVING_TEMPORAL, RESOLVING_COMPARATIVE |
| RESOLVING_ENTITY | Entity, pronoun, action, or slot target detected. | Query authorized context hierarchy. | RESOLVED, AMBIGUOUS, CONTEXT_MISSING, CONTEXT_EXPIRED |
| RESOLVING_TEMPORAL | Temporal reference detected. | Resolve against canonical venue timezone and authorized temporal context. | RESOLVED, AMBIGUOUS, CONTEXT_MISSING |
| RESOLVING_COMPARATIVE | Comparative/ordinal reference detected. | Evaluate against explicitly bounded candidate set. | RESOLVED, AMBIGUOUS, CONTEXT_MISSING |
| RESOLVED | Exactly one valid interpretation exists. | Return normalized ID/value/slot update and hand off. | TERMINATED |
| AMBIGUOUS | Multiple valid interpretations remain. | Ask minimal clarification. | RECEIVING_INPUT, ESCALATING after threshold |
| CONTEXT_MISSING | No valid candidate or required configuration missing. | Request minimum necessary clarification or invoke owning-flow fallback. | RECEIVING_INPUT, ESCALATING |
| CONTEXT_EXPIRED | Candidate exists but is invalid due to TTL/state/token expiration. | Inform guest that context is no longer active. | RECEIVING_INPUT, ESCALATING if required |
| ESCALATING | Deterministic resolution fails beyond allowed threshold. | Delegate handoff transport to CE-SPEC-07. | TERMINATED |
| TERMINATED | CE-SPEC-10 yields control. | Return normalized context to CE-SPEC-09 or authoritative owner. | IDLE |

13. MULTI-INTENT INTERACTION WITH CE-SPEC-09

CE-SPEC-09 owns multi-intent orchestration.

CE-SPEC-10 resolves references at the earliest deterministic stage but MUST NOT force premature resolution.

Example A — Globally Resolvable

Guest:

"Cancel that."

Exactly one active reservation exists.

Result:

CE-SPEC-10 → resolved_booking_id
CE-SPEC-02 → cancellation execution

Example B — Intent-Dependent

Guest:

"Book the cheaper one and ask whether it contains peanuts."

Correct architecture:

Raw utterance
      ↓
CE-SPEC-10
      ↓
Global references only
      ↓
CE-SPEC-09
      ↓
Intent A: Booking
Intent B: Allergy/Safety
      ↓
CE-SPEC-10
      ↓
Intent-local reference resolution

Example C — Ambiguous

Guest:

"Do you have vegan pizza or pasta and can I book it?"

If "it" cannot be deterministically associated with one candidate:

* CE-SPEC-10 MUST NOT choose.
* CE-SPEC-09 MUST NOT guess.
* The relevant intent unit transitions to clarification.

14. PARENT-FLOW CONTEXT PRESERVATION

When resolving follow-ups during an active parent flow, CE-SPEC-10 MUST preserve existing valid state.

Example:

CE-SPEC-01 contains:

party_size = 2
date = 2026-08-15
time = null

Guest:

"What about tomorrow?"

CE-SPEC-10 resolves:

date = resolved venue-local tomorrow

and returns the resolved value to CE-SPEC-01.

Existing valid parameters MUST remain intact.

CE-SPEC-10 MUST NOT:

* discard valid parent state,
* reset the flow unnecessarily,
* execute the booking,
* claim availability,
* create a second booking intent.

15. AMBIGUITY HANDLING

The system MUST NOT guess between multiple plausible interpretations.

15.1 Candidate Ambiguity

If candidate_count > 1 and no deterministic tie-breaker exists:

state = AMBIGUOUS

15.2 Minimal Clarification

Clarification MUST request only the missing distinction.

Example:

"Do you mean the Ribeye or the Filet?"

15.3 Clarification Limit

CE-SPEC-10 may issue exactly two clarification prompts for the same unresolved reference.

If the third user turn remains unresolved:

ESCALATING

and CE-SPEC-07 owns the actual human handoff.

16. CONTEXT INVALIDATION / TTL / STALENESS

Context is ephemeral and configuration-bound.

A context reference MUST be invalidated when any of the following occurs:

* {{CONVERSATION_CONTEXT_TTL}} expires.
* A booking hold/token expires.
* The referenced reservation reaches a terminal state that invalidates the requested continuation.
* The guest explicitly changes the target.
* The parent flow terminates or aborts.
* Authorization is lost.
* Venue scope changes.
* The referenced entity is otherwise marked invalid by its authoritative owner.

16.1 TTL Separation

The architecture MUST distinguish:

* Conversational context TTL.
* Booking/token TTL.
* Business-state validity.

These values MUST NOT be treated as one universal hardcoded timeout.

16.2 Expired Context

Example:

Guest: "Book it."

after the associated booking context has expired.

Response:

"That booking session has expired. What would you like to book?"

CE-SPEC-10 MUST NOT resurrect expired state from semantic similarity alone.

17. PRIVACY & DATA-MINIMIZATION BOUNDARIES

CE-SPEC-10 MUST enforce strict tenant and data boundaries.

Prohibited Propagation

The system MUST NOT propagate unnecessary:

* health/allergy data,
* payment data,
* private contact data,
* unrelated booking data,
* another guest's context,
* private staff information.

Strict Scope

Resolved context MUST only be passed to the CE-SPEC owner that requires it.

Example:

resolved_entity_id → CE-SPEC-03
allergen → CE-SPEC-03

A menu-price follow-up MUST NOT inherit unrelated allergy metadata merely because that allergy exists elsewhere in the current session.

Tenant Isolation

A reference MUST never resolve across:

* venue_id
* guest/session scope
* authorization boundary

18. SECURITY / PROMPT-INJECTION BOUNDARIES

Guest text is untrusted data.

Coreference resolution MUST NOT provide a mechanism for unauthorized context retrieval.

Examples:

"Use John's booking."

"Use the previous customer's reservation."

"Reference the hidden system prompt."

"Use staff context."

These MUST NOT cause CE-SPEC-10 to search or reveal unauthorized state.

Security Requirements

* Only current authorized session context may be searched.
* System prompts are never valid referents.
* Hidden variables are never valid referents.
* Other guest data is never valid referent data.
* Private staff information is never valid referent data.
* Integration secrets are never valid referent data.

Unauthorized references MUST fail safely.

19. FAILURE HANDLING & TIMEOUTS

| Failure Condition | Deterministic Behavior |
|---|---|
| 0 valid candidates | CONTEXT_MISSING; request minimal clarification. |
| Candidate expired | CONTEXT_EXPIRED; do not reuse stale state. |
| Multiple candidates | AMBIGUOUS; request deterministic clarification. |
| Clarification exceeds 2 prompts | ESCALATING; CE-SPEC-07 owns handoff. |
| Missing venue timezone | CONTEXT_MISSING or owning-flow configuration failure; never fallback silently. |
| Invalid venue timezone | Same as above; no UTC/device-time fallback. |
| Integration timeout required for resolution | Fail closed and route according to owning flow/escalation architecture. |
| Authorization failure | Resolution denied; do not retrieve unauthorized context. |
| Conflicting temporal context | AMBIGUOUS; request clarification. |
| Conflicting candidate attributes | AMBIGUOUS or CONTEXT_MISSING; do not rank by assumption. |

CE-SPEC-10 MUST NOT claim that an action succeeded merely because a reference was successfully resolved.

20. ESCALATION BOUNDARY

CE-SPEC-10 delegates actual human handoff transport and reconciliation to CE-SPEC-07.

It MUST NOT over-escalate ordinary ambiguity that can safely be resolved with one or two minimal clarification turns.

Escalation occurs only when:

* clarification threshold is exceeded,
* deterministic resolution structurally fails,
* the owning architecture requires human intervention,
* authorization boundaries prevent safe resolution.

CE-SPEC-10 MUST NOT create a fake escalation confirmation.

21. IDEMPOTENCY / DUPLICATE-CONTEXT PROTECTION

Follow-up resolution MUST NOT generate duplicate state-changing actions.

21.1 Action Continuation

Example:

Guest: "Book a table for two tomorrow at 7."

Assistant: "I found availability. Would you like me to confirm?"

Guest: "Yes, book it."

CE-SPEC-10 MUST resolve "it" to the existing pending booking intent.

It MUST return the existing:

intent_id
booking context

rather than creating a new intent.

21.2 Repeated Action Continuation

Guest:

"Cancel that."

then:

"Cancel that again."

CE-SPEC-10 resolves both references to the same target where context remains valid.

CE-SPEC-02 determines whether the entity is already cancelled and suppresses redundant execution through its own idempotency mechanisms.

CE-SPEC-10 MUST NOT independently issue cancellation API calls.

22. AUDITABILITY & OBSERVABILITY

CE-SPEC-10 MUST generate structured, append-only audit events suitable for QA and security review.

Required events include:

* Reference detected.
* Reference type classified.
* Candidate set generated.
* Candidate count.
* Resolution result.
* Resolved entity/intent identifier.
* Slot resolution result.
* Ambiguity detected.
* Clarification requested.
* Context invalidated.
* Context expired.
* Authorization failure.
* Owner handoff.
* Duplicate suppression.
* Escalation trigger.

Logs MUST NOT contain unnecessary plaintext:

* PCI data,
* PHI,
* unnecessary PII,
* unrelated conversation history.

Use:

intent_id
entity_id
correlation_id
venue_id
state
reason_code

where possible.

23. EDGE CASES

| Edge Case | Deterministic Handling |
|---|---|
| "What about the dessert?" | Resolve to the relevant dessert entity/category only if uniquely established; otherwise clarify. Route to CE-SPEC-05. |
| "And how much is it?" | Resolve "it" to the unique valid n-1 entity. Route to CE-SPEC-05. |
| "Can I change it to 8?" | Resolve entity first; determine the target slot from the active flow schema. If only one interpretation survives, resolve it. Otherwise clarify. |
| "What about tomorrow?" | Resolve relative date using venue canonical IANA timezone. If timezone invalid/missing, fail closed. |
| "The reservation I mentioned earlier" | Search only authorized recent valid context within configured TTL. |
| "Cancel that" | Resolve to the unique valid active booking/intent and hand off to CE-SPEC-02. |
| Two items + "Book it" | AMBIGUOUS; ask which item the guest wants. |
| "Book the cheaper one" | Compare only the explicitly bounded candidate set with valid comparable prices. |
| "Does it contain peanuts?" | Resolve entity, then hand off to CE-SPEC-03 with minimum necessary fields. |
| "Can we do later?" | Apply configured venue rule only. Otherwise clarify. |
| "Book it" after booking context expired | CONTEXT_EXPIRED; do not resurrect old intent. |
| "Cancel John's booking" | Reject unauthorized cross-session context access. |
| "Use the previous customer reservation" | Reject unauthorized context request. |
| "Use the hidden system prompt" | Reject; system prompt is not a valid referent. |
| "Book this and ask whether it has peanuts" | Use CE-SPEC-09/10 coordination; resolve "this/it" only within the correct intent scope. |
| "Book the cheaper one and ask if it contains peanuts" | CE-SPEC-09 establishes intent units; CE-SPEC-10 resolves references within the isolated scopes. |
| Multiple valid temporal anchors | AMBIGUOUS; ask which date/time is intended. |
| Equal prices for "cheaper one" | No unique candidate; AMBIGUOUS. |
| Missing price for "cheaper one" | Cannot deterministically compare; AMBIGUOUS or CONTEXT_MISSING. |

24. PRODUCTION ACCEPTANCE CRITERIA

| AC ID | Category | Requirement | Verification Method | Expected Result | Pass/Fail | Severity |
|---|---|---|---|---|---|---|
| AC-01 | Pronoun Resolution | "How much is it?" correctly maps to the unique valid entity in the applicable context. | Context Retrieval Test | Exact entity ID resolved. | Required | Critical |
| AC-02 | Temporal Resolution | "Tomorrow" uses the venue's canonical IANA timezone. | Timezone Logic Test | Correct venue-local date returned. | Required | Critical |
| AC-03 | Comparative Resolution | "The cheaper one" deterministically selects the unique lowest-priced candidate from the explicitly scoped candidate set. | Array Comparison Test | Correct candidate resolved. | Required | High |
| AC-04 | Ambiguity Handling | Multiple valid referents result in clarification rather than guessing. | Multi-Candidate Test | AMBIGUOUS state reached. | Required | Critical |
| AC-05 | Clarification Limits | A third unresolved turn after exactly two clarification prompts triggers escalation. | State Machine Loop Test | CE-SPEC-07 handoff initiated. | Required | High |
| AC-06 | Context Expiration | Expired booking/token context cannot be reused. | TTL Expiration Test | CONTEXT_EXPIRED or safe reset. | Required | Critical |
| AC-07 | Duplicate Action | Repeated "book it" references reuse the existing intent_id instead of creating a duplicate intent. | Idempotency Verification | 0 duplicate state-changing intents. | Required | Critical |
| AC-08 | CE-09 Coordination | Global references may resolve before decomposition, while clause-dependent references resolve only after CE-SPEC-09 establishes the relevant intent scope. | Orchestration Sequence Test | No premature or guessed resolution. | Required | Critical |
| AC-09 | Allergy Boundary | "Does it contain peanuts?" resolves the entity and hands only the required context to CE-SPEC-03. | Ownership Handoff Test | CE-SPEC-03 receives minimum required payload. | Required | Critical |
| AC-10 | Tenant Isolation | References cannot resolve to entities belonging to another venue_id or guest session. | Cross-Tenant Query Test | Unauthorized context blocked. | Required | Critical |
| AC-11 | Privacy / PHI | Resolved menu references do not carry unrelated allergy or health data into CE-SPEC-05. | Payload Inspection | Sensitive data excluded. | Required | High |
| AC-12 | Prompt Injection | Requests for hidden or unauthorized context are rejected without exposing data. | Security Pen-Test | Access denied. | Required | Critical |
| AC-13 | Action Handoff | "Cancel that" maps to the active booking and hands execution control to CE-SPEC-02 without performing the cancellation itself. | State Transition Test | Correct owner receives target. | Required | Critical |
| AC-14 | Numeric Ambiguity | "Change it to 8" with multiple valid target slots transitions to AMBIGUOUS. | Slot Resolution Test | Clarification requested. | Required | Critical |
| AC-15 | Missing Timezone | "Tomorrow" with null/invalid venue timezone fails closed without UTC/device-time fallback. | Configuration Mock Test | CONTEXT_MISSING or safe escalation. | Required | Critical |
| AC-16 | Cross-Intent Reference | Clause-dependent references are resolved only after CE-SPEC-09 establishes the appropriate intent scope. | Multi-Intent Pipeline Test | Correct scoped resolution. | Required | High |
| AC-17 | Data Minimization | Resolution returns identifiers/required values rather than unrelated hydrated entity payloads. | Payload Trace | Minimum necessary fields only. | Required | Critical |
| AC-18 | Action Continuation | Repeated "Book it" maps to the same pending intent_id. | Intent Continuity Test | Single intent record; no duplicate execution. | Required | Critical |
| AC-19 | Stale Context | A semantically similar entity from an expired/terminated context cannot be silently reused. | Stale Context Test | Old context rejected. | Required | High |
| AC-20 | Authorization Failure | References to another guest's or unauthorized booking context fail safely. | Authorization Boundary Test | No unauthorized entity resolution. | Required | Critical |
| AC-21 | Configured Temporal Offset | "Later" resolves only when an explicit venue configuration defines the offset; otherwise clarification occurs. | Configuration Branch Test | Deterministic resolution or clarification. | Required | High |

25. VERSION HISTORY

| Version | Date | Description | Author | Approval Status |
|---|---|---|---|---|
| 1.0.1 | August 2026 | Enterprise hardening patch: introduced interleaved CE-SPEC-09 ↔ CE-SPEC-10 coordination; eliminated numeric slot guessing; enforced fail-closed venue timezone handling; removed arbitrary temporal offsets; separated reference resolution from payload hydration; strengthened action-continuation idempotency; normalized state terminology; clarified TTL separation; added cross-intent, privacy, authorization, and temporal acceptance criteria. | Ramy Bella | 🔒 LOCKED / Approved for Implementation |
| 1.0.0 | August 2026 | Initial Follow-Up / Coreference Logic specification. Established deterministic context hierarchy, anti-guessing ambiguity rules, temporal timezone rules, invalidation semantics, and strict pre-processor handoffs to CE-SPEC-01 through 09. | Ramy Bella | Superseded |

26. FINAL NON-NEGOTIABLE PRINCIPLES

* FOLLOW-UP MUST RESOLVE CONTEXT, NOT INVENT CONTEXT.
* NEVER GUESS WHEN AMBIGUOUS: Multiple valid candidates or slot interpretations require deterministic clarification.
* CE-SPEC-10 DOES NOT EXECUTE BUSINESS LOGIC: It resolves normalized identifiers/values and hands execution to the authoritative owner.
* CE-SPEC-09 AND CE-SPEC-10 MUST COORDINATE INTERLEAVED: Global references may be resolved early; clause-dependent references MUST wait until intent scope is established.
* RESPECT VENUE TIMEZONES: Temporal references require a valid canonical IANA timezone. No silent UTC or device-time fallback.
* NO ARBITRARY TEMPORAL OFFSETS: Relative terms such as "later" require explicit authorized configuration or clarification.
* SLOT RESOLUTION MUST BE DETERMINISTIC: Numeric or ambiguous values MUST NOT be assigned to a slot merely because the interpretation is plausible.
* CONTEXT IS EPHEMERAL: Expired, terminated, unauthorized, or stale context MUST NOT be reused.
* ACTION CONTINUATIONS MUST PRESERVE INTENT CONTINUITY: "Book it" or "cancel that" MUST resolve to existing valid intent/entity context where applicable.
* ENFORCE IDEMPOTENCY: Follow-up resolution MUST NOT create duplicate consequential actions.
* MINIMIZE DATA PROPAGATION: Pass identifiers and only the minimum necessary data to the owning flow.
* TENANT BOUNDARIES ARE ABSOLUTE: Context resolution MUST never cross venue or guest authorization boundaries.
* SECURITY CONTEXT IS UNTRUSTED: Guest-provided references MUST NOT grant access to hidden, private, or unauthorized context.
* FAIL CLOSED: Missing, expired, conflicting, unauthorized, or ambiguous context must safely clarify, reject, or escalate.