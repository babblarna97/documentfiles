PE-SPEC-08: Prompt Template Architecture
1. DOCUMENT CONTROL
| Attribute | Value |
|---|---|
| Document Title | 08 Prompt Templates.md |
| Document ID | PE-SPEC-08 |
| Version | 1.0.0 |
| Status | DRAFT / Implementation Specification |
| Author | Ramy Bella |
| Classification | Confidential / Enterprise Proprietary |
| Target Audience | AI Architects, Prompt Engineers, Backend Engineers, Platform Architects |
| Parent Document | PE-SPEC-01 |
| Related Documents | PE-SPEC-02, PE-SPEC-04, PE-SPEC-05, PE-SPEC-06, PE-SPEC-07, CE-SPEC-01 through CE-SPEC-12 |
| System | Restaurant AI System |
| Phase | Phase 4 — Prompt Engineering |
| Last Updated | August 2026 |
2. EXECUTIVE PURPOSE
The Prompt Template Architecture (PE-SPEC-08) defines the canonical structure, lifecycle, versioning, validation, composition, and integrity requirements for reusable prompt templates within the Restaurant AI System.
Where:
 * PE-SPEC-06 determines which prompt components are required.
 * PE-SPEC-07 resolves the runtime values assigned to declared slots.
 * PE-SPEC-05 resolves and classifies contextual information.
 * PE-SPEC-08 defines the immutable template structures from which those components are instantiated.
 * PE-SPEC-04 compiles the resulting logical representation into the final LLM payload.
PE-SPEC-08 therefore acts as the Template Definition Layer.
Its purpose is to ensure that prompt templates are:
 * deterministic,
 * immutable at runtime,
 * explicitly versioned,
 * structurally validated,
 * dependency-aware,
 * slot-safe,
 * tenant-safe,
 * provenance-aware,
 * resistant to instruction injection,
 * reproducible,
 * auditable,
 * and independently testable.
A prompt template MUST be treated as a versioned executable specification of instruction structure, not as an arbitrary string.
3. PURPOSE AND SCOPE
Scope: What PE-SPEC-08 Controls
 * Template definition and canonical structure.
 * Template metadata and identity.
 * Template versioning.
 * Template immutability.
 * Declared slots and mount references.
 * Template component types.
 * Static versus dynamic template boundaries.
 * Template dependency declarations.
 * Template compatibility requirements.
 * Template schema validation.
 * Template integrity and checksums.
 * Template lifecycle and activation status.
 * Template composition rules.
 * Template-level security metadata.
 * Template-level provenance declarations.
 * Template compatibility with PE-SPEC-06 and PE-SPEC-07.
 * Deterministic template retrieval.
Scope: What PE-SPEC-08 Explicitly Does NOT Control
 * Business state or business decisions — Owned by Phase 3.
 * Intent precedence — Owned by CE-SPEC-09 / Phase 3.
 * Dynamic component selection — Owned by PE-SPEC-06.
 * Runtime variable retrieval — Owned by PE-SPEC-07.
 * Context retrieval or trust classification — Owned by PE-SPEC-05.
 * Final serialization or escaping — Owned by PE-SPEC-04.
 * Token truncation — Owned by PE-SPEC-04.
 * LLM inference.
 * Tool execution.
 * Authorization decisions.
 * Booking decisions.
 * Safety decisions originating from Phase 3.
 * Runtime inference of missing values.
PE-SPEC-08 defines templates. It does not execute them.
4. ARCHITECTURAL POSITION
The Prompt Template Registry provides the authoritative source of immutable prompt component definitions.
[PHASE 3 / RUNTIME]
        |
        | Authoritative State
        v
[PE-SPEC-06: BLUEPRINT ASSEMBLY]
        |
        | Selects Template IDs / Components
        v
[PE-SPEC-08: TEMPLATE REGISTRY]
        |
        | Resolves immutable template definitions
        v
[PE-SPEC-06: ASSEMBLY BLUEPRINT]
        |
        +----------------------+
        |                      |
        v                      v
[PE-SPEC-05]              [PE-SPEC-07]
Context Resolution        Variable Hydration
        |                      |
        +----------+-----------+
                   |
                   v
          [PE-SPEC-04: COMPILER]
                   |
                   v
             [LLM API]

Boundary Rule:
PE-SPEC-06 may request:
 * template_id = Booking_Time_Request
 * version = 1.4.2
