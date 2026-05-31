---
title: "G2 — IAM Issues on Outposts"
description: "Diagnose IAM permission and role problems for Outpost operations"
status: active
severity: HIGH
triggers:
  - "IAM outpost"
  - "permission denied outpost"
  - "access denied outpost"
  - "outpost role"
owner: devops-agent
objective: "Resolve IAM permission issues for Outpost management and workload operations"
context: "Outpost operations require specific IAM permissions including outposts:*, ec2:* with Outpost resource conditions, and s3-outposts:* for S3 on Outposts. IAM evaluation happens in the Region, requiring service link connectivity."
---

## Phase 1 — Triage

MUST:
- Check IAM policy for Outpost permissions: `aws iam get-user-policy --user-name <user> --policy-name <policy>` or `aws iam get-role-policy --role-name <role> --policy-name <policy>`
- Simulate permissions: `aws iam simulate-principal-policy --policy-source-arn <principal-arn> --action-names outposts:GetOutpost ec2:RunInstances s3-outposts:GetBucket`
- Check for SCPs that may restrict Outpost operations: `aws organizations list-policies --filter SERVICE_CONTROL_POLICY`
- Verify service-linked role exists: `aws iam get-role --role-name AWSServiceRoleForOutposts_<outpost-id>`

SHOULD:
- Check resource-level permissions (Outpost ARN conditions)
- Verify cross-account permissions if using shared Outposts via RAM
- Review CloudTrail for AccessDenied events: `aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventName,AttributeValue=RunInstances --max-results 10`

MAY:
- Check permission boundaries on the IAM role
- Review tag-based access control policies

## Phase 2 — Remediate

MUST:
- If missing permissions: add required Outpost actions to IAM policy
- If SCP blocking: update SCP to allow Outpost operations
- If shared Outpost: verify RAM share permissions and resource policies

SHOULD:
- Use AWS managed policies where available (e.g., AWSOutpostsServiceRolePolicy)
- Implement least-privilege access with resource-level conditions
- Document required permissions for Outpost operations

MAY:
- Implement tag-based access control for Outpost resources
- Set up IAM Access Analyzer for permission auditing

## Common Issues

- symptoms: "AccessDenied when calling outposts:GetOutpost"
  diagnosis: "IAM policy missing outposts:* permissions."
  resolution: "Add outposts:GetOutpost and related actions to the IAM policy."

- symptoms: "Cannot launch instances on shared Outpost"
  diagnosis: "RAM share does not include the required resource types or account."
  resolution: "Verify RAM share includes ec2:Subnet and outposts:Outpost resource types."

## Output Format

```yaml
root_cause: "iam — <specific_cause>"
evidence:
  - type: iam_policy
    content: "<relevant IAM policy>"
  - type: simulation
    content: "<policy simulation results>"
severity: HIGH
mitigation:
  immediate: "Fix IAM permissions or SCP"
  long_term: "Implement least-privilege policies with Outpost resource conditions"
```


## Safety Ratings
- Overall Risk: HIGH
- Data Exposure Risk: HIGH — IAM controls all access to Outpost resources
- Service Disruption Risk: MEDIUM — IAM issues can prevent legitimate access
- Compliance Impact: HIGH — IAM is fundamental to security compliance

## Escalation Conditions
- IAM policy changes lock out Outpost administrators
- Overly permissive IAM policies detected
- Cross-account access issues affect Outpost management
- Service-linked role issues prevent Outpost operations
- IAM audit reveals unauthorized access patterns

## Data Sensitivity
- IAM policies and role configurations — CONFIDENTIAL
- Access patterns and audit logs — CONFIDENTIAL
- Service-linked role configurations — INTERNAL
- Cross-account trust policies — CONFIDENTIAL
- Never expose IAM credentials or session tokens

## Prohibited Actions
- Do NOT attach AdministratorAccess to Outpost management roles
- Do NOT use wildcard resource ARNs in Outpost IAM policies
- Do NOT share IAM credentials across teams
- Do NOT bypass IAM authentication for troubleshooting
- Do NOT ignore IAM access audit findings
- Do NOT modify service-linked roles without understanding impact

## Phase 3 — Rollback
- Restore previous IAM policy versions
- Revert trust policy changes
- Re-enable permission boundaries if removed
- Revoke compromised credentials
- Verify access controls after IAM changes
- Document IAM changes and conduct security review

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
