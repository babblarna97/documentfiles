PE-SPEC-06: Dynamic Prompt Assembly Architecture
1. DOCUMENT CONTROL
| Attribute | Value |
|---|---|
| Document Title | 06 Dynamic Prompt Assembly.md |
| Document ID | PE-SPEC-06 |
| Version | 1.0.2 |
| Status | DRAFT / Implementation Specification |
| Author | Ramy Bella |
| Classification | Confidential / Enterprise Proprietary |
| Target Audience | AI Architects, Prompt Engineers, Backend Engineers, Platform Architects |
| Parent Document | PE-SPEC-01 |
| Related Documents | PE-SPEC-02, PE-SPEC-04, PE-SPEC-05, PE-SPEC-07, CE-SPEC-01 through CE-SPEC-12 |
| System | Restaurant AI System |
| Phase | Phase 4 — Prompt Engineering |
| Last Updated | August 2026 |
2. EXECUTIVE PURPOSE
The Dynamic Prompt Assembly Architecture (PE-SPEC-06) establishes the deterministic rules engine for selecting, ordering, and orchestrating prompt components turn-by-turn.
Where PE-SPEC-04 is the Compiler (handling string serialization, escaping, and token limits) and PE-SPEC-05 is the Information Supply Chain (handling context retrieval and trust), PE-SPEC-06 is the Blueprint Orchestrator. It evaluates the active authoritative state provided by Phase 3 (Conversation Engine) and deterministically maps it to the exact set of instruction blocks, conditional directives, and output contracts required for the current transaction.
This specification ensures that the LLM is never burdened with evaluating conditional business logic, guaranteeing that the model only receives the exact, flat instructions relevant to the active context.
3. PURPOSE AND SCOPE
Scope: What PE-SPEC-06 Controls
 * Component Resolution: Selecting which immutable templates, dynamic blocks, and output schemas to include based on Phase 3 state.
 * Assembly Blueprinting: Generating the logical graph (Abstract Syntax Tree) of the prompt before passing it to the compiler.
 * Conditional Sections: Resolving intent-dependent and state-dependent prompt inclusions at runtime. The LLM must never receive unresolved <If>/<Then> logic.
 * Component Precedence: Resolving structural conflicts when assembling multiple dynamic components.
 * Dependency Validation: Deterministically detecting cycles, detecting missing dependencies, preserving dependency ordering, and failing closed if unresolvable.
 * Cache-Aware Ordering: Structuring the logical blueprint to optimize LLM Prefix Caching efficiency, strictly subordinated to semantic correctness.
Scope: What PE-SPEC-06 Explicitly Does NOT Control
 * Compilation & Serialization: Escaping, token truncation, and final LLM payload formatting are strictly owned by PE-SPEC-04.
 * Context Filtering & Trust: Trust classification and RAG data minimization are strictly owned by PE-SPEC-05.
 * Slot/Variable Hydration: The resolution of specific variables into template slots is owned by PE-SPEC-07.
 * Business Logic: Phase 3 dictates the state; PE-SPEC-06 maps state to prompt components. It DOES NOT calculate, execute, infer, mutate, authorize, or route business state.
4. ARCHITECTURAL POSITION
The Prompt Assembly Orchestrator sits at the nexus of Phase 3 state and Phase 4 compilation.
[PHASE 3 / RUNTIME] -> Emits Active State (e.g., Intents, Booking Status, Auth, Precedence)
       ↓
[PE-SPEC-06: DYNAMIC ASSEMBLY] -> Maps State to Prompt Blocks, evaluates conditionals, resolves dependencies, and creates the Logical Assembly Blueprint declaring required context/variable mounts.
       ↓
[PE-SPEC-05 / PE-SPEC-07] -> Resolves classified context (05) and slots/variables (07) based purely on the Blueprint's declared mounts. They MUST NOT mutate PE-SPEC-06 directives, state, precedence, or business semantics.
       ↓
[PE-SPEC-04: COMPILER] -> Safely encodes, truncates, and serializes the Blueprint into the CIR.
       ↓
