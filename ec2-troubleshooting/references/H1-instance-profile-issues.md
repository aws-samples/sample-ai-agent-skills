---
title: "H1 — Instance Profile / IAM Role Issues"
description: "Diagnose IAM permission failures for EC2 instances"
status: active
severity: HIGH
triggers:
  - "AccessDenied"
  - "UnauthorizedAccess"
  - "no instance profile"
  - "AssumeRole.*failed"
owner: devops-agent
objective: "Identify the IAM permission gap and restore API access"
context: "EC2 instances use instance profiles to assume IAM roles. The instance profile wraps an IAM role. Credentials are delivered via IMDS. Common issues: no instance profile attached, role missing required permissions, SCP blocking, or permission boundary limiting."
---

## Phase 1 — Triage

MUST:
- Check if instance has an instance profile: `aws ec2 describe-instances --instance-ids <id>` → IamInstanceProfile
- If no profile: that's the root cause — no IAM credentials available
- Check the role's policies: `aws iam list-attached-role-policies --role-name <role>` and `aws iam list-role-policies --role-name <role>`
- Verify the specific API action is allowed by the role's policies

SHOULD:
- Check for explicit deny in SCPs (organization level)
- Check for permission boundaries on the role
- Check resource-based policies on the target resource (S3 bucket policy, KMS key policy)
- Use IAM Policy Simulator to test the specific action

MAY:
- Check CloudTrail for the exact AccessDenied event with error details
- Check if the role's trust policy allows ec2.amazonaws.com to assume it

## Common Issues

- symptoms: "AccessDenied but role appears to have correct permissions"
  diagnosis: "SCP, permission boundary, or resource-based policy is denying. Or condition keys don't match."
  resolution: "Check SCPs, permission boundaries, and resource policies. Use CloudTrail for the exact denial reason."

- symptoms: "No credentials available on instance"
  diagnosis: "No instance profile attached, or IMDS unreachable."
  resolution: "Attach instance profile: `aws ec2 associate-iam-instance-profile`. Check IMDS accessibility (see H2)."

- symptoms: "Credentials work for some APIs but not others"
  diagnosis: "Role has partial permissions. Missing specific actions or resource ARNs."
  resolution: "Add the missing IAM permissions to the role's policy."

## Safety Ratings
Rate each remediation action in Phase 2:
- GREEN: read-only diagnostic commands (describe-*, get-*, list-*)
- YELLOW: state-changing but recoverable (modify security group, restart instance, modify parameter)
- RED: destructive or irreversible (terminate instance, delete volume, modify key infrastructure)

Format as:
```
safety_ratings:
  - "describe-instances, list-attached-role-policies, list-role-policies: GREEN — read-only"
  - "IAM Policy Simulator: GREEN — read-only policy evaluation"
  - "CloudTrail AccessDenied event review: GREEN — read-only"
  - "Associate instance profile: YELLOW — grants IAM credentials to instance, recoverable"
  - "Add IAM policy to role: YELLOW — expands permissions, recoverable by detaching"
  - "Modify KMS key policy: YELLOW — changes key access, recoverable by reverting policy"
  - "Modify IAM role trust policy: RED — may break assume-role for all instances using this role"
```

## Escalation Conditions
- Fix requires modifying IAM instance profile in a production environment
- AccessDenied is caused by an SCP at the organization level (requires org admin)
- Permission boundary is blocking and cannot be modified by the team
- Cross-account resource access requires coordination with another team
- IAM role is shared across multiple instances and policy change affects all of them

## Data Sensitivity
- HIGH: list-attached-role-policies, get-role-policy (reveals IAM permissions and access scope)
- HIGH: CloudTrail AccessDenied events (reveals attempted actions, resource ARNs, caller identity)
- HIGH: describe-instances IamInstanceProfile (reveals role name and instance-role mapping)
- MEDIUM: IAM Policy Simulator results (reveals effective permissions)

## Prohibited Actions
- NEVER suggest attaching AdministratorAccess or PowerUserAccess to fix permission issues
- NEVER suggest modifying SCPs without organization administrator approval
- NEVER suggest embedding long-lived credentials on the instance instead of using instance profiles
- NEVER suggest disabling permission boundaries to work around access issues

## Phase 3 — Rollback
- If IAM policy was added to role: detach policy with `detach-role-policy`
- If inline policy was added: delete with `delete-role-policy`
- If instance profile was associated: disassociate with `disassociate-iam-instance-profile`
- If KMS key policy was modified: update key policy to previous version
- If role trust policy was modified: revert trust policy to previous version

## Output Format

```yaml
root_cause: "<no_profile|missing_permission|scp_deny|permission_boundary|resource_policy>"
evidence:
  - type: iam_policy
    content: "<policy analysis showing the gap>"
severity: HIGH
mitigation:
  immediate: "Attach profile or add missing permissions"
  long_term: "Use least-privilege policies, test with IAM Policy Simulator"
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
