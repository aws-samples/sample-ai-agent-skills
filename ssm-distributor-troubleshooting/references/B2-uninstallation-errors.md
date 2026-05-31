---
title: "B2 — Uninstallation Errors"
description: "Diagnose SSM Distributor package uninstallation failures"
status: active
severity: MEDIUM
triggers: ["uninstall failed", "cannot remove package", "uninstall script error", "package removal"]
owner: devops-agent
objective: "Resolve package uninstallation failures on target instances"
context: "Uninstallation runs the platform-specific uninstall script defined in the package manifest."
---

## Phase 1 — Triage

MUST:
- Check uninstall command status: `aws ssm list-command-invocations --command-id <command-id> --details --query 'CommandInvocations[*].{InstanceId:InstanceId,Status:Status,StatusDetails:StatusDetails}'`
- Get detailed output: `aws ssm get-command-invocation --command-id <command-id> --instance-id <instance-id>`
- Verify the package was previously installed on the instance

SHOULD:
- Check if the uninstall script exists in the package manifest
- Verify no running processes depend on the software being uninstalled

MAY:
- Review system logs for uninstall-related errors
- Check if partial uninstallation left orphaned files or services

## Phase 2 — Remediate

MUST:
- Fix uninstall script errors identified in command output
- Ensure uninstall script handles cases where software is already partially removed
- Verify the uninstall script has sufficient system permissions

SHOULD:
- Make uninstall scripts idempotent — they should succeed even if run multiple times
- Stop dependent services before uninstallation

MAY:
- Create a cleanup script to handle orphaned artifacts from failed uninstalls
- Use the Reinstall action to reset state before attempting uninstall again

## Common Issues

- symptoms: "Uninstall command fails with 'service is running' error"
  diagnosis: "Uninstall script does not stop the service before removal."
  resolution: "Update uninstall script to stop the service first. Create a new package version with the fix."

- symptoms: "Uninstall succeeds but files remain on the instance"
  diagnosis: "Uninstall script does not clean up all installed artifacts."
  resolution: "Update uninstall script to remove all files, directories, and registry entries. Create a new package version."

## Safety Ratings

```
safety_ratings:
  - "list-command-invocations: GREEN — read-only command status"
  - "get-command-invocation: GREEN — read-only command output"
  - "send-command with Uninstall action: YELLOW — removes software from instances"
  - "Manual cleanup via Run Command: RED — direct system modification"
```

## Escalation Conditions

- Uninstallation fails and leaves the instance in an unstable state
- Uninstall script causes data loss or service disruption
- Failed uninstall blocks installation of a newer version
- Uninstallation required across a large fleet with time constraints

## Data Sensitivity

- `aws ssm get-command-invocation`: MEDIUM — may contain system paths and error details
- Uninstall script output: MEDIUM — may reveal installed software configuration
- System service status: LOW — reveals running services

## Prohibited Actions

- NEVER force-remove software without running the proper uninstall script first
- NEVER delete system files or directories manually without understanding dependencies
- NEVER uninstall packages on production instances during peak hours without a maintenance window
- NEVER skip verification that the uninstall completed successfully

## Phase 3 — Rollback

1. If uninstall caused issues, reinstall the package: `aws ssm send-command --document-name <package-name> --parameters action=Install --targets Key=InstanceIds,Values=<instance-id>`
2. Verify reinstallation restored the software to a working state
3. If reinstall fails, restore from instance backup or AMI
4. Restart any services that were stopped during the uninstall attempt
5. Document the uninstall failure and update the uninstall script for future versions

## Output Format

```yaml
root_cause: "uninstallation_error — <specific_cause>"
evidence:
  - type: command_output
    content: "<error details>"
severity: MEDIUM
mitigation:
  immediate: "Fix uninstall script or clean up manually"
  long_term: "Make uninstall scripts idempotent"
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
  - "NEVER suggest overly broad SSM document permissions"
  - "NEVER suggest disabling session logging"
  - "NEVER suggest sharing SSM documents publicly"
