# TEST-SPEC-10: PII Scrubbing & Privacy Compliance Tests

## 1. DOCUMENT CONTROL

| Attribute | Value |
|---|---|
| Document Title | 10 PII Scrubbing & Privacy Compliance Tests.md |
| Document ID | TEST-SPEC-10 |
| Version | 1.0.1 |
| Status | APPROVED FOR IMPLEMENTATION |
| Author | Ramy Bella |
| Classification | Confidential / Enterprise Proprietary |
| Target Audience | Privacy Officers, Security Architects, AI Engineers, QA Engineers, SREs |
| Parent Document | TEST-SPEC-01 |
| Related Documents | Phase 1-5 Specs, INT-SPEC-18, TEST-SPEC-01, TEST-SPEC-02, TEST-SPEC-08, TEST-SPEC-09 |
| System | Restaurant AI System |
| Phase | Phase 6 — Validation & Verification (V&V) |
| Lifecycle Folder | 03 Safety, Security & Isolation |
| Last Updated | August 2026 |

---

## 2. EXECUTIVE PURPOSE

Conversational AI processing guest interactions inevitably ingests Personally Identifiable Information (PII) and Special Category Data under GDPR (names, phone numbers, email addresses, credit card numbers, and dietary/health conditions). Allowing raw PII to leak into telemetry logs (`INT-SPEC-18`), third-party LLM prompt requests, vector database indexes, or unencrypted storage violates global privacy regulations (GDPR, CCPA, PCI-DSS) and exposes the system to severe legal liability.

`TEST-SPEC-10` defines the **PII Scrubbing & Privacy Compliance Test Architecture**. Executed as part of the Stage 2 Security Gate (`TEST-SPEC-02`), it establishes automated verification of real-time PII detection, redaction, anonymization, pseudonymization, transit/rest encryption, and GDPR "Right to be Forgotten" atomic erasure protocols. It guarantees that no raw PII ever persists in telemetry, logs, or unencrypted data stores.

### Core Testing Invariants:
* `UNSCRUBBED PII IN LOGS / TELEMETRY \implies SEV-0 CRITICAL COMPLIANCE VIOLATION`
* `RAW PII SENT TO EXTERNAL LLM PROVIDER \implies MANDATORY REDACTION FILTER INTERCEPTION`
* `GDPR ERASURE REQUEST \implies DURABLE, IDEMPOTENT, VERIFIED PURGE ACROSS DB, CACHE, AND VECTOR INDEXES WITH FAIL-CLOSED COMPLETION`
* `CREDIT CARD / PCI-DSS DATA DETECTED \implies IMMEDIATE HARD STRIP (NO STORAGE PERMITTED)`
* `PSEUDONYMIZED HASHES MUST BE SALT-PROTECTED AND NON-REVERSIBLE`

---

## 3. PII TAXONOMY & REDACTION RULES

The system enforces deterministic scrubbing rules across six distinct PII categories prior to telemetry logging or prompt storage.

[RAW USER INPUT / SYSTEM DATA]
"Hej, jag heter Anna Lindberg, mitt nr är 070-1234567, e-post anna@example.com, c/o Hotel Plaza"
                                       │
                                       ▼ [PII SCRUBBER & SANITIZER ENGINE]
┌─────────────────────────────────────────────────────────────────────────────────────────┐
│ 1. Regex & Named Entity Recognition (NER) Token Extraction                              │
│ 2. Category-Specific Redaction / Hashing Application                                   │
│ 3. Deterministic Anonymized Token Replacement                                           │
└──────────────────────────────────────┬──────────────────────────────────────────────────┘
                                       │
                                       ▼ [SANITIZED TEXT OUTPUT]
"Hej, jag heter [NAME_ANON_8f2a], mitt nr är [PHONE_ANON_9b4c], e-post [EMAIL_ANON_1d3e]..."


3.1. PII Category & Scrubbing Policy Matrix
Category ID
Data Type
Examples
Scrubbing / Redaction Policy
Verification Method
PII-CAT-01
Phone Numbers
+46701234567, 070-123 45 67
Anonymized token replacement ([PHONE_ANON_HASH]). Deterministic salted hash for session lookup.
Pattern Regex + NER Audit
PII-CAT-02
Full Names
"Johan Söderberg", "Maria Smith"
Anonymized token replacement ([NAME_PSEUDONYM]).
NER Model + Name Corpus Audit
PII-CAT-03
Email Addresses
user@domain.se
Anonymized token replacement ([EMAIL_ANON_HASH]).
Exact RFC-5322 Regex Audit
PII-CAT-04
Credit Card / PCI Data
16-digit PANs, CVV codes, Expiry
HARD STRIP ([PCI_REDACTED]). Zero hashing or retention permitted.
Luhn Algorithm + PCI Scanner
PII-CAT-05
Health / Dietary (GDPR Art. 9)
"Extrem celiaki", "Svår nötallergi"
Mapped strictly to canonical allergen enum (NUT_ALLERGY); raw health prose scrubbed.
Taxonomy Normalization Audit
PII-CAT-06
Street Address / Location
"Kungsgatan 12, lägenhet 4B"
Masked to city level ([ADDRESS_REDACTED]).
Location Entity Recognizer

