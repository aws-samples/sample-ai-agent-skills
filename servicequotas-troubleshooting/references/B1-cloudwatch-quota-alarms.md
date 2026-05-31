---
title: "B1 — CloudWatch Quota Alarms"
description: "Set up and troubleshoot CloudWatch alarms for quota utilization"
status: active
severity: MEDIUM
triggers:
  - "quota alarm"
  - "usage alarm"
  - "quota monitoring"
  - "approaching limit"
owner: devops-agent
objective: "Configure CloudWatch alarms to alert before hitting quota limits"
context: "Some quotas publish usage metrics to CloudWatch under AWS/Usage namespace. Alarms can be created on these metrics to alert when utilization approaches the quota limit. Not all quotas have CloudWatch metrics."
---

## Phase 1 — Triage

MUST:
- Check if quota has usage metric: `aws service-quotas get-service-quota --service-code <service-code> --quota-code <quota-code> --query 'Quota.UsageMetric'`
- Check existing alarms: `aws cloudwatch describe-alarms --alarm-name-prefix <prefix>`
- Get current usage: `aws cloudwatch get-metric-statistics --namespace AWS/Usage --metric-name ResourceCount --dimensions Name=Type,Value=Resource Name=Resource,Value=<resource> Name=Service,Value=<service> Name=Class,Value=None --start-time <start> --end-time <end> --period 3600 --statistics Maximum`

SHOULD:
- Verify the metric dimensions match the quota
- Check alarm threshold vs current quota value
- Review alarm actions (SNS topic, etc.)

MAY:
- Check alarm history for previous triggers
- Compare usage trends over time

## Phase 2 — Remediate

MUST:
- Create alarm if none exists for critical quotas
- Set threshold at 80% of quota value for early warning
- Configure alarm actions (SNS notification)

SHOULD:
- Use math expressions to calculate percentage: `usage / quota * 100`
- Create alarms for all critical quotas
- Test alarm by temporarily lowering threshold

MAY:
- Create a dashboard for quota utilization
- Set up automated quota increase requests on alarm

## Common Issues

- symptoms: "Cannot create alarm — no usage metric"
  diagnosis: "This quota doesn't publish CloudWatch metrics."
  resolution: "Use service-specific APIs to monitor utilization manually."

- symptoms: "Alarm never triggers despite high usage"
  diagnosis: "Wrong metric dimensions or threshold too high."
  resolution: "Verify metric dimensions match. Set threshold at 80% of applied quota."

## Output Format

```yaml
root_cause: "quota_alarm — <specific_cause>"
evidence:
  - type: usage_metric
    content: "<usage metric availability>"
  - type: alarm_config
    content: "<alarm configuration>"
severity: MEDIUM
mitigation:
  immediate: "Create or fix quota alarms"
  long_term: "Implement comprehensive quota monitoring"
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
