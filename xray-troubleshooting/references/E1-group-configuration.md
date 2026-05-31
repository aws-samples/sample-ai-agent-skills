---
title: "E1 — Group Configuration Issues"
description: "Diagnose X-Ray group configuration problems"
status: active
severity: LOW
triggers:
  - "X-Ray group not working"
  - "group configuration error"
  - "cannot create group"
  - "group filter not matching"
owner: devops-agent
objective: "Identify and resolve X-Ray group configuration issues"
context: "X-Ray groups organize traces using filter expressions. Groups can have CloudWatch metrics and insights enabled. Group filter expressions must use valid X-Ray filter syntax. Groups are useful for monitoring specific services or workflows."
---

## Phase 1 — Triage

MUST:
- List existing groups: `aws xray get-groups`
- Check the group's filter expression: `aws xray get-group --group-name <group-name>`
- Verify the filter expression matches expected traces
- Test the filter expression directly: `aws xray get-trace-summaries --start-time <start> --end-time <end> --filter-expression '<expression>'`

SHOULD:
- Check if insights are enabled for the group
- Verify CloudWatch metrics are being generated for the group
- Check for filter expression syntax errors

MAY:
- Review group quotas (25 groups per account per region)
- Check CloudTrail for group modification events

## Phase 2 — Remediate

MUST:
- Fix the filter expression syntax
- Create or update the group: `aws xray create-group --group-name <name> --filter-expression '<expression>'`
- Verify traces match the group filter

SHOULD:
- Enable insights on the group for anomaly detection
- Enable CloudWatch metrics for the group

MAY:
- Create groups for each microservice or workflow
- Document group naming conventions

## Common Issues

- symptoms: "Group shows no traces"
  diagnosis: "Filter expression doesn't match any traces."
  resolution: "Test the filter expression with get-trace-summaries. Fix syntax or criteria."

- symptoms: "Cannot create group — quota exceeded"
  diagnosis: "Maximum 25 groups per account per region."
  resolution: "Delete unused groups or consolidate filter expressions."

- symptoms: "Group insights not generating"
  diagnosis: "Insights not enabled on the group, or insufficient trace volume."
  resolution: "Enable insights: aws xray update-group --group-name <name> --insights-configuration InsightsEnabled=true."

## Output Format

```yaml
root_cause: "group_configuration_issue — <specific_cause>"
evidence:
  - type: group_config
    content: "<group filter expression>"
  - type: matching_traces
    content: "<traces matching the filter>"
  - type: insights_config
    content: "<insights configuration>"
severity: LOW
mitigation:
  immediate: "Fix group filter expression"
  long_term: "Standardize group naming and filter conventions"
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
