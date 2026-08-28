PE-SPEC-13: Prompt Tool Instructions

1. DOCUMENT CONTROL
| Attribute | Value |
|---|---|
| Document Title | 13 Prompt Tool Instructions.md |
| Document ID | PE-SPEC-13 |
| Version | 1.0.0 |
| Status | DRAFT / Implementation Specification |
| Author | Ramy Bella |
| Classification | Confidential / Enterprise Proprietary |
| Target Audience | AI Architects, Prompt Engineers, Backend Engineers, Tool/API Engineers, Security Architects, QA Engineers |
| Parent Document | PE-SPEC-01 |
| Related Documents | PE-SPEC-04, PE-SPEC-06, PE-SPEC-07, PE-SPEC-08, PE-SPEC-09, PE-SPEC-10, PE-SPEC-11, PE-SPEC-12, PE-SPEC-14, PE-SPEC-15, CE-SPEC-01 through CE-SPEC-12 |
| System | Restaurant AI System |
| Phase | Phase 4 — Prompt Engineering |
| Lifecycle Folder | EXECUTION CONTRACTS |
| Last Updated | August 2026 |


2. EXECUTIVE PURPOSE
The Prompt Tool Instructions Architecture (PE-SPEC-13) defines the deterministic architecture for representing authorized tool capabilities to the Large Language Model (LLM).
To interact with the physical restaurant environment (e.g., booking a table, looking up a menu, issuing a refund), the LLM must generate structured tool calls. However, giving the model a tool schema is fraught with security and consistency risks if not strictly governed. PE-SPEC-13 establishes the precise contracts, parameter requirements, usage constraints, and boundary definitions necessary for the model to generate valid tool proposals, while guaranteeing that the model never gains the authority to execute those tools.
The central invariant of this architecture is absolute:
MODEL TOOL PROPOSAL \neq TOOL AUTHORIZATION \neq TOOL EXECUTION \neq BUSINESS SUCCESS.


3. PURPOSE AND SCOPE
Scope: What PE-SPEC-13 Controls



Tool Capability Representation: How tools are structurally described and instructed to the LLM.

Tool Schemas: Formal definitions of required input and output structures.

Parameter Contracts: Strict enforcement of data types, enums, and required/optional fields.

Usage Constraints: Instructions governing when a tool proposal is structurally appropriate.

Tool Result Handling: How the outcomes of executed tools are represented back to the model.

Prevention of Hallucinated Tools: Strict mechanisms preventing the LLM from inventing tools or parameters.

Prevention of Unauthorized Execution: Delineating the boundary between prompt-level proposal and runtime-level authorization.
Scope: What PE-SPEC-13 Explicitly Does NOT Control

Authorizing Tools: Phase 3/Runtime dictates if a user is permitted to invoke a capability.

Executing Tools: The backend API executes the tool.

Determining Business Outcomes: PE-SPEC-13 formats the result; it does not interpret its business meaning.

Changing CE-SPEC State: Tool results alter Phase 3 state, not PE-SPEC-13.

Creating Permissions: Instructing the model that a tool exists does not grant RBAC permissions.

Retrieving Arbitrary Context: PE-SPEC-05 governs RAG and context retrieval.

Deciding if an Action is Allowed: The runtime is the sole arbiter of legality.


4. ARCHITECTURAL POSITION
PE-SPEC-13 sits between dynamic assembly (PE-SPEC-06) and compilation (PE-SPEC-04), defining the capability schemas injected into the prompt payload.
[PHASE 3 / RUNTIME]  <-- Defines what is currently authorized and required.
↓
[PE-SPEC-06]         <-- Assembles the blueprint, indicating required tools.
↓
[PE-SPEC-13]         <-- Formats the authorized tool capability contracts.
↓
[PE-SPEC-04]         <-- Compiles and escapes the schemas into the payload.
↓
[LLM]                <-- Processes prompt and generates a response.
↓
[MODEL TOOL PROPOSAL]<-- The LLM asks to use a tool.
↓
[RUNTIME AUTHORIZATION]<- Validates the proposal against actual RBAC and State.
↓
[TOOL EXECUTION]     <-- The backend API fires.
↓
[AUTHORITATIVE RESULT]<- The backend API returns the truth.
↓
[PHASE 3 STATE UPDATE]<- Business state changes based on the result.



