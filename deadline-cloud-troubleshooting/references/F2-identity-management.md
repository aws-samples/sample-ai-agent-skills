---
title: "F2 — Identity Management"
description: "Diagnose identity and access management issues for Deadline Cloud"
status: active
severity: HIGH
triggers: ["access denied", "IAM error", "identity center issue", "permission denied", "role assumption failed"]
owner: devops-agent
objective: "Resolve identity and access management issues"
context: "Deadline Cloud uses IAM roles for API access and service operations, and IAM Identity Center for monitor portal access. Different roles are needed for farm admin, queue execution, fleet workers, and submitters."
---

## Phase 1 — Triage

MUST:
- Check IAM role permissions: `aws iam get-role --role-name <role-name>`
- Verify role trust policy allows Deadline Cloud service: `aws iam get-role --role-name <role-name> --query 'Role.AssumeRolePolicyDocument'`
- Check for explicit deny in IAM policies or SCPs

SHOULD:
- Verify queue role has S3 permissions for job attachments
- Check fleet role has EC2 permissions for service-managed fleets
- Review Identity Center user assignments for monitor access

MAY:
- Use IAM Policy Simulator to test permissions
- Review CloudTrail for access denied events: `aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventName,AttributeValue=CreateJob --max-results 10`

## Phase 2 — Remediate

MUST:
- Fix IAM role trust policy to include deadline.amazonaws.com or credentials.deadline.amazonaws.com
- Add required permissions to role policies
- Ensure no SCPs blocking Deadline Cloud actions

SHOULD:
- Use least-privilege IAM policies scoped to specific farms/queues
- Configure separate roles for different functions (admin, submitter, worker)

MAY:
- Implement permission boundaries for Deadline Cloud roles
- Set up IAM Access Analyzer to review permissions

## Safety Ratings

```
safety_ratings:
  - "iam get-role: GREEN — read-only role details"
  - "iam get-role-policy: GREEN — read-only policy details"
  - "iam put-role-policy: YELLOW — modifies role permissions"
  - "iam update-assume-role-policy: YELLOW — changes who can assume the role"
  - "iam delete-role: RED — removes role, breaks all services using it"
```

## Escalation Conditions

- IAM role changes require security team approval
- SCP modifications require organization admin
- Identity Center configuration requires SSO admin
- Cross-account access requires coordination between account owners

## Data Sensitivity

- IAM role policies: HIGH — reveal permission scope and access patterns
- Trust policies: HIGH — reveal which services and principals can assume roles
- CloudTrail access denied events: MEDIUM — reveal attempted actions and principals
- Identity Center assignments: MEDIUM — reveal user access patterns

## Prohibited Actions

- NEVER grant AdministratorAccess to Deadline Cloud roles
- NEVER modify IAM roles used by multiple services without impact analysis
- NEVER share IAM role credentials or access keys
- NEVER disable SCPs to work around permission issues

## Phase 3 — Rollback

1. If IAM policy was changed incorrectly, revert to previous policy version
2. If trust policy was modified, restore previous trust relationship
3. If Identity Center assignment was changed, restore previous assignments
4. Verify all Deadline Cloud operations work after permission changes

## Common Issues

- symptoms: "CreateJob returns AccessDeniedException"
  diagnosis: "Submitter IAM role lacks deadline:CreateJob permission on the queue."
  resolution: "Add deadline:CreateJob permission scoped to the specific farm and queue ARN."

- symptoms: "Fleet workers cannot register with Deadline Cloud"
  diagnosis: "Fleet IAM role trust policy does not allow credentials.deadline.amazonaws.com."
  resolution: "Update role trust policy to allow credentials.deadline.amazonaws.com as trusted principal."

## Output Format

```yaml
root_cause: "identity_management — <specific_cause>"
evidence:
  - type: iam_detail
    content: "<IAM role configuration and error>"
severity: HIGH
mitigation:
  immediate: "Fix IAM permissions"
  long_term: "Implement least-privilege IAM policies with IaC"
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
