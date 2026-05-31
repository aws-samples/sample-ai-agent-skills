---
title: "D1 — EC2 Limits"
description: "Troubleshoot common EC2 quota issues"
status: active
severity: HIGH
triggers:
  - "EC2 limit reached"
  - "cannot launch instance"
  - "vCPU limit"
  - "InstanceLimitExceeded"
owner: devops-agent
objective: "Resolve EC2 quota issues preventing instance launches"
context: "EC2 On-Demand limits are vCPU-based per instance family (Standard, Accelerated, etc.). Spot instances have separate limits. EBS volume limits, EIP limits, and security group limits are also common bottlenecks."
---

## Phase 1 — Triage

MUST:
- Check On-Demand vCPU quota: `aws service-quotas get-service-quota --service-code ec2 --quota-code L-1216C47A`
- Check current vCPU usage: `aws ec2 describe-instances --filters Name=instance-state-name,Values=running --query 'Reservations[*].Instances[*].[InstanceType]' --output text | sort | uniq -c`
- Check specific error message from launch failure
- Check EBS volume limits: `aws service-quotas get-service-quota --service-code ebs --quota-code L-D18FCD1D`

SHOULD:
- Check Spot vCPU quota: `aws service-quotas get-service-quota --service-code ec2 --quota-code L-34B43A08`
- Check EIP limit: `aws service-quotas get-service-quota --service-code ec2 --quota-code L-0263D0A3`
- Calculate vCPU usage per instance family
- Check security group limits: `aws service-quotas get-service-quota --service-code vpc --quota-code L-E79EC296`

MAY:
- Check instance type availability in the AZ
- Review EC2 usage trends

## Phase 2 — Remediate

MUST:
- Request vCPU quota increase: `aws service-quotas request-service-quota-increase --service-code ec2 --quota-code L-1216C47A --desired-value <value>`
- Use the correct quota code for the instance family
- Monitor request status

SHOULD:
- Request increases proactively before hitting limits
- Consider using Spot instances for non-critical workloads
- Distribute workloads across regions if needed

MAY:
- Use smaller instance types to fit within vCPU limits
- Implement auto-scaling with appropriate limits

## Common Issues

- symptoms: "InstanceLimitExceeded"
  diagnosis: "On-Demand vCPU limit reached for the instance family."
  resolution: "Request vCPU quota increase for the specific instance family."

- symptoms: "Cannot launch in specific AZ"
  diagnosis: "Instance type not available in that AZ (capacity, not quota)."
  resolution: "Try a different AZ or instance type. This is capacity, not quota."

- symptoms: "EIP limit reached"
  diagnosis: "Elastic IP address quota exhausted."
  resolution: "Release unused EIPs or request quota increase (L-0263D0A3)."

## Output Format

```yaml
root_cause: "ec2_limits — <specific_cause>"
evidence:
  - type: vcpu_quota
    content: "<vCPU quota and usage>"
  - type: error_message
    content: "<launch failure error>"
severity: HIGH
mitigation:
  immediate: "Request quota increase for the specific limit"
  long_term: "Implement proactive quota monitoring"
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
