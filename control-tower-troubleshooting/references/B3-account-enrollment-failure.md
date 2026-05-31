---
title: "B3 — Account Enrollment Failure"
description: "Diagnose failures when enrolling existing AWS accounts into Control Tower"
status: active
severity: HIGH
triggers:
  - "Account enrollment failed"
  - "Cannot enroll account"
  - "Enrollment error"
owner: devops-agent
objective: "Identify why account enrollment failed and complete enrollment"
context: "Enrolling an existing account brings it under CT governance by deploying baseline StackSets, Config rules, and CloudTrail configuration. The account must have the AWSControlTowerExecution role and be in a registered OU."
---

## Phase 1 — Triage

MUST:
- Verify the AWSControlTowerExecution role exists in the target account with correct trust policy
- Check that the target OU is registered with Control Tower
- Check for pre-existing Config recorders or delivery channels in the target account
- Verify the account is a member of the Organization (not standalone)

SHOULD:
- Check if the account has existing SCPs that might block CT operations
- Verify no CloudFormation stacks conflict with CT baseline stacks
- Check if the account was previously enrolled and unenrolled (residual resources)

MAY:
- Check CloudTrail for EnrollAccount API call details
- Verify the account is not suspended or in a pending close state

## Phase 2 — Remediate

MUST:
- Create AWSControlTowerExecution role if missing (trust management account)
- Move account to a registered OU if not already there
- Remove conflicting Config recorders if present
- Clean up residual CT resources from previous enrollment

SHOULD:
- Delete existing AWSControlTower* stacks in the target account before retrying
- Verify the account can assume the AWSControlTowerExecution role from management account

MAY:
- Use the CT console "Enroll account" feature for guided enrollment
- Contact AWS Support for accounts stuck in enrollment

## Common Issues

- symptoms: "Enrollment fails with role not found"
  diagnosis: "AWSControlTowerExecution role does not exist in the target account."
  resolution: "Create the role with trust policy allowing the management account to assume it."

- symptoms: "Enrollment fails with Config recorder conflict"
  diagnosis: "The account already has an AWS Config recorder."
  resolution: "Delete the existing Config recorder and delivery channel, then retry."

- symptoms: "Enrollment fails with OU not registered"
  diagnosis: "The target OU is not registered with Control Tower."
  resolution: "Register the OU with Control Tower first, then enroll the account."

## Safety Ratings

```
safety_ratings:
  - "describe-organization: GREEN — read-only org metadata"
  - "list-policies-for-target: GREEN — read-only SCP listing"
  - "create AWSControlTowerExecution role: YELLOW — recoverable, can be deleted if needed"
  - "delete Config recorder: YELLOW — recoverable, CT will recreate on enrollment"
  - "move account to registered OU: YELLOW — recoverable, account can be moved back"
  - "enroll existing account: RED — deploys CT baselines, modifies Config/CloudTrail/IAM in the account"
```

## Escalation Conditions

- Fix requires updating the landing zone version
- Fix requires modifying CT-managed SCPs
- Account has existing governance from another tool that conflicts with CT
- Fix requires re-registering OUs in production
- Enrollment fails repeatedly with no clear error

## Data Sensitivity

- HIGH: Organization account list (all account IDs and emails)
- HIGH: SCP policy documents (expose security boundaries)
- MEDIUM: OU structure, control enablement status
- MEDIUM: IAM role trust policies (expose cross-account access patterns)

## Prohibited Actions

- "NEVER suggest directly modifying CT-managed CloudFormation stacks"
- "NEVER suggest directly editing CT-managed SCPs"
- "NEVER suggest deleting the Security OU or shared accounts"
- "NEVER suggest disabling CloudTrail organization trail"
- "NEVER suggest removing CT IAM roles"

## Phase 3 — Rollback

- If enrollment fails partway, clean up residual CT resources (AWSControlTower* stacks) in the target account before retrying
- "If OU re-registration fails, document the error and do NOT retry without investigating"
- "If landing zone update fails, do NOT attempt to roll back manually — contact AWS Support"
- Document the account's pre-enrollment state (existing Config, CloudTrail, IAM roles) before starting enrollment
- If the AWSControlTowerExecution role was created but enrollment failed, leave the role in place for retry — do NOT delete it prematurely

## Output Format

```yaml
root_cause: "enrollment_failure — <specific cause>"
evidence:
  - type: iam_role_check
    content: "<role existence and trust policy>"
  - type: config_state
    content: "<Config recorder status>"
severity: HIGH
mitigation:
  immediate: "Fix prerequisites and retry enrollment"
  long_term: "Create enrollment checklist and pre-validation script"
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
  - "NEVER suggest disabling mandatory controls"
  - "NEVER suggest removing the audit account"
  - "NEVER modify landing zone without understanding blast radius"
