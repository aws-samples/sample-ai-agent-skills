---
title: "A1 — Workspace Creation Failures"
description: "Diagnose and resolve AMP workspace creation failures"
status: active
severity: HIGH
triggers:
  - "workspace creation failed"
  - "CreateWorkspace"
  - "CREATION_FAILED"
  - "cannot create workspace"
owner: devops-agent
objective: "Identify why the AMP workspace failed to create and resolve the issue"
context: "AMP workspace creation provisions a managed Prometheus-compatible backend including ingestion, storage, and query infrastructure. Failures typically stem from IAM permissions, service quotas, or tagging policy violations."
---

## Phase 1 — Triage

MUST:
- Check workspace status: `aws amp list-workspaces --query 'workspaces[?alias==`<alias>`]'`
- Check CloudTrail for CreateWorkspace errors: `aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventName,AttributeValue=CreateWorkspace --max-results 5`
- Verify IAM permissions for the calling principal: `aws iam simulate-principal-policy --policy-source-arn <caller-arn> --action-names aps:CreateWorkspace`
- Check service quotas for workspace limit: `aws service-quotas get-service-quota --service-code aps --quota-code L-1B5E2E42`

SHOULD:
- Check if a workspace with the same alias already exists: `aws amp list-workspaces --alias <alias>`
- Verify no SCP or permission boundary blocks `aps:CreateWorkspace`
- Check for tag policy violations if tags are specified
- Verify the region supports AMP: `aws amp list-workspaces --region <region>`

MAY:
- Check AWS Health Dashboard for AMP service events
- Review organization-level SCPs that may restrict AMP operations
- Check if the account has been recently created (some services have delayed availability)

## Phase 2 — Remediate

MUST:
- If IAM permission denied: add `aps:CreateWorkspace` to the caller's IAM policy
- If quota exceeded: request increase via `aws service-quotas request-service-quota-increase --service-code aps --quota-code L-1B5E2E42 --desired-value <new-limit>`
- If tag policy violation: fix tags to comply with organization tag policies
- If region not supported: use a supported region

SHOULD:
- Use IaC (CloudFormation, Terraform, CDK) for workspace creation with validated configurations
- Set a meaningful alias for the workspace for easier identification
- Apply consistent tagging for cost allocation and organization

MAY:
- Configure logging configuration at creation time
- Set up workspace tags for resource grouping

## Common Issues

- symptoms: "AccessDeniedException when calling CreateWorkspace"
  diagnosis: "Calling principal lacks aps:CreateWorkspace permission."
  resolution: "Add aps:CreateWorkspace to the IAM policy. If using a service role, update the role's policy."

- symptoms: "ServiceQuotaExceededException — workspace limit reached"
  diagnosis: "Account has reached the maximum number of AMP workspaces per region."
  resolution: "Request a quota increase or delete unused workspaces."

- symptoms: "ValidationException — invalid tag key or value"
  diagnosis: "Tags do not comply with organization tag policies or AWS tag restrictions."
  resolution: "Fix tag keys/values. Tag keys must be 1-128 chars, values 0-256 chars. Check org tag policies."

- symptoms: "Workspace stuck in CREATING status for extended period"
  diagnosis: "Backend provisioning delay or transient service issue."
  resolution: "Wait up to 5 minutes. If still CREATING, check AWS Health Dashboard and retry."

## Safety Ratings

```
safety_ratings:
  - "list-workspaces: GREEN — read-only enumeration"
  - "simulate-principal-policy: GREEN — read-only IAM check"
  - "service-quotas get-service-quota: GREEN — read-only quota check"
  - "CreateWorkspace: YELLOW — provisions new managed infrastructure with ongoing cost"
  - "request-service-quota-increase: YELLOW — changes account limits"
```

## Escalation Conditions

- Workspace creation fails repeatedly after IAM and quota fixes
- Quota increase needed urgently for production deployment
- SCP or permission boundary blocks workspace creation at the organization level
- Workspace stuck in CREATING status for more than 5 minutes

## Data Sensitivity

- `aws amp list-workspaces`: LOW — workspace metadata only
- `aws iam simulate-principal-policy`: LOW — tests permissions without changes
- `aws cloudtrail lookup-events`: MEDIUM — reveals API call history and principal identities

## Prohibited Actions

- NEVER suggest deleting a workspace with active collectors writing to it
- NEVER suggest basic auth (only SigV4)
- NEVER create workspaces without meaningful aliases for identification
- NEVER ignore tag policy violations — fix tags to comply with organizational policies

## Phase 3 — Rollback

1. If a workspace was created in error, delete it: `aws amp delete-workspace --workspace-id <workspace-id>` (only if no collectors are writing to it)
2. If a quota increase was requested in error, it cannot be reversed but won't cause harm
3. If tags were applied incorrectly, update them: `aws amp tag-resource --resource-arn <arn> --tags <correct-tags>`
4. Verify no collectors are configured to write to the workspace before deletion
5. Document the creation failure and root cause for future attempts

## Output Format

```yaml
root_cause: "Workspace creation failure — <specific_cause>"
evidence:
  - type: cloudtrail_event
    content: "<CreateWorkspace error details>"
  - type: iam_policy
    content: "<permission or quota issue>"
severity: HIGH
mitigation:
  immediate: "Fix the identified permission or quota issue and retry workspace creation"
  long_term: "Use IaC with validated configurations and pre-flight permission checks"
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
