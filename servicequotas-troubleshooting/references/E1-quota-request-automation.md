---
title: "E1 — Quota Request Automation"
description: "Automate quota increase requests"
status: active
severity: LOW
triggers:
  - "automate quota"
  - "automatic increase"
  - "quota automation"
  - "proactive quota"
owner: devops-agent
objective: "Set up automated quota increase request workflows"
context: "Quota management can be automated using Service Quotas API, CloudWatch alarms, EventBridge, and Lambda. Automated workflows can request increases when utilization approaches limits, reducing manual intervention."
---

## Phase 1 — Triage

MUST:
- Identify quotas that need automated management
- Check which quotas have CloudWatch usage metrics
- Verify Service Quotas API access: `aws service-quotas list-services`
- Check existing automation (Lambda functions, EventBridge rules)

SHOULD:
- Review current quota utilization levels
- Identify quotas that frequently need increases
- Check IAM permissions for automated quota requests

MAY:
- Review existing CloudWatch alarms for quota metrics
- Check for existing automation frameworks

## Phase 2 — Remediate

MUST:
- Create CloudWatch alarms on quota usage metrics (80% threshold)
- Create Lambda function to request quota increases
- Connect alarm → SNS → Lambda for automated requests

SHOULD:
- Include approval workflow for large increases
- Log all automated requests for audit
- Set maximum auto-increase limits to prevent runaway costs

MAY:
- Use AWS Config rules to monitor quota compliance
- Create dashboards for quota automation status
- Implement Slack/Teams notifications for quota events

## Common Issues

- symptoms: "Automation requests denied"
  diagnosis: "Requested value too high or quota not adjustable."
  resolution: "Add validation logic. Check adjustable field before requesting."

- symptoms: "Alarm triggers but no increase requested"
  diagnosis: "Lambda function error or IAM permission issue."
  resolution: "Check Lambda logs. Verify IAM role has servicequotas:RequestServiceQuotaIncrease."

## Output Format

```yaml
root_cause: "quota_automation — <specific_cause>"
evidence:
  - type: automation_config
    content: "<current automation setup>"
  - type: quotas_managed
    content: "<quotas under automated management>"
severity: LOW
mitigation:
  immediate: "Fix automation configuration"
  long_term: "Expand automation to cover all critical quotas"
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

- Quota utilization reveals infrastructure scale and capacity
- Service usage patterns may indicate business activity levels
- Organization quota templates reveal multi-account structure
- Classification: INTERNAL - operational capacity data

## Prohibited Actions

- NEVER reduce service quotas below current utilization levels
- NEVER delete CloudWatch alarms monitoring quota utilization
- NEVER modify organization-level quota templates without member account review
- NEVER cancel pending quota increase requests without requester approval

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
