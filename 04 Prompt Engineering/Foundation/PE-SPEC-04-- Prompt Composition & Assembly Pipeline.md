PE-SPEC-04: Prompt Composition & Assembly Pipeline
1. DOCUMENT CONTROL
| Attribute | Value |
|---|---|
| Document Title | 04 Prompt Composition.md |
| Document ID | PE-SPEC-04 |
| Version | 1.1.0 |
| Status | DRAFT / Implementation Specification |
| Author | Ramy Bella |
| Classification | Confidential / Enterprise Proprietary |
| Target Audience | AI Engineers, Backend Engineers, Prompt Engineers, Platform Architects, Security Architects |
| Parent Document | PE-SPEC-01 |
| Related Documents | PE-SPEC-02, PE-SPEC-03, CE-SPEC-01 through CE-SPEC-12 |
| System | Restaurant AI System |
| Phase | Phase 4 — Prompt Engineering |
| Last Updated | August 2026 |
2. EXECUTIVE PURPOSE
The Prompt Composition & Assembly Pipeline (PE-SPEC-04) defines the deterministic mechanical compilation process for generating a prompt payload. It is the engine that aggregates immutable templates, Phase 3 authoritative state, runtime context, authorized tool schemas, and untrusted guest input into a single, canonical, provider-agnostic representation before adapting it for Large Language Model (LLM) inference.
While PE-SPEC-02 defines the logical architecture and instruction hierarchy of the System Prompt, PE-SPEC-04 strictly controls the mechanics of assembly. It ensures that payloads are constructed safely, reproducibly, and securely, enforcing strict data formatting, context-safe escaping, token governance, and fail-closed validation. The prompt compiler MUST NEVER invent, reinterpret, or modify business logic.
3. PURPOSE AND SCOPE
Scope: What PE-SPEC-04 Controls
 * The deterministic compiler pipeline and execution phases.
 * Canonical Intermediate Representation (CIR) of prompts.
 * Context-safe escaping and encoding mechanisms.
 * Data serialization formatting without semantic mutation.
 * Token accounting, strict budgeting, and deterministic truncation algorithms.
 * Null, missing, and default-value handling during compilation.
 * Provider-agnostic payload mapping via Provider Adapters.
 * Compilation validation, integrity checks, and failure behaviors.
 * Compilation observability and reproducibility constraints.
Scope: What PE-SPEC-04 Explicitly Does NOT Control
 * System Prompt Architecture: (Owned by PE-SPEC-02).
 * Role & Identity Constraints: (Owned by PE-SPEC-03).
 * Business Logic & State Transitions: (Owned by CE-SPEC-01 through 12).
 * Runtime Authorization: The compiler integrates schemas authorized by the runtime; it does NOT grant authorization.
 * Security Enforcement: The compiler escapes and structures data, but runtime evaluation validates LLM compliance.
4. ARCHITECTURAL POSITION
PE-SPEC-04 functions as the strict compiler between the Phase 3 / Runtime environment and the LLM API.
[PHASE 3 / RUNTIME] -> Outputs JSON State, Tool Schemas, User Input, KB Payloads
       ↓
[PE-SPEC-04 COMPILER]
   ├─ Preflight & Compatibility Validation
   ├─ Tenant & Scope Validation
   ├─ Canonical Serialization & Escaping
   ├─ Token Budget Evaluation & Truncation
   └─ Canonical Intermediate Representation (CIR)
       ↓
[PROVIDER ADAPTER] -> Formats CIR into Vendor-Specific API Payload (OpenAI, Anthropic, etc.)
       ↓
[LLM INFERENCE API]