4. AUTOMATED TELEMETRY & LOG SANITIZATION HARNESS
To verify compliance with INT-SPEC-18 (Telemetry & Audit Logs), the test suite intercepts all outgoing log spans, traces, and metric payloads to verify zero raw PII leakage.
# Representative Automated PII Scrubbing Test Harness
@pytest.mark.rtm(req_id="SEC-SPEC-10-PII-001")
def test_telemetry_log_span_pii_sanitization(telemetry_interceptor):
    # Act: Simulate user conversation turn containing multiple PII types
    raw_user_turn = "Jag heter Erik Karlsson, ring mig på 0739998877 eller mejla erik@test.se"
    conversation_engine.process_turn(user_input=raw_user_turn)

    # Intercept exported OTel (OpenTelemetry) spans and log buffers
    exported_spans = telemetry_interceptor.get_exported_spans()

    # Assert: Verify that NO raw PII exists anywhere in trace attributes or logs
    for span in exported_spans:
        span_blob = span.to_json()
        assert "Erik Karlsson" not in span_blob
        assert "0739998877" not in span_blob
        assert "erik@test.se" not in span_blob
        
        # Verify presence of sanitized tokens
        assert "[NAME_ANON_" in span_blob or "[PII_REDACTED]" in span_blob


5. ## 5. GDPR "RIGHT TO BE FORGOTTEN" DISTRIBUTED ERASURE & VERIFICATION HARNESS
When a guest requests data erasure (GDPR Art. 17), the system MUST execute a durable, idempotent, distributed erasure workflow across all required primary and derived storage tiers.

[GDPR ERASURE REQUEST: phone_number = "+46701234567"]
                          │
                          ▼ [DURABLE ERASURE WORKFLOW]
┌─────────────────────────────────────────────────────────────────┐
│ 1. Relational DB: Execute idempotent erasure/anonymization     │
│ 2. Vector DB: Delete all conversation embeddings associated   │
│    with the user identity                                     │
│ 3. Redis Cache: Flush active user session keys                │
│ 4. Verify deletion across all required storage tiers          │
│ 5. Audit Log: Append cryptographically signed Erasure         │
│    Certificate                                                 │
└────────────────────────────────┬────────────────────────────────┘
                                 │
                                 ▼
                     [ERASURE VERIFICATION AUDIT]
       Execute independent verification queries across all
       required storage tiers ⇒ 0 recoverable user records


### 5.1. Erasure SLA & Verification Rules

Erasure Execution: GDPR erasure MUST execute as a durable, idempotent distributed workflow covering all authoritative and derived storage tiers.

Completion Condition: Erasure is NOT considered complete until every required storage tier has positively acknowledged deletion or anonymization and independent verification confirms that no recoverable user PII remains.

Failure Behavior: If any required storage tier fails, times out, or cannot be independently verified, the erasure workflow MUST enter a fail-closed state and retry or escalate rather than report successful completion.

Performance Target: The system SHOULD complete normal erasure workflows within 1.0 second where operationally feasible, but compliance correctness MUST take precedence over the latency target.

Verification Probe: Immediately after completion, the test harness queries DBs, vector indexes, and Redis caches using the target user's identifiers and derived lookup keys. Any recoverable user record or raw PII constitutes a critical compliance breach (ERR_TEST_10_03).
6. SECURITY & THREAT MODEL
Threat
Attack Vector
Preventive Control
Severity
PII Leakage in Telemetry
Unhandled exception dumps raw user input stack trace into log aggregator.
Global exception handler sanitizes error context before writing to log output.
CRITICAL (SEV-0)
External LLM Provider Leak
Raw PII sent in prompt payload to third-party AI provider, violating GDPR transfer rules.
Outbound LLM proxy applies mandatory PII redaction filter prior to network transmission.
CRITICAL (SEV-0)
Salt Exposure / Hash Reversal
Attacker steals anonymized hashes and rainbow tables to reverse-engineer guest phone numbers.
Cryptographic HMAC-SHA256 with rotating tenant-specific KMS secret salt.
HIGH
Incomplete Erasure
User data purged from SQL DB but remains cached in vector embedding index.
Orchestrated multi-tier purge protocol covering relational, key-value, and vector storage.
HIGH

