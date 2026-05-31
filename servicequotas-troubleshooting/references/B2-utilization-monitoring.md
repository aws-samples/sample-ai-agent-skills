---
title: "B2 — Utilization Monitoring"
description: "Monitor quota utilization across services"
status: active
severity: MEDIUM
triggers:
  - "quota utilization"
  - "how much used"
  - "capacity remaining"
  - "usage percentage"
owner: devops-agent
objective: "Monitor and report on quota utilization across AWS services"
context: "Quota utilization monitoring helps prevent hitting limits. Some quotas have CloudWatch metrics, others require service-specific API calls. Proactive monitoring prevents service disruptions."
---

## Phase 1 — Triage

MUST:
- Check quotas with usage metrics: `aws service-quotas list-service-quotas --service-code <service-code> --query 'Quotas[?UsageMetric!=null].{Name:QuotaName,Code:QuotaCode,Value:Value,Usage:UsageMetric}'`
- Get current utilization for key quotas
- Compare utilization against applied quota values
- Identify quotas above 80% utilization

SHOULD:
- Check EC2 vCPU utilization: `aws ec2 describe-instance-type-offerings --location-type region --query 'InstanceTypeOfferings | length(@)'`
- Check Lambda concurrency: `aws lambda get-account-settings --query 'AccountLimit.ConcurrentExecutions'`
- Check VPC counts: `aws ec2 describe-vpcs --query 'Vpcs | length(@)'`

MAY:
- Create utilization reports across all services
- Track utilization trends over time
- Compare utilization across regions

## Phase 2 — Remediate

MUST:
- Request increases for quotas above 80% utilization
- Set up alarms for critical quotas
- Document current utilization levels

SHOULD:
- Create a regular quota review process
- Automate utilization checks with scripts
- Set up dashboards for visibility

MAY:
- Implement automated quota increase workflows
- Create capacity planning reports

## Common Issues

- symptoms: "Don't know current utilization"
  diagnosis: "Need to check service-specific APIs for usage."
  resolution: "Use CloudWatch AWS/Usage metrics where available. Use service APIs for others."

- symptoms: "Utilization suddenly spiked"
  diagnosis: "New deployment or workload increase."
  resolution: "Request quota increase if sustained. Investigate if unexpected."

## Output Format

```yaml
root_cause: "utilization — <specific_cause>"
evidence:
  - type: utilization
    content: "<current utilization vs quota>"
  - type: trend
    content: "<utilization trend>"
severity: MEDIUM
mitigation:
  immediate: "Request increases for high-utilization quotas"
  long_term: "Implement automated utilization monitoring"
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