Architectural Boundary: The LLM can propose. The Runtime decides. The Tool executes.
5. TOOL DEFINITION MODEL
Every tool represented to the LLM MUST adhere to a canonical, logical ToolDefinition schema.
{
"tool_id": "booking_create",
"tool_version": "1.2.0",
"description": "Proposes the creation of a new restaurant reservation.",
"purpose": "To submit parsed booking constraints to the runtime for validation and execution.",
"authorized_scope": "TRANSACTION_ACTIVE",
"input_schema": {
"type": "object",
"properties": {
"party_size": {"type": "integer"},
"target_date": {"type": "string", "format": "date"},
"target_time": {"type": "string", "format": "time"}
},
"required": ["party_size", "target_date", "target_time"]
},
"output_schema": {"type": "object", "properties": {"status": {"type": "string"}}},
"side_effect_class": "STATE_CHANGING",
"idempotency_requirement": "REQUIRED",
"required_context": ["venue_id", "session_id"],
"tenant_binding": "REQUIRED",
"session_binding": "REQUIRED",
"security_classification": "PRIVILEGED",
"provenance": "CE-SPEC-01",
"checksum": "sha256:abcd1234efgh..."
}

Constraint: Authorization parameters (like session tokens or RBAC headers) MUST NOT live inside the LLM prompt. Authorization is entirely a runtime concern.
6. TOOL CLASSIFICATION
Tools MUST be explicitly classified to inform runtime handling and structural prompt instructions. This classification informs how the tool is presented, but DOES NOT itself grant permission.

READ_ONLY: Fetching static public data (e.g., getting venue hours).

STATE_QUERY: Fetching dynamic data (e.g., checking table availability).

STATE_CHANGING: Modifying internal business state (e.g., creating a booking).

EXTERNAL_SIDE_EFFECT: Actions that affect external systems or guests (e.g., sending an SMS, charging a card).

PRIVILEGED: Actions requiring elevated or specific staff authorization.

SAFETY_CRITICAL: Actions interacting with life-safety domains (e.g., allergy overrides).


7. TOOL AVAILABILITY
A tool MUST only appear in the model-facing tool set when the authoritative runtime has explicitly authorized it for the current turn, state, and tenant scope.



PE-SPEC-13 MUST NOT decide authorization independently.

If a tool is not explicitly authorized by Phase 3:

It MUST NOT be exposed as an available tool in the prompt.

The model MUST NOT be instructed to use or mention it.

The runtime MUST reject the unauthorized proposal and FAIL CLOSED if the model somehow hallucinates the tool call.



8. TOOL PARAMETER CONTRACT
Parameter schemas communicated to the LLM MUST be strictly defined.
Requirements:



Explicit Types: Strings, integers, booleans, arrays MUST be explicitly typed.

Required/Optional: All parameters must define requirement status. PE-SPEC-13 MUST NOT allow the LLM to skip required parameters.

Enums: Constrained options MUST be represented as strict enums (e.g., ["LUNCH", "DINNER"]).

Format Validation: String types representing times, dates, or UUIDs must define formats.

Bindings: The schema MUST enforce tenant_binding and session_binding requirements at the runtime validation layer.

Entity IDs: Favor opaque identifiers (e.g., menu_item_id) over natural language strings when acting on database entities.

No Undeclared Parameters: The runtime MUST reject proposals containing parameters not in the schema.

No Implicit Defaults: PE-SPEC-13 MUST NOT infer missing parameters or instruct the LLM to invent business defaults.


9. NO BUSINESS LOGIC IN TOOL INSTRUCTIONS
Tool descriptions MUST describe mechanical capabilities and structural contracts, not create new business policies. Phase 3 remains authoritative for business behavior.



Prohibited (Business Logic): "If the guest sounds upset, automatically use this tool to refund them up to $50."

Allowed (Capability Description): "This tool proposes a refund request. It requires a valid booking_id and an amount. Execution is subject to runtime authorization."


10. TOOL SELECTION BOUNDARY
PE-SPEC-13 communicates which authorized tools are structurally available to the model, but defines strict boundaries around the model's agency:



LLM tool selection remains a proposal.

The LLM CANNOT grant itself a tool.