[LLM INFERENCE API] -> Only executes the final resolved instruction set.

Constraint: The LLM MUST NEVER perform conditional inclusion. All conditional logic MUST evaluate during PE-SPEC-06 assembly, delivering a flat, resolved directive to the model.
5. THE ASSEMBLY BLUEPRINT MODEL
Prompt assembly operates on a strictly modular architecture. PE-SPEC-06 constructs a logical AssemblyBlueprint consisting of categorized components.
 * Mandatory Components (Defined by the active prompt profile):
   * e.g., Global_Identity (Persona, transparency).
   * e.g., Global_Governance (Untrusted data handling, refusal boundaries).
 * Intent-Specific Directives (State-Triggered):
   * e.g., Booking_Modification_Rules (Triggered by CE-SPEC-02).
   * e.g., Allergy_Evaluation_Rules (Triggered by CE-SPEC-03).
 * Context Placeholders (State-Triggered):
   * Explicit nodes declaring where PE-SPEC-05 must attach data (e.g., [Mount: KB_Policies]).
 * Output Contracts (State-Triggered):
   * e.g., JSON_Schema_Booking_Update vs Markdown_Conversational_Response.
6. RUNTIME-DRIVEN COMPOSITION
PE-SPEC-06 utilizes a deterministic mapping matrix to resolve Phase 3 state into prompt components.
6.1 State-to-Component Mapping
The assembly engine reads the ActiveFlows array from Phase 3 and selects the corresponding instruction blocks.
 * Example 1:
   * Phase 3 State: CE-SPEC-01 (Booking) = AWAITING_TIME
   * PE-SPEC-06 Action: Include Block_Booking_Time_Request. Set Output Contract to Schema_Clarification.
6.2 Multi-Intent Composition & Precedence
Multi-intent precedence MUST come exclusively from CE-SPEC-09 / Phase 3.
 * Constraint: PE-SPEC-06 MUST consume the CE-SPEC-09 decision. It MUST NOT independently calculate dominance, redefine routing priority, or evaluate business precedence. The assembly engine merely inserts a Multi_Intent_Governance block that explicitly instructs the model on how to address the intents according to Phase 3's authoritative hierarchy.
7. CONDITIONAL SECTIONS & DEPENDENCY RESOLUTION
Prompt templates contain declarative dependencies, resolved at runtime prior to compilation.
7.1 Server-Side Conditional Evaluation
Conditional logic is evaluated by the PE-SPEC-06 engine, NOT the LLM. The LLM MUST NEVER receive unresolved <If>/<Then> logic.
 * Template Syntax (Logical): <If state="auth.is_logged_in"> Include Block_Loyalty_Greeting </If>
 * Assembly Behavior: The engine evaluates the condition. If true, the block is appended. If false, the block is excluded entirely.
Explicit Rule: CONDITIONAL LOGIC ≠ BUSINESS LOGIC
Conditional evaluation in PE-SPEC-06 is limited strictly to prompt-component selection. It MUST NOT implement, infer, mutate, or replace Phase 3 business logic. PE-SPEC-06 MUST NOT:
 * calculate business state
 * modify Phase 3 state
 * infer authorization
 * make safety decisions
 * make transactional decisions
 * determine routing
 * determine booking outcomes
 * override CE-SPEC decisions
7.2 Dependency Trees
Prompt blocks may declare prerequisites.
 * Resolution Rule: PE-SPEC-06 deterministically walks the dependency tree, detects cycles, detects missing dependencies, and preserves dependency ordering.
 * Failure Rule: If a dependency cycle is detected, or if a required block is missing from the registry, assembly MUST FAIL CLOSED.
8. COMPONENT PRECEDENCE & CONFLICT RESOLUTION
When dynamically combining multiple components, conflicting instructions may arise. PE-SPEC-06 enforces a strict precedence hierarchy (derived from PE-SPEC-02 logic) during assembly.
Precedence Hierarchy (Highest to Lowest):
 * Global Identity & Safety Guardrails (Overrides all).
 * Emergency Directives (e.g., CE-SPEC-12 inclusions override standard flow instructions).
 * Active Primary Intent Directives (As defined by Phase 3).
 * Secondary Intent Directives (As defined by Phase 3).
