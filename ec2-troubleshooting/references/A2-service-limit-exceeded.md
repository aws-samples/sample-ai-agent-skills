---
title: "A2 — Service Limit / Quota Exceeded"
description: "Diagnose EC2 launch failures due to service quotas (vCPU limits, EIP limits, etc.)"
status: active
severity: HIGH
triggers:
  - "InstanceLimitExceeded"
  - "vCPU.*limit"
  - "You have requested more instances than your current instance limit"
  - "AddressLimitExceeded"
owner: devops-agent
objective: "Identify the exhausted quota and either optimize usage or request an increase"
context: "EC2 quotas are per-region and per-instance-family for On-Demand vCPUs. Spot has separate vCPU limits. Quotas are NOT per-AZ. Common limits: On-Demand vCPUs, Spot vCPUs, EIPs, security groups per VPC, ENIs per region."
---

## Phase 1 — Triage

MUST:
- Identify which quota is exceeded from the error message
- Check current quota usage: `aws service-quotas get-service-quota --service-code ec2 --quota-code <code>`
- Check current usage: `aws ec2 describe-instances --filters Name=instance-state-name,Values=running` and count vCPUs
- Distinguish between On-Demand vCPU limits (L-1216C47A) and Spot vCPU limits (L-34B43A08)

SHOULD:
- Check if stopped instances are consuming quotas (they don't consume vCPU quota but do consume other limits)
- Review running instances for unused or oversized resources that could be right-sized

MAY:
- Check quota history: `aws service-quotas list-requested-service-quota-change-history --service-code ec2`

## Phase 2 — Remediate

MUST:
- Request quota increase if legitimately needed: `aws service-quotas request-service-quota-increase --service-code ec2 --quota-code <code> --desired-value <value>`
- If urgent: contact AWS Support for expedited quota increase

SHOULD:
- Terminate unused instances to free up quota
- Right-size over-provisioned instances to reduce vCPU consumption
- Use Spot or Reserved Instances to optimize costs while managing capacity

MAY:
- Consolidate workloads onto fewer, larger instances
- Distribute across regions if single-region quota is insufficient

## Common Issues

- symptoms: "InstanceLimitExceeded when launching On-Demand instances"
  diagnosis: "On-Demand vCPU quota exhausted for this instance family in this region."
  resolution: "Request quota increase via Service Quotas console or API. Typical approval: 1-3 business days."

- symptoms: "Launch fails but running instance count seems low"
  diagnosis: "vCPU quota is per-family. A few large instances (e.g., 4x m5.16xlarge = 256 vCPUs) can exhaust the default 64 vCPU limit."
  resolution: "Check vCPU count, not instance count. Request increase for the specific instance family."

## Safety Ratings
Rate each remediation action in Phase 2:
- GREEN: read-only diagnostic commands (describe-*, get-*, list-*)
- YELLOW: state-changing but recoverable (modify security group, restart instance, modify parameter)
- RED: destructive or irreversible (terminate instance, delete volume, modify key infrastructure)

Format as:
```
safety_ratings:
  - "Request quota increase via service-quotas API: GREEN — administrative request, no infrastructure change"
  - "Terminate unused instances to free quota: RED — permanently destroys instances and data"
  - "Right-size over-provisioned instances: YELLOW — requires stop+start, changes instance type"
  - "Consolidate workloads onto fewer instances: RED — requires migration planning, risk of data loss"
```

## Escalation Conditions
- Quota increase request is denied or delayed beyond SLA
- Terminating instances to free quota would impact production workloads
- Multiple quotas are simultaneously exhausted across instance families
- Quota exhaustion is caused by runaway automation or compromised credentials
- Critical production launch blocked and no unused instances can be safely terminated

## Data Sensitivity
- HIGH: describe-instances (returns instance IPs, security group IDs, IAM roles, key pair names)
- MEDIUM: service-quotas get-service-quota (reveals account capacity and usage patterns)
- MEDIUM: list-requested-service-quota-change-history (reveals scaling plans)

## Prohibited Actions
- NEVER suggest terminating instances without first confirming they are unused or non-production
- NEVER suggest reducing quotas below current usage levels
- NEVER suggest distributing across regions without understanding data residency requirements
- NEVER suggest modifying Reserved Instance commitments to free quota

## Phase 3 — Rollback
- If instances were terminated to free quota: restore from latest AMI/snapshot if termination was premature
- If instance type was changed (right-sizing): stop instance, change back to original type, restart
- If quota increase causes unexpected billing: request quota decrease back to previous value
- If workloads were consolidated: redeploy from backup/IaC to original instance configuration

## Output Format

```yaml
root_cause: "Quota exceeded — <quota_name> at <current>/<limit>"
evidence:
  - type: service_quota
    content: "<quota details from service-quotas API>"
severity: HIGH
mitigation:
  immediate: "Request quota increase or terminate unused instances"
  long_term: "Implement quota monitoring alerts, right-size instances"
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
  - command: "describe-instances"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"
  - command: "get-console-output"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"
  - command: "ssm send-command"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"

## Prohibited Actions

prohibited_actions:
  - "NEVER suggest 0.0.0.0/0 inbound security group rules as a fix"
  - "NEVER suggest disabling instance metadata service"
  - "NEVER terminate instances without confirmation"