The LLM CANNOT invent a new tool.

The LLM CANNOT alter or bypass a tool's schema.

The LLM CANNOT bypass a required confirmation or authorization state defined by Phase 3 (e.g., the model cannot generate a booking confirmation tool call without the required payment_token secured out-of-band).


11. TOOL CALL CONTRACT
When the LLM proposes a tool execution, the output MUST conform to the canonical logical structure of a tool proposal.
{
"tool_id": "booking_create",
"tool_version": "1.2.0",
"arguments": {
"party_size": 4,
"target_date": "2026-08-13",
"target_time": "19:00"
},
"correlation_id": "txn-8899-abc"
}



Clarification: This object is a PROPOSAL only. It has zero business impact until the runtime validates the schema, checks authorization, verifies idempotency, and executes it against the backend.
12. TOOL RESULT BOUNDARY
A clear architectural distinction exists between proposing an action and achieving a business result.

Tool Proposal: LLM outputs a JSON tool call.

Runtime Authorization Decision: Backend evaluates if the user can do this.

Tool Execution: Backend calls the database/integration.

Tool Result: Backend returns a status to the prompt cycle.

Business Success: Phase 3 registers the finalized state.
Invariant: A model MUST NOT tell a guest that a tool action succeeded solely because it generated a tool call. Only an authoritative runtime TOOL_RESULT indicating SUCCESS authorizes the model to confirm the action to the user.


13. TOOL RESULT TRUST CLASSIFICATION
When returning the outcome of a tool execution back to the prompt, PE-SPEC-13 dictates strict handling to prevent indirect prompt injection.



Structured Authoritative Metadata: Execution flags (e.g., status: success, http_code: 200, booking_id: 123) are classified as AUTHORITATIVE_FACT.

Untrusted Textual Content: Any raw text embedded in the tool result (e.g., a review scraped by a tool, or notes written by a third party) MUST be classified as UNTRUSTED_TOOL_CONTENT (Per PE-SPEC-12).

Boundary Defense: Untrusted textual content MUST remain data, must be fenced, and MUST NOT become system instructions. Reference PE-SPEC-11 for injection defenses.


14. SIDE-EFFECT TOOL SAFETY
For STATE_CHANGING and EXTERNAL_SIDE_EFFECT tools, PE-SPEC-13 requires specific capability representations:



Confirmation State: The tool schema MUST reflect whether Phase 3 requires explicit user confirmation before the model may propose the tool.

Correlation IDs: The tool proposal MUST include a correlation_id tying the proposal to the exact active conversational turn to prevent replay attacks.

Exact Target Entity: Tools acting on specific records must require a verified entity_id.
PE-SPEC-13 does NOT invent new confirmation rules; it only communicates the authoritative rules supplied by Phase 3.


15. IDEMPOTENCY / DUPLICATE TOOL CALLS
Tool instructions MUST support the existing architecture's idempotency requirements.



If a guest says "Yes, book it," the LLM generates a tool proposal.

If the guest immediately repeats "Book it", the LLM may generate a second identical tool proposal.

Rule: This MUST NOT cause an unrelated duplicate operation. Phase 3 / Runtime manages the idempotency key (often tied to the correlation_id or active session intent) to suppress duplicate executions.

Tool-level idempotency belongs to the backend. PE-SPEC-13 ensures the prompt instructs the model to provide the required tracking parameters to allow the backend to do its job.


16. TENANT / SESSION ISOLATION
Every tenant-sensitive or session-sensitive tool MUST be strictly bound to the authorized context.



Venue ID Validation: Tool schemas must implicitly or explicitly bind to the active venue_id. The LLM cannot substitute a different venue's identifier into a tool call unless operating in a globally authorized cross-venue mode established by Phase 3.

Session ID Validation: The tool proposal is bound to the active session_id.

Enforcement: PE-SPEC-13 does not determine authorization. The runtime MUST reject proposals with cross-tenant target substitution or cross-session guessing.


17. SECRET / CREDENTIAL BOUNDARY
Tool instructions MUST NEVER expose security credentials to the LLM.



API keys, bearer tokens, database credentials, signing keys, and internal secrets MUST NOT appear in the tool schema or descriptions.

Tool schemas describe parameters for the business action, not credentials for the API transport.

