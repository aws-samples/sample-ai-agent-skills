---
title: "G1 — Access Key Rotation"
description: "Diagnose access key rotation issues, deactivation, and last-used tracking"
status: active
severity: MEDIUM
triggers:
  - "access key"
  - "key rotation"
  - "old access key"
  - "deactivate key"
  - "access key not working"
owner: devops-agent
objective: "Manage access key lifecycle including rotation, deactivation, and cleanup"
context: "IAM users can have up to 2 access keys. Best practice is to rotate keys regularly: create new key, update applications, verify new key works, deactivate old key, then delete old key. Keys remain active until explicitly deactivated. Last-used data helps identify unused keys."
---

## Phase 1 — Triage

MUST:
- List access keys for the user: `aws iam list-access-keys --user-name <user>`
- Check key status (Active/Inactive): look at Status field in list-access-keys output
- Check when each key was last used: `aws iam get-access-key-last-used --access-key-id <key-id>`
- Check key age: compare CreateDate with current date

SHOULD:
- Generate a credential report for all users: `aws iam generate-credential-report` then `aws iam get-credential-report`
- Identify keys older than 90 days (common rotation policy)
- Check if the user has console access in addition to programmatic access

MAY:
- Use AWS Config rule `access-keys-rotated` to monitor key age across the account
- Check IAM Access Analyzer for unused access keys

## Phase 2 — Remediate

MUST:
- To rotate: create new key first: `aws iam create-access-key --user-name <user>`
- Update all applications/services using the old key with the new key
- Verify the new key works in all locations
- Deactivate the old key: `aws iam update-access-key --user-name <user> --access-key-id <old-key> --status Inactive`
- After confirming no issues, delete the old key: `aws iam delete-access-key --user-name <user> --access-key-id <old-key>`

SHOULD:
- Never delete the old key immediately — deactivate first and wait
- Use IAM roles (instance profiles, IRSA, task roles) instead of access keys where possible
- Implement automated key rotation alerts

MAY:
- Use AWS Secrets Manager for automatic key rotation
- Implement a key rotation Lambda function on a schedule

## Common Issues

- symptoms: "Access key stopped working suddenly"
  diagnosis: "The key was deactivated or deleted, or the user's IAM policies changed."
  resolution: "Check key status with list-access-keys. If Inactive, reactivate or use the other key."

- symptoms: "Cannot create new access key — LimitExceeded"
  diagnosis: "The user already has 2 access keys (maximum)."
  resolution: "Delete one of the existing keys before creating a new one."

- symptoms: "Rotated key but application still uses old key"
  diagnosis: "The application configuration was not updated, or credentials are cached."
  resolution: "Update the application configuration and restart to clear cached credentials."

## Output Format

```yaml
root_cause: "access_key — <specific_cause>"
evidence:
  - type: key_status
    content: "<key_id>: <Active|Inactive>, created: <date>, last_used: <date>"
  - type: key_age
    content: "<days> days old"
severity: MEDIUM
mitigation:
  immediate: "Rotate or reactivate the key as needed"
  long_term: "Migrate to IAM roles and implement automated rotation alerts"
```


## Safety Ratings

safety_ratings:
  - "Access key rotation (key deactivation and deletion): YELLOW — Deactivating or deleting an access key can immediately break applications and services that depend on it"

## Escalation Conditions

escalation_conditions:
  - "Escalate to a human if the access key is used by a production application or CI/CD pipeline"
  - "Escalate if the key is the only active key for the user and deactivation would lock out access"
  - "Escalate if the key is older than 1 year (may be deeply embedded in multiple systems)"

## Rollback

If remediation makes things worse:
1. Document the change that was made
2. Reactivate the old access key using `aws iam update-access-key --user-name <user> --access-key-id <old-key> --status Active`
3. Verify access is restored to pre-change state

## Data Sensitivity

- `aws iam list-access-keys` returns access key IDs and creation dates — classification: MEDIUM
- `aws iam get-access-key-last-used` reveals when and from which service/region the key was last used — classification: MEDIUM
- `aws iam get-credential-report` returns a full credential report for all IAM users including key ages, MFA status, and last activity — classification: HIGH

## Prohibited Actions

- NEVER suggest `"Action": "*"` or `"Resource": "*"` in any policy fix
- NEVER suggest granting AdministratorAccess to resolve permission issues
- NEVER suggest removing permission boundaries to fix access denied
- Always identify the SPECIFIC denied action from CloudTrail before suggesting a fix
- Always scope fixes to the minimum required permissions (least privilege)

## Cleanup

- Delete the deactivated old access key after confirming the new key works: `aws iam delete-access-key --user-name <user> --access-key-id <old-key>`
- Remove any cached or stored copies of the old access key from configuration files, secrets managers, or environment variables

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
