Review
PE-SPEC-04: OpenAI Integration

1. DOCUMENT CONTROL
| Attribute | Value |
|---|---|
| Document Title | 04 OpenAI Integration.md |
| Document ID | PE-SPEC-04 |
| Version | 1.0.1 |
| Status | DRAFT / Implementation Specification |
| Author | Ramy Bella |
| Classification | Confidential / Enterprise Proprietary |
| Target Audience | Integration Architects, Backend Engineers, Platform Engineers, OpenAI Integration Engineers, Security Architects, QA Architects, Reliability Engineers |
| Parent Document | PE-SPEC-03 |
| Related Documents | PE-SPEC-01, PE-SPEC-02, PE-SPEC-03, PE-SPEC-05 through PE-SPEC-20, Phase 4 Specifications |
| System | Restaurant AI System |
| Phase | Phase 5 — Integrations |
| Lifecycle Folder | PROVIDER INTEGRATIONS |
| Last Updated | August 2026 |


2. EXECUTIVE PURPOSE
The Restaurant AI System leverages Large Language Models to drive conversational interactions, intent extraction, and structural reasoning. PE-SPEC-04 defines the concrete architecture for integrating OpenAI as a designated LLM provider strictly through the canonical Phase 5 abstraction layer.
This specification enforces the mechanical boundary between abstract integration contracts and proprietary OpenAI APIs, SDKs, and data schemas. It details request construction, response normalization, model configuration, and structured-output adaptation, ensuring that the system can utilize OpenAI's capabilities without absorbing its architectural dependencies.
Core Architectural Invariant:
OPENAI IMPLEMENTATION \neq PROVIDER ABSTRACTION \neq BUSINESS AUTHORITY.
OpenAI MUST NOT become a special architectural authority merely because it is the first provider implementation. It remains a replaceable execution dependency bound entirely by PE-SPEC-03.


3. PURPOSE AND SCOPE
Scope: What PE-SPEC-04 Controls



OpenAI Provider Adapter Architecture: Implementation of the adapter bridging PE-SPEC-03 to OpenAI.

OpenAI Capability Mapping: Correlating abstract Phase 5 capabilities to OpenAI endpoints.

OpenAI Request Construction: Deterministically mapping canonical inputs to OpenAI payload structures.

OpenAI Response Normalization: Transforming OpenAI completions, structured outputs, and tool calls into PE-SPEC-02 contracts.

OpenAI-Specific Request/Response Schema Handling: Translating schema constraints into OpenAI's native syntax.

OpenAI Model Configuration References: Binding execution to explicit model IDs and temperatures.

OpenAI API Version Compatibility: Managing compatibility with explicit OpenAI API releases.

OpenAI SDK/HTTP Transport Isolation: Restricting OpenAI client libraries exclusively to this adapter.

OpenAI Provider Error Normalization: Translating native OpenAI exceptions to canonical IntegrationError objects.

OpenAI Provider Compatibility: Confirming compliance with PE-SPEC-02/03.

Adapter Testability & Failure States: Definition of explicit failure modes and adapter verification.
Scope: What PE-SPEC-04 Explicitly Does NOT Control

Business logic \rightarrow Phase 3

Canonical integration contracts \rightarrow PE-SPEC-02

Provider abstraction interfaces \rightarrow PE-SPEC-03

Security implementation \rightarrow PE-SPEC-10

Authentication/authorization \rightarrow PE-SPEC-11

Data transformation policy \rightarrow PE-SPEC-12

Tenant/environment enforcement \rightarrow PE-SPEC-13

Recovery & Idempotency orchestration \rightarrow PE-SPEC-14/15

Webhooks \rightarrow PE-SPEC-16

Rate limits/resilience \rightarrow PE-SPEC-17

Observability \rightarrow PE-SPEC-18

Certification \rightarrow PE-SPEC-19

Lifecycle/registry \rightarrow PE-SPEC-20

Prompt architecture & evaluation \rightarrow Phase 4