If a tool requires a token to execute, that token is injected by the runtime HTTP client out-of-band of the LLM.


18. DATA MINIMIZATION
Tool instructions MUST expose only the absolute minimum parameters needed for the authorized action.



Avoid requiring full user objects (user_name, user_email, user_phone) as input parameters when a single opaque entity_id (user_123) is sufficient for the backend to perform the action.

Reference PE-SPEC-12 (Prompt Data Boundaries) for broader data-minimization rules and PII/PHI exclusions.


19. VERSIONING / INTEGRITY
Tool definitions MUST be versioned and integrity-verified.



PE-SPEC-10 is authoritative for lifecycle/version governance.

PE-SPEC-13 MUST NOT redefine PE-SPEC-10.

Production tool instructions MUST use explicit, pinned versions (e.g., booking_create@1.2.0). The compiler (PE-SPEC-04) will reject unpinned or "latest" tool versions.


20. TOOL COMPATIBILITY
Tool definitions MUST declare architectural compatibility.
A ToolDefinition must be compatible with:



PE-SPEC-04 (Supported serialization formats, e.g., JSON Schema vs. XML schemas).

PE-SPEC-07 (Compatible variable hydration bindings).

PE-SPEC-08 / PE-SPEC-09 (Compatible templates and routes).

Runtime API Schema (1:1 mapping with the actual backend API).
Incompatibility detected during resolution MUST FAIL CLOSED.


21. PROMPT INJECTION DEFENSE
PE-SPEC-13 works in conjunction with PE-SPEC-11 to prevent manipulation via tool definitions.
User input MUST NEVER be able to:



Invent a new tool.

Modify an existing tool schema.

Enable an unauthorized tool.

Alter tool parameter validation rules outside the schema.

Change a tool version.

Bypass runtime authorization checks via prompt pleading.
The LLM is structurally constrained to the injected schemas.


22. OBSERVABILITY / AUDIT
Audit metadata MUST track the tool proposal lifecycle for observability.
Required Fields:



tool_id, tool_version

authorization_reference (Phase 3 intent ID)

correlation_id

proposal_status (Valid/Invalid)

runtime_authorization_result (Approved/Denied)

execution_result (Success/Failure)

failure_code (If applicable)
Constraint: Do NOT log unnecessary PII, PHI, PCI, secrets, or full sensitive payloads in standard observability tools. Log the structural identifiers and operational outcomes.


23. FAILURE ARCHITECTURE
Failure handling within PE-SPEC-13 MUST be deterministic. All critical security and authorization violations MUST FAIL CLOSED.
| Failure ID | Condition | System Response | Runtime Handoff | Severity |
|---|---|---|---|---|
| ERR_TOOL_01 | Requested tool is unavailable/missing. | Abort compilation; FAIL CLOSED. | System Error | Critical |
| ERR_TOOL_02 | Requested tool is unauthorized. | Exclude tool; DO NOT expose to LLM. | Security Alert | Critical |
| ERR_TOOL_03 | Invalid tool schema definition. | Abort compilation; FAIL CLOSED. | System Error | Critical |
| ERR_TOOL_04 | Missing required parameter in proposal. | Reject proposal. | Re-prompt LLM | High |
| ERR_TOOL_05 | Invalid parameter type in proposal. | Reject proposal. | Re-prompt LLM | High |
| ERR_TOOL_06 | Undeclared parameter in proposal. | Reject proposal. | Re-prompt LLM | High |
| ERR_TOOL_07 | Tenant/Venue mismatch in target. | Reject proposal; FAIL CLOSED. | Security Alert | Critical |
| ERR_TOOL_08 | Session mismatch. | Reject proposal; FAIL CLOSED. | Security Alert | Critical |
| ERR_TOOL_09 | Tool version mismatch / unpinned. | Abort compilation; FAIL CLOSED. | System Error | Critical |
| ERR_TOOL_10 | Checksum mismatch for ToolDefinition. | Abort compilation; FAIL CLOSED. | SecOps Alert | Critical |
| ERR_TOOL_11 | Tool proposal rejected by Runtime Auth. | Block execution; Generate refusal. | CE-SPEC-08 | Critical |
| ERR_TOOL_12 | Tool execution failure (Backend API down). | Bubble error to prompt context. | CE-SPEC-07 | High |
| ERR_TOOL_13 | Invalid tool result payload returned. | Treat as ERR_TOOL_12. | CE-SPEC-07 | High |
| ERR_TOOL_14 | Unauthorized tool result content. | Fence content; tag untrusted. | PE-SPEC-11 | High |
| ERR_TOOL_15 | Duplicate state-changing attempt. | Suppress execution via Idempotency. | Return Cached Result | Medium |


