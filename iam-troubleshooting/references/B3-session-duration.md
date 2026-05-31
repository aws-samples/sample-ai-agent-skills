---
title: "B3 — Session Duration Issues"
description: "Diagnose issues with role session duration limits and MaxSessionDuration"
status: active
severity: MEDIUM
triggers:
  - "session duration"
  - "MaxSessionDuration"
  - "session expired too soon"
  - "DurationSeconds"
  - "token expired"
owner: devops-agent
objective: "Identify and fix session duration misconfigurations causing premature credential expiry"
context: "Role sessions have configurable duration via MaxSessionDuration (role setting, 1-12 hours) and DurationSeconds (AssumeRole parameter). The effective duration is the minimum of the two. Chained roles are limited to 1 hour. Federation sessions have separate limits."
---

## Phase 1 — Triage

MUST:
- Check the role's MaxSessionDuration: `aws iam get-role --role-name <role> --query 'Role.MaxSessionDuration'`
- Check what DurationSeconds was requested in the AssumeRole call (CloudTrail): `aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventName,AttributeValue=AssumeRole`
- Verify if this is a chained role assumption (role assuming another role — limited to 1 hour)

SHOULD:
- Check if the session is from federation (SAML/OIDC have different limits)
- Verify the application is refreshing credentials before expiry
- Check if a session policy was passed that might affect behavior

MAY:
- Review application logs for credential refresh patterns
- Check SDK configuration for credential provider chain settings

## Phase 2 — Remediate

MUST:
- If MaxSessionDuration is too low, increase it: `aws iam update-role --role-name <role> --max-session-duration <seconds>`
  - Valid range: 3600 (1 hour) to 43200 (12 hours)
- If DurationSeconds in AssumeRole call exceeds MaxSessionDuration, reduce DurationSeconds or increase MaxSessionDuration
- For chained roles: the 1-hour limit cannot be changed — redesign to avoid chaining

SHOULD:
- Implement credential refresh logic in applications before session expiry
- Set MaxSessionDuration to the minimum needed for the use case
- Use instance profiles or IRSA for long-running workloads instead of manual AssumeRole

MAY:
- Use AWS SSO/Identity Center for longer session management
- Implement automatic credential rotation in application code

## Common Issues

- symptoms: "AssumeRole fails with 'DurationSeconds exceeds the MaxSessionDuration'"
  diagnosis: "The requested DurationSeconds is greater than the role's MaxSessionDuration."
  resolution: "Either increase MaxSessionDuration on the role or decrease DurationSeconds in the call."

- symptoms: "Credentials expire after 1 hour even though MaxSessionDuration is 12 hours"
  diagnosis: "This is a chained role assumption. Chained roles are hard-limited to 1 hour."
  resolution: "Assume the target role directly from the original identity instead of chaining through an intermediate role."

- symptoms: "SAML federation session expires after 1 hour"
  diagnosis: "SAML SessionDuration attribute is not set or set to 3600. Default is 1 hour."
  resolution: "Set the SessionDuration SAML attribute in the IdP to the desired value (up to 12 hours)."

## Output Format

```yaml
root_cause: "session_duration — <specific_cause>"
evidence:
  - type: role_config
    content: "MaxSessionDuration: <value>"
  - type: assume_role_call
    content: "DurationSeconds: <value>"
  - type: session_type
    content: "direct | chained | federation"
severity: MEDIUM
mitigation:
  immediate: "Adjust MaxSessionDuration or DurationSeconds"
  long_term: "Implement proper credential refresh and avoid role chaining"
```


## Safety Ratings

| Action | Rating | Justification |
|--------|--------|---------------|
| `aws iam get-role` | GREEN | Read-only diagnostic |
| `aws cloudtrail lookup-events` | GREEN | Read-only diagnostic |
| `aws sts get-caller-identity` | GREEN | Read-only diagnostic |
| `aws iam update-role --max-session-duration` | YELLOW | Extends credential validity window — security implications |

## Escalation Conditions

- Remediation requires modifying IAM policies in a production account
- Fix involves granting cross-account access
- Root cause involves the management account
- Multiple principals are affected
- The denied action involves security-sensitive services (KMS, STS, Organizations)

## Rollback

1. **Before any change**: Document current state with `aws iam get-policy-version` or `aws iam get-role`
2. **After change**: Verify the fix resolves the issue without granting excessive permissions
3. **If change causes issues**: Revert to the previous policy version or detach the modified policy
4. **Cleanup**: Remove any temporary policies or test principals created during investigation

## Data Sensitivity

| Command | Sensitivity | Handling |
|---------|------------|----------|
| `get-account-authorization-details` | HIGH | Contains all IAM entities — redact from stored output |
| `get-policy-version` | HIGH | Contains full policy document — summarize, don't reproduce |
| `simulate-principal-policy` | MEDIUM | Shows effective permissions — summarize results |
| `list-*` | LOW | Resource listings — safe to include |

## Prohibited Actions

- NEVER suggest `"Action": "*"` or `"Resource": "*"` in any policy fix
- NEVER suggest granting `AdministratorAccess` to resolve permission issues
- NEVER suggest removing permission boundaries to fix access denied
- NEVER suggest `"Principal": "*"` without restrictive Condition keys
- NEVER suggest disabling MFA requirements as a troubleshooting step
- ALWAYS identify the specific denied action from CloudTrail before suggesting a fix
- ALWAYS use least-privilege: grant only the specific action on the specific resource

## Cleanup

- No temporary resources are created during session duration troubleshooting

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
  - command: "get-account-authorization-details"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"
  - command: "get-policy-version"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"
  - command: "get-role"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"

## Prohibited Actions

prohibited_actions:
  - "NEVER suggest Action: * or Resource: * in any policy"
  - "NEVER suggest removing permission boundaries"
  - "NEVER suggest AdministratorAccess for troubleshooting"

## Phase 3 � Rollback

- Document the current state before making any changes
- If remediation worsens the issue, revert to the documented pre-change state
- Remove any temporary resources created during investigation
- Verify the system returns to its previous functional state after rollback
