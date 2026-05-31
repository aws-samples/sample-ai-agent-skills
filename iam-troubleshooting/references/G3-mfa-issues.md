---
title: "G3 — MFA Enforcement Issues"
description: "Diagnose MFA enforcement, condition keys, and GetSessionToken requirements"
status: active
severity: HIGH
triggers:
  - "MFA"
  - "multi-factor"
  - "MFA required"
  - "aws:MultiFactorAuthPresent"
  - "GetSessionToken"
  - "MFA denied"
owner: devops-agent
objective: "Resolve MFA enforcement issues including condition key misuse and CLI MFA workflows"
context: "MFA enforcement uses IAM condition keys: aws:MultiFactorAuthPresent and aws:MultiFactorAuthAge. MFA conditions only work with temporary credentials (STS). Long-term access keys do NOT include MFA context — users must call GetSessionToken with MFA first, then use the temporary credentials."
---

## Phase 1 — Triage

MUST:
- Check if the policy requires MFA: look for `aws:MultiFactorAuthPresent` or `aws:MultiFactorAuthAge` conditions
- Determine credential type: long-term access key or temporary (STS) credentials
- Check if the user has MFA configured: `aws iam list-mfa-devices --user-name <user>`
- If using CLI: verify the user called GetSessionToken with MFA first

SHOULD:
- Check the exact condition operator: `BoolIfExists` vs `Bool` behaves differently
- Verify the MFA device is active and synced
- Check CloudTrail for the denied event to see if MFA context was present

MAY:
- Check if the policy uses `aws:MultiFactorAuthAge` for time-based MFA requirements
- Verify virtual MFA device configuration

## Phase 2 — Remediate

MUST:
- For CLI users: obtain MFA-authenticated temporary credentials first:
  ```bash
  aws sts get-session-token \
    --serial-number arn:aws:iam::<account>:mfa/<user> \
    --token-code <mfa-code>
  ```
  Then use the returned temporary credentials for subsequent calls
- Ensure the MFA condition uses `BoolIfExists` (not `Bool`) if you want to allow service roles without MFA

SHOULD:
- Use `aws:MultiFactorAuthPresent` with `BoolIfExists` for Deny statements:
  ```json
  {
    "Effect": "Deny",
    "Action": "*",
    "Resource": "*",
    "Condition": {"BoolIfExists": {"aws:MultiFactorAuthPresent": "false"}}
  }
  ```
- Allow users to manage their own MFA without MFA (bootstrap exception)
- Configure CLI profiles with MFA using `mfa_serial` in `~/.aws/config`

MAY:
- Use hardware MFA for high-security accounts
- Implement MFA for specific high-risk actions only (e.g., IAM changes, billing)

## Common Issues

- symptoms: "MFA policy denies access even when using CLI with MFA device configured"
  diagnosis: "Long-term access keys don't carry MFA context. Must use GetSessionToken first."
  resolution: "Call sts get-session-token with --serial-number and --token-code, then use the temporary credentials."

- symptoms: "MFA condition denies service roles and Lambda functions"
  diagnosis: "Using `Bool` instead of `BoolIfExists`. Service roles don't have MFA context, so `Bool` treats missing key as not matching."
  resolution: "Use `BoolIfExists` which treats a missing key as matching the condition."

- symptoms: "MFA works in console but not in CLI"
  diagnosis: "Console automatically prompts for MFA. CLI requires explicit GetSessionToken call."
  resolution: "Configure CLI profile with mfa_serial or use a credential helper that prompts for MFA."

## Output Format

```yaml
root_cause: "mfa_issue — <specific_cause>"
evidence:
  - type: mfa_condition
    content: "<condition from policy>"
  - type: credential_type
    content: "long_term | temporary"
  - type: mfa_device
    content: "<MFA device ARN and status>"
severity: HIGH
mitigation:
  immediate: "Use GetSessionToken with MFA or fix condition key usage"
  long_term: "Implement CLI MFA profiles and use BoolIfExists consistently"
```


## Safety Ratings

safety_ratings:
  - "MFA configuration changes (modifying MFA enforcement policies or device settings): YELLOW — MFA changes affect user authentication requirements; incorrect changes can lock users out or weaken security"

## Escalation Conditions

escalation_conditions:
  - "Escalate to a human if the MFA policy is enforced organization-wide via SCP"
  - "Escalate if the change involves removing or weakening MFA requirements"
  - "Escalate if the user's MFA device is lost or compromised (requires identity verification)"

## Rollback

If remediation makes things worse:
1. Document the change that was made
2. Restore the original MFA enforcement policy by reverting the policy version using `aws iam create-policy-version --set-as-default` with the original policy document, or re-enable the MFA device using `aws iam enable-mfa-device`
3. Verify access is restored to pre-change state

## Data Sensitivity

- `aws iam list-mfa-devices` returns MFA device serial numbers and types — classification: MEDIUM
- MFA condition policies reveal the organization's authentication requirements — classification: MEDIUM
- `aws sts get-session-token` with MFA returns temporary credentials — classification: HIGH

## Prohibited Actions

- NEVER suggest `"Action": "*"` or `"Resource": "*"` in any policy fix
- NEVER suggest granting AdministratorAccess to resolve permission issues
- NEVER suggest removing permission boundaries to fix access denied
- Always identify the SPECIFIC denied action from CloudTrail before suggesting a fix
- Always scope fixes to the minimum required permissions (least privilege)

## Cleanup

- Deactivate and delete old MFA devices that have been replaced: `aws iam deactivate-mfa-device --user-name <user> --serial-number <arn>` followed by `aws iam delete-virtual-mfa-device --serial-number <arn>`

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