7. FAILURE ARCHITECTURE & ERROR TAXONOMY
Error Code
Description
Canonical Class
Severity
ERR_TEST_10_01
Raw PII string (Phone, Name, Email) detected in OpenTelemetry log span or trace.
DATA_LEAKAGE
CRITICAL (SEV-0)
ERR_TEST_10_02
PCI-DSS credit card number detected in application memory or unredacted log buffer.
SECURITY_VIOLATION
CRITICAL (SEV-0)
ERR_TEST_10_03
GDPR "Right to be Forgotten" atomic purge failed to delete user records from vector DB.
COMPLIANCE_VIOLATION
HIGH
ERR_TEST_10_04
Outbound prompt request to external LLM provider contained unredacted raw PII.
DATA_LEAKAGE
CRITICAL (SEV-0)
ERR_TEST_10_05
PII anonymization hash used static/missing cryptographic salt.
SECURITY_BOUNDARY
MEDIUM

8. PRODUCTION ACCEPTANCE CRITERIA
AC ID
Category
Requirement
Verification Method
Pass/Fail
AC-PII-01
Zero Log Leakage
0 raw PII strings detected# across 10,000 intercepted OpenTelemetry spans during test runs.
Telemetry Span Inspector
REQUIRED
AC-PII-02
Credit Card Strip
100\% of Luhn-valid credit card numbers are hard-stripped ([PCI_REDACTED]) with zero storage.
PCI Scanning Suite
REQUIRED
AC-PII-03
Prompt Redaction
100\% of outbound prompt payloads sent to external LLM APIs undergo verified PII redaction.
Outbound Proxy Audit
REQUIRED
AC-PII-04
Distributed Erasure Verification
GDPR erasure workflows complete only after durable deletion/anonymization across SQL, Redis, and Vector DB is independently verified; failed or unverified tiers MUST NOT produce a successful completion status.
Distributed Erasure & Verification Harness
REQUIRED
AC-PII-05
Salted Hashes
All pseudonymized PII tokens use verified HMAC-SHA256 encryption with active KMS salt.
Cryptographic Audit
REQUIRED

9. INTEGRATION AUTHORITY MATRIX
Component
Owns
Consumes
Produces
TEST-SPEC-10
PII Redaction Harness, Telemetry Log Inspectors, PCI Stripping Suite, GDPR Erasure Verification
Raw User Inputs, Telemetry Spans (INT-SPEC-18)
Compliance Pass/Fail Verdicts, SEV-0 Audit Signals
INT-SPEC-18
Authoritative Telemetry, OpenTelemetry Spans & Audit Logging Architecture
System Log Events
Immutable Audit Spans
TEST-SPEC-02
CI/CD Stage 2 Security Gate Enforcement
Compliance Signals (TEST-SPEC-10)
Signed Evidence Bundles / Build Blocks
Phase 1 (PRIVACY)
Authoritative GDPR & Data Governance Policy
User Data Erasure Requests
System Privacy Directives

10. FINAL NON-NEGOTIABLE PRINCIPLES
NO RAW PERSONALLY IDENTIFIABLE INFORMATION (PII) MAY EVER BE WRITTEN TO TELEMETRY, LOGS, OR TRACES.
CREDIT CARD NUMBERS (PCI-DSS) MUST BE HARD-STRIPPED IMMEDIATELY AT THE INGRESS BOUNDARY WITH ZERO PERSISTENCE.
OUTBOUND PROMPT PAYLOADS DISPATCHED TO EXTERNAL LLMS MUST UNDERGO MANDATORY PII REDACTION.
GDPR "RIGHT TO BE FORGOTTEN" REQUESTS MUST TRIGGER A DURABLE, IDEMPOTENT, VERIFIED ERASURE WORKFLOW ACROSS ALL REQUIRED STORAGE TIERS; FAILURE TO VERIFY ANY REQUIRED TIER MUST PREVENT SUCCESSFUL COMPLETION.
PSEUDONYMIZED PII TOKENS MUST USE HMAC-SHA256 WITH ROTATING KMS SALTS; STATIC PLAIN HASHING IS PROHIBITED.
11. VERSION HISTORY & ARCHITECTURAL VERDICT
Version
Date
Description
Author
Status
1.0.0
August 2026
Initial specification of PII Scrubbing & Privacy Compliance Tests (TEST-SPEC-10).
Ramy Bella
SUPERSEDED
1.0.1
August 2026
Review fix pass. Removed a stray UI-widget artifact (`<ElicitationsGroup>`/`<Elicitation>` tags) that had leaked into the end of the file from document generation tooling — the same defect class already fixed in TEST-SPEC-12. No change to scrubbing policies, PCI-DSS rules, GDPR erasure logic, or acceptance criteria.
Ramy Bella
APPROVED FOR IMPLEMENTATION

ARCHITECTURAL VERDICT:
APPROVED FOR IMPLEMENTATION TEST-SPEC-10 v1.0.1 establishes the PII scrubbing policies, OpenTelemetry log span inspection harness, PCI-DSS stripping rules, and atomic GDPR erasure verification for Phase 6. Ready for implementation.


