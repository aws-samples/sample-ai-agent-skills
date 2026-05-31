---
title: "Z1 — General X-Ray Troubleshooting"
description: "Catch-all runbook for X-Ray issues not covered by specific runbooks"
status: active
severity: MEDIUM
triggers:
  - "X-Ray not working"
  - "tracing problem"
  - "X-Ray issue"
  - "something wrong with X-Ray"
owner: devops-agent
objective: "Systematically diagnose unspecified X-Ray issues"
context: "This catch-all runbook provides a systematic approach when the specific issue is unclear. It covers the most common X-Ray problems and guides toward the appropriate specific runbook."
---

## Phase 1 — Triage

MUST:
- Check service graph: `aws xray get-service-graph --start-time <start> --end-time <end>`
- Check trace summaries: `aws xray get-trace-summaries --start-time <start> --end-time <end>`
- Check sampling rules: `aws xray get-sampling-rules`
- Check encryption config: `aws xray get-encryption-config`

SHOULD:
- Check sampling statistics: `aws xray get-sampling-statistic-summaries`
- List groups and insights: `aws xray get-groups`
- Check active insights: `aws xray get-insight-summaries --start-time <start> --end-time <end> --states ACTIVE`
- Review CloudTrail for X-Ray events: `aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventSource,AttributeValue=xray.amazonaws.com --max-results 20`

MAY:
- Check IAM permissions for the affected service
- Verify daemon status for non-Lambda services
- Check application logs for X-Ray SDK errors

## Phase 2 — Remediate

MUST:
- Address the identified issue based on triage findings
- Redirect to the appropriate specific runbook

SHOULD:
- Redirect based on findings:
  - Missing traces → A1
  - Incomplete traces → A2
  - Sampling issues → A3
  - SDK problems → B1-B3
  - Integration issues → C1-C3
  - Service map gaps → D1
  - Group/filter issues → E1-E2
  - Insights issues → F1-F2
  - Permission issues → G1
  - Encryption issues → G2

MAY:
- Perform a full X-Ray configuration audit
- Document findings for future reference

## Common Issues

- symptoms: "General X-Ray not working"
  diagnosis: "Multiple possible causes — follow systematic triage."
  resolution: "Check service graph, traces, sampling, permissions in order."

- symptoms: "X-Ray was working but stopped"
  diagnosis: "Recent configuration change, permission change, or daemon issue."
  resolution: "Check CloudTrail for recent changes. Verify daemon and permissions."

- symptoms: "X-Ray works for some services but not others"
  diagnosis: "Per-service configuration issue."
  resolution: "Check each service's tracing config, IAM permissions, and SDK instrumentation."

## Output Format

```yaml
root_cause: "general_xray_issue — <specific_cause>"
evidence:
  - type: service_graph
    content: "<service graph summary>"
  - type: trace_count
    content: "<traces in time range>"
  - type: sampling_rules
    content: "<active sampling rules>"
  - type: encryption
    content: "<encryption configuration>"
severity: MEDIUM
mitigation:
  immediate: "Address the identified issue"
  long_term: "Implement comprehensive X-Ray monitoring"
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
