---
title: "D1 — Permissions Errors"
description: "Diagnose CloudShell IAM permissions errors"
status: active
severity: HIGH
triggers: ["access denied", "not authorized", "permission denied", "cannot access CloudShell"]
owner: devops-agent
objective: "Resolve IAM permission issues preventing CloudShell access or operations"
context: "CloudShell requires cloudshell:* permissions to access. Operations inside CloudShell use the console user's IAM permissions. SCPs can block access."
---

## Phase 1 — Triage

MUST:
- Check CloudShell access permissions: `aws iam simulate-principal-policy --policy-source-arn <user-arn> --action-names cloudshell:CreateEnvironment cloudshell:StartEnvironment cloudshell:PutCredentials`
- Check for explicit deny in policies: `aws iam list-attached-user-policies --user-name <user>` and `aws iam list-user-policies --user-name <user>`
- For role-based access, check role policies: `aws iam list-attached-role-policies --role-name <role>`

SHOULD:
- Check SCPs that may block CloudShell: `aws organizations list-policies-for-target --target-id <account-id> --filter SERVICE_CONTROL_POLICY`
- Verify permission boundaries: `aws iam get-user --user-name <user> --query 'User.PermissionsBoundary'`

MAY:
- Review CloudTrail for authorization failures: `aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventName,AttributeValue=CreateEnvironment --max-results 10`
- Check if IP-based conditions in policies restrict CloudShell access

## Phase 2 — Remediate

MUST:
- Attach AWSCloudShellFullAccess managed policy for full access: `aws iam attach-user-policy --user-name <user> --policy-arn arn:aws:iam::aws:policy/AWSCloudShellFullAccess`
- Or create custom policy with specific cloudshell:* actions
- Remove explicit deny statements blocking CloudShell if appropriate

SHOULD:
- Use IAM groups for consistent CloudShell access across developers
- Update permission boundaries to allow cloudshell:* actions

MAY:
- Create a restricted CloudShell policy that allows access but blocks file transfer
- Set up CloudShell access as part of developer onboarding automation

## Safety Ratings

```
safety_ratings:
  - "simulate-principal-policy: GREEN — read-only permission test"
  - "list-attached-user-policies: GREEN — read-only policy enumeration"
  - "attach-user-policy: YELLOW — grants additional permissions"
  - "modify SCP: RED — affects all accounts in the organizational unit"
```

## Escalation Conditions

- SCP blocking CloudShell requires organization management account changes
- Permission boundary set by security team prevents CloudShell access
- IP-based conditions in policies conflict with user's network
- Multiple users affected by the same permission issue

## Data Sensitivity

- `aws iam simulate-principal-policy`: LOW — permission simulation
- `aws iam list-attached-user-policies`: MEDIUM — reveals policy assignments
- `aws iam get-policy-version`: MEDIUM — reveals policy content
- `aws organizations list-policies-for-target`: MEDIUM — reveals SCP configuration

## Prohibited Actions

- NEVER attach AdministratorAccess to fix CloudShell permission issues
- NEVER modify SCPs without organization admin approval
- NEVER remove permission boundaries without security team approval
- NEVER create inline policies when managed policies are available

## Phase 3 — Rollback

1. If wrong policy was attached: `aws iam detach-user-policy --user-name <user> --policy-arn <policy-arn>`
2. If SCP was modified: restore original SCP content from backup
3. If permission boundary was changed: restore original boundary ARN
4. If group membership was changed: `aws iam remove-user-from-group --user-name <user> --group-name <group>`
5. Document all IAM changes and notify security team

## Common Issues

- symptoms: "User can access AWS console but CloudShell shows 'You do not have required permissions'"
  diagnosis: "User lacks cloudshell:CreateEnvironment or cloudshell:StartEnvironment permission."
  resolution: "Attach AWSCloudShellFullAccess managed policy or add cloudshell:* to user's policy."

## Output Format

```yaml
root_cause: "permissions — <specific_cause>"
evidence:
  - type: permission_error
    content: "<error details>"
severity: HIGH
mitigation:
  immediate: "Grant required CloudShell permissions"
  long_term: "Include CloudShell access in standard developer IAM policies"
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