5. FORMAL COMPILER INVARIANTS
The compilation pipeline is governed by the following absolute invariants. Any violation of these invariants MUST result in an immediate FAIL CLOSED state.
 * INVARIANT-01: Compilation MUST NOT modify Phase 3 semantics.
 * INVARIANT-02: Compilation MUST NOT create authoritative facts.
 * INVARIANT-03: Compilation MUST NOT create authorization.
 * INVARIANT-04: Untrusted data MUST NOT become trusted instructions through serialization. Serialization provides structure; it does not grant trust.
 * INVARIANT-05: Immutable components MUST NOT be overwritten or mutated by dynamic components.
 * INVARIANT-06: Tenant boundaries MUST be validated before tenant-scoped compilation begins.
 * INVARIANT-07: Missing required data MUST NOT be replaced with invented defaults (e.g., silent fallback to UTC for missing timezones is prohibited).
 * INVARIANT-08: Token optimization/truncation MUST NOT remove required authoritative state or immutable instructions.
 * INVARIANT-09: If the minimum valid prompt cannot fit within the provider token budget, compilation MUST FAIL CLOSED.
 * INVARIANT-10: The same valid compiler inputs and versions MUST produce the exact same Canonical Intermediate Representation (Reproducibility).
 * INVARIANT-11: Tool schemas MUST originate solely from an authoritative runtime authorization context.
 * INVARIANT-12: Provider-specific formatting (via adapters) MUST NOT alter canonical semantics.
6. FORMAL COMPILER MODEL (PIPELINE)
The prompt compilation MUST execute sequentially through the following deterministic phases:
 * INPUTS RECEPTION: Receive state, context, and input from Runtime/Phase 3.
 * PREFLIGHT VALIDATION: Verify existence of required base templates.
 * COMPATIBILITY VALIDATION: Cross-check PE-SPEC template versions against incoming CE-SPEC schema versions.
 * TENANT / SCOPE VALIDATION: Verify venue_id explicitly matches the current authorized session.
 * STATE VALIDATION: Validate that incoming authoritative state conforms to strict schemas.
 * CANONICAL SERIALIZATION: Convert objects into canonical text representations preserving type and order.
 * TRUST BOUNDARY ENCODING & ESCAPING: Apply context-safe encoding to untrusted payloads.
 * TOOL SCHEMA COMPILATION: Serialize authorized tool schemas.
 * CONTEXT ASSEMBLY: Construct the full payload structure.
 * TOKEN BUDGET EVALUATION: Calculate payload token size against dynamic budget.
 * DETERMINISTIC TRUNCATION: Shed lower-priority tokens if needed (and permitted).
 * CANONICAL INTERMEDIATE REPRESENTATION (CIR): Finalize the provider-neutral prompt envelope.
 * PROVIDER ADAPTER: Map CIR to the target LLM API format.
 * FINAL API PAYLOAD DISPATCH: Send to LLM.
7. CANONICAL INTERMEDIATE REPRESENTATION (CIR)
To maintain provider independence, the compiler produces a CanonicalPromptEnvelope. This ensures prompt semantics remain stable regardless of the ultimate LLM vendor (e.g., OpenAI, Anthropic, Google).
Structure of CanonicalPromptEnvelope:
{
  "compilation_metadata": {
    "prompt_id": "sys_booking_v1",
    "prompt_version": "1.2.0",
    "compiler_version": "1.0.1",
    "serialization_version": "1.0.0",
    "provider_adapter": "openai_chat_v2",
    "integrity_hash": "sha256-..."
  },
  "tenant_scope": "v_778",
  "correlation_id": "txn-992",
  "system_directives": [
    {"type": "IMMUTABLE_IDENTITY", "content": "..."},
    {"type": "INSTRUCTION_HIERARCHY", "content": "..."}
  ],
  "authoritative_state": "...",
  "authorized_tools": [],
  "authoritative_context": "...",
  "conversation_context": [],
  "untrusted_guest_input": "...",
  "token_budget": {
    "limit": 16384,
    "consumed": 2048,
    "truncated": false
  }
}

