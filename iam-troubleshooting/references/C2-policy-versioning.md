---
title: "C2 — Policy Versioning Issues"
description: "Diagnose issues with IAM managed policy versions, the 5-version limit, and default version management"
status: active
severity: MEDIUM
triggers:
  - "policy version"
  - "5 version limit"
  - "default version"
  - "wrong policy version"
  - "policy not taking effect"
owner: devops-agent
objective: "Resolve policy versioning issues including version limit errors and incorrect default version"
context: "IAM managed policies support up to 5 versions. Only the default version is active. When creating a new version at the limit, you must delete an old version first. A common mistake is creating a new version without setting it as default."
---

## Phase 1 — Triage

MUST:
- List policy versions: `aws iam list-policy-versions --policy-arn <arn>`
- Check which version is the default: look for `IsDefaultVersion: true` in the output
- Get the default version content: `aws iam get-policy-version --policy-arn <arn> --version-id <default-version>`

SHOULD:
- Compare the default version with the latest version to see if the intended changes are active
- Check if the policy has 5 versions (at the limit): count versions in list-policy-versions output
- Verify the policy ARN is correct — different policies can have similar names

MAY:
- Review all versions to understand the change history
- Check CloudTrail for CreatePolicyVersion events to see who made changes

## Phase 2 — Remediate

MUST:
- If wrong default version: `aws iam set-default-policy-version --policy-arn <arn> --version-id <correct-version>`
- If at 5-version limit, delete an old non-default version first: `aws iam delete-policy-version --policy-arn <arn> --version-id <old-version>`
- Then create the new version: `aws iam create-policy-version --policy-arn <arn> --policy-document file://policy.json --set-as-default`

SHOULD:
- Always use `--set-as-default` when creating new versions to avoid forgetting
- Clean up old versions proactively to stay below the limit
- Document version changes for audit purposes

MAY:
- Automate version cleanup in CI/CD pipelines
- Use infrastructure-as-code (CloudFormation/Terraform) to manage policy versions

## Common Issues

- symptoms: "Created new policy version but permissions haven't changed"
  diagnosis: "The new version was not set as default. Only the default version is active."
  resolution: "Set the new version as default: `aws iam set-default-policy-version --policy-arn <arn> --version-id <new-version>`"

- symptoms: "Cannot create new policy version — LimitExceeded"
  diagnosis: "The policy already has 5 versions (maximum)."
  resolution: "Delete a non-default version first, then create the new version."

- symptoms: "Policy changes keep reverting"
  diagnosis: "An automation tool (CloudFormation, Terraform) is resetting the default version."
  resolution: "Make changes through the IaC tool, not directly via CLI/console."

## Output Format

```yaml
root_cause: "policy_versioning — <specific_cause>"
evidence:
  - type: version_list
    content: "<list of versions with default marker>"
  - type: default_version
    content: "<current default version content>"
severity: MEDIUM
mitigation:
  immediate: "Set correct default version or clean up old versions"
  long_term: "Manage policies through IaC and automate version cleanup"
```


## Safety Ratings

safety_ratings:
  - "Policy version management (changing default version, deleting old versions): YELLOW — Setting the wrong default version can immediately change effective permissions for all attached principals"

## Escalation Conditions

escalation_conditions:
  - "Escalate to a human if the policy is attached to production roles or users"
  - "Escalate if the policy has Deny statements and the version change could remove them"
  - "Escalate if the policy is managed by IaC (CloudFormation/Terraform) and manual changes could cause drift"

## Rollback

If remediation makes things worse:
1. Document the change that was made
2. Set the previous version as default using `aws iam set-default-policy-version --policy-arn <arn> --version-id <previous-version-id>`
3. Verify access is restored to pre-change state

## Data Sensitivity

- `aws iam get-policy-version` returns full policy documents which may contain sensitive resource ARNs and account IDs — classification: HIGH
- `aws iam list-policy-versions` reveals the policy change history — classification: MEDIUM

## Prohibited Actions

- NEVER suggest `"Action": "*"` or `"Resource": "*"` in any policy fix
- NEVER suggest granting AdministratorAccess to resolve permission issues
- NEVER suggest removing permission boundaries to fix access denied
- Always identify the SPECIFIC denied action from CloudTrail before suggesting a fix
- Always scope fixes to the minimum required permissions (least privilege)

## Cleanup

- Delete old non-default policy versions that are no longer needed to stay below the 5-version limit: `aws iam delete-policy-version --policy-arn <arn> --version-id <old-version>`

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
