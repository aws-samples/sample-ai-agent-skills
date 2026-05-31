---
title: "A1 — Resource Share Creation Failures"
description: "Diagnose why a RAM resource share creation is failing"
status: active
severity: HIGH
triggers:
  - "resource share creation failed"
  - "cannot create share"
  - "CreateResourceShare error"
  - "share creation denied"
owner: devops-agent
objective: "Identify why resource share creation is failing and resolve the issue"
context: "Resource share creation can fail due to IAM permission issues, unsupported resource types, invalid principal ARNs, resource already shared, or organization sharing not enabled. The caller needs ram:CreateResourceShare permission and appropriate permissions for the resource being shared."
---

## Phase 1 — Triage

MUST:
- Check IAM permissions: `aws iam simulate-principal-policy --policy-source-arn <caller-arn> --action-names ram:CreateResourceShare`
- Verify resource type is shareable: `aws ram list-resource-types`
- Check if resource is already in another share: `aws ram list-resources --resource-owner SELF --resource-arns <resource-arn>`
- Verify principal ARNs are valid

SHOULD:
- Check CloudTrail for creation failure: `aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventName,AttributeValue=CreateResourceShare --max-results 10`
- Verify allowExternalPrincipals setting matches intended sharing scope
- Check for SCPs blocking RAM operations
- Verify the resource exists and is in the correct region

MAY:
- Check RAM service quotas
- Verify organization sharing is enabled if sharing with org principals

## Phase 2 — Remediate

MUST:
- Fix IAM permissions to include ram:CreateResourceShare
- Use supported resource types only
- Provide valid principal ARNs (account IDs, org ARN, OU ARN)

SHOULD:
- Set allowExternalPrincipals appropriately for the sharing scope
- Add appropriate RAM permissions to the share
- Test with a single resource and principal before expanding

MAY:
- Automate resource share creation with IaC
- Set up monitoring for share creation events

## Common Issues

- symptoms: "Unsupported resource type error"
  diagnosis: "The resource type cannot be shared through RAM."
  resolution: "Check supported types: aws ram list-resource-types. Use resource policies for unsupported types."

- symptoms: "Invalid principal error"
  diagnosis: "Principal ARN format is incorrect or account doesn't exist."
  resolution: "Use correct format: account ID, arn:aws:organizations::account-id:organization/o-xxx, or OU ARN."

## Output Format

```yaml
root_cause: "share_creation_failure — <specific_cause>"
evidence:
  - type: iam_permissions
    content: "<IAM policy evaluation>"
  - type: resource_type
    content: "<resource type support status>"
severity: HIGH
mitigation:
  immediate: "Fix the specific creation blocker"
  long_term: "Automate resource share creation and validate prerequisites"
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

- Resource share configurations may contain account IDs and organizational structure
- Principal ARNs reveal cross-account relationships
- Permission policies may expose internal access patterns
- Classification: INTERNAL - contains organizational topology data

## Prohibited Actions

- NEVER modify or delete resource shares in production without owner approval
- NEVER change RAM permissions that could revoke cross-account access to shared resources
- NEVER disable organization sharing without coordinating with all member accounts
- NEVER remove principals from active resource shares without impact analysis

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
