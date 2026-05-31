---
title: "D1 — Service Map Issues"
description: "Diagnose X-Ray service map gaps and display problems"
status: active
severity: MEDIUM
triggers:
  - "service map incomplete"
  - "service not showing in map"
  - "service map gaps"
  - "dependency map missing"
owner: devops-agent
objective: "Identify and resolve service map display issues"
context: "The service map is generated from trace data. Missing services indicate missing traces. The map defaults to 6 hours. Low sampling rates cause incomplete maps. Services must be instrumented to appear."
---

## Phase 1 — Triage

MUST:
- Get the service graph: `aws xray get-service-graph --start-time <start> --end-time <end>`
- Verify the time range covers the period when the missing service was active
- Check if traces exist for the missing service: `aws xray get-trace-summaries --start-time <start> --end-time <end> --filter-expression 'service("<service-name>")'`
- Verify the missing service has X-Ray instrumentation

SHOULD:
- Check sampling rules that may filter out the missing service's traces
- Verify the service is actually receiving traffic during the time range
- Check if the service name in the map matches expectations

MAY:
- Increase the time range to see if the service appears in a broader window
- Temporarily increase sampling rate for the missing service

## Phase 2 — Remediate

MUST:
- Instrument the missing service with X-Ray SDK
- Ensure traces are being generated for the service
- Adjust the time range to cover the relevant period

SHOULD:
- Increase sampling rates for services that should always appear in the map
- Verify trace ID propagation between all services in the chain

MAY:
- Create X-Ray groups to focus on specific service subsets
- Set up dashboards for service map monitoring

## Common Issues

- symptoms: "Service appears intermittently in the map"
  diagnosis: "Low sampling rate causes the service to not appear in some time windows."
  resolution: "Increase the sampling rate or reservoir size for the service."

- symptoms: "Service shows as 'unknown' in the map"
  diagnosis: "Service not properly named in X-Ray SDK configuration."
  resolution: "Set the service name in the X-Ray SDK segment configuration."

- symptoms: "Connection between services missing"
  diagnosis: "Trace ID not propagated between the services."
  resolution: "Ensure X-Amzn-Trace-Id header is forwarded between services."

## Output Format

```yaml
root_cause: "service_map_issue — <specific_cause>"
evidence:
  - type: service_graph
    content: "<services present in the graph>"
  - type: missing_service_traces
    content: "<trace search results for missing service>"
  - type: sampling_config
    content: "<sampling rules for the service>"
severity: MEDIUM
mitigation:
  immediate: "Fix instrumentation or sampling for missing services"
  long_term: "Ensure all services are instrumented and properly sampled"
```


## Safety Ratings

| Phase 2 Action | Rating | Rationale |
|---|---|---|
| Investigate and collect diagnostic data | GREEN | Read-only analysis; no state change |
| Modify configuration settings | YELLOW | Configuration change; reversible with backup |
| Restart or reboot components | YELLOW | Service interruption; recoverable |
| Delete or remove resources | RED | Potentially destructive; requires confirmation and backup |

## Escalation Conditions

- Production environment impact detected or imminent
- Multiple failed remediation attempts (>2 cycles without resolution)
- Security-related root cause identified (unauthorized access, data exposure)
- Issue requires cross-service or cross-account coordination
- Customer-facing service degradation exceeding SLA thresholds
- Regulatory or compliance implications discovered during investigation

## Data Sensitivity

- Trace data reveals application request flow and dependencies
- Segment metadata may contain request/response details
- Service maps expose complete application architecture
- Classification: CONFIDENTIAL - application architecture and request flow data

## Prohibited Actions

- NEVER modify sampling rules that could cause production trace data loss
- NEVER delete X-Ray groups used by production monitoring dashboards
- NEVER change encryption settings without coordinating with security team
- NEVER disable tracing on production services without approval

## Phase 3 -- Rollback

MUST:
- Document all changes made during Phase 2 remediation
- Verify rollback procedures exist before applying any fix
- Confirm service health after rollback using the same diagnostic steps from Phase 1
- Notify stakeholders if rollback is initiated in production

SHOULD:
- Maintain a timeline of all actions taken for post-incident review
- Capture before/after configuration snapshots
- Test rollback in non-production environment when possible
- Update runbook with lessons learned after incident resolution

MAY:
- Automate rollback procedures for frequently occurring issues
- Create CloudWatch alarms to detect regression after rollback

## Safety Ratings

safety_ratings:
  - "Phase 1 triage commands (describe/get/list): GREEN — read-only"
  - "Phase 2 configuration changes: YELLOW — state-changing but recoverable"
  - "Phase 2 resource deletion or security changes: RED — destructive or irreversible"

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
  - "NEVER suggest overly broad IAM permissions as a fix"
  - "NEVER suggest disabling security controls for troubleshooting"
  - "NEVER suggest deleting resources without confirmation"