Conflict Resolution Rule:
If the assembly engine detects that two intent blocks require mutually exclusive Output Contracts, PE-SPEC-06 MUST NOT independently resolve the conflict. CE-SPEC-09 / Phase 3 is authoritative for multi-intent precedence and dominant output-contract selection. PE-SPEC-06 MUST consume that authoritative decision. If Phase 3 has not provided a resolution for the conflict, assembly MUST FAIL CLOSED.
9. CACHE-AWARE ASSEMBLY ORDERING
Modern LLM inference APIs utilize Prefix Caching (KV Caching) to optimize costs and latency. PE-SPEC-06 SHOULD sequence the logical blueprint to maximize cache hit rates, but ONLY when it does not violate structural dependencies or architectural priorities.
Canonical Logical Order (for Blueprinting):
 * [STATIC] Immutable System Identity.
 * [STATIC] Global Governance & Security Rules.
 * [STATIC] Base Tool Definitions (Commonly used APIs).
 * [SEMI-STATIC] Venue Global Policies (Operating hours).
 * [DYNAMIC] Intent-Specific Instructions (Active CE-SPEC rules).
 * [DYNAMIC] Current Output Contract Schema.
 * [VOLATILE] Declared Mounts for Context/Variables.
Invariant: Cache-aware ordering is strictly an optimization. Semantic correctness, dependency ordering, safety/security precedence, CE-SPEC authority, and architectural boundaries ALWAYS take precedence over cache optimization. Note: PE-SPEC-06 defines the logical ordering blueprint; PE-SPEC-04 remains responsible for final serialization and encoding.
10. CONCURRENCY & SESSION ISOLATION
The Assembly Orchestrator operates in a high-concurrency runtime environment.
 * Stateless Execution: The assembly function MUST be entirely stateless. It MUST NOT rely on shared global memory.
 * Absolute Tenant Isolation: Tenant isolation MUST be explicit. Every resolved component MUST be validated against the active venue_id. Cross-tenant component resolution MUST FAIL CLOSED.
11. REPRODUCIBILITY & COMPONENT INTEGRITY
To guarantee auditability and safe rollbacks, dynamic assembly must be 100% reproducible.
 * Determinism Guarantee: Identical inputs + identical prompt version + identical registry state MUST produce an identical AssemblyBlueprint.
 * Version Pinning: A request to assemble a prompt MUST include the explicit prompt_version mapped to the current runtime release.
 * Component Checksums: PE-SPEC-06 SHOULD verify cryptographic hashes of requested prompt blocks against a known-good manifest before adding them to the Blueprint.
