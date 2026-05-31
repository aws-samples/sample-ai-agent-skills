---
title: "F2 — Latency Analysis Issues"
description: "Diagnose X-Ray latency analysis and performance investigation problems"
status: active
severity: MEDIUM
triggers:
  - "latency analysis"
  - "slow traces"
  - "response time analysis"
  - "performance investigation"
owner: devops-agent
objective: "Use X-Ray to identify and analyze latency issues"
context: "X-Ray captures response times for each segment and subsegment. The service map shows latency distribution. Trace analysis can filter by response time. Latency issues may be in the service itself or in downstream calls."
---

## Phase 1 — Triage

MUST:
- Get service graph with latency data: `aws xray get-service-graph --start-time <start> --end-time <end>`
- Find slow traces: `aws xray get-trace-summaries --start-time <start> --end-time <end> --filter-expression 'responsetime > <threshold>'`
- Get full trace details for slow traces: `aws xray batch-get-traces --trace-ids <trace-id>`
- Identify which segment or subsegment contributes most to latency

SHOULD:
- Compare latency across different time periods
- Check if latency is in the service itself or downstream calls
- Look for patterns (specific endpoints, time of day, etc.)

MAY:
- Use X-Ray analytics for latency distribution analysis
- Check CloudWatch metrics for correlated resource utilization

## Phase 2 — Remediate

MUST:
- Identify the specific segment/subsegment causing latency
- Investigate the root cause (slow database query, external API, cold start, etc.)
- Address the specific latency bottleneck

SHOULD:
- Add custom subsegments around suspected slow operations
- Use annotations to tag traces with relevant context
- Set up latency-based sampling rules to capture more slow traces

MAY:
- Create X-Ray groups for latency monitoring
- Set up CloudWatch alarms based on X-Ray latency metrics

## Common Issues

- symptoms: "Overall latency high but individual segments look normal"
  diagnosis: "Latency in network transit or uninstrumented code between segments."
  resolution: "Add custom subsegments to identify uninstrumented latency sources."

- symptoms: "Latency spikes at specific times"
  diagnosis: "Cold starts, scaling events, or downstream service degradation."
  resolution: "Correlate with CloudWatch metrics. Check for Lambda cold starts or auto-scaling."

- symptoms: "Downstream service subsegment shows high latency"
  diagnosis: "Slow database query, external API timeout, or throttling."
  resolution: "Investigate the specific downstream service. Check for throttling or resource constraints."

## Output Format

```yaml
root_cause: "latency_issue — <specific_cause>"
evidence:
  - type: slow_traces
    content: "<slow trace IDs and response times>"
  - type: latency_breakdown
    content: "<segment/subsegment latency breakdown>"
  - type: service_graph
    content: "<service graph latency data>"
severity: MEDIUM
mitigation:
  immediate: "Address the specific latency bottleneck"
  long_term: "Set up latency monitoring and alerting with X-Ray groups"
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
