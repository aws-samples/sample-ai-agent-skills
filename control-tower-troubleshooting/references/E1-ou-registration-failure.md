---
title: "E1 — OU Registration Failure"
description: "Diagnose failures when registering an OU with Control Tower"
status: active
severity: HIGH
triggers:
  - "OU registration failed"
  - "Cannot register OU"
  - "RegisterOrganizationalUnit error"
owner: devops-agent
objective: "Identify why OU registration failed and complete registration"
context: "Registering an OU deploys CT baseline StackSets, Config rules, and SCPs to all accounts in the OU. Failures are typically due to account-level issues, StackSet deployment failures, or missing prerequisites."
---

## Phase 1 — Triage

MUST:
- Check the registration operation status
- Verify the OU exists in Organizations: `aws organizations describe-organizational-unit --organizational-unit-id <ou-id>`
- Check if accounts in the OU have the AWSControlTowerExecution role
- Check StackSet deployment status for the OU's accounts

SHOULD:
- Verify no concurrent CT operations are running
- Check if any accounts in the OU have conflicting resources
- Verify the OU is a direct child of a registered parent (for nested OUs)

MAY:
- Check CloudTrail for the registration API call
- Review individual account enrollment status

## Phase 2 — Remediate

MUST:
- Ensure all accounts in the OU have the AWSControlTowerExecution role
- Fix any failed StackSet instances
- Retry registration after fixing prerequisites

SHOULD:
- Enroll problematic accounts individually before OU registration
- Remove accounts with issues from the OU temporarily

MAY:
- Register the OU with fewer accounts and add more later
- Contact AWS Support for persistent registration failures

## Common Issues

- symptoms: "Registration fails with role assumption error"
  diagnosis: "One or more accounts lack the AWSControlTowerExecution role."
  resolution: "Create the role in all accounts in the OU with trust to the management account."

- symptoms: "Registration times out"
  diagnosis: "Too many accounts or StackSet deployment is slow."
  resolution: "Wait for completion. Large OUs take longer. Check StackSet operation status."

## Safety Ratings

```
safety_ratings:
  - "describe-organizational-unit: GREEN — read-only OU metadata"
  - "list-stack-instances: GREEN — read-only StackSet instance listing"
  - "create AWSControlTowerExecution role: YELLOW — recoverable, can be deleted"
  - "fix failed StackSet instances: YELLOW — recoverable, CT will redeploy on re-register"
  - "register OU: RED — deploys CT baselines to all accounts in the OU, modifies Config/CloudTrail/IAM"
```

## Escalation Conditions

- Fix requires updating the landing zone version
- Fix requires modifying CT-managed SCPs
- Drift affects multiple OUs or accounts
- Fix requires re-registering OUs in production
- Registration fails repeatedly with no clear error

## Data Sensitivity

- HIGH: SCP policy documents (expose security boundaries)
- HIGH: Organization account list (all account IDs and emails)
- MEDIUM: OU structure, control enablement status
- MEDIUM: StackSet deployment details (account IDs, regions, failure reasons)

## Prohibited Actions

- "NEVER suggest directly modifying CT-managed CloudFormation stacks"
- "NEVER suggest directly editing CT-managed SCPs"
- "NEVER suggest deleting the Security OU or shared accounts"
- "NEVER suggest disabling CloudTrail organization trail"
- "NEVER suggest removing CT IAM roles"

## Phase 3 — Rollback

- "If OU re-registration fails, document the error and do NOT retry without investigating"
- "If landing zone update fails, do NOT attempt to roll back manually — contact AWS Support"
- If registration fails partway, check which accounts received baselines and which did not before retrying
- Document the OU membership and account states before starting registration
- If problematic accounts were removed from the OU to unblock registration, plan to add them back and enroll individually after

## Output Format

```yaml
root_cause: "ou_registration_failure — <specific cause>"
evidence:
  - type: controltower_operation
    content: "<registration status>"
  - type: stackset_status
    content: "<deployment details>"
severity: HIGH
mitigation:
  immediate: "Fix prerequisites and retry registration"
  long_term: "Pre-validate accounts before OU registration"
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
