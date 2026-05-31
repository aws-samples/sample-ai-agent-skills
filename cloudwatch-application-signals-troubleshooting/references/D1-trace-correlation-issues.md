---
title: "D1 — Trace Correlation Issues"
description: "Diagnose CloudWatch Application Signals trace correlation failures"
status: active
severity: MEDIUM
triggers: ["trace not correlated", "missing traces", "trace ID not found", "X-Ray correlation"]
owner: devops-agent
objective: "Resolve trace correlation issues between Application Signals and X-Ray"
context: "Application Signals uses X-Ray trace IDs to correlate metrics, traces, and logs across services."
---

## Phase 1 — Triage

MUST:
- Check X-Ray traces: `aws xray get-trace-summaries --start-time $(date -d '-1 hour' -u +%Y-%m-%dT%H:%M:%S) --end-time $(date -u +%Y-%m-%dT%H:%M:%S)`
- Verify X-Ray tracing is enabled in the ADOT agent configuration
- Check that trace IDs are present in Application Signals metric dimensions

SHOULD:
- Get a specific trace: `aws xray batch-get-traces --trace-ids <trace-id>`
- Verify trace context propagation headers between services
- Check X-Ray sampling rules: `aws xray get-sampling-rules`

MAY:
- Review ADOT agent exporter configuration for X-Ray
- Check for trace ID format mismatches between services

## Phase 2 — Remediate

MUST:
- Enable X-Ray tracing in ADOT agent configuration
- Ensure trace context propagation is configured for all service communication
- Verify X-Ray daemon/agent is accessible from the application

SHOULD:
- Configure consistent sampling rules across all services
- Verify W3C Trace Context or X-Ray propagation format is used consistently

MAY:
- Increase sampling rate for critical service paths
- Add custom attributes to traces for better correlation

## Common Issues

- symptoms: "Application Signals shows metrics but clicking 'View traces' shows no results"
  diagnosis: "X-Ray tracing is not enabled or traces are not being exported."
  resolution: "Enable X-Ray exporter in ADOT agent. Verify traces appear in X-Ray console."

- symptoms: "Traces show gaps between services in the trace timeline"
  diagnosis: "Trace context is not propagated between services."
  resolution: "Ensure HTTP clients propagate trace context headers. Use OpenTelemetry context propagation."

## Safety Ratings

```
safety_ratings:
  - "xray get-trace-summaries: GREEN — read-only trace listing"
  - "xray batch-get-traces: GREEN — read-only trace retrieval"
  - "xray get-sampling-rules: GREEN — read-only sampling configuration"
  - "Sampling rule changes: YELLOW — affects trace collection volume"
```

## Escalation Conditions

- Trace correlation fails despite correct X-Ray and ADOT configuration
- Trace context propagation blocked by a proxy or middleware
- Sampling rules cause critical traces to be dropped
- Cross-account trace correlation required

## Data Sensitivity

- `aws xray get-trace-summaries`: MEDIUM — reveals service communication patterns
- `aws xray batch-get-traces`: MEDIUM — contains request details and timing
- X-Ray sampling rules: LOW — contains sampling configuration
- Trace context headers: LOW — contain trace and span IDs

## Prohibited Actions

- NEVER disable X-Ray tracing to troubleshoot Application Signals
- NEVER set sampling rate to 100% in production without understanding the cost impact
- NEVER modify trace context headers manually in application code
- NEVER delete X-Ray sampling rules without understanding their purpose

## Phase 3 — Rollback

1. Revert ADOT agent X-Ray exporter configuration if changes caused issues
2. Restore original X-Ray sampling rules: `aws xray update-sampling-rule --sampling-rule-update <original-rule>`
3. Verify traces resume flowing after configuration revert
4. Check Application Signals trace correlation is restored
5. Document the correlation issue and configuration attempted

## Output Format

```yaml
root_cause: "trace_correlation — <specific_cause>"
evidence:
  - type: trace_error
    content: "<correlation details>"
severity: MEDIUM
mitigation:
  immediate: "Fix X-Ray configuration or trace propagation"
  long_term: "Standardize tracing configuration across services"
```

## Escalation Conditions

escalation_conditions:
  - "Remediation requires modifying IAM policies in a production account"
  - "Remediation requires disabling a security control even temporarily"
  - "Root cause cannot be identified after 3 hypothesis pivots"
  - "Blast radius affects more than one account or region"
  - "Issue involves potential data loss or exposure"

## Data Sensitivity

data_sensitivity:
  - command: "describe-* commands"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"
  - command: "get-* commands"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"
  - command: "list-* commands"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"

## Prohibited Actions

prohibited_actions:
  - "NEVER suggest deleting alarms without review"
  - "NEVER reduce log retention below compliance requirements"
  - "NEVER disable detailed monitoring in production"
