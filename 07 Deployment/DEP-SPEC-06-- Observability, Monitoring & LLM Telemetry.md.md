# DEP-SPEC-06: Observability, Monitoring & LLM Telemetry

## 1. DOCUMENT CONTROL

| Attribute | Value |
|---|---|
| Document Title | 06 Observability, Monitoring & LLM Telemetry.md |
| Document ID | DEP-SPEC-06 |
| Version | 1.0.0 |
| Status | APPROVED FOR IMPLEMENTATION |
| Author | Ramy Bella |
| Classification | Confidential / Enterprise Proprietary |
| Target Audience | SREs, Platform Engineers, AI Observability Engineers, Cost/FinOps Analysts |
| Parent Document | DEP-SPEC-01 |
| Related Documents | DEP-SPEC-01, DEP-SPEC-03, DEP-SPEC-04, DEP-SPEC-05, DEP-SPEC-07, TEST-SPEC-10, TEST-SPEC-19 |
| System | Restaurant AI System |
| Phase | Phase 7 — Deployment & Production Operations |
| Lifecycle Folder | 07 Deployment & Infrastructure |
| Last Updated | August 2026 |

---

## 2. EXECUTIVE PURPOSE & SCOPE BOUNDARIES

`DEP-SPEC-03`'s circuit breaker, `DEP-SPEC-04`'s T+15/T+30 audits and PONR health check, and `DEP-SPEC-05`'s disaster detection all *consume* telemetry as an automated decision input. None of them owns what that telemetry actually is, how it's measured, or what proves it can be trusted. `DEP-SPEC-06` is that source of truth: the LLM-specific metrics, the distributed tracing backbone every other document's signal rides on, the SLO/error-budget engine, and the `TelemetryHealthReport@1.0.0` contract. If this document's numbers are wrong, every automated rollback and disaster declaration built on top of them is wrong too — silently, since none of those consumers re-derive the metric themselves.

**Owns:** LLM & API telemetry (TTFT, P50/P95/P99 latency, per-turn token consumption, per-tenant cost tracking); distributed tracing via OpenTelemetry; the SLO/SLA dashboard and alert-threshold definitions (grounded in `TEST-SPEC-19` benchmarks); multi-tenant cost accounting; the `TelemetryHealthReport@1.0.0` contract.

**Delegates:** incident handling, PagerDuty routing, escalation, and incident-command playbooks → `DEP-SPEC-07`; the *decision* to halt a rollout or cutover based on a telemetry alert → `DEP-SPEC-03` and `DEP-SPEC-04` (this document supplies the signal, it never makes the rollback decision itself); performance/load benchmark testing in CI/CD → `TEST-SPEC-19`; the underlying cloud infrastructure for the tracing/metrics backend → `DEP-SPEC-01`.

### Core Operational Invariants

- `EVERY GUEST-FACING REQUEST ⇒ MANDATORY END-TO-END TRACE ID PROPAGATION ACROSS ALL SPANS, NO EXCEPTION`
- `P95 LATENCY > 500 MS FOR MORE THAN 2% OF A ROLLING 15-MINUTE WINDOW ⇒ ERROR BUDGET FLAGGED CONSUMED`
- `5XX ERROR RATE ABOVE 0% ⇒ SLO BREACH SIGNAL EMITTED, NEVER SUPPRESSED, AVERAGED, OR SMOOTHED AWAY`
- `PII OR SECRET DETECTED IN AN EMITTED SPAN ⇒ SEV-0 SCRUBBING-BOUNDARY FAILURE, HARD INGESTION BLOCK FOR THE OFFENDING SOURCE`
- `TELEMETRYHEALTHREPORT = PROOF THAT NO SPANS WERE DROPPED AND ALL METRICS ARE VALID, NEVER AN ASSUMED-HEALTHY DEFAULT`

---

## 3. TELEMETRY & TRACING ARCHITECTURE

### 3.1 OpenTelemetry LLM Span Architecture

