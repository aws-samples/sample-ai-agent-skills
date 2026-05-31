---
title: "A1 — Session Launch Failures"
description: "Diagnose CloudShell session launch failures"
status: active
severity: HIGH
triggers: ["cloudshell won't open", "session failed to start", "unable to start environment"]
owner: devops-agent
objective: "Resolve CloudShell session launch issues"
context: "CloudShell sessions require IAM permissions, regional availability, and may be blocked by SCPs or network policies."
---

## Phase 1 — Triage

MUST:
- Verify CloudShell is available in the current region
- Check IAM permissions: `aws iam simulate-principal-policy --policy-source-arn <user-arn> --action-names cloudshell:CreateEnvironment cloudshell:StartEnvironment cloudshell:GetEnvironmentStatus`
- Check for SCPs blocking CloudShell: `aws organizations list-policies --filter SERVICE_CONTROL_POLICY`

SHOULD:
- Review CloudTrail for CreateEnvironment errors: `aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventName,AttributeValue=CreateEnvironment --max-results 10`
- Check concurrent environment count against the 10-environment limit

MAY:
- Test from a different browser or incognito mode to rule out browser issues
- Check browser console for WebSocket connection errors

## Phase 2 — Remediate

MUST:
- Attach AWSCloudShellFullAccess managed policy if missing: `aws iam attach-user-policy --user-name <user> --policy-arn arn:aws:iam::aws:policy/AWSCloudShellFullAccess`
- Remove SCP restrictions blocking cloudshell:* actions if appropriate
- Switch to a supported region if current region lacks CloudShell

SHOULD:
- Delete unused environments if at the 10-environment limit: `aws cloudshell delete-environment --environment-id <env-id>`
- Clear browser cache and cookies for the AWS console domain

MAY:
- Configure VPN or proxy exceptions for CloudShell WebSocket endpoints
- Use AWS CLI directly if CloudShell console access is persistently blocked

## Safety Ratings

```
safety_ratings:
  - "simulate-principal-policy: GREEN — read-only permission check"
  - "attach-user-policy: YELLOW — modifies IAM permissions"
  - "delete-environment: RED — permanently removes environment and persistent storage"
  - "organizations list-policies: GREEN — read-only SCP enumeration"
```

## Escalation Conditions

- Launch failures persist after IAM permission fixes across multiple users
- SCP modifications require organization management account access
- Network firewall rules blocking WebSocket connections require infrastructure team
- Regional service outage suspected — check AWS Health Dashboard

## Data Sensitivity

- `aws iam simulate-principal-policy`: LOW — tests permissions without changes
- `aws cloudtrail lookup-events`: MEDIUM — may reveal user identities and actions
- `aws iam attach-user-policy`: HIGH — modifies IAM permissions
- `aws cloudshell delete-environment`: HIGH — destroys persistent storage

## Prohibited Actions

- NEVER delete CloudShell environments without confirming backup of $HOME contents
- NEVER attach AdministratorAccess to resolve CloudShell permission issues
- NEVER disable SCPs without organization admin approval
- NEVER share CloudShell session URLs or screenshots containing credentials

## Phase 3 — Rollback

1. If IAM policy was attached incorrectly: `aws iam detach-user-policy --user-name <user> --policy-arn <policy-arn>`
2. If an environment was deleted by mistake, create a new one (data is not recoverable)
3. If SCP was modified: restore the original SCP from version history or backup
4. Document the rollback action and verify the user can still access required services
5. Notify the security team if IAM changes were made to production accounts

## Common Issues

- symptoms: "CloudShell shows 'Unable to start the environment' with no details"
  diagnosis: "IAM user lacks cloudshell:CreateEnvironment or cloudshell:StartEnvironment permission."
  resolution: "Attach AWSCloudShellFullAccess managed policy or add specific cloudshell:* permissions."

## Output Format

```yaml
root_cause: "session_launch — <specific_cause>"
evidence:
  - type: launch_error
    content: "<error details>"
severity: HIGH
mitigation:
  immediate: "Fix launch blocker"
  long_term: "Ensure IAM policies include CloudShell access for developer roles"
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
