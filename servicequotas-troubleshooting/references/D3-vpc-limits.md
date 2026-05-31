---
title: "D3 — VPC Limits"
description: "Troubleshoot common VPC quota issues"
status: active
severity: HIGH
triggers:
  - "VPC limit reached"
  - "cannot create subnet"
  - "security group limit"
  - "route table limit"
owner: devops-agent
objective: "Resolve VPC quota issues preventing resource creation"
context: "VPC quotas include VPCs per region, subnets per VPC, security groups per VPC, rules per security group, route tables per VPC, network interfaces per region, and more. Some are per-VPC, others per-region."
---

## Phase 1 — Triage

MUST:
- Check VPCs per region: `aws service-quotas get-service-quota --service-code vpc --quota-code L-F678F1CE`
- Check subnets per VPC: `aws service-quotas get-service-quota --service-code vpc --quota-code L-407747CB`
- Check security groups per VPC: `aws service-quotas get-service-quota --service-code vpc --quota-code L-E79EC296`
- Check current usage: `aws ec2 describe-vpcs --query 'Vpcs | length(@)'`

SHOULD:
- Check rules per security group: `aws service-quotas get-service-quota --service-code vpc --quota-code L-0EA8095F`
- Check route tables per VPC: `aws service-quotas get-service-quota --service-code vpc --quota-code L-589F43AA`
- Check network interfaces per region: `aws service-quotas get-service-quota --service-code vpc --quota-code L-DF5E4CA3`
- Count resources in the specific VPC

MAY:
- Check NAT gateway limits
- Check VPC peering connection limits
- Review VPC endpoint limits

## Phase 2 — Remediate

MUST:
- Request quota increase for the specific limit hit
- Clean up unused resources (empty security groups, unused subnets)
- Identify the specific quota code from the error message

SHOULD:
- Consolidate security group rules where possible
- Remove unused VPCs and subnets
- Plan VPC architecture to stay within limits

MAY:
- Distribute workloads across multiple VPCs
- Use Transit Gateway to reduce peering connections

## Common Issues

- symptoms: "Cannot create security group — limit reached"
  diagnosis: "Security groups per VPC quota exhausted."
  resolution: "Delete unused security groups. Request increase (L-E79EC296)."

- symptoms: "Cannot add rule to security group"
  diagnosis: "Rules per security group quota reached (default 60 inbound + 60 outbound)."
  resolution: "Consolidate rules. Use prefix lists. Request increase (L-0EA8095F)."

- symptoms: "Cannot create subnet"
  diagnosis: "Subnets per VPC quota reached."
  resolution: "Delete unused subnets. Request increase (L-407747CB)."

## Output Format

```yaml
root_cause: "vpc_limits — <specific_cause>"
evidence:
  - type: quota_value
    content: "<specific VPC quota and usage>"
  - type: error_message
    content: "<resource creation error>"
severity: HIGH
mitigation:
  immediate: "Clean up unused resources or request quota increase"
  long_term: "Plan VPC architecture within quota constraints"
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