8. COMPONENT SERIALIZATION & STATE FIDELITY
The compiler serializes structured JSON state into model-readable string formats (e.g., XML or Markdown blocks).
State Fidelity Invariants:
 * The compiler MUST serialize; it MUST NOT infer, summarize, reinterpret, merge, split, correct, or normalize semantics.
 * Stable Ordering: Serializers MUST process fields in a deterministic order (e.g., alphabetical key sort) to ensure reproducibility, unless ordering is semantically meaningful in the source array, in which case exact original ordering MUST be preserved.
 * Explicit Type Preservation: Numbers, booleans, and strings must be explicitly distinguishable in the serialized format.
 * Explicit Null Semantics: The compiler MUST distinguish between missing, null, unavailable, empty collection [], and empty string "".
   * If a value is null, it serializes explicitly as [NULL].
   * If a collection is empty, it serializes as [EMPTY_COLLECTION].
   * Missing required fields trigger a compilation abort.
 * Redaction: If the compiler performs redaction, it MUST do so only according to an already-authorized security/privacy policy payload passed from runtime. The compiler does not invent redaction policies.
9. SYNTACTICAL FENCING & CONTEXT-SAFE ESCAPING
Serialization structures the prompt; however, serialization does NOT equal trust. Syntactical boundaries (tags, delimiters) are representations, not hardened security perimeters.
Context-Safe Escaping Requirements:
The compiler MUST NOT rely on simplistic string-replacement or naive regex to sanitize inputs. It MUST use context-safe encoding:
 * XML Escaping: Untrusted text injected into XML-like tags MUST be strictly XML-escaped (< becomes &lt;, > becomes &gt;, & becomes &amp;).
 * JSON Escaping: Text injected into JSON blocks MUST be strictly JSON-escaped (handling quotes, backslashes, and control characters).
 * Control Characters & Unicode: The compiler MUST neutralize or strip non-printable control characters and homoglyphs that could manipulate tokenizer behavior.
 * Delimiter Collision Prevention: The compiler must mathematically guarantee that no sequence of characters in the untrusted input can be parsed as a closing boundary for the current container.
10. TOOL SCHEMA COMPILATION
 * TOOL SCHEMA ≠ TOOL AUTHORIZATION.
 * The PE-SPEC-04 compiler ONLY compiles the JSON schemas supplied by the authoritative runtime authorization layer. It CANNOT independently decide to inject a tool schema.
 * Tool Results: Tool results returning from the backend MUST NOT automatically become system instructions. The compiler MUST preserve the trust classification supplied by the runtime.
   * Authoritative structured fields (e.g., status: success) are serialized as AUTHORITATIVE_FACT.
   * Untrusted textual payloads inside tool results (e.g., a review scraped from the web) MUST be escaped and fenced identically to untrusted guest input.
11. TOKEN GOVERNANCE & TRUNCATION
LLM context windows are finite. The compiler MUST enforce a deterministic token-budget model.
Token Allocation Priority (0 = Highest, 8 = Lowest):
 * Priority 0: Immutable system governance (Identity, Core Rules).
 * Priority 1: Authoritative Phase 3 state (CE-SPEC parameters).
 * Priority 2: Required tool schemas.
 * Priority 3: Current user input.
 * Priority 4: Required authoritative context (KB-SPEC targeted data).
 * Priority 5: Immediate prior turn (Assistant/User n-1).
 * Priority 6: Recent history.
 * Priority 7: Old history.
 * Priority 8: Optional contextual enrichment.
Truncation Algorithm:
 * If the projected token count exceeds the budget_limit, the compiler truncates starting at Priority 8 and works upwards.
 * Hard Minimum Required Content: Priority 0 through Priority 5 represent the minimum valid payload.
 * Fail-Closed Rule: Token optimization MUST NOT silently remove Priority 0-5 content. If the minimum valid prompt cannot fit within the budget, compilation MUST FAIL CLOSED.
12. CACHING COMPATIBILITY
Prefix caching (where supported by LLM providers) is an optimization, not a correctness requirement.
 * Rule: Correctness, security, state fidelity, and authorization ALWAYS take precedence over caching optimization.
 * Structure: If compatible with the Provider Adapter, the CIR SHOULD structure static components (Priority 0-2) at the beginning of the envelope. The compiler MUST NOT reorder semantically dependent information merely to improve cache hit rates.
