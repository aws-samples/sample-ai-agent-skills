---
title: "E1 — Settings API Errors"
description: "Diagnose Bottlerocket settings API (apiclient) errors and configuration failures"
status: active
severity: MEDIUM
triggers: ["apiclient error", "settings api failed", "cannot set configuration", "invalid settings"]
owner: devops-agent
objective: "Resolve Bottlerocket settings API errors and restore configuration management"
context: "All Bottlerocket configuration goes through the settings API (apiclient). Settings are validated before being applied."
---

## Phase 1 — Triage

MUST:
- Test settings API connectivity: `aws ssm start-session --target <instance-id> --document-name AWS-StartInteractiveCommand --parameters command="apiclient get settings"`
- Check specific setting that failed: `apiclient get settings.<category>`
- Verify the setting key and value format are valid

SHOULD:
- Check if the setting requires a reboot to take effect
- Review user data TOML for syntax errors in the launch template

MAY:
- Collect logs for API errors: `logdog`

## Phase 2 — Remediate

MUST:
- Fix TOML syntax errors in user data (validate with a TOML parser)
- Use correct setting paths: `apiclient set <category>.<key>=<value>`
- Verify settings after applying: `apiclient get settings.<category>`

SHOULD:
- Reboot if the setting requires it: `apiclient reboot`
- Document valid setting paths for the Bottlerocket variant in use

MAY:
- Use `apiclient raw` for advanced API interactions

## Safety Ratings

```
safety_ratings:
  - "apiclient get settings: GREEN — read-only settings enumeration"
  - "apiclient set: YELLOW — modifies system configuration"
  - "apiclient raw: RED — direct API access bypassing validation helpers"
  - "apiclient reboot: RED — causes instance reboot"
```

## Escalation Conditions

- Settings API is completely unresponsive (apiserver process crashed)
- Setting changes cause boot loops or service failures
- User data TOML is too large for EC2 user data limits (16KB)
- Settings conflict between user data and runtime changes

## Data Sensitivity

- `apiclient get settings`: MEDIUM — reveals full system configuration including cluster details
- `apiclient set`: MEDIUM — modifies system configuration
- User data TOML: HIGH — may contain cluster certificates and API endpoints
- `apiclient raw` output: MEDIUM — raw API responses with system details

## Prohibited Actions

- NEVER use apiclient raw to bypass safety validations without understanding the consequences
- NEVER modify settings that affect the control container without a backup access method
- NEVER apply untested TOML user data to production instances
- NEVER ignore apiclient validation errors — they indicate real configuration problems

## Phase 3 — Rollback

1. Revert individual settings: `apiclient set <category>.<key>=<original-value>`
2. If multiple settings were changed, reboot to re-apply user data defaults: `apiclient reboot`
3. If settings API is unresponsive, stop and start the instance to reset
4. If user data was changed in launch template, revert to previous version
5. Terminate and relaunch if settings are in an unrecoverable state

## Common Issues

- symptoms: "apiclient set returns 'invalid setting' error"
  diagnosis: "The setting key path is incorrect or not available in this Bottlerocket variant."
  resolution: "Check available settings with apiclient get settings. ECS and EKS variants have different settings."

- symptoms: "Settings applied via apiclient set are lost after reboot"
  diagnosis: "User data TOML overrides runtime settings on boot."
  resolution: "Update the user data TOML in the launch template to include the desired settings."

## Output Format

```yaml
root_cause: "settings_api — <specific_cause>"
evidence:
  - type: api_error
    content: "<error details>"
severity: MEDIUM
mitigation:
  immediate: "Fix settings syntax or path"
  long_term: "Validate TOML user data in CI/CD pipeline"
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
  - "NEVER suggest overly broad IAM permissions as a fix"
  - "NEVER suggest disabling security controls for troubleshooting"
  - "NEVER suggest deleting resources without confirmation"