12. FAILURE ARCHITECTURE
Failure behavior within PE-SPEC-06 MUST be deterministic and FAIL CLOSED. There MUST be no implicit recovery, guessed defaults, hidden fallback blocks, or invented business behavior. All unmapped, invalid, ambiguous, or structurally inconsistent Phase 3 states MUST hand off to the appropriate Phase 3/runtime error handler.
| Failure ID | Condition | Detection Stage | System Response | Runtime Handoff | Severity |
|---|---|---|---|---|---|
| ERR_ASM_01 | Required prompt block missing from registry | Block Retrieval | Abort assembly; FAIL CLOSED | CE-SPEC-07 / System Err | Critical |
| ERR_ASM_02 | Dependency cycle detected | Tree Resolution | Abort assembly; FAIL CLOSED | Platform Alert | Critical |
| ERR_ASM_03 | Conflicting unresolvable output contracts | Schema Resolution | Abort assembly; FAIL CLOSED | CE-SPEC-07 / System Err | High |
| ERR_ASM_04 | Unmapped, invalid, or ambiguous Phase 3 State | State-to-Block Mapping | Abort assembly; FAIL CLOSED | CE-SPEC-08 / Runtime | Critical |
| ERR_ASM_05 | Component Hash Mismatch (Tampering) | Integrity Check | Abort assembly; FAIL CLOSED | SecOps / Alert | Critical |
| ERR_ASM_06 | Cross-tenant component detected | Scope Validation | Abort assembly; FAIL CLOSED | Security Runtime | Critical |
13. BOUNDARY CLARIFICATION
To ensure no architectural overlap, the implementation MUST adhere to these exact handoffs:
 * State Evaluation: Phase 3 decides what the system is doing.
 * Dynamic Assembly (PE-SPEC-06): Selects and orders the instruction blocks for what is happening. Evaluates If/Then logic on the server side to determine component selection only. Defines the required logical mounts and dependencies.
 * Context & Variable Resolution (PE-SPEC-05 / PE-SPEC-07): PE-SPEC-05 (Context) and PE-SPEC-07 (Variables) resolve their respective inputs based on the Blueprint's declared mounts. They MUST NOT mutate PE-SPEC-06 directives, state, precedence, or business semantics.
 * Compilation (PE-SPEC-04): Takes the final hydrated Blueprint, escapes strings, applies token truncation, and formats the JSON payload for the LLM.
14. PRODUCTION ACCEPTANCE CRITERIA
| AC ID | Category | Requirement | Verification Method | Expected Result | Pass/Fail | Severity |
|---|---|---|---|---|---|---|
| AC-01 | Conditional Logic | Template conditions (<If>) strictly map component inclusion and do not mutate or infer Phase 3 business state. The LLM receives NO unresolved <If> tags. | Blueprint Output Audit | Flat, resolved instructions; state unchanged. | Required | Critical |
| AC-02 | Cycle Detection | If a dependency cycle is introduced in the component registry (A requires B, B requires A), the assembly engine deterministically fails closed. | Dependency Cycle Mock | Compilation aborts; ERR_ASM_02. | Required | Critical |
| AC-03 | Cache Optimization | Cache-aware ordering prioritizes semantic dependencies; cache logic never overrides dependency correctness or architectural precedence. | Sequence Dependency Test | Semantics supersede cache ordering. | Required | High |
| AC-04 | Missing Block | If the runtime requests a state mapping for a prompt block that has been deleted or is missing, assembly fails closed. | Registry Null Mock | Compilation aborts; ERR_ASM_01. | Required | Critical |
| AC-05 | Dependency Order | If Block A requires Block B, invoking Block A automatically appends Block B to the Blueprint, strictly preserving declared dependency ordering. | Tree Resolution Test | Dependent blocks successfully resolved in order. | Required | High |
| AC-06 | Precedence Auth | PE-SPEC-06 relies exclusively on CE-SPEC-09's precedence signals to resolve output contract selection, never calculating intent dominance independently. | Dominance Handoff Mock | Blueprint obeys CE-SPEC-09 schema exactly. | Required | Critical |
| AC-07 | Tenant Isolation | Every resolved component is validated against the active venue_id. Any cross-tenant component resolution deterministically fails closed. | Cross-Tenant Inject Test | Compilation aborts; ERR_ASM_06. | Required | Critical |
| AC-08 | Reproducibility | Identical Phase 3 state + identical prompt version + identical registry state yields the exact same logical Blueprint hash. | Determinism Test | Hashes match perfectly. | Required | Critical |
| AC-09 | Unmapped State | If Phase 3 emits a state that lacks a valid PE-SPEC-06 mapping, assembly fails closed. The compiler MUST NOT invent fallback business behavior. | Unmapped State Mock | Compilation aborts; ERR_ASM_04. | Required | Critical |
| AC-10 | Semantic Boundary | PE-SPEC-05 and PE-SPEC-07 successfully resolve their declared Blueprint mounts without modifying the PE-SPEC-06 logical directives, precedence, or state intent. | Integration Semantic Test | Mounts resolved; directives unchanged. | Required | Critical |
15. VERSION HISTORY
| Version | Date | Description | Author | Approval Status |
|---|---|---|---|---|
| 1.0.0 | August 2026 | Initial Dynamic Prompt Assembly Architecture specification. Established state-to-component mapping, server-side conditional evaluation, and dependency resolution boundaries. | Ramy Bella | DRAFT / Implementation Specification |
| 1.0.1 | August 2026 | Targeted architectural hardening: clarified fail-closed handling for unmapped Phase 3 states, established CE-SPEC-09 authority for multi-intent precedence, restricted conditional evaluation strictly to prompt-component selection, clarified PE-SPEC-05/PE-SPEC-07 integration boundaries, and subordinated cache optimization to semantic correctness. | Ramy Bella | DRAFT / Implementation Specification |
| 1.0.2 | August 2026 | MAX-LEVEL architectural hardening: Enforced that the LLM MUST NEVER receive unresolved conditional logic, formalized component-level tenant validation, mandated cycle detection and deterministic dependency ordering, completely banned invented fallbacks for invalid states, and secured the explicit boundaries between Blueprint Assembly (PE-06), Context (PE-05), Variables (PE-07), Compilation (PE-04), and Phase 3 Business Logic. | AI Architect | DRAFT / Implementation Specification |
16. FINAL NON-NEGOTIABLE PRINCIPLES
 * PE-SPEC-06 ASSEMBLES THE PROMPT BLUEPRINT; IT DOES NOT EXECUTE BUSINESS LOGIC.
 * PHASE 3 DECIDES BUSINESS STATE; CE-SPEC-09 OWNS MULTI-INTENT PRECEDENCE.
 * PE-SPEC-05 CLASSIFIES CONTEXT; PE-SPEC-07 RESOLVES VARIABLES; PE-SPEC-04 COMPILES.
 * THE LLM ONLY EXECUTES THE FINAL RESOLVED INSTRUCTION SET; IT NEVER RECEIVES UNRESOLVED LOGIC.
 * UNMAPPED, INVALID, OR AMBIGUOUS STATE MUST FAIL CLOSED; NEVER INVENT FALLBACK BUSINESS BEHAVIOR.
 * CONDITIONAL PROMPT LOGIC MAY SELECT COMPONENTS ONLY; IT MUST NEVER CREATE OR MODIFY BUSINESS STATE.
 * DEPENDENCY RESOLUTION MUST DETECT CYCLES, PRESERVE ORDER, AND FAIL CLOSED IF UNRESOLVABLE.
 * TENANT ISOLATION IS ABSOLUTE; EVERY COMPONENT MUST BE VALIDATED.
 * CACHE OPTIMIZATION IS ALWAYS SECONDARY TO SEMANTIC CORRECTNESS AND SECURITY.