13. PROVIDER ADAPTER MAPPING
The CanonicalPromptEnvelope (CIR) MUST be mapped to the specific LLM API via a Provider Adapter.
 * The Provider Adapter handles the conversion of CIR logical blocks into the vendor's specific schema (e.g., mapping system directives to OpenAI's system role array, or Anthropic's top-level system string).
 * Invariant: Provider-specific payload mapping MUST NOT alter canonical semantics. If a provider's API structure cannot safely represent a critical boundary from the CIR (e.g., inability to separate system vs. user roles), the adapter MUST reject the compilation.
14. SECURITY THREAT MODEL
The compiler pipeline implements specific technical controls against prompt-level threats.
| Threat | Attack Surface | Preventive Control (PE-SPEC-04) | Detection | Compiler Response | Runtime Owner | Audit Event | Severity |
|---|---|---|---|---|---|---|---|
| Direct Prompt Injection | Guest Input | Context-safe escaping; Structural XML fencing. | Provider Adapter pre-flight | Enclose safely | CE-SPEC-08 | INPUT_ESCAPED | High |
| Delimiter Injection | Guest Input / Tool Data | Strict XML/JSON encoding of < > & ". | Serialization Layer | Encode characters | Security Auth | DELIMITER_ENCODED | Critical |
| Cross-Tenant Contam. | Phase 3 Context | Tenant boundary validation pre-compilation. | Tenant/Scope Validator | FAIL CLOSED | CE-SPEC-01 | TENANT_VIOLATION | Critical |
| Tool-Result Poisoning | Tool API Payloads | Explicitly separating metadata vs. untrusted text inside tool returns. | Serialization Layer | Fence untrusted tool text | Runtime | TOOL_POISON_MITIGATED | High |
| Encoding Attacks | Control Characters | Neutralize non-printable/Unicode homoglyphs before serialization. | Preflight validation | Strip control chars | Security Auth | ENCODING_SANITIZED | High |
| Component Substitution | Immutable Templates | Cryptographic hashes checked on template retrieval. | Template Resolver | FAIL CLOSED | Platform | TEMPLATE_HASH_FAIL | Critical |
| Version Downgrade | PE/CE versions | Compatibility matrix enforced pre-assembly. | Compatibility Validator | FAIL CLOSED | Platform | VERSION_MISMATCH | High |
15. FAILURE ARCHITECTURE
Every compilation failure MUST be handled deterministically.
| Failure ID | Condition | Detection Stage | Compiler Response | Retry Policy | Runtime Handoff | Audit Event | Severity |
|---|---|---|---|---|---|---|---|
| ERR_COMP_01 | Missing template component | Component Resolution | Abort compilation | No | CE-SPEC-07 / System Err | TEMPLATE_MISSING | Critical |
| ERR_COMP_02 | Version incompatibility | Compatibility Valid. | Abort compilation | No | CE-SPEC-07 | INCOMPATIBLE_VERSION | Critical |
| ERR_COMP_03 | Missing Tenant ID | Scope Validation | Abort compilation | No | CE-SPEC-07 | TENANT_NULL | Critical |
| ERR_COMP_04 | Tenant mismatch | Scope Validation | Abort compilation | No | CE-SPEC-12 (Sec) | TENANT_MISMATCH | Critical |
| ERR_COMP_05 | Invalid Phase 3 state | State Validation | Abort compilation | No | CE-SPEC-07 | INVALID_CE_STATE | High |
| ERR_COMP_06 | Escaping / Serialization fail | Canonical Serialization | Abort compilation | No | CE-SPEC-07 | SERIALIZATION_ERROR | Critical |
| ERR_COMP_07 | Unauthorized tool schema | Tool Compilation | Exclude / Abort | No | Security Runtime | UNAUTH_TOOL_SCHEMA | Critical |
| ERR_COMP_08 | Minimum prompt overflow | Token Budget Eval | Abort compilation | No | CE-SPEC-07 | TOKEN_HARD_OVERFLOW | High |
| ERR_COMP_09 | Unresolved Timezone | Component Resolution | Abort compilation | No | CE-SPEC-07 | NULL_TIMEZONE_FATAL | High |
| ERR_COMP_10 | Integrity hash mismatch | Final CIR Validation | Abort compilation | No | Security Runtime | INTEGRITY_VIOLATION | Critical |
16. OBSERVABILITY & AUDIT
Compilation telemetry MUST be deterministic, proving exactly what was compiled without logging raw PII.
Required Telemetry Output:
 * correlation_id
 * prompt_id, prompt_version, compiler_version, serialization_version
 * ce_spec_versions (Active dependencies)
 * tenant_reference (Masked/Hashed)
 * tool_schema_ids (List of tools successfully injected)
 * token_counts (Budget limit, consumed, priority truncated)
 * compilation_result (SUCCESS / FAILURE)
 * integrity_hash (SHA256 of the Canonical Prompt Envelope)