PE-SPEC-08 resolves that identifier to an immutable template definition.
PE-SPEC-07 subsequently resolves only the slots declared by that template/blueprint.
PE-SPEC-08 MUST NOT dynamically determine what business state means.
5. TEMPLATE MODEL
A Prompt Template is an immutable, versioned logical instruction component.
A template consists of:
 * Identity.
 * Version.
 * Metadata.
 * Instruction structure.
 * Declared slots.
 * Declared context mounts.
 * Dependencies.
 * Compatibility constraints.
 * Security classification.
 * Integrity metadata.
A template MUST NOT contain unresolved business decisions disguised as template logic.
6. CANONICAL TEMPLATE CONTRACT
Every registered template MUST conform to a canonical schema.
Example:
{
  "template_id": "booking_time_request",
  "template_version": "1.4.2",
  "status": "ACTIVE",
  "tenant_scope": "GLOBAL",
  "component_class": "INTENT_DIRECTIVE",
  "description": "Requests missing booking time from the guest.",
  "dependencies": [
    "global_identity",
    "booking_governance"
  ],
  "slots": [
    {
      "slot_id": "slot_booking_date_01",
      "variable_name": "target_booking_date",
      "expected_type": "Date",
      "required": true,
      "source": "CE-SPEC-01.active_state",
      "scope": "TRANSACTION",
      "venue_binding": "REQUIRED",
      "session_binding": "REQUIRED",
      "freshness_requirement": "ACTIVE_TURN",
      "formatting_rule": "ISO8601_DATE",
      "null_policy": "FAIL_CLOSED",
      "provenance_tracking": true,
      "sensitivity_classification": "NONE"
    }
  ],
  "context_mounts": [],
  "instruction_nodes": [
    {
      "node_type": "instruction",
      "instruction_id": "booking_time_request_instruction",
      "content": "Request the missing booking time from the guest.",
      "authority": "SYSTEM"
    }
  ],
  "output_contract": "Schema_Clarification",
  "security_classification": "SYSTEM_INSTRUCTION",
  "checksum": "sha256:d2a8b9c..."
}

The exact serialization format is implementation-specific. The logical contract is not.
7. TEMPLATE IDENTITY
Every template MUST have a globally unique immutable template_id.
Examples:
 * booking_time_request
 * allergy_disclosure_policy
 * global_identity
 * booking_confirmation
 * multi_intent_governance
 * emergency_safety_directive
Template IDs:
 * MUST be unique.
 * MUST NOT change after publication.
 * MUST NOT be reused for semantically different templates.
 * MUST NOT encode mutable business state.
 * MUST NOT depend on tenant-controlled user input.
If a template's semantic purpose changes materially, a new template identity SHOULD be created rather than silently changing the meaning of the existing identity.
8. TEMPLATE VERSIONING
Every template MUST have an explicit immutable version.
Recommended format: MAJOR.MINOR.PATCH
Example: booking_confirmation@2.1.0
 * MAJOR: A breaking semantic or structural change.
   * Examples: Required slot removed. Slot type changed. Output contract changed. Instruction semantics materially changed. Dependency contract changed.
 * MINOR: Backward-compatible functionality added.
   * Examples: Additional optional slot. Additional compatible instruction node. Additional metadata.
 * PATCH: Non-semantic correction.
   * Examples: Typographical correction. Formatting correction. Non-semantic metadata correction.
Once a template version is activated, its content MUST be immutable. A modified template MUST produce a new version.
9. TEMPLATE IMMUTABILITY
Active templates MUST be immutable. Runtime processes MUST NEVER mutate template definitions.
The following are prohibited:
 * Runtime instruction modification.
 * Runtime slot creation.
 * Runtime dependency modification.
 * Runtime precedence modification.
 * Runtime security classification changes.
 * Runtime output-contract modification.
 * Runtime text rewriting.
If a runtime transaction requires a different instruction structure, PE-SPEC-06 MUST select a different template/version.
10. TEMPLATE COMPONENT CLASSES
Templates SHOULD declare their component class. Canonical classes include:
 * 10.1 GLOBAL_IDENTITY: Defines stable system identity and foundational behavior.
 * 10.2 GLOBAL_GOVERNANCE: Defines global security, trust, and interaction boundaries.
 * 10.3 TOOL_GOVERNANCE: Defines tool-use constraints where applicable.
 * 10.4 INTENT_DIRECTIVE: Defines instructions for a specific active intent.
 * 10.5 MULTI_INTENT_GOVERNANCE: Defines how Phase 3-authorized multi-intent decisions are expressed to the model.
 * 10.6 EMERGENCY_DIRECTIVE: Represents explicitly selected emergency or safety directives.
 * 10.7 CONTEXT_INSTRUCTION: Defines how declared contextual mounts should be interpreted.
 * 10.8 OUTPUT_CONTRACT: Defines the logical output requirements associated with the active contract.
 * 10.9 CLARIFICATION_DIRECTIVE: Defines instructions for requesting missing information.
11. TEMPLATE CONTENT MODEL
Templates MUST be represented as logical nodes rather than relying exclusively on raw text.
A logical template MAY contain:
Template
 ├── Metadata
 ├── Dependencies
 ├── Instruction Nodes
 ├── Slot Declarations
 ├── Context Mount Declarations
 ├── Output Contract
 └── Security Metadata

Instruction nodes SHOULD support explicit semantic classification.
Example:
{
  "node_type": "instruction",
  "instruction_id": "booking_time_request_instruction",
  "content": "Request the missing booking time from the guest.",
  "authority": "SYSTEM"
}

This prevents arbitrary strings from being mistaken for structural instructions.
12. STATIC AND DYNAMIC TEMPLATE CONTENT
Templates MAY contain:
Static Content: Content that is immutable across runtime executions.
 * Example: You are the restaurant booking assistant.
Slot References: Declared runtime values.
 * Example: The requested booking time is: [slot:target_booking_time]
Context Mount References: Declared contextual sources.
 * Example: [mount:KB_BOOKING_POLICY]
Structural Directives: Logical instructions represented within the template schema.
Templates MUST NOT contain unresolved runtime business conditionals.
For example, this is prohibited:
<If booking_confirmed>
    Confirm the booking.
</If>

The selection of the appropriate component MUST already have occurred in PE-SPEC-06.
13. SLOT DECLARATION BOUNDARY
A template MAY declare slots. However:
PE-SPEC-08 defines the slot contract; PE-SPEC-07 resolves the slot value.
PE-SPEC-08 MUST NOT:
 * retrieve the value,
 * infer the value,
 * default the value,
 * validate current business meaning,
 * query unrelated runtime state.
Every slot declaration MUST be compatible with the PE-SPEC-07 slot contract. If a template declares an invalid slot contract, the template MUST NOT become ACTIVE.
14. CONTEXT MOUNT DECLARATION
Templates MAY declare context mounts intended for PE-SPEC-05.
Example:
{
  "mount_id": "KB_BOOKING_POLICY",
  "source_class": "POLICY_KB",
  "required": true,
  "trust_classification": "VERIFIED_POLICY"
}

PE-SPEC-08 defines what mount is expected.
PE-SPEC-05 determines:
 * how context is retrieved,
 * whether it is available,
 * how it is classified,
 * whether it is trustworthy,
 * and how it is minimized.
PE-SPEC-08 MUST NOT perform RAG retrieval.
15. TEMPLATE DEPENDENCIES
Templates MAY declare dependencies.
Example:
booking_confirmation
    ├── global_identity
    ├── global_governance
    └── booking_governance

Dependencies MUST be:
 * explicitly declared,
 * version-resolvable,
 * immutable,
 * tenant-compatible,
 * recursively resolvable.
PE-SPEC-06 is responsible for dependency resolution during assembly. PE-SPEC-08 is responsible for ensuring that dependency declarations themselves are valid. A template with an invalid dependency declaration MUST NOT be activated.
16. CYCLIC DEPENDENCY PROHIBITION
Template dependency cycles are prohibited.
Example:
 * Template A requires B
 * Template B requires A
The registry MUST reject such a template graph.
PE-SPEC-06 MUST additionally detect cycles during runtime resolution.
Defense MUST therefore exist at both:
 * Registry validation.
 * Runtime assembly.
A detected cycle MUST FAIL CLOSED.
17. TEMPLATE PRECEDENCE
Templates MUST NOT independently establish business precedence.
A template MAY carry a declared architectural class, but it MUST NOT claim authority over Phase 3 decisions.
For example: PRIMARY_INTENT, SECONDARY_INTENT, EMERGENCY_DIRECTIVE, GLOBAL_GOVERNANCE are classifications. They are not runtime decision mechanisms.
PE-SPEC-06 consumes Phase 3's authoritative precedence decision. PE-SPEC-08 merely defines the template associated with that decision.
18. OUTPUT CONTRACT COMPATIBILITY
Templates that define or require output contracts MUST explicitly declare the expected contract.
Example:
{
  "output_contract": "BookingConfirmationSchema"
}