24. SECURITY / TOOL THREAT MODEL
| Threat | Attack Surface | Preventive Control | Detection | Response | Owner | Severity |
|---|---|---|---|---|---|---|
| Tool Hallucination | LLM Output | Strict schema enforcement; Runtime rejects undeclared tools. | Runtime Validation | ERR_TOOL_01 | Platform | High |
| Unauthorized Use | LLM Output | Runtime RBAC; Schema excluded from unauthorized prompts. | Runtime Auth | ERR_TOOL_02 | Phase 3 Sec | Critical |
| Schema Manipulation | Input Injection | Tool schemas are immutable components. | Compiler validation | ERR_TOOL_10 | Architecture | Critical |
| Parameter Injection | LLM Output | Strict JSON schema typing and parameter restriction. | Schema Validation | ERR_TOOL_06 | Platform | High |
| Privilege Escalation | LLM Output | Model proposals possess zero implicit authorization. | Runtime Auth | ERR_TOOL_11 | Phase 3 Sec | Critical |
| Cross-Tenant Attack | LLM Output | Hard parameter bindings to active venue_id. | Scope Validation | ERR_TOOL_07 | Runtime Sec | Critical |
| Secret Exposure | Tool Schemas | Credentials removed from LLM-facing schemas. | Payload Inspection | ERR_SEC_01 | SecOps | Critical |
| Malicious Output | Tool Result | Distinct trust classifications for metadata vs. raw text. | Trust Classifier | ERR_TOOL_14 | PE-SPEC-11 | High |
| Duplicate Side-Effect | LLM Output | correlation_id + backend idempotency keys. | Backend Execution | ERR_TOOL_15 | Backend Eng | Medium |


25. PRODUCTION ACCEPTANCE CRITERIA
| AC ID | Category | Requirement | Verification Method | Expected Result | Pass/Fail | Severity |
|---|---|---|---|---|---|---|
| AC-TOOL-01 | Auth Exposure | Unauthorized tools are never serialized into the prompt payload. | Payload Trace Test | Schema completely absent | Required | Critical |
| AC-TOOL-02 | Auth Authority | A valid tool proposal generated by the LLM is blocked if runtime RBAC denies the action. | RBAC Execution Mock | Execution rejected | Required | Critical |
| AC-TOOL-03 | Schema Rejection | Proposals containing an undeclared, fabricated parameter are deterministically rejected. | Proposal Validator Test | ERR_TOOL_06 | Required | High |
| AC-TOOL-04 | Type Rejection | Passing a string "three" to a required integer parameter yields immediate rejection. | Proposal Validator Test | ERR_TOOL_05 | Required | High |
| AC-TOOL-05 | Tenant Isolation | A tool proposal substituting a different venue_id fails closed. | Cross-Tenant Mock | ERR_TOOL_07 | Required | Critical |
| AC-TOOL-06 | Secret Exclusion | Tool definition schemas injected into prompts contain zero API keys or authentication secrets. | Schema Audit | Zero secrets present | Required | Critical |
| AC-TOOL-07 | No Hallucination | The runtime successfully rejects an LLM attempt to invoke an invented tool_id. | Invented Tool Mock | ERR_TOOL_01 | Required | Critical |
| AC-TOOL-08 | Version Pinning | Tool schemas referenced without an exact version pin abort compilation. | Resolution Validator | ERR_TOOL_09 | Required | Critical |
| AC-TOOL-09 | Checksum Verify | Tampered ToolDefinition objects fail checksum and abort compilation. | Integrity Simulation | ERR_TOOL_10 | Required | Critical |
| AC-TOOL-10 | Result Handling | An LLM proposal must wait for a backend SUCCESS return before confirming the action to the guest. | State Sequence Test | Business confirmation delayed | Required | Critical |
| AC-TOOL-11 | Idempotency | Identical duplicate proposals generated in immediate succession trigger backend idempotency without crashing. | Duplicate Output Mock | Execution suppressed safely | Required | High |
| AC-TOOL-12 | Business Logic | Tool descriptions contain mechanical definitions, completely devoid of dynamic business rules. | Semantic Audit | Zero business logic | Required | Critical |


