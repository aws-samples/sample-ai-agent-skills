---
title: "A2 — Landing Zone Update Failure"
description: "Diagnose failures when updating Control Tower landing zone to a new version"
status: active
severity: CRITICAL
triggers:
  - "Landing zone update failed"
  - "UpdateLandingZone failed"
  - "Cannot update landing zone"
owner: devops-agent
objective: "Identify why the landing zone update failed and complete the update"
context: "Landing zone updates apply new CT versions with updated controls, StackSets, and configurations. Updates fail when drift is present, StackSets have failed instances, or the update introduces breaking changes."
---

## Phase 1 — Triage

MUST:
- Check current landing zone version: `aws controltower get-landing-zone --landing-zone-identifier <arn>`
- Check for drift: landing zone status should be ACTIVE with no drift
- Check failed operations: `aws controltower list-landing-zone-operations --filter statuses=FAILED`
- Check StackSet instance status: `aws cloudformation list-stack-instances --stack-set-name AWSControlTowerBP-BASELINE-CLOUDWATCH --filters Name=DETAILED_STATUS,Values=FAILED`
- Review CloudFormation events for AWSControlTower* stacks

SHOULD:
- Check release notes for the target version for breaking changes
- Verify all OUs are in a clean state (no pending operations)
- Check if any member accounts have been suspended or closed

MAY:
- Compare current and target version feature differences
- Check AWS forums or documentation for known update issues

## Phase 2 — Remediate

MUST:
- Resolve all drift before retrying the update
- Fix any failed StackSet instances (delete and let CT recreate)
- Ensure no concurrent CT operations are running

SHOULD:
- Re-register OUs after the update if required by release notes
- Update Account Factory product versions after landing zone update
- Verify controls are still enabled and compliant after update

MAY:
- Plan the update during a maintenance window
- Notify account owners of potential temporary disruptions

## Common Issues

- symptoms: "Update fails with drift detected"
  diagnosis: "Landing zone has unresolved drift from manual changes."
  resolution: "Resolve all drift (re-register OUs, restore SCPs) before retrying update."

- symptoms: "Update fails with StackSet operation in progress"
  diagnosis: "A previous StackSet operation is still running or stuck."
  resolution: "Wait for the operation to complete or cancel it, then retry."

- symptoms: "Update fails with stack instance failures"
  diagnosis: "One or more member accounts have failed stack instances."
  resolution: "Delete failed stack instances and retry. Check the target account for blocking conditions."

## Safety Ratings

```
safety_ratings:
  - "get-landing-zone: GREEN — read-only landing zone metadata"
  - "list-landing-zone-operations: GREEN — read-only operation listing"
  - "list-stack-instances: GREEN — read-only StackSet instance listing"
  - "re-register OU after update: YELLOW — recoverable, resets CT baselines"
  - "delete failed stack instances: YELLOW — recoverable, CT will recreate on re-register"
  - "UpdateLandingZone (retry): RED — updates foundational governance structure, can break controls across all OUs"
```

## Escalation Conditions

- Fix requires updating the landing zone version
- Fix requires modifying CT-managed SCPs
- Drift affects multiple OUs or accounts
- Update fails with no clear error after resolving drift and StackSet issues
- Update introduces breaking changes affecting production accounts

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

- "If landing zone update fails, do NOT attempt to roll back manually — contact AWS Support"
- If StackSet instances were deleted to unblock the update, document which accounts/regions were affected for re-registration
- "If OU re-registration fails, document the error and do NOT retry without investigating"
- If the update partially completed, do NOT retry without checking the current landing zone state first
- Document the pre-update landing zone version and configuration before attempting any update

## Output Format

```yaml
root_cause: "landing_zone_update_failure — <specific cause>"
evidence:
  - type: controltower_operation
    content: "<operation status and error>"
  - type: stackset_instances
    content: "<failed instance details>"
severity: CRITICAL
mitigation:
  immediate: "Resolve blocking condition and retry update"
  long_term: "Implement pre-update checklist: drift check, StackSet health, release notes review"
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
