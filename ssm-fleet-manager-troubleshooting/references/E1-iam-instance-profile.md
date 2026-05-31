---
title: "E1 — IAM Instance Profile Issues"
description: "Diagnose IAM instance profile misconfiguration preventing SSM management"
status: active
severity: HIGH
triggers: ["instance profile missing", "IAM permissions for SSM", "AmazonSSMManagedInstanceCore"]
owner: devops-agent
objective: "Resolve IAM instance profile issues blocking SSM Fleet Manager functionality"
context: "EC2 instances require an IAM instance profile with SSM permissions. The AmazonSSMManagedInstanceCore managed policy provides minimum required permissions. Custom policies must include ssm, ssmmessages, and ec2messages actions."
---

## Phase 1 — Triage

MUST:
- Check if instance has a profile: `aws ec2 describe-instances --instance-ids <instance-id> --query 'Reservations[*].Instances[*].IamInstanceProfile'`
- If profile exists, check the role policies: `aws iam list-attached-role-policies --role-name <role-name>`
- Verify the role has SSM permissions: `aws iam simulate-principal-policy --policy-source-arn <role-arn> --action-names ssm:UpdateInstanceInformation ssm:ListInstanceAssociations`

SHOULD:
- Check for inline policies that might deny SSM actions
- Verify the instance profile is not restricted by SCPs

MAY:
- Review CloudTrail for AssumeRole failures from the instance
- Check if the instance metadata service (IMDS) is accessible

## Phase 2 — Remediate

MUST:
- Create instance profile if missing: `aws iam create-instance-profile --instance-profile-name SSMInstanceProfile`
- Attach `AmazonSSMManagedInstanceCore` to the role: `aws iam attach-role-policy --role-name <role-name> --policy-arn arn:aws:iam::aws:policy/AmazonSSMManagedInstanceCore`
- Associate profile with instance: `aws ec2 associate-iam-instance-profile --instance-id <instance-id> --iam-instance-profile Name=SSMInstanceProfile`

SHOULD:
- Restart SSM Agent after attaching profile to pick up new credentials
- Verify the instance can assume the role via IMDS

MAY:
- Create a custom policy with least-privilege SSM permissions instead of the managed policy
- Enforce instance profiles via launch templates or Auto Scaling groups

## Common Issues

- symptoms: "Instance has a profile but SSM shows 'Insufficient permissions'"
  diagnosis: "Role is missing AmazonSSMManagedInstanceCore or has explicit deny."
  resolution: "Attach AmazonSSMManagedInstanceCore policy. Check for deny statements in inline or SCP policies."

- symptoms: "Instance profile attached but SSM Agent cannot get credentials"
  diagnosis: "IMDS (Instance Metadata Service) is disabled or restricted."
  resolution: "Verify IMDS is enabled. For IMDSv2, ensure SSM Agent supports token-based metadata requests."

## Safety Ratings

```
safety_ratings:
  - "describe-instances: GREEN — read-only instance metadata"
  - "list-attached-role-policies: GREEN — read-only policy listing"
  - "attach-role-policy: YELLOW — adds permissions to IAM role"
  - "associate-iam-instance-profile: YELLOW — modifies instance IAM configuration"
```

## Escalation Conditions

- SCP at the organization level blocks SSM permissions and cannot be modified
- Instance profile role is shared with other services and adding SSM policy may violate least privilege
- IMDS is disabled by security policy and cannot be re-enabled
- Custom IAM policy required due to compliance restrictions on managed policies

## Data Sensitivity

- `aws ec2 describe-instances`: MEDIUM — reveals instance configuration and IAM profile
- `aws iam list-attached-role-policies`: MEDIUM — reveals IAM policy attachments
- `aws iam simulate-principal-policy`: LOW — tests permissions without making changes
- `aws cloudtrail lookup-events`: MEDIUM — reveals IAM activity and principal identities

## Prohibited Actions

- NEVER attach AdministratorAccess policy to resolve SSM permission issues
- NEVER remove existing policies from a shared role without understanding dependencies
- NEVER disable IMDS to troubleshoot credential issues
- NEVER create instance profiles with overly broad permissions (s3:*, ec2:*)

## Phase 3 — Rollback

1. If an incorrect policy was attached, detach it: `aws iam detach-role-policy --role-name <role-name> --policy-arn <policy-arn>`
2. If an instance profile was associated incorrectly, disassociate: `aws ec2 disassociate-iam-instance-profile --association-id <id>`
3. If a new role was created, delete it after removing all policies and instance profile associations
4. Verify the instance returns to its previous IAM configuration
5. Document all IAM changes for security audit trail

## Output Format

```yaml
root_cause: "iam_instance_profile — <specific_cause>"
evidence:
  - type: iam_error
    content: "<error details>"
severity: HIGH
mitigation:
  immediate: "Fix IAM instance profile"
  long_term: "Enforce SSM profiles via launch templates"
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
  - "NEVER suggest overly broad SSM document permissions"
  - "NEVER suggest disabling session logging"
  - "NEVER suggest sharing SSM documents publicly"