```text
[GUEST MESSAGE RECEIVED]
           │
           ▼ trace_id GENERATED SERVER-SIDE (never accepted from client input)
┌─────────────────────────────────────────────────────────────────┐
│ SPAN: prompt_compiler        (TEST-SPEC-16)                      │
│ SPAN: rag_vector_search      (TEST-SPEC-12)                      │
│ SPAN: llm_api_call           (model_id, prompt/completion tokens)│
│ SPAN: pos_adapter_dispatch                                       │
│                                                                    │
│ Every span above shares the same trace_id and carries the same  │
│ standardized attribute set (Section 3.1.1) before export.        │
└──────────────────────────────┬────────────────────────────────────┘
                               │
                               ▼
              [GUEST RESPONSE RETURNED — TRACE CLOSED]
```

#### 3.1.1 Standardized Span Attributes

| Attribute | Description |
|---|---|
| `model_id` | Identifier of the LLM model/version that served the span. |
| `prompt_tokens` | Token count of the compiled prompt for this span. |
| `completion_tokens` | Token count of the model's response for this span. |
| `total_cost_usd` | Computed cost of this span at the provider's current rate. |
| `rag_chunk_count` | Number of RAG context chunks injected for this span. |
| `tenant_id` | Owning tenant, for cost aggregation and isolation auditing. |

Every attribute value passes through the `TEST-SPEC-10` scrubbing ruleset **before** a span is serialized or exported — this document does not define its own scrubbing rules, it reuses TEST-SPEC-10's as the authoritative source, applied at the point of span creation rather than after the fact.

### 3.2 SLO / Error Budget Engine

SLO targets are grounded in `TEST-SPEC-19` benchmark baselines: P95 latency ≤ 500 ms, 5xx error rate = 0%. The error budget is a rate-over-window calculation, deliberately not a single-request check: if P95 latency exceeds 500 ms for more than 2% of requests within a rolling 15-minute window, the error budget for that window is flagged `CONSUMED`. A single slow outlier request inside an otherwise healthy window MUST NOT consume the budget — this mirrors the same "sustained signal, not one data point" principle `DEP-SPEC-03` applies to canary-stage advancement.

The 15-minute window and 2% threshold are fixed evaluation parameters owned exclusively by this document. No downstream consumer may substitute a different window or threshold to avoid a flag (Section 6, `ERR_DEP_06_05`).

### 3.3 Multi-Tenant Cost Accounting

Token cost is aggregated per `tenant_id` in near-real-time, not on a batched delay — the purpose is detecting an active anomaly (a tenant stuck in a conversational loop, or a crash causing retry storms) while it is still happening, not reconstructing it after the invoice arrives. Anomaly detection compares each tenant's current cost rate against that tenant's own rolling baseline rather than a single fixed dollar figure shared across all tenants, since a high-volume tenant's normal usage is another tenant's runaway loop.

### 3.4 PII & Secret Leak Prevention in Telemetry

This document applies the same defense-in-depth posture `TEST-SPEC-08` applies to prompt injection: a primary structural boundary, backed by a verification layer that assumes the boundary can fail.

1. **Primary boundary** — every span attribute is scrubbed against the `TEST-SPEC-10` ruleset at span-creation time, before serialization. In the normal case, PII and secrets never exist in span form at all.
2. **Verification layer** — a continuous, automated scanner inspects ingested spans for residual PII/secret patterns as a safety net. A detection at this layer does not mean "scrub now" — it means the primary boundary already failed, which is itself a SEV-0 event: the offending source is hard-blocked from further ingestion pending investigation, not warned and allowed to continue (Section 6, `ERR_DEP_06_02`).

---

## 4. CONTRACT SCHEMA (`TelemetryHealthReport@1.0.0`)