4. ARCHITECTURAL POSITION
The OpenAI adapter operates fully encapsulated within the Phase 5 abstraction hierarchy.
[PHASE 3 / BUSINESS AUTHORITY]
↓
[PHASE 4 / AUTHORIZED AI REQUEST]
↓
[PE-SPEC-02 / CANONICAL CONTRACT]
↓
[PE-SPEC-03 / PROVIDER ABSTRACTION]
↓
=====================================================
[PE-SPEC-04 / OPENAI ADAPTER]
↓
[OPENAI CLIENT / TRANSPORT] (e.g., openai-python/HTTP)
↓
[OPENAI API] (Explicitly pinned/governed endpoint configuration)
↓
[OPENAI RESPONSE]
↓
[PE-SPEC-04 RESPONSE VALIDATOR]
↓
[PE-SPEC-04 NORMALIZATION ENGINE]
=====================================================
↓
[PE-SPEC-03 NORMALIZED RESPONSE]
↓
[PE-SPEC-02 CANONICAL VALIDATION]
↓
[PHASE 3 / RUNTIME]



Non-Negotiable Isolation Constraints:

Core application code MUST NOT import OpenAI SDKs directly.

OpenAI-specific types (e.g., ChatCompletionMessage, FinishReason) MUST terminate inside PE-SPEC-04.

The LLM itself does NOT possess provider credentials.

OpenAI output DOES NOT automatically establish business truth.


5. OPENAI CAPABILITY MODEL
The adapter maps OpenAI's APIs to formally declared abstract capabilities.
Supported Capabilities (Subject to Registry Configuration):



RESPONSE_GENERATION (Standard conversational output)

STRUCTURED_OUTPUT_GENERATION (JSON-schema constrained output)

TOOL_PROPOSAL_GENERATION (Capabilities leveraging tools and tool_choice)

EMBEDDING_GENERATION (Vector creation, if required by architecture)

PROVIDER_HEALTH (API status verification)
Do NOT assume every OpenAI capability is enabled for production. Each capability MUST be explicitly registered.
Logical Registration Mapping Example:
{
"provider": "OPENAI",
"capability": "STRUCTURED_OUTPUT_GENERATION",
"adapter_version": "1.0.0",
"provider_api_version": "PINNED_PROVIDER_API_VERSION_REFERENCE",
"model_reference": "PINNED_MODEL_ID",
"request_contract": "LLM_REQUEST@1.0.0",
"response_contract": "LLM_RESPONSE@1.0.0",
"status": "ACTIVE"
}


6. OPENAI ADAPTER CONTRACT
The OpenAI adapter fulfills strict mapping and enforcement responsibilities.
The Adapter MUST:



Receive a canonical PE-SPEC-02 request.