A template MUST NOT dynamically switch output contracts. If two templates require incompatible contracts, PE-SPEC-06/Phase 3 MUST resolve the conflict. PE-SPEC-08 MUST NOT independently decide which contract wins.
19. SECURITY CLASSIFICATION
Every template MUST carry a security classification.
Examples: SYSTEM_INSTRUCTION, SECURITY_GOVERNANCE, INTENT_INSTRUCTION, DATA_INTERPRETATION, OUTPUT_CONTRACT.
Templates containing system-level instructions MUST be protected against unauthorized modification. Tenant-authored content MUST NOT automatically become a system-level template.
20. TENANT SCOPE
Templates MUST explicitly declare tenant scope.
Supported values SHOULD include:
 * GLOBAL
 * VENUE
 * TENANT_GROUP
A GLOBAL template may be used across authorized venues.
A VENUE template MUST contain an explicit venue binding.
A venue-specific template MUST NEVER be resolved for another venue.
Cross-tenant template resolution MUST FAIL CLOSED.
21. TEMPLATE INTEGRITY
Every active template SHOULD have a cryptographic integrity hash.
 * Recommended: SHA-256
 * The checksum SHOULD cover the canonical logical representation of: template identity, version, instruction nodes, slot definitions, context mounts, dependencies, output contract, and security metadata.
 * The checksum MUST be calculated from canonicalized content. Whitespace-only serialization differences MUST NOT produce semantically different hashes if canonicalization defines them as equivalent.
22. REGISTRY INTEGRITY
The Template Registry MUST provide:
 * immutable versions,
 * deterministic lookup,
 * checksum verification,
 * activation state,
 * tenant scope,
 * dependency metadata,
 * compatibility metadata,
 * audit history.
An active template MUST NOT be retrieved from an untrusted or unauthorized registry source.
If registry integrity cannot be established: FAIL CLOSED.
23. TEMPLATE ACTIVATION LIFECYCLE
Templates SHOULD follow a controlled lifecycle:
DRAFT \rightarrow VALIDATING \rightarrow VALIDATED \rightarrow APPROVED \rightarrow ACTIVE \rightarrow DEPRECATED \rightarrow RETIRED
Only ACTIVE versions may be selected for normal production assembly.
DRAFT, VALIDATING, and RETIRED templates MUST NOT be used by production runtime unless an explicitly authorized non-production execution mode exists.
24. TEMPLATE VALIDATION
Before activation, a template MUST pass deterministic validation.
Validation MUST include:
 * Schema validation.
 * Template ID validation.
 * Version validation.
 * Slot contract validation.
 * Dependency validation.
 * Dependency cycle validation.
 * Tenant scope validation.
 * Output-contract compatibility.
 * Security classification validation.
 * Integrity checksum generation.
 * PE-SPEC-06 compatibility.
 * PE-SPEC-07 slot compatibility.
 * PE-SPEC-04 compilation compatibility.
A template that fails any mandatory validation MUST NOT become ACTIVE.
25. PROMPT INJECTION BOUNDARY
Template content is trusted system configuration, but template values are not automatically trusted merely because they appear inside a template.
Templates MUST NOT provide a mechanism by which runtime data can become executable instruction structure.
For example, this is prohibited: {{guest_note}} being dynamically interpreted as a new instruction block.
The correct representation is: [slot:guest_note] where the value remains DATA.
PE-SPEC-07 hydrates the value. PE-SPEC-04 performs final structural escaping.
26. TEMPLATE ESCAPING BOUNDARY
PE-SPEC-08 MUST NOT perform final LLM serialization.
Template definitions remain logical structures until compilation.
PE-SPEC-04 owns: escaping, fencing, serialization, token accounting, truncation, and final JSON payload generation.
This prevents PE-SPEC-08 from duplicating compiler behavior.
27. DETERMINISTIC TEMPLATE RESOLUTION
Given: template_id, template_version, registry_state, and tenant_scope, the same request MUST always resolve to the same immutable template definition.
 * No runtime process may randomly select between compatible template versions.
 * No implicit "latest" resolution is permitted for production execution.
 * Production assembly MUST use explicit version pinning.
28. TEMPLATE COMPATIBILITY
A template MUST declare its compatibility requirements where applicable.
Example:
{
  "requires": {
    "pe_spec_06": ">=1.0.2",
    "pe_spec_07": ">=1.0.1",
    "pe_spec_04": ">=1.0.0"
  }
}

