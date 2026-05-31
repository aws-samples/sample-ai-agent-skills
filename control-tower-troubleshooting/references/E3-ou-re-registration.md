---
title: "E3 — OU Re-Registration Issues"
description: "Diagnose issues when re-registering an OU with Control Tower"
status: active
severity: MEDIUM
triggers:
  - "Re-register OU failed"
  - "OU re-registration"
  - "Update provisioned product for OU"
owner: devops-agent
objective: "Successfully re-register the OU to restore CT governance"
context: "Re-registering an OU resets CT-managed resources (StackSets, SCPs, Config rules) to the expected state. It's the primary mechanism for resolving OU-level drift and applying landing zone updates to existing OUs."
---

## Phase 1 — Triage

MUST:
- Check the re-registration operation status
- Verify the OU is currently registered (can't re-register an unregistered OU)
- Check for failed StackSet instances that might block re-registration
- Verify accounts in the OU have the AWSControlTowerExecution role

SHOULD:
- Check if there are concurrent CT operations
- Verify the landing zone is in ACTIVE state (not drifted)
- Check if any accounts in the OU are suspended or closing

MAY:
- Review what triggered the need for re-registration
- Check if the OU has a large number of accounts (affects duration)

## Phase 2 — Remediate

MUST:
- Fix any blocking conditions (failed stacks, missing roles)
- Retry re-registration after fixes
- Verify controls and baselines are restored after re-registration

SHOULD:
- Re-register OUs one at a time to isolate failures
- Monitor StackSet operations during re-registration
- Verify compliance status after re-registration

MAY:
- Schedule re-registration during low-activity periods
- Notify account owners of temporary disruptions

## Common Issues

- symptoms: "Re-registration fails with stack instance errors"
  diagnosis: "Failed stack instances in member accounts block the operation."
  resolution: "Delete failed stack instances, fix the underlying issue, then retry."

- symptoms: "Re-registration takes very long"
  diagnosis: "Large number of accounts in the OU causes slow StackSet operations."
  resolution: "This is expected. Monitor progress via StackSet operation status."

## Safety Ratings

```
safety_ratings:
  - "list-enabled-controls: GREEN — read-only control listing"
  - "list-stack-instances: GREEN — read-only StackSet instance listing"
  - "delete failed stack instances: YELLOW — recoverable, CT will recreate on re-register"
  - "re-register OU: YELLOW — recoverable, resets CT baselines for the OU"
```

## Escalation Conditions

- Fix requires updating the landing zone version
- Fix requires modifying CT-managed SCPs
- Drift affects multiple OUs or accounts
- Fix requires re-registering OUs in production
- Re-registration fails repeatedly with no clear error
- Landing zone is in drifted state blocking re-registration

## Data Sensitivity

- HIGH: SCP policy documents (expose security boundaries)
- HIGH: Organization account list (all account IDs and emails)
- MEDIUM: OU structure, control enablement status
- MEDIUM: StackSet instance details (account IDs, regions, failure reasons)

## Prohibited Actions

- "NEVER suggest directly modifying CT-managed CloudFormation stacks"
- "NEVER suggest directly editing CT-managed SCPs"
- "NEVER suggest deleting the Security OU or shared accounts"
- "NEVER suggest disabling CloudTrail organization trail"
- "NEVER suggest removing CT IAM roles"

## Phase 3 — Rollback

- "If OU re-registration fails, document the error and do NOT retry without investigating"
- "If landing zone update fails, do NOT attempt to roll back manually — contact AWS Support"
- If re-registration fails partway, check which accounts were updated and which were not
- Document the OU's current state (controls, StackSet instances) before starting re-registration
- If failed stack instances were deleted to unblock re-registration, document which accounts/regions were affected

## Output Format

```yaml
root_cause: "ou_re_registration — <specific cause>"
evidence:
  - type: controltower_operation
    content: "<re-registration status>"
severity: MEDIUM
mitigation:
  immediate: "Fix blocking conditions and retry"
  long_term: "Maintain clean StackSet state to enable smooth re-registration"
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