A `TelemetryHealthReport` is generated only once its reporting window has fully closed — unlike `DEP-SPEC-04`/`DEP-SPEC-05`'s lifecycle records, every field here is always a known, computable value at creation time, so none of them need to be conditionally optional. The `allOf`/`if`-`then` rules below instead enforce cross-field **value consistency** — proving the report's own numbers don't contradict each other or the invariants above.

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "title": "TelemetryHealthReport@1.0.0",
  "type": "object",
  "additionalProperties": false,
  "properties": {
    "report_id": { "type": "string", "minLength": 1 },
    "target_environment_id": { "type": "string", "minLength": 1 },
    "window_start": { "type": "string", "format": "date-time" },
    "window_end": { "type": "string", "format": "date-time" },
    "spans_expected": { "type": "integer", "minimum": 0 },
    "spans_received": { "type": "integer", "minimum": 0 },
    "spans_dropped": { "type": "integer", "minimum": 0 },
    "trace_continuity_verified": { "type": "boolean" },
    "p95_latency_ms": { "type": "number", "minimum": 0 },
    "error_rate_5xx_pct": { "type": "number", "minimum": 0, "maximum": 100 },
    "slo_breached": { "type": "boolean" },
    "error_budget_status": { "type": "string", "enum": ["AVAILABLE", "CONSUMED"] },
    "pii_leak_detected": { "type": "boolean" },
    "status": { "type": "string", "enum": ["HEALTHY", "DEGRADED", "PIPELINE_FAILURE"] }
  },
  "required": [
    "report_id",
    "target_environment_id",
    "window_start",
    "window_end",
    "spans_expected",
    "spans_received",
    "spans_dropped",
    "trace_continuity_verified",
    "p95_latency_ms",
    "error_rate_5xx_pct",
    "slo_breached",
    "error_budget_status",
    "pii_leak_detected",
    "status"
  ],
  "allOf": [
    {
      "if": {
        "properties": { "error_rate_5xx_pct": { "exclusiveMinimum": 0 } },
        "required": ["error_rate_5xx_pct"]
      },
      "then": {
        "properties": { "slo_breached": { "const": true } },
        "required": ["slo_breached"]
      }
    },
    {
      "if": {
        "properties": { "spans_dropped": { "exclusiveMinimum": 0 } },
        "required": ["spans_dropped"]
      },
      "then": {
        "properties": { "trace_continuity_verified": { "const": false } },
        "required": ["trace_continuity_verified"]
      }
    },
    {
      "if": {
        "properties": { "pii_leak_detected": { "const": true } },
        "required": ["pii_leak_detected"]
      },
      "then": {
        "properties": { "status": { "const": "PIPELINE_FAILURE" } },
        "required": ["status"]
      }
    }
  ]
}
```

### Contract Semantics

Any report with `error_rate_5xx_pct` above zero MUST carry `slo_breached = true` — a schema-level enforcement of Core Invariant 3, so "the error rate was nonzero but nobody was told" is not a state the contract can represent.

Any report with `spans_dropped` above zero MUST carry `trace_continuity_verified = false` — a report cannot claim an unbroken trace chain while simultaneously admitting spans were lost.

Any report with `pii_leak_detected = true` MUST carry `status = "PIPELINE_FAILURE"` — a detected leak can never coexist with a `HEALTHY` or merely `DEGRADED` status, consistent with Section 3.4 treating detection itself as a SEV-0 event.

---

## 5. AUTOMATED TELEMETRY TEST HARNESS

```python
# Representative Automated Trace Propagation & PII-Free Test
@pytest.mark.rtm(req_id="DEP-SPEC-06-OBS-001")
def test_trace_context_propagates_without_pii_leak():
    guest_message = "Boka bord for Johan, telefon 070-123 45 67, kl 19."

    result = request_pipeline.execute_guest_turn(
        message=guest_message,
        tenant_id="TENANT-4021"
    )

    spans = tracing_backend.get_spans(trace_id=result.trace_id)

    # All expected spans exist, share one trace_id, and none were dropped
    expected_span_names = {
        "prompt_compiler",
        "rag_vector_search",
        "llm_api_call",
        "pos_adapter_dispatch"
    }
    assert {span.name for span in spans} == expected_span_names
    assert all(span.trace_id == result.trace_id for span in spans)

    # The raw phone number must never appear in any span attribute or log
    raw_phone_number = "070-123 45 67"
    for span in spans:
        for attribute_value in span.attributes.values():
            assert raw_phone_number not in str(attribute_value)

    health_report = telemetry_engine.get_health_report(
        trace_id=result.trace_id
    )
    assert health_report.pii_leak_detected is False
    assert health_report.trace_continuity_verified is True


