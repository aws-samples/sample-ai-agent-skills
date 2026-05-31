---
title: "A1 — Service Network Creation Failures"
description: "Diagnose VPC Lattice service network creation and update issues"
status: active
severity: HIGH
triggers:
  - "service network creation failed"
  - "cannot create service network"
  - "service network error"
owner: devops-agent
objective: "Identify and resolve service network creation failures"
context: "A service network is the top-level grouping in VPC Lattice. It connects VPCs and services. Creation failures typically involve IAM permissions, service quotas, or naming conflicts."
---

## Phase 1 — Triage

MUST:
- Check IAM permissions: `aws iam simulate-principal-policy --policy-source-arn <role-arn> --action-names vpc-lattice:CreateServiceNetwork`
- List existing service networks: `aws vpc-lattice list-service-networks`
- Check service quotas: `aws service-quotas get-service-quota --service-code vpc-lattice --quota-code L-1A2B3C4D`
- Review CloudTrail for error details: `aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventName,AttributeValue=CreateServiceNetwork --max-results 5`

SHOULD:
- Verify no naming conflict with existing service networks
- Check if account has VPC Lattice available in the region

## Phase 2 — Remediate

MUST:
- Ensure IAM role has `vpc-lattice:CreateServiceNetwork` permission
- Verify service quota not exceeded (default: 10 service networks per region)
- Use unique service network name

SHOULD:
- Tag service networks for organization
- Configure auth type during creation (NONE or AWS_IAM)

## Output Format

```yaml
root_cause: "service-network-creation — <specific_cause>"
evidence:
  - type: iam_check
    content: "<permission evaluation result>"
  - type: quota_check
    content: "<current vs max service networks>"
severity: HIGH
mitigation:
  immediate: "Fix permissions or request quota increase"
  long_term: "Implement IaC for consistent service network creation"
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

- Service network configurations reveal microservice architecture
- Auth policies contain inter-service access control rules
- Access logs reveal service-to-service communication patterns
- Classification: INTERNAL - service mesh topology and access patterns

## Prohibited Actions

- NEVER delete service networks with active service associations
- NEVER modify auth policies that could disrupt inter-service communication
- NEVER remove target groups serving production traffic
- NEVER change listener rules without testing routing behavior first

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
  - command: "describe-security-groups"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"
  - command: "describe-network-acls"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"
  - command: "describe-route-tables"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"

## Prohibited Actions

prohibited_actions:
  - "NEVER suggest 0.0.0.0/0 inbound rules as a fix"
  - "NEVER suggest disabling NACLs to troubleshoot"
  - "NEVER remove all route table entries"