Strict Logging Prohibition:
Raw guest content, raw conversational history, and raw compiled prompts MUST NOT be recorded in default operational logs. Raw prompt capture MAY occur ONLY through explicitly authorized, access-controlled debugging/incident tooling with strict retention and redaction policies.
17. EDGE CASES
| Edge Case | Deterministic Handling |
|---|---|
| Missing Current Time/Timezone | If required by the active CE-SPEC, the compiler MUST NOT silently default to UTC, device time, or host time. Transitions to ERR_COMP_09 and FAILS CLOSED. |
| Empty User Message | If runtime logic defines an empty-input response, compiler executes it. Otherwise, compilation returns a deterministic error to the runtime rather than inventing conversational behavior. |
| Null vs. Empty Array | Explicitly serialized differently. A missing array triggers compilation failure (if required) or is omitted. An empty array serializes as [EMPTY_COLLECTION]. |
| Missing Optional Context | Evaluates to explicit [UNAVAILABLE]. The model is forced to process the explicit lack of data. |
| Extremely Large Tool Result | Subject to Priority 4 truncation. If the tool result causes a minimum-payload overflow, compilation fails closed. |
| Unsupported Provider Capability | If the adapter cannot map a required CIR directive (e.g., missing system role support), compilation FAILS CLOSED. |
18. PRODUCTION ACCEPTANCE CRITERIA
| AC ID | Category | Requirement | Verification Method | Expected Result | Pass/Fail | Severity |
|---|---|---|---|---|---|---|
| AC-01 | Semantic Parity | Compiling identical Phase 3 state payloads must yield exactly the same CIR semantic output. | Reproducibility Unit Test | 100% match on structural semantics. | Required | Critical |
| AC-02 | Determ. Compilation | The same inputs, tool schemas, and compiler version produce identical prompt envelope hashes. | Hash Comparison Test | Hashes match exactly. | Required | Critical |
| AC-03 | Escaping Security | Guest input containing <System> and & is context-safe encoded (e.g., &lt;System&gt;) before fencing. | Injection Compiler Test | Payload safely escaped; no syntactical breakout. | Required | Critical |
| AC-04 | Null Semantics | Missing optional fields render as [UNAVAILABLE]; missing required fields abort compilation. | Null Boundary Test | Correct explicit representation or abort. | Required | High |
| AC-05 | Tool Auth | Only tool schemas explicitly flagged as authorized by the runtime context are injected into the CIR. | Auth Payload Audit | Unauthorized tools entirely excluded. | Required | Critical |
| AC-06 | Tenant Isolation | Pre-flight validation aborts compilation immediately if state.venue_id differs from auth.venue_id. | Mismatch Simulation | Compilation aborted; ERR_COMP_04. | Required | Critical |
| AC-07 | Min Payload Overflow | Exceeding the token budget via Priority 1 (CE-SPEC State) triggers a compilation abort, not a truncation. | Token Budget Mock | Compilation aborted; ERR_COMP_08. | Required | Critical |
| AC-08 | Version Matrix | Compiling with a CE-SPEC-01 schema version not supported by the template matrix aborts compilation. | Matrix Mock Test | Compilation aborted; ERR_COMP_02. | Required | High |
| AC-09 | Timezone Invariant | Passing a payload requiring CurrentTime with a null venue_timezone aborts compilation. | Timezone Null Test | Compilation aborted; ERR_COMP_09. | Required | Critical |
| AC-10 | Provider Mapping | The Provider Adapter accurately maps the CIR system directives without altering logical constraints. | Adapter Output Test | Roles and fences map accurately to LLM API. | Required | High |
| AC-11 | Audit Telemetry | Operational logs record metadata and integrity_hash, but zero raw user prompt text. | Telemetry Inspection | No raw PII/Prompt strings present. | Required | Critical |
| AC-12 | Immutable Override | Dynamic input containing keys identical to immutable template keys fails to overwrite the template values. | Key Collision Test | Immutable content remains untouched. | Required | Critical |
19. VERSION HISTORY
| Version | Date | Description | Author | Approval Status |
|---|---|---|---|---|
| 1.0.0 | August 2026 | Initial Prompt Composition & Assembly Pipeline specification. | Ramy Bella | DRAFT / Implementation Specification |
| 1.1.0 | August 2026 | MAX-LEVEL Architectural Upgrade: Replaced generic serialization with the Canonical Intermediate Representation (CIR) and Provider Adapters. Enforced context-safe encoding (over regex), strict Token Governance priorities (0-8), explicit fail-closed null/timezone handling, immutable compilation invariants, and deterministic reproducibility criteria. | Ramy Bella | DRAFT / Implementation Specification |
20. FINAL NON-NEGOTIABLE PRINCIPLES
 * SERIALIZATION DOES NOT EQUAL TRUST: Structure guides attention; runtime validation guarantees security.
 * ZERO SEMANTIC MUTATION: The compiler formats data; it does not change, infer, or guess its meaning.
 * MISSING DATA FAILS CLOSED: If required structural variables or configurations (like timezones) are missing, compilation aborts.
 * CONTEXT-SAFE ESCAPING IS MANDATORY: All untrusted data undergoes strict encoding to prevent delimiter and control-character attacks.
 * AUTHORITATIVE STATE IS IMMUNE TO TRUNCATION: Token shedding applies only to lower-priority conversational history and context.
 * TOOL SCHEMAS ARE NOT TOOL AUTHORIZATION: The compiler only serializes schemas for tools the runtime explicitly authorizes.
 * REPRODUCIBILITY IS REQUIRED: The same inputs and versions must compile to the exact same canonical representation.