If an active runtime does not satisfy required compatibility constraints, the template MUST NOT be assembled. No automatic downgrade or upgrade may occur.
29. ATOMIC TEMPLATE PUBLICATION
Template publication MUST be atomic.
A template version MUST NOT become partially visible.
The registry MUST guarantee:
UNPUBLISHED \rightarrow VALIDATED \rightarrow ATOMIC ACTIVATE \rightarrow VISIBLE
Consumers MUST either receive:
 * the complete valid template, or
 * no template.
Partial template definitions MUST NEVER be resolvable.
30. TEMPLATE ROLLBACK
Rollback MUST be version-based. The system MUST NOT mutate an active template back into an earlier state.
Instead:
booking_confirmation@2.2.0 \rightarrow deactivate
booking_confirmation@2.1.0 \rightarrow activate
The rollback MUST be auditable and deterministic.
31. OBSERVABILITY & AUDIT
Every template resolution SHOULD produce an auditable event containing:
 * template_id
 * template_version
 * registry_version
 * checksum
 * tenant_scope
 * venue_id
 * activation_state
 * resolution_timestamp
 * dependency_versions
The audit record MUST NOT unnecessarily contain sensitive hydrated variable values. Template observability MUST distinguish: template identity, template content version, runtime hydration, and final compilation.
32. FAILURE ARCHITECTURE
PE-SPEC-08 MUST FAIL CLOSED when template integrity or structural validity cannot be established.
| Failure ID | Condition | System Response | Runtime Handoff | Severity |
|---|---|---|---|---|
| ERR_TPL_01 | Template not found | Abort resolution; FAIL CLOSED | System Error | Critical |
| ERR_TPL_02 | Requested version unavailable | Abort resolution; FAIL CLOSED | System Error | Critical |
| ERR_TPL_03 | Template schema invalid | Reject template | Deployment/Registry | Critical |
| ERR_TPL_04 | Dependency invalid | Reject template / FAIL CLOSED | Platform Alert | Critical |
| ERR_TPL_05 | Dependency cycle detected | Reject / FAIL CLOSED | Platform Alert | Critical |
| ERR_TPL_06 | Checksum mismatch | Abort resolution; FAIL CLOSED | Security Alert | Critical |
| ERR_TPL_07 | Unauthorized tenant scope | Abort resolution; FAIL CLOSED | Security Runtime | Critical |
| ERR_TPL_08 | Incompatible PE specification version | Abort resolution; FAIL CLOSED | System Error | High |
| ERR_TPL_09 | Invalid slot declaration | Reject template | PE-SPEC-07 / Registry | Critical |
| ERR_TPL_10 | Invalid output-contract declaration | Reject template | PE-SPEC-06 / Registry | High |
| ERR_TPL_11 | Inactive template requested for production | Abort resolution; FAIL CLOSED | System Error | High |
| ERR_TPL_12 | Unauthorized template mutation detected | Abort resolution; FAIL CLOSED | Security Alert | Critical |
33. SECURITY THREAT MODEL
| Threat | Attack Surface | Preventive Control | Response | Owner |
|---|---|---|---|---|
| Template Tampering | Registry | Cryptographic checksum verification | ERR_TPL_06 | SecOps |
| Version Confusion | Runtime resolution | Explicit version pinning | ERR_TPL_02 | Platform |
| Cross-Tenant Template Leakage | Registry lookup | Explicit tenant-scope validation | ERR_TPL_07 | Runtime Sec |
| Malicious Template Injection | Registry publication | Approval + schema validation | ERR_TPL_03 | Platform |
| Dependency Cycle | Template graph | Static + runtime cycle detection | ERR_TPL_05 | Platform |
| Slot Injection | Template definitions | Strict slot contract | ERR_TPL_09 | Prompt Eng |
| Output Contract Conflict | Template metadata | Contract validation | ERR_TPL_10 | Architecture |
| Unauthorized Mutation | Registry | Immutable versions + audit | ERR_TPL_12 | SecOps |
| Stale Template | Deployment | Explicit lifecycle/activation state | ERR_TPL_11 | Platform |
34. PRODUCTION ACCEPTANCE CRITERIA
| AC ID | Category | Requirement | Verification Method | Expected Result | Severity |
|---|---|---|---|---|---|
| AC-01 | Identity | Every production template has immutable template_id and explicit version. | Registry Audit | Unique versioned identity | Critical |
| AC-02 | Immutability | Active template content cannot be modified in place. | Mutation Test | Mutation rejected | Critical |
| AC-03 | Schema | Invalid template structure cannot become ACTIVE. | Schema Test | Activation rejected | Critical |
| AC-04 | Slot Contract | Invalid slot definitions are rejected before activation. | Slot Validation Test | ERR_TPL_09 | Critical |
| AC-05 | Dependency | Cyclic dependencies are detected. | Cycle Test | ERR_TPL_05 | Critical |
| AC-06 | Integrity | Checksum mismatch prevents template resolution. | Tamper Test | ERR_TPL_06 | Critical |
| AC-07 | Tenant | Venue-specific templates cannot resolve cross-tenant. | Tenant Injection Test | ERR_TPL_07 | Critical |
| AC-08 | Version | Production never implicitly resolves "latest". | Version Resolution Test | Explicit version required | Critical |
| AC-09 | Comp. Boundary | PE-SPEC-08 does not serialize final LLM payloads. | Architecture Test | PE-04 owns serialization | High |
| AC-10 | Var. Boundary | Template cannot directly retrieve runtime variables. | Runtime Isolation Test | PE-07 owns hydration | Critical |
| AC-11 | Ctx. Boundary | Template cannot independently perform RAG/context retrieval. | Context Isolation Test | PE-05 owns retrieval | High |
| AC-12 | Bus. Boundary | Template cannot mutate Phase 3 state or precedence. | Semantic Isolation Test | State unchanged | Critical |
| AC-13 | Atomic Pub. | Partially published templates cannot be resolved. | Registry Transaction Test | Complete-or-absent | Critical |
| AC-14 | Reproducibility | Same template ID/version/registry state resolves identically. | Determinism Test | Identical hashes | Critical |
| AC-15 | Rollback | Rollback occurs through explicit version activation, not mutation. | Rollback Test | Auditable version switch | High |
35. VERSION HISTORY
| Version | Date | Description | Author | Approval Status |
|---|---|---|---|---|
| 1.0.0 | August 2026 | Initial Prompt Template Architecture. Defined immutable versioned templates, canonical template contracts, registry integrity, dependency declarations, slot/context boundaries, tenant scope, lifecycle management, and compilation separation. | Ramy Bella | DRAFT / Implementation Specification |
36. FINAL NON-NEGOTIABLE PRINCIPLES
 * PE-SPEC-08 DEFINES TEMPLATES; IT DOES NOT EXECUTE BUSINESS LOGIC.
 * EVERY PRODUCTION TEMPLATE MUST HAVE AN EXPLICIT IMMUTABLE VERSION.
 * ACTIVE TEMPLATE CONTENT MUST NEVER BE MUTATED IN PLACE.
 * PE-SPEC-06 DECIDES WHICH TEMPLATE COMPONENTS ARE REQUIRED.
 * PE-SPEC-08 PROVIDES THE IMMUTABLE TEMPLATE DEFINITION.
 * PE-SPEC-07 HYDRATES DECLARED TEMPLATE SLOTS.
 * PE-SPEC-05 RESOLVES DECLARED CONTEXT MOUNTS.
 * PE-SPEC-04 COMPILES THE FINAL PAYLOAD.
 * TEMPLATES MUST NEVER RECEIVE OR EXECUTE UNRESOLVED BUSINESS LOGIC.
 * TEMPLATES MUST NEVER INVENT RUNTIME VALUES.
 * TEMPLATES MUST NEVER SELECT ALTERNATIVE VARIABLE SOURCES.
 * TEMPLATES MUST NEVER OVERRIDE PHASE 3 PRECEDENCE.
 * TEMPLATE DEPENDENCIES MUST BE EXPLICIT AND ACYCLIC.
 * CHECKSUM FAILURE MUST FAIL CLOSED.
 * CROSS-TENANT TEMPLATE RESOLUTION MUST FAIL CLOSED.
 * PRODUCTION TEMPLATE VERSION RESOLUTION MUST NEVER DEFAULT TO "LATEST".
 * TEMPLATE VALUES ARE DATA; THEY MUST NEVER BECOME EXECUTABLE INSTRUCTION STRUCTURE.
 * TEMPLATE PUBLICATION MUST BE ATOMIC.
 * ROLLBACK MUST BE VERSION-BASED, NEVER MUTATION-BASED.
 * DETERMINISM AND REPRODUCIBILITY ARE REQUIRED.
ARCHITECTURAL VERDICT
APPROVED — DRAFT / IMPLEMENTATION SPECIFICATION