26. INTEGRATION CONTRACTS
PE-SPEC-13 interfaces precisely with the surrounding architecture:



PE-SPEC-04: Handles the mechanical compilation, escaping, and final serialization of PE-SPEC-13 schemas.

PE-SPEC-05: Resolves context, which may act as input constraints for tools.

PE-SPEC-06: Assembles the Blueprint, declaring which authorized tools are needed.

PE-SPEC-07: Hydrates variables required to evaluate tool dependencies.

PE-SPEC-08: Houses the immutable prompt templates that reference these tools.

PE-SPEC-09: Routes the prompt flow, dictating the chronological availability of tools.

PE-SPEC-10: Controls the versioning, integrity, and lifecycle governance of ToolDefinitions.

PE-SPEC-11: Owns the security boundaries, defending against malicious tool proposals.

PE-SPEC-12: Owns data boundaries, minimizing PII/PHI/PCI exposed within tool parameters.

PE-SPEC-14: Dictates the final output contract governing how the LLM formats its proposal (e.g., JSON).

PE-SPEC-15: Governs the error handling for when the LLM makes an invalid proposal.

Phase 3 / Runtime: Determines AUTHORIZATION, performs EXECUTION, and returns the AUTHORITATIVE RESULT.


27. VERSION HISTORY
| Version | Date | Description | Author | Approval Status |
|---|---|---|---|---|
| 1.0.0 | August 2026 | Initial Prompt Tool Instructions specification. Established strict separation between LLM tool proposals and runtime execution authority. Defined parameter schemas, idempotency boundaries, and robust failure behaviors. | Ramy Bella | DRAFT / Implementation Specification |


28. FINAL NON-NEGOTIABLE PRINCIPLES



TOOL INSTRUCTIONS COMMUNICATE CAPABILITIES; THEY DO NOT GRANT AUTHORIZATION.

MODEL TOOL PROPOSAL IS NOT TOOL EXECUTION.

TOOL EXECUTION IS NOT BUSINESS SUCCESS UNTIL RUNTIME CONFIRMS IT.

THE LLM MUST NEVER INVENT TOOLS.

THE LLM MUST NEVER MODIFY TOOL SCHEMAS.

UNDECLARED PARAMETERS MUST BE REJECTED.

TOOL VERSIONS MUST BE EXPLICITLY PINNED.

TENANT AND SESSION BINDINGS ARE ABSOLUTE.

SECRETS MUST NEVER ENTER TOOL INSTRUCTIONS.

TOOL RESULTS MUST NOT BECOME SYSTEM INSTRUCTIONS.

DUPLICATE SIDE-EFFECTS MUST BE CONTROLLED BY RUNTIME IDEMPOTENCY.

PE-SPEC-13 MUST NOT CHANGE PHASE 3 BUSINESS LOGIC.

PE-SPEC-13 MUST NOT OVERRIDE PE-SPEC-11 SECURITY.

PE-SPEC-13 MUST NOT OVERRIDE PE-SPEC-12 DATA BOUNDARIES.

PE-SPEC-13 MUST NOT REDEFINE PE-SPEC-10 VERSION GOVERNANCE.

CRITICAL TOOL SECURITY VIOLATIONS MUST FAIL CLOSED.
ARCHITECTURAL VERDICT
APPROVED — DRAFT / IMPLEMENTATION SPECIFICATION
Implementation Readiness Statement:
READY FOR ENTERPRISE IMPLEMENTATION. PE-SPEC-13 establishes a mathematically rigorous, fail-closed contract for LLM tool integration. By unequivocally separating the LLM's capacity to propose an action from the runtime's authority to execute an action, this specification nullifies privilege escalation attacks, prevents hallucinated parameter injections, and guarantees that tool schemas strictly enforce enterprise data minimization and tenant isolation boundaries. It seamlessly integrates into the Phase 4 assembly pipeline without usurping Phase 3 business logic.