---
title: "D1 — AWS Chatbot Channel IAM Role Issues"
description: "Diagnose channel IAM role configuration problems"
status: active
severity: HIGH
triggers:
  - "channel role"
  - "IAM role error"
  - "role not found"
  - "trust policy"
  - "chatbot role"
owner: devops-agent
objective: "Identify and resolve channel IAM role issues affecting Chatbot operations"
context: "Each Chatbot channel configuration has an IAM role that determines what AWS actions can be performed via CLI commands. The role must trust chatbot.amazonaws.com and have appropriate permissions."
---

## Phase 1 — Triage

MUST:
- Get channel IAM role ARN: `aws chatbot describe-slack-channel-configurations --chat-configuration-arn <arn>` (look at iamRoleArn)
- Check role exists: `aws iam get-role --role-name <role-name>`
- Check trust policy: `aws iam get-role --role-name <role-name> --query 'Role.AssumeRolePolicyDocument'`
- List attached policies: `aws iam list-attached-role-policies --role-name <role-name>`
- List inline policies: `aws iam list-role-policies --role-name <role-name>`

SHOULD:
- Verify trust policy includes chatbot.amazonaws.com
- Check for permission boundaries
- Verify policies grant required permissions for intended commands

MAY:
- Compare with AWS managed Chatbot policies
- Check for SCPs that may restrict the role

## Phase 2 — Remediate

MUST:
- Ensure trust policy allows chatbot.amazonaws.com to assume the role
- Attach policies with required permissions for CLI commands
- Use the correct role ARN in the channel configuration

SHOULD:
- Follow least-privilege principle — only grant permissions needed for intended commands
- Use separate roles for different channels with different permission needs
- Document role purpose and permissions

MAY:
- Use AWS managed policies as a starting point (e.g., ReadOnlyAccess)
- Implement role permission auditing

## Common Issues

- symptoms: "Channel configuration fails with role error"
  diagnosis: "IAM role doesn't exist or trust policy missing chatbot.amazonaws.com."
  resolution: "Create role with chatbot.amazonaws.com in trust policy."

- symptoms: "Commands fail with assume role error"
  diagnosis: "Trust policy doesn't allow chatbot.amazonaws.com."
  resolution: "Update trust policy: {\"Effect\":\"Allow\",\"Principal\":{\"Service\":\"chatbot.amazonaws.com\"},\"Action\":\"sts:AssumeRole\"}"

- symptoms: "Role exists but commands return no permissions"
  diagnosis: "Role has no attached policies or policies are empty."
  resolution: "Attach appropriate IAM policies to the role."

## Output Format

```yaml
root_cause: "channel_role_issue — <specific_cause>"
evidence:
  - type: role_config
    content: "<trust policy and attached policies>"
  - type: channel_config
    content: "<channel IAM role ARN>"
severity: HIGH
mitigation:
  immediate: "Fix trust policy or attach required permissions"
  long_term: "Standardize Chatbot IAM roles with IaC"
```


## Safety Ratings

| Phase 2 Action | safety_rating |
|---|---|
| Ensure trust policy allows chatbot.amazonaws.com to assume the role | YELLOW |
| Attach policies with required permissions for CLI commands | YELLOW |
| Use the correct role ARN in the channel configuration | GREEN |
| Follow least-privilege principle | GREEN |
| Use separate roles for different channels with different permission needs | GREEN |
| Document role purpose and permissions | GREEN |
| Use AWS managed policies as a starting point | GREEN |
| Implement role permission auditing | GREEN |

## Escalation Conditions

- Fix requires modifying IAM role permissions for chat commands

## Data Sensitivity

- **MEDIUM**: notification content, channel configurations
- IAM role trust policies and permissions define the security boundary for chat-based AWS access

## Prohibited Actions

- NEVER suggest granting AdministratorAccess to chatbot role

## Phase 3 — Rollback

1. If the trust policy was modified, revert to the previous trust policy document
2. If policies were attached to the role, detach them and restore previous policy attachments
3. If a new role was created, delete it and update channel configuration to use the previous role
4. Verify rollback by confirming `aws iam get-role --role-name <role>` shows expected configuration

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