Validate required provider-specific compatibility (e.g., confirming the requested canonical tool schema can be expressed in OpenAI's JSON Schema dialect).

Construct the OpenAI API request payload.

Apply approved provider configuration (Model ID, parameters).

Invoke OpenAI through the authorized transport layer.

Validate the raw OpenAI response against expected HTTP/SDK bounds.

Normalize the response into the canonical response contract.

Normalize provider errors into Phase 5 canonical integration errors.

Preserve correlation (correlation_id) and provenance metadata.

Return ONLY canonical data upward.
The Adapter MUST NOT:

Make business decisions.

Select business intent.

Invent prompt policies.

Expose provider credentials in logs or payloads.

Bypass Phase 4 (PE-SPEC-14) output contracts.

Bypass PE-SPEC-11 security or PE-SPEC-12 data boundaries.

Directly modify Phase 3 state.

Silently switch model/provider versions or fallback to undocumented model IDs.


7. MODEL CONFIGURATION BOUNDARY
To ensure determinism, OpenAI model identity and configuration MUST be explicit.
Explicit References Required:



model_id (e.g., PINNED_MODEL_ID, not generic or implicit identifiers).

Model configuration version (governing default temperature, top_p, etc.).

Provider API version (e.g., matching a pinned OpenAPI spec or SDK version).

Capability support flags (e.g., supports_strict_json: true).
Prohibitions:

Do NOT hardcode a "latest model" policy.

The adapter MUST explicitly fail if a configuration mismatch occurs.
Architectural Distinction:
MODEL ID \neq MODEL VERSION GOVERNANCE \neq PROVIDER API VERSION \neq APPLICATION PROMPT VERSION. PE-SPEC-20 owns production lifecycle and registry governance; PE-SPEC-04 merely executes the pinned configuration it is given.


8. REQUEST TRANSLATION
The adapter translates the generalized integration request into the exact shape required by OpenAI.
[CANONICAL REQUEST] → [OPENAI REQUEST MAPPER] → [OPENAI REQUEST]



Translation Rules:

System/Developer Instructions: Mapped to OpenAI {"role": "system"} or {"role": "developer"} arrays as dictated by the canonical request and specific OpenAI API version.

User Content: Mapped to OpenAI {"role": "user"}.

Conversation Context: Prior normalized turns mapped to alternating user/assistant / tool objects.

Tools: Canonical tool definitions mapped to OpenAI's tools array structure.

Generation Controls: Explicit mapping of canonical temperature or token limits to temperature, max_tokens, etc.

Metadata: correlation_id passed via custom headers or transport metadata where supported, but NEVER injected into the prompt text itself.
Provider-specific parameters MUST NOT leak into canonical contracts unless explicitly represented through a governed extension mechanism. No hidden defaults. No implicit latest. No arbitrary provider-specific options injected from user input.


9. RESPONSE NORMALIZATION
The adapter isolates proprietary output formats and normalizes them into strict canonical states.
[RAW OPENAI RESPONSE] → [SCHEMA VALIDATION] → [OPENAI RESULT EXTRACTION] → [CANONICAL RESPONSE]



Extraction & Mapping:
Provider-specific response indicators MUST be validated and deterministically normalized into the canonical PE-SPEC-02 result/error structures.
For example, the adapter MUST distinguish and correctly map implementation details such as the OpenAI finish_reason and response payload:

A successful generation indicator (e.g., stop) \rightarrow Successful generation.

A tool proposal indicator (e.g., tool_calls) \rightarrow Tool-call proposal state.

A context limit indicator (e.g., length) \rightarrow Incomplete response (Context/Token limit exceeded).

A safety filter indicator (e.g., content_filter) \rightarrow Provider refusal/safety response.

Provider HTTP Error \rightarrow Normalized provider error.
Invariant: Do NOT equate a syntactically successful OpenAI response (HTTP 200) with business success. The adapter normalizes the technical result. PE-SPEC-14 remains responsible for validating whether the generated content fulfills the required prompt output contract.


10. TOOL-CALL INTEGRATION BOUNDARY
OpenAI's function-calling mechanism is abstracted into the Phase 4 / Phase 5 lifecycle.
Execution Flow:



OpenAI generates a response containing a tool-call indicator (e.g., finish_reason: "tool_calls").

PE-SPEC-04 validates the proprietary tool call array and normalizes it into the canonical Phase 5 TOOL_PROPOSAL representation.

The normalized result is passed upward.

Phase 4 (PE-SPEC-13/14) evaluates the proposal against system prompt schemas.

Runtime performs authorization and executes the actual integration.
Rules:

OpenAI does NOT authorize execution. It only proposes.

Provider-specific tool-call formats (e.g., stringified JSON arguments inside an OpenAI function object) MUST be safely parsed and validated by the adapter, not leaked raw into Phase 3.

Fabricated or undeclared tools returned by OpenAI MUST be caught and normalized into contract mismatch errors, remaining rejected by the existing architecture.


11. STRUCTURED OUTPUT INTEGRATION
The adapter handles the proprietary mechanics of forcing OpenAI to return structured schemas.
Rules:



The canonical output contract remains owned by PE-SPEC-14.

PE-SPEC-04 translates the PE-SPEC-14 schema into OpenAI's supported representation (e.g., response_format: { type: "json_schema", ... }).

OpenAI's provider-specific schema syntax quirks MUST be handled transparently by the adapter's translation layer. OpenAI's dialect MUST NOT become the canonical contract.

If the selected pinned OpenAI model/API configuration does not support the requested structured-output capability, the adapter MUST produce a deterministic compatibility error (ERR_OPENAI_11).

The adapter MUST NOT silently downgrade to free-form output when a strict structured contract is mandatory.


12. AUTHENTICATION & CREDENTIAL BOUNDARY
PE-SPEC-04 observes strict credential hygiene.



API keys / Auth credentials NEVER enter prompts.

Credentials NEVER enter PE-SPEC-02 canonical payloads.

Credentials NEVER enter LLM outputs.

Credentials are injected entirely out-of-band by the secure integration transport initialized by PE-SPEC-11.

Production credential references MUST be explicit and environment-scoped.

Provider credentials MUST be isolated per tenant/environment policy where applicable.
Note: PE-SPEC-11 owns detailed authentication and authorization implementation.


13. DATA & PRIVACY BOUNDARY
OpenAI is an external data-processing boundary.
Strict Rules:



Only authorized canonical fields may be mapped to the OpenAI request.

No arbitrary conversation history; context must be explicitly passed down from PE-05.

No raw PCI data may be sent.

No system credentials may be sent.

Unnecessary PII/PHI must be excluded or tokenized before entering the adapter.

Tenant isolation MUST remain intact; requests must not bleed tenant data into a shared context window.

Provider payloads MUST be generated from canonical approved data only.
Do NOT invent legal/compliance claims here. PE-SPEC-04 enforces the technical data boundary. Retention/privacy configuration requirements are governed by PE-SPEC-12, lifecycle specifications, and the actual external vendor contract/configuration.


14. ERROR NORMALIZATION
PE-SPEC-04 maps native OpenAI exceptions to canonical Phase 5 error classes.
| OpenAI Native Condition | Canonical Integration Error Class |
|---|---|
| HTTP 401 Unauthorized | AUTHENTICATION |
| HTTP 403 Forbidden / Invalid Org | AUTHORIZATION |
| HTTP 400 Bad Request (Invalid Param) | VALIDATION |
| Model ID does not exist | CONFIGURATION |
| HTTP 429 Rate Limit Exceeded | RATE_LIMIT |
| HTTP 500/503 Internal Server Error | PROVIDER_UNAVAILABLE |
| Network timeout during request | TIMEOUT |
| Safety filter triggered | PROVIDER_REJECTED |
| Context/Token Limit Exceeded | VALIDATION |
| Invalid JSON in structured output | CONTRACT_MISMATCH |
| Hallucinated Tool Call | CONTRACT_MISMATCH |
Do NOT define retry policy here. PE-SPEC-14/15 own recovery behavior based on the mapped error class.


15. TIMEOUT / UNKNOWN EXECUTION STATE
For inference operations, PE-SPEC-04 must explicitly distinguish execution states:
REQUEST NOT SENT \neq REQUEST REJECTED \neq REQUEST TIMED OUT \neq REQUEST SUCCESSFULLY RECEIVED



If the adapter times out before completing the dispatch to OpenAI, it returns TRANSIENT.

If the adapter times out while waiting for the OpenAI completion, it returns TIMEOUT.

The adapter MUST NOT fabricate a completion status.

Where provider semantics allow safe determination of delivery/result, use authoritative transport response metadata.

Where the outcome remains uncertain (e.g., connection drop during a streaming response), the adapter MUST return the canonical UNKNOWN state to trigger safe upper-level error handling.


16. OPENAI PROVIDER HEALTH
The adapter exposes normalized provider health states compliant with PE-SPEC-03.



HEALTHY: OpenAI API reachable, latency nominal, authentication successful.

DEGRADED: Elevated HTTP 5xx rates, elevated 429s, or high latency.

UNAVAILABLE: OpenAI API unreachable or 100% failure rate.

UNKNOWN: Health unverified.
Constraint: Do not turn health state into business decisions, circuit-breaker policies, or business-state changes. PE-SPEC-17 owns resilience, throttling, and circuit-breaker behavior based on this health signal.


17. VERSION AND COMPATIBILITY
The adapter asserts deterministic compatibility constraints.
Tracked Metadata:



adapter_version

openai_sdk_version (if applicable/used under the hood)

openai_api_version (e.g., explicit headers or endpoint paths)

model_id

canonical_contract_version supported

capability supported
Rules:

All production references MUST be explicitly pinned.

NO implicit "latest".

Unsupported model/API combinations MUST fail deterministically (ERR_OPENAI_03).

Provider SDK changes MUST be tested before promotion.

PE-SPEC-20 owns lifecycle and registry authority.


18. SECURITY / THREAT MODEL
| Threat | Attack Surface | Preventive Control | Detection | Response | Owner | Severity |
|---|---|---|---|---|---|---|
| API Credential Leakage | Adapter Config | Injected out-of-band; zero logging | Secret Scanner | Fail Closed | SecOps | Critical |
| Prompt/Data Leakage | Output Payload | PE-12 constraints; Output validation | Payload Audit | Scrub/Block | Privacy Eng | Critical |
| Malicious Response | OpenAI Output | Validation of all outputs before Phase 3 | Schema Val | ERR_OPENAI_10 | Architecture | High |
| Model Spoofing | Resolution Layer | Explicit Model ID pinning; Integrity hashes | Config Check | ERR_OPENAI_03 | Platform | Critical |
| Provider Schema Drift | OpenAI API | Strict response schema validation in adapter | Contract Val | ERR_OPENAI_10 | QA Arch | High |
| Tool-Call Manipulation | OpenAI Output | Normalization limits output to approved schemas | Tool Val | ERR_OPENAI_12 | Arch/SecOps | High |
| Tenant Data Crossover | Context Assembly | Strict session/tenant binding | Scope Audit | ERR_OPENAI_15 | Privacy Eng | Critical |
| Supply-Chain Compromise | OpenAI SDK | Dependency pinning and CI/CD scanning | Dep. Scanners | Block Build | SecOps | Critical |
| Provider Outage | External Network | Timeout bounds; Health abstraction | Health Check | Map Error | SRE | High |
| Excessive Data Trans. | Context Input | Prompt bounding rules; Token limits | Data Validation | ERR_OPENAI_06 | Prompt Eng | High |


19. FAILURE ARCHITECTURE
OpenAI-specific integration failures mapping to canonical recovery states:
| Failure ID | Condition | Canonical Error Class | Severity |
|---|---|---|---|
| ERR_OPENAI_01 | Adapter unavailable or improperly initialized | CONFIGURATION | Critical |
| ERR_OPENAI_02 | Unsupported capability requested | CONTRACT_MISMATCH | Critical |
| ERR_OPENAI_03 | Model/configuration mismatch | CONFIGURATION | Critical |
| ERR_OPENAI_04 | Provider authentication failure | AUTHENTICATION | Critical |
| ERR_OPENAI_05 | Provider authorization failure | AUTHORIZATION | Critical |
| ERR_OPENAI_06 | Invalid provider request (Adapter logic bug) | VALIDATION | High |
| ERR_OPENAI_07 | Provider rate-limit response | RATE_LIMIT | Medium |
| ERR_OPENAI_08 | Provider timeout | TIMEOUT | Medium |
| ERR_OPENAI_09 | Provider unavailable | PROVIDER_UNAVAILABLE | High |
| ERR_OPENAI_10 | Malformed provider response | CONTRACT_MISMATCH | High |
| ERR_OPENAI_11 | Structured-output incompatibility | CONTRACT_MISMATCH | High |
| ERR_OPENAI_12 | Tool-call normalization failure | CONTRACT_MISMATCH | High |
| ERR_OPENAI_13 | Provider config integrity failure | CONFIGURATION | Critical |
| ERR_OPENAI_14 | Data-boundary violation during mapping | DATA_BOUNDARY | Critical |
| ERR_OPENAI_15 | Tenant/environment mismatch | TENANT_VIOLATION | Critical |
Note: The adapter maps to the canonical class. PE-SPEC-15 defines retry execution for that class.


20. PRODUCTION ACCEPTANCE CRITERIA
| AC ID | Category | Requirement | Verification Method | Expected Result | Pass/Fail | Severity |
|---|---|---|---|---|---|---|
| AC-OAI-01 | Isolation | The OpenAI adapter compiles and runs without leaking OpenAI-specific SDK imports into core domain code. | Codebase Audit | Zero SDK leakage | Required | Critical |
| AC-OAI-02 | Version Pin | The adapter strictly rejects any configuration requesting the latest model identifier. | Config Unit Test | ERR_OPENAI_03 | Required | Critical |
| AC-OAI-03 | Credentials | API keys are passed to the transport layer without being exposed in canonical request payloads. | Payload Inspection | Keys Absent | Required | Critical |
| AC-OAI-04 | Normalization | Provider-specific tool indicators are successfully translated into PE-SPEC-02 canonical proposals. | Translation Mock | Valid Canonical Output | Required | High |
| AC-OAI-05 | Struct Output | The adapter translates standard PE-SPEC-14 JSON schemas into OpenAI's native requirement without silent fallback. | Capability Test | Strict formatting applied | Required | High |
| AC-OAI-06 | Error Map | An OpenAI HTTP 429 response is successfully mapped to ERR_OPENAI_07 / RATE_LIMIT. | Fault Injection | Mapped correctly | Required | High |
| AC-OAI-07 | Timeout | A simulated infinite network hang results in a mapped TIMEOUT state without crashing the adapter. | Network Mock | Controlled Timeout | Required | High |
| AC-OAI-08 | Validation | A malformed completion response yields ERR_OPENAI_10 rather than passing corrupted data upward. | Malformed Mock | ERR_OPENAI_10 | Required | High |
| AC-OAI-09 | Health | Provider health checks reflect UNAVAILABLE when the OpenAI mock endpoint returns 503s. | Health Check Test | State = UNAVAILABLE | Required | Medium |
| AC-OAI-10 | Auth Handoff | The adapter prevents tool calls from acting as authorization; it merely returns the proposal. | Arch Verification | Handoff preserved | Required | Critical |
| AC-OAI-11 | Replaceability | Standard tests continue to pass when the OpenAI adapter is swapped for a mock PE-SPEC-03 adapter. | Adapter Swap Test | Tests Pass | Required | Critical |


21. INTEGRATION AUTHORITY MATRIX
| Component | Owns | Consumes | Produces | Must Not Override |
|---|---|---|---|---|
| Phase 3 | Business state & meaning | Normalized LLM Output | Business Actions | Adapter implementation |
| Phase 4 | Prompt content & schemas | Business Intent | Canonical Requests | Native API logic |
| PE-SPEC-01 | Master Architecture | System Design | Arch Rules | Specific Adapter Logic |
| PE-SPEC-02 | Canonical Contracts | System Intent | Validated Contracts | Provider Native Schemas |
| PE-SPEC-03 | Abstraction Layer | Canonical Contracts | Adapter Selection | Adapter Logic |
| PE-SPEC-04 | OpenAI Integration | Canonical Payload | Native OpenAI Sync | Canonical Validation Rules |
| OpenAI API | Remote Inference | Native OpenAI Payload | Native AI Response | System Business Truth |
| PE-SPEC-14 | Output Struct Validation | Normalized LLM Output | Validated Schemas | Native API Parsing |
| PE-SPEC-20 | Lifecycle / Registry | Provider Meta/Config | Promotion Actions | Adapter Execution |


22. INTEGRATION CONTRACTS / RELATIONSHIPS
PE-SPEC-04 implements the OpenAI-specific translation logic beneath the integration architecture.



PE-SPEC-01/02/03: Provide the boundaries, canonical contracts, and abstract interface that PE-SPEC-04 must satisfy.

PE-SPEC-05 through 09: Sibling specifications for other capabilities (Booking, Email, etc.); completely isolated from PE-SPEC-04.

PE-SPEC-10/11 (Security/Auth): Provide the runtime injection mechanisms for OpenAI API keys.

PE-SPEC-12 (Data Mapping): Governs the PII/PHI rules for the data fed into PE-SPEC-04.

PE-SPEC-13 (Isolation): Governs which OpenAI project/tenant config PE-SPEC-04 may load.

PE-SPEC-14/15 (Errors/Recovery): Consume the normalized errors PE-SPEC-04 produces to execute retries or fallback processing.

PE-SPEC-16 (Webhooks): Dictates async limits if OpenAI features like batch API webhooks are adopted.

PE-SPEC-17 (Resilience): Imposes the rate limit and circuit-breaker envelopes over PE-SPEC-04 network dispatch.

PE-SPEC-18 (Observability): Consumes the metadata and structural telemetry PE-SPEC-04 emits.

PE-SPEC-19/20 (Testing/Lifecycle): Govern the validation, certification, and version promotion of PE-SPEC-04.
Clarification: PE-SPEC-04 is specifically the OpenAI provider implementation and does not become the general LLM architecture.


23. FINAL NON-NEGOTIABLE PRINCIPLES



PHASE 3 REMAINS THE BUSINESS AUTHORITY.

PE-SPEC-02 DEFINES THE CANONICAL CONTRACT.

PE-SPEC-03 DEFINES THE PROVIDER ABSTRACTION.

PE-SPEC-04 IMPLEMENTS THE OPENAI PROVIDER ADAPTER.

CORE BUSINESS LOGIC MUST NOT DEPEND DIRECTLY ON OPENAI SDKS.

OPENAI CREDENTIALS MUST NEVER ENTER PROMPT OR BUSINESS PAYLOADS.

OPENAI-SPECIFIC SCHEMAS MUST NOT LEAK INTO CANONICAL CONTRACTS.

MODEL SELECTION MUST BE EXPLICIT AND VERSION-GOVERNED.

NO IMPLICIT "LATEST" MODEL OR API VERSION.

OPENAI RESPONSES MUST BE VALIDATED BEFORE NORMALIZATION.

OPENAI TOOL CALLS ARE PROPOSALS, NOT AUTHORIZATION.

PROVIDER RESPONSE SUCCESS MUST NOT BE CONFUSED WITH BUSINESS SUCCESS.

UNKNOWN EXECUTION STATE MUST NEVER BE FABRICATED AS SUCCESS OR FAILURE.

DATA SENT TO OPENAI MUST RESPECT PE-SPEC-12.

TENANT AND ENVIRONMENT BOUNDARIES MUST BE PRESERVED.

PE-SPEC-04 MUST NOT MODIFY PHASE 3 BUSINESS STATE.

PE-SPEC-04 MUST NOT BYPASS PE-SPEC-10 THROUGH PE-SPEC-20.

OPENAI MUST REMAIN REPLACEABLE THROUGH PE-SPEC-03.


24. VERSION HISTORY
| Version | Date | Description | Author | Approval Status |
|---|---|---|---|---|
| 1.0.0 | August 2026 | Initial OpenAI Integration specification. Established the concrete OpenAI provider adapter, capability mapping, canonical request/response translation, model/configuration boundaries, structured-output and tool-call integration, credential isolation, error normalization, provider health, and deterministic compatibility boundaries. | Ramy Bella | DRAFT / Implementation Specification |
| 1.0.1 | August 2026 | Targeted provider-compatibility precision pass: removed outdated concrete model/API-version examples, generalized endpoint/version references, and clarified that provider-specific response indicators are implementation details normalized into canonical contracts. | Ramy Bella | DRAFT / Implementation Specification |
ARCHITECTURAL VERDICT
APPROVED — DRAFT / IMPLEMENTATION SPECIFICATION
Implementation Readiness Statement:
PE-SPEC-04 provides the concrete OpenAI implementation required by the Phase 5 provider layer while preserving the PE-SPEC-02 canonical contract and PE-SPEC-03 provider abstraction boundaries. Security, authentication/authorization, data transformation, tenant isolation, resilience, observability, testing/certification, and lifecycle governance remain owned by their respective Phase 5 specifications. PE-SPEC-04 implements only the OpenAI-specific provider adapter and does not assume authority over those concerns.