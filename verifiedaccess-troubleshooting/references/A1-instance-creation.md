---
title: "A1 — Instance Creation Issues"
description: "Diagnose Verified Access instance creation failures"
status: active
severity: HIGH
triggers:
  - "instance creation failed"
  - "cannot create Verified Access instance"
  - "CreateVerifiedAccessInstance error"
owner: devops-agent
objective: "Identify why Verified Access instance creation is failing and resolve the issue"
context: "Verified Access instances are the top-level resource that groups and endpoints belong to. Instance creation requires IAM permissions and may fail due to service limits, permission errors, or configuration issues. Trust providers are attached after instance creation."
---

## Phase 1 — Triage

MUST:
- Check existing instances: `aws ec2 describe-verified-access-instances`
- Verify IAM permissions for ec2:CreateVerifiedAccessInstance
- Check service quotas for Verified Access instances
- Check CloudTrail for creation errors: `aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventName,AttributeValue=CreateVerifiedAccessInstance --max-results 10`

SHOULD:
- Verify the region supports Verified Access
- Check for SCP restrictions blocking the operation
- Review account-level limits

MAY:
- Check if there are organization-level restrictions
- Review IAM policy conditions that may block creation

## Phase 2 — Remediate

MUST:
- Ensure IAM permissions include ec2:CreateVerifiedAccessInstance and ec2:CreateTags
- Create instance: `aws ec2 create-verified-access-instance --description "My VA Instance" --tag-specifications 'ResourceType=verified-access-instance,Tags=[{Key=Name,Value=my-instance}]'`

SHOULD:
- Attach trust providers after creation: `aws ec2 attach-verified-access-trust-provider --verified-access-instance-id <instance-id> --verified-access-trust-provider-id <provider-id>`
- Enable logging immediately after creation

MAY:
- Use CloudFormation/Terraform for repeatable deployments
- Tag instances for cost allocation and management

## Common Issues

- symptoms: "Access denied creating instance"
  diagnosis: "IAM policy missing ec2:CreateVerifiedAccessInstance permission."
  resolution: "Add the required EC2 Verified Access permissions to the IAM policy."

- symptoms: "Service quota exceeded"
  diagnosis: "Account has reached the Verified Access instance limit."
  resolution: "Request quota increase or delete unused instances."

## Output Format

```yaml
root_cause: "instance_creation — <specific_cause>"
evidence:
  - type: iam_permissions
    content: "<IAM policy evaluation>"
  - type: service_quotas
    content: "<current instance count vs limit>"
severity: HIGH
mitigation:
  immediate: "Fix IAM permissions or quota"
  long_term: "Use IaC for Verified Access deployment"
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

- Trust provider configurations contain identity provider details
- Access logs reveal user access patterns and device information
- Cedar policies define authorization logic for applications
- Classification: CONFIDENTIAL - access control policies and user activity data

## Prohibited Actions

- NEVER modify trust provider configurations during active user sessions
- NEVER change Verified Access policies that could lock out production users
- NEVER delete endpoints serving active application traffic
- NEVER modify Cedar policies without testing in a non-production environment first

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
