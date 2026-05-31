---
title: "F3 — Cloud-Init / User-Data Failures"
description: "Diagnose cloud-init and user-data execution failures during instance initialization"
status: active
severity: MEDIUM
triggers:
  - "cloud-init.*error"
  - "user-data.*failed"
  - "cloud-init.*timeout"
  - "cc_scripts_user.*failed"
owner: devops-agent
objective: "Identify the cloud-init failure and fix the initialization script"
context: "Cloud-init runs during first boot (and optionally on every boot). User-data scripts execute as root. Failures can leave the instance in a partially configured state. Logs are in /var/log/cloud-init.log and /var/log/cloud-init-output.log."
---

## Phase 1 — Triage

MUST:
- Check system log for cloud-init errors: `aws ec2 get-console-output --instance-id <id>`
- If SSM available: check `/var/log/cloud-init.log` and `/var/log/cloud-init-output.log`
- Check user-data content: `aws ec2 describe-instance-attribute --instance-id <id> --attribute userData`
- Verify user-data format (must start with `#!/bin/bash`, `#cloud-config`, or be base64 encoded)

SHOULD:
- Check if user-data script has syntax errors
- Verify network connectivity during cloud-init (some scripts need internet access)
- Check cloud-init status: `cloud-init status --long` via SSM

MAY:
- Re-run cloud-init: `cloud-init clean && cloud-init init` via SSM (for testing)

## Common Issues

- symptoms: "User-data script doesn't execute"
  diagnosis: "Missing shebang line (#!/bin/bash), or user-data not base64 encoded when required."
  resolution: "Add shebang line. Ensure proper encoding. Check instance metadata for user-data availability."

- symptoms: "Cloud-init times out waiting for metadata"
  diagnosis: "IMDS unreachable. Check if IMDS is enabled and hop limit is sufficient."
  resolution: "Enable IMDS, set hop limit to 2 if using containers. Check network path to 169.254.169.254."

- symptoms: "User-data runs but fails partway through"
  diagnosis: "Script error, missing dependency, or network issue during execution."
  resolution: "Check /var/log/cloud-init-output.log for the specific error. Fix the script."

## Safety Ratings
Rate each remediation action in Phase 2:
- GREEN: read-only diagnostic commands (describe-*, get-*, list-*)
- YELLOW: state-changing but recoverable (modify security group, restart instance, modify parameter)
- RED: destructive or irreversible (terminate instance, delete volume, modify key infrastructure)

Format as:
```
safety_ratings:
  - "get-console-output, describe-instance-attribute (userData): GREEN — read-only"
  - "Check cloud-init logs via SSM: GREEN — read-only diagnostics"
  - "cloud-init status via SSM: GREEN — read-only status check"
  - "Re-run cloud-init (cloud-init clean && init): YELLOW — re-executes all init scripts"
  - "Modify user-data and relaunch: YELLOW — requires new instance or stop+start"
  - "Fix IMDS configuration: YELLOW — changes metadata access, recoverable"
```

## Escalation Conditions
- Cloud-init failure left the instance in a partially configured state with security gaps
- User-data script contains secrets that may have been logged to cloud-init-output.log
- Cloud-init failure affects an Auto Scaling group launch configuration (all new instances fail)
- IMDS is unreachable and instance cannot retrieve credentials or metadata
- Cloud-init failure is caused by a network dependency that is intermittently unavailable

## Data Sensitivity
- HIGH: describe-instance-attribute --attribute userData (may contain secrets, API keys, passwords in scripts)
- HIGH: /var/log/cloud-init-output.log (may contain script output with credentials)
- HIGH: /var/log/cloud-init.log (may contain metadata, instance identity details)
- MEDIUM: get-console-output (may contain cloud-init error messages with configuration details)

## Prohibited Actions
- NEVER suggest logging user-data script output to publicly accessible locations
- NEVER suggest embedding long-lived credentials in user-data scripts (use IAM roles instead)
- NEVER suggest disabling IMDS entirely to troubleshoot cloud-init issues
- NEVER suggest running cloud-init clean on a production instance without understanding re-initialization impact

## Phase 3 — Rollback
- If cloud-init was re-run: some modules are not idempotent — may need to manually undo changes
- If user-data was modified: update launch template/configuration to previous version
- If IMDS was reconfigured: revert with `modify-instance-metadata-options` to previous settings
- If instance was relaunched: terminate new instance and restore from previous instance/AMI if needed

## Output Format

```yaml
root_cause: "<script_error|missing_shebang|imds_timeout|network_during_init|encoding> — <detail>"
evidence:
  - type: cloud_init_log
    content: "<error from cloud-init logs>"
severity: MEDIUM
mitigation:
  immediate: "Fix user-data script and relaunch or re-run cloud-init"
  long_term: "Test user-data scripts in staging, use cloud-init modules over raw scripts"
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
  - command: "describe-instances"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"
  - command: "get-console-output"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"
  - command: "ssm send-command"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"

## Prohibited Actions

prohibited_actions:
  - "NEVER suggest 0.0.0.0/0 inbound security group rules as a fix"
  - "NEVER suggest disabling instance metadata service"
  - "NEVER terminate instances without confirmation"