ARCHITECTURAL VERDICT
APPROVED
CHANGES APPLIED
 * Unresolved Logic Ban: Explicitly mandated that the LLM must never receive unresolved <If>/<Then> syntax (Sections 3, 4, 7.1).
 * Dependency Rigor: Added explicit requirements for cycle detection, missing dependency detection, and preservation of dependency ordering (Section 7.2, AC-02, AC-05).
 * Tenant Isolation: Upgraded tenant isolation to apply explicitly at the component level. Every resolved component must match the active venue_id or fail closed (Section 10, AC-07).
 * Reproducibility: Hardened the determinism guarantee to explicitly require identical inputs, versions, and registry state to produce an identical blueprint (Section 11, AC-08).
 * Fail-Closed Purity: Reworded ERR_ASM_04 to explicitly cover invalid, ambiguous, or structurally inconsistent Phase 3 states, strictly banning implicit recovery or invented fallbacks (Section 12, AC-09).
 * Boundary Enforcement: Re-clarified the precise integration boundary between Phase 3, PE-SPEC-06, PE-SPEC-05, PE-SPEC-07, and PE-SPEC-04 in Section 13 and the Final Principles.
 * Acceptance Criteria: Added explicit, testable criteria for cycle detection, tenant component isolation, and unresolved logic bans.
