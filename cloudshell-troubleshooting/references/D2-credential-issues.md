---
title: "D2 — Credential Issues"
description: "Diagnose CloudShell credential and authentication issues"
status: active
severity: HIGH
triggers: ["expired credentials", "invalid credentials", "token expired", "assume role failed in CloudShell"]
owner: devops-agent
objective: "Resolve credential and authentication issues within CloudShell sessions"
context: "CloudShell inherits console user credentials. Temporary credentials rotate automatically. Cross-account assume role requires proper trust policies."
---

## Phase 1 — Triage

MUST:
- Verify current identity: `aws sts get-caller-identity` (inside CloudShell)
- Check credential expiration: `aws sts get-session-token` will fail if credentials are expired
- For assume role issues: `aws sts assume-role --role-arn <role-arn> --role-session-name test --dry-run 2>&1`

SHOULD:
- Check if MFA is required for the operation: review IAM policy conditions
- Verify cross-account role trust policy includes the user's account

MAY:
- Review CloudTrail for AssumeRole failures: `aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventName,AttributeValue=AssumeRole --max-results 10`
- Check if session duration limits are causing premature expiration

## Phase 2 — Remediate

MUST:
- For expired credentials: close and reopen CloudShell session to get fresh credentials
- For assume role failures: verify trust policy on target role allows the CloudShell user
- Ensure role session duration is within allowed limits

SHOULD:
- Configure AWS CLI profiles in $HOME/.aws/config for cross-account access
- Use `aws sts assume-role` with appropriate session duration

MAY:
- Set up credential helper scripts in $HOME for frequently assumed roles
- Configure SSO profiles if using AWS IAM Identity Center

## Safety Ratings

```
safety_ratings:
  - "sts get-caller-identity: GREEN — read-only identity check"
  - "sts assume-role: YELLOW — obtains temporary credentials for another role"
  - "iam update-assume-role-policy: RED — modifies role trust relationships"
  - "aws configure: YELLOW — stores credential configuration locally"
```

## Escalation Conditions

- Credentials expire faster than expected (possible session policy issue)
- Cross-account role trust policy changes require target account admin
- MFA requirements cannot be satisfied in CloudShell
- Credential issues affect multiple users simultaneously

## Data Sensitivity

- `aws sts get-caller-identity`: LOW — returns account ID and ARN
- `aws sts assume-role`: HIGH — obtains temporary credentials
- `$HOME/.aws/config`: MEDIUM — contains role ARNs and account IDs
- `aws cloudtrail lookup-events`: MEDIUM — reveals authentication attempts

## Prohibited Actions

- NEVER store long-term access keys in CloudShell $HOME/.aws/credentials
- NEVER share assumed role credentials between users
- NEVER modify role trust policies to use wildcard principals
- NEVER bypass MFA requirements by modifying IAM policies

## Phase 3 — Rollback

1. If trust policy was modified incorrectly: restore original trust policy document
2. If AWS config was misconfigured: `rm $HOME/.aws/config` and reconfigure
3. If credentials were cached incorrectly: close all CloudShell sessions and reopen
4. If role was assumed with excessive permissions: revoke session with `aws iam put-role-policy` adding deny
5. Document credential configuration changes

## Common Issues

- symptoms: "ExpiredTokenException when running AWS CLI commands in CloudShell"
  diagnosis: "CloudShell session credentials have expired, usually after session timeout."
  resolution: "Close and reopen the CloudShell session. Credentials are automatically refreshed on new sessions."

## Output Format

```yaml
root_cause: "credentials — <specific_cause>"
evidence:
  - type: credential_error
    content: "<error details>"
severity: HIGH
mitigation:
  immediate: "Restore valid credentials"
  long_term: "Configure proper cross-account access patterns and credential management"
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
