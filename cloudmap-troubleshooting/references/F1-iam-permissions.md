---
title: "F1 — Cloud Map IAM Permission Issues"
description: "Diagnose IAM permission problems for Cloud Map operations"
status: active
severity: HIGH
triggers:
  - "access denied"
  - "permission denied"
  - "not authorized"
  - "IAM error"
owner: devops-agent
objective: "Identify and resolve IAM permission issues affecting Cloud Map operations"
context: "Cloud Map operations require servicediscovery permissions. DNS namespaces also require route53 permissions. ECS/EKS integration requires additional permissions for the respective services."
---

## Phase 1 — Triage

MUST:
- Check caller permissions: `aws iam simulate-principal-policy --policy-source-arn <caller-arn> --action-names servicediscovery:ListNamespaces servicediscovery:RegisterInstance servicediscovery:DiscoverInstances`
- Check CloudTrail for access denied events: `aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventSource,AttributeValue=servicediscovery.amazonaws.com --max-results 10`
- Verify the specific action that failed from the error message
- Check for SCPs or permission boundaries

SHOULD:
- Verify Route 53 permissions for DNS namespace operations
- Check ECS task role permissions for service discovery
- Verify IRSA permissions for EKS controllers

MAY:
- Check resource-level permissions (specific namespace/service ARNs)
- Review IAM policy conditions that may restrict access

## Phase 2 — Remediate

MUST:
- Grant required servicediscovery permissions to the calling identity
- For DNS namespaces, grant route53 permissions (route53:ChangeResourceRecordSets, etc.)
- For ECS, grant permissions to the ECS task execution role

SHOULD:
- Use least-privilege policies scoped to specific namespaces
- Document required permissions for each integration
- Use IAM policy simulator to validate permissions

MAY:
- Create managed policies for common Cloud Map access patterns
- Implement permission auditing

## Common Issues

- symptoms: "AccessDeniedException on RegisterInstance"
  diagnosis: "Caller lacks servicediscovery:RegisterInstance permission."
  resolution: "Add servicediscovery:RegisterInstance to the IAM policy."

- symptoms: "DiscoverInstances returns access denied"
  diagnosis: "Caller lacks servicediscovery:DiscoverInstances permission."
  resolution: "Add servicediscovery:DiscoverInstances permission."

- symptoms: "Namespace creation fails with Route 53 error"
  diagnosis: "Caller lacks route53:CreateHostedZone permission."
  resolution: "Add Route 53 permissions for DNS namespace operations."

## Output Format

```yaml
root_cause: "iam_permission_issue — <specific_cause>"
evidence:
  - type: permission_test
    content: "<simulate-principal-policy results>"
  - type: error_message
    content: "<specific access denied error>"
  - type: cloudtrail
    content: "<access denied events>"
severity: HIGH
mitigation:
  immediate: "Add missing permissions to the IAM policy"
  long_term: "Implement least-privilege policies and regular audits"
```


## Safety Ratings

| Phase 2 Action | safety_rating |
|---|---|
| Grant servicediscovery permissions | YELLOW |
| Grant route53 permissions for DNS namespaces | YELLOW |
| Grant permissions to ECS task execution role | YELLOW |
| Use least-privilege policies | GREEN |
| Document required permissions | GREEN |
| Create managed policies | GREEN |

## Escalation Conditions

- Fix requires modifying production IAM roles or policies
- Fix involves granting broad service discovery permissions

## Data Sensitivity

- **HIGH**: IAM configurations control access to all Cloud Map operations
- Permission changes affect service registration and discovery across the infrastructure

## Prohibited Actions

- NEVER suggest deleting namespaces with active service registrations

## Phase 3 — Rollback

1. If IAM permissions were broadened, revert to the previous policy version
2. If Route 53 permissions were added, remove if no longer needed
3. If ECS task role was modified, restore previous policy
4. Verify rollback by confirming IAM policies match expected least-privilege configuration

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