# Representative Automated Error Budget Engine Test
@pytest.mark.rtm(req_id="DEP-SPEC-06-OBS-002")
def test_error_budget_consumed_only_on_sustained_p95_breach():
    window_id = telemetry_engine.open_evaluation_window(
        target_environment_id="PROD-EU-01",
        duration_minutes=15
    )

    # A single slow outlier request inside an otherwise healthy window
    latency_feed.record_requests(
        window_id=window_id,
        total_requests=1000,
        over_threshold_count=5  # 0.5% — below the 2% budget-consuming rate
    )
    telemetry_engine.evaluate_error_budget(window_id=window_id)
    report = telemetry_engine.get_health_report_for_window(window_id)
    assert report.error_budget_status == "AVAILABLE"

    # Now the breach rate sustains above 2% for the same window
    latency_feed.record_requests(
        window_id=window_id,
        total_requests=1000,
        over_threshold_count=25  # 2.5% — above the budget-consuming rate
    )
    telemetry_engine.evaluate_error_budget(window_id=window_id)
    report = telemetry_engine.get_health_report_for_window(window_id)
    assert report.error_budget_status == "CONSUMED"
```

---

## 6. SECURITY & THREAT MODEL

| Threat | Attack Vector | Preventive Control | Severity |
|---|---|---|---|
| PII Leak via Unscrubbed Span Attribute | A guest's free-text note (e.g., a special request) containing a phone number or name is logged verbatim into a span. | Primary scrubbing boundary reuses the TEST-SPEC-10 ruleset at span-creation time; a verification-layer scanner treats any residual detection as SEV-0 and hard-blocks the source. | CRITICAL (SEV-0) |
| Trace ID Spoofing | A malicious or malfunctioning client supplies its own `trace_id`, enabling cross-tenant trace correlation or log injection. | `trace_id` is generated exclusively server-side at ingress; any client-supplied value is discarded, never trusted or propagated. | HIGH (SEV-1) |
| Cost Accounting Blind Spot | A runaway tenant loop consumes excessive tokens without tripping any alert because cost aggregation lags real-time. | Cost aggregation is streaming/near-real-time specifically so per-tenant anomaly detection can catch a loop while it is happening. | HIGH (SEV-1) |
| Error Budget Gaming | Sustained SLO breaches are smoothed by evaluating against a longer window than the owned 15-minute/2% parameters, avoiding a CONSUMED flag. | The evaluation window and threshold are fixed and owned exclusively by this document; no downstream consumer may substitute its own parameters. | MEDIUM (SEV-2) |

---

## 7. FAILURE ARCHITECTURE & ERROR TAXONOMY

| Error Code | Description | Canonical Class | Severity |
|---|---|---|---|
| ERR_DEP_06_01 | One or more spans in a trace were dropped before reaching the tracing backend. | DATA_INTEGRITY | CRITICAL (SEV-0) |
| ERR_DEP_06_02 | PII or a secret was detected in an emitted span after the primary scrubbing boundary. | SECURITY_VIOLATION | CRITICAL (SEV-0) |
| ERR_DEP_06_03 | A trace_id was accepted from client input rather than generated server-side. | SECURITY_VIOLATION | HIGH (SEV-1) |
| ERR_DEP_06_04 | TelemetryHealthReport failed schema or conditional-validation rules. | CONTRACT_MISMATCH | HIGH (SEV-1) |
| ERR_DEP_06_05 | Error budget evaluation used a window duration or threshold other than the fixed parameters owned by this document. | CONFIGURATION_DRIFT | MEDIUM (SEV-2) |

---

## 8. PRODUCTION ACCEPTANCE CRITERIA

| AC ID | Category | Requirement | Verification Method | Pass/Fail |
|---|---|---|---|---|
| AC-OBS-01 | Trace Completeness | 100% of guest-facing requests produce a complete trace with 0 dropped spans across the full chain. | Trace Continuity Audit | REQUIRED |
| AC-OBS-02 | PII/Secret-Free Telemetry | 0 instances of unscrubbed PII or secrets detected across 10,000+ sampled requests. | Telemetry Test Harness | REQUIRED |
| AC-OBS-03 | SLO Enforcement | 100% of windows with 5xx error rate > 0% produce an SLO breach signal; 0 suppressed or averaged-away breaches. | SLO Engine Audit | REQUIRED |
| AC-OBS-04 | Error Budget Accuracy | 100% of 15-minute windows with P95 > 500 ms for more than 2% of requests are flagged CONSUMED; 0 single-request false triggers. | Error Budget Test Suite | REQUIRED |
| AC-OBS-05 | Schema Compliance | 100% of TelemetryHealthReport instances validate against TelemetryHealthReport@1.0.0, including all conditional rules. | Schema Inspector | REQUIRED |
| AC-OBS-06 | Cost Accounting Latency | Per-tenant cost aggregation reflects usage within a bounded near-real-time delay sufficient to surface an active runaway loop before it requires DEP-SPEC-07 escalation. | Cost Pipeline Latency Audit | REQUIRED |

---

## 9. INTEGRATION AUTHORITY MATRIX

| Component | Owns | Consumes | Produces |
|---|---|---|---|
| DEP-SPEC-06 | LLM/API telemetry, distributed tracing, SLO/error-budget engine, multi-tenant cost accounting, TelemetryHealthReport | Guest request chain spans, TEST-SPEC-19 benchmark baselines, TEST-SPEC-10 scrubbing rules | TelemetryHealthReport records, SLO breach signals, error-budget status, cost anomaly signals |
| DEP-SPEC-07 | PagerDuty & incident command | SLO breach / error-budget / cost anomaly signals (this document) | On-call escalation |
| DEP-SPEC-03 | Canary staging & automatic rollback | Telemetry signals (this document) as one circuit-breaker input | Rollback decisions — this document supplies the signal, never the decision |
| DEP-SPEC-04 | Cutover sequencing, PONR health validation | Telemetry signals (this document) for T+15/T+30 audits and PONR health confirmation | Cutover/PONR go-forward decisions |
| DEP-SPEC-05 | Disaster recovery & HA | Telemetry signals (this document) for provider/region health detection | Disaster declarations |
| DEP-SPEC-01 | Environment tiers & cloud resources | — | Provisioned tracing/metrics backend infrastructure this document runs on |
| TEST-SPEC-19 | Concurrency, latency & backpressure soak tests | — | SLO baseline benchmarks this document's targets are derived from |
| TEST-SPEC-10 | PII scrubbing & privacy compliance rules | — | Scrubbing ruleset this document's primary span-scrubbing boundary reuses |

---

## 10. FINAL NON-NEGOTIABLE PRINCIPLES

EVERY GUEST-FACING REQUEST MUST PRODUCE A COMPLETE, UNBROKEN TRACE FROM FIRST MESSAGE TO FINAL RESPONSE, WITH ZERO DROPPED SPANS.

NO SPAN OR LOG MAY EVER CONTAIN UNSCRUBBED PII OR SECRETS; DETECTION OF EITHER IS ITSELF A SEV-0 SCRUBBING-BOUNDARY FAILURE.

ANY 5XX ERROR RATE ABOVE 0% MUST EMIT AN SLO BREACH SIGNAL; THE SIGNAL MUST NEVER BE SUPPRESSED, AVERAGED, OR SMOOTHED AWAY.

TRACE IDS MUST BE GENERATED EXCLUSIVELY SERVER-SIDE; A CLIENT-SUPPLIED TRACE ID MUST NEVER BE TRUSTED OR PROPAGATED.

A TELEMETRYHEALTHREPORT MUST PROVE PIPELINE COMPLETENESS AND METRIC VALIDITY; HEALTH MUST NEVER BE THE ASSUMED DEFAULT.

---

## 11. VERSION HISTORY & ARCHITECTURAL VERDICT

| Version | Date | Description | Author | Status |
|---|---|---|---|---|
| 1.0.0 | August 2026 | Initial specification of Observability, Monitoring & LLM Telemetry (DEP-SPEC-06). Establishes the OpenTelemetry LLM span architecture, the rate-over-window SLO/error-budget engine grounded in TEST-SPEC-19 baselines, near-real-time multi-tenant cost accounting, the two-layer PII/secret prevention model reusing TEST-SPEC-10's scrubbing ruleset, and the TelemetryHealthReport@1.0.0 contract. | Ramy Bella | APPROVED FOR IMPLEMENTATION |

**ARCHITECTURAL VERDICT:**
APPROVED FOR IMPLEMENTATION. DEP-SPEC-06 establishes the telemetry and tracing backbone every other Phase 7 document's automated decisions depend on: standardized OTel spans, a sustained-signal SLO/error-budget engine, near-real-time cost accounting, a defense-in-depth PII/secret prevention model, and the TelemetryHealthReport@1.0.0 contract proving pipeline health rather than assuming it. Ready for implementation.
