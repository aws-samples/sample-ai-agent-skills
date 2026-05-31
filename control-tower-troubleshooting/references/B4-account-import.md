---
title: "B4 — Account Import Issues"
description: "Diagnose issues when importing accounts from Organizations into Control Tower"
status: active
severity: MEDIUM
triggers:
  - "Account import failed"
  - "Cannot import account"
  - "Extend governance"
owner: devops-agent
objective: "Successfully import existing Organization accounts into Control Tower governance"
context: "Account import (extend governance) brings existing Organization member accounts under CT governance. This differs from enrollment in that it handles accounts already in the Organization but not yet governed by CT."
---

## Phase 1 — Triage

MUST:
- Verify the account is a member of the Organization
- Check that the target OU is registered with Control Tower
- Verify AWSControlTowerExecution role exists in the target account
- Check for conflicting resources (Config, CloudTrail, CT stacks)

SHOULD:
- Check if the account has existing governance from another tool
- Verify the account is in an active state (not suspended)
- Check for SCP conflicts that might block baseline deployment

MAY:
- Review the account's existing CloudFormation stacks for conflicts
- Check if the account has resources in non-governed regions

## Phase 2 — Remediate

MUST:
- Ensure all prerequisites are met (role, OU registration, no conflicts)
- Move the account to a registered OU if needed
- Clean up conflicting resources

SHOULD:
- Import accounts in batches to avoid throttling
- Verify governance after import (controls, Config rules, StackSets)

MAY:
- Use the CT console for guided import
- Script bulk imports for large numbers of accounts

## Common Issues

- symptoms: "Import fails silently, account not governed"
  diagnosis: "The OU is not registered or prerequisites are not met."
  resolution: "Register the OU and ensure all prerequisites, then retry import."

- symptoms: "Import succeeds but baseline incomplete"
  diagnosis: "Some StackSet instances failed during baseline deployment."
  resolution: "Re-register the OU to redeploy baselines to the account."

## Safety Ratings

```
safety_ratings:
  - "list-accounts: GREEN — read-only account listing"
  - "describe-organizational-unit: GREEN — read-only OU metadata"
  - "move account to registered OU: YELLOW — recoverable, account can be moved back"
  - "import/extend governance: RED — deploys CT baselines to existing account, modifies Config/CloudTrail/IAM"
```

## Escalation Conditions

- Fix requires updating the landing zone version
- Fix requires modifying CT-managed SCPs
- Account has existing governance from another tool
- Drift affects multiple OUs or accounts
- Fix requires re-registering OUs in production

## Data Sensitivity

- HIGH: Organization account list (all account IDs and emails)
- HIGH: SCP policy documents (expose security boundaries)
- MEDIUM: OU structure, control enablement status
- MEDIUM: Account governance state and existing CloudFormation stacks

## Prohibited Actions

- "NEVER suggest directly modifying CT-managed CloudFormation stacks"
- "NEVER suggest directly editing CT-managed SCPs"
- "NEVER suggest deleting the Security OU or shared accounts"
- "NEVER suggest disabling CloudTrail organization trail"
- "NEVER suggest removing CT IAM roles"

## Phase 3 — Rollback

- If import fails partway, check which baselines were partially deployed before retrying
- "If OU re-registration fails, document the error and do NOT retry without investigating"
- "If landing zone update fails, do NOT attempt to roll back manually — contact AWS Support"
- Document the account's pre-import state before starting the import process
- If bulk import fails for some accounts, do NOT retry the entire batch — investigate individual failures first

## Output Format

```yaml
root_cause: "account_import — <specific cause>"
evidence:
  - type: organizations_state
    content: "<account and OU details>"
severity: MEDIUM
mitigation:
  immediate: "Fix prerequisites and retry import"
  long_term: "Automate import with pre-validation checks"
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