ARCHITECTURAL VERDICT
APPROVED
 * Critical corrections made: Introduced the Canonical Intermediate Representation (CIR) and Provider Adapters to decouple core compilation architecture from vendor-specific payload structures (e.g., OpenAI vs Anthropic). Replaced naive regex sanitization with strict, context-safe encoding rules (XML/JSON escaping).
 * Security improvements made: Formalized the separation between serialization (structure) and security (runtime validation). Differentiated between authoritative tool execution metadata and untrusted textual payloads returned inside tool results. Prohibited fallback to unsafe temporal anchors (UTC/Device time).
 * Determinism improvements made: Implemented strict explicit null semantics (missing \neq null \neq empty array). Established a mathematical Token Governance budget (Priorities 0-8) where truncation of required layers instantly forces a fail-closed state. Added the reproducibility invariant.
 * Runtime boundary improvements made: Clarified that PE-SPEC-04 compiles tool schemas, but the runtime controls tool authorization and invocation. Prevented the compiler from inventing fallback conversational behavior for empty messages.
 * Cross-document issues discovered: Reconciled caching references with performance realities; correctly subordinated caching to security and authorization priorities.
 * Remaining external dependencies: The exact configuration schemas of CE-SPEC Phase 3 states and the specific target API payload structures (e.g., OpenAI Chat Completion API definitions) must be provided by the runtime/integration layers during actual software implementation.
 * Implementation-ready: Yes. The specification now defines a deterministic compiler pipeline free of semantic mutation, tightly integrated with enterprise safety rules, and capable of generating objective, auditable artifacts for every LLM invocation.
