---
title: "Z1 — General OpsCenter Troubleshooting"
description: "Catch-all runbook for OpsCenter issues"
status: active
severity: MEDIUM
triggers: ["OpsCenter error", "OpsCenter not working"]
owner: devops-agent
objective: "Systematic investigation of unclassified OpsCenter issues"
context: "Use when the issue doesn't clearly match a specific runbook."
---

## Phase 1 — Triage

MUST:
- Verify OpsCenter is enabled: `aws ssm get-service-setting --setting-id /ssm/opsCenter`
- List recent OpsItems: `aws ssm describe-ops-items --ops-item-filters "Key=Status,Values=Open,Operator=Equal"`
- Check CloudTrail: `aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventSource,AttributeValue=ssm.amazonaws.com --max-results 20`

SHOULD:
- Check IAM permissions
- Verify EventBridge rules

MAY:
- Check AWS Health Dashboard

## Phase 2 — Remediate

MUST:
- Identify problem domain and pivot to specific runbook
- Enable OpsCenter if disabled

SHOULD:
- Document findings

MAY:
- Set up monitoring

## Output Format

```yaml
root_cause: "general_opscenter — <specific_cause>"
evidence:
  - type: investigation_summary
    content: "<findings>"
severity: MEDIUM
mitigation:
  immediate: "Address identified issue"
  long_term: "Implement monitoring"
```


## Safety Ratings
- Overall Risk: LOW
- Data Exposure Risk: LOW — general troubleshooting uses read-only operations
- Service Disruption Risk: LOW — diagnostic commands do not affect resources
- Compliance Impact: LOW — standard troubleshooting procedures

## Escalation Conditions
- Issue cannot be resolved using standard troubleshooting procedures
- OpsCenter service becomes unavailable
- Multiple unrelated issues occur simultaneously
- AWS Health Dashboard shows Systems Manager service event
- Issue requires AWS Support engagement

## Data Sensitivity
- OpsCenter configuration — INTERNAL
- CloudWatch metrics — INTERNAL
- Service health data — INTERNAL
- Error messages — INTERNAL
- Never expose account details in external communications

## Prohibited Actions
- Do NOT make changes to production OpsCenter configurations without approval
- Do NOT run destructive operations during initial diagnosis
- Do NOT share OpsCenter details in public forums without sanitizing
- Do NOT bypass established escalation procedures
- Do NOT attempt fixes outside OpsCenter service boundaries
- Do NOT ignore recurring issues

## Phase 3 — Rollback
- Revert any diagnostic changes made during troubleshooting
- Restore OpsCenter to last known good configuration
- Document all troubleshooting steps and outcomes
- Verify OpsCenter health after troubleshooting
- Update runbooks with new findings
- Close out temporary access granted for troubleshooting

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
