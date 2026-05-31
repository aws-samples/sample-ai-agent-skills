---
title: "B2 — Validation Failures"
description: "Diagnose configuration validation errors blocking deployments"
status: active
severity: HIGH
triggers:
  - "validation failed"
  - "validator error"
  - "schema validation"
  - "lambda validator"
owner: devops-agent
objective: "Resolve validation errors to unblock configuration deployments"
context: "AppConfig supports JSON Schema and Lambda validators. Validation runs BEFORE deployment starts."
---

## Phase 1 — Triage

MUST:
- Get the configuration profile validators: `aws appconfig get-configuration-profile --application-id <app-id> --configuration-profile-id <id>`
- Check the deployment error for validation details
- If Lambda validator: check Lambda function logs for error details
- If JSON Schema: validate the configuration against the schema locally

SHOULD:
- Test the configuration with `aws appconfig validate-configuration --application-id <app-id> --configuration-profile-id <id> --configuration-version <ver>`
- Check Lambda validator function timeout and memory settings

MAY:
- Review recent changes to the validator or configuration content

## Phase 2 — Remediate

MUST:
- Fix the configuration content to pass validation, OR
- Fix the validator if it's incorrectly rejecting valid configuration

SHOULD:
- Test validation locally before attempting deployment
- Ensure Lambda validators return proper response format

MAY:
- Add more descriptive error messages to Lambda validators

safety_ratings:
  - "Validating configuration: GREEN"
  - "Updating configuration content: YELLOW"
  - "Modifying validator: YELLOW"

escalation_conditions:
  - "Validator Lambda is broken and blocking all deployments"
  - "Schema change required affects multiple configuration profiles"

data_sensitivity:
  - command: "validate-configuration"
    sensitivity: MEDIUM
    contains: "Configuration content, validation results"

prohibited_actions:
  - "NEVER remove validators to bypass validation errors in production"
  - "NEVER deploy unvalidated configuration to production environments"

## Phase 3 — Rollback

- If validator was modified, restore previous validator version
- If configuration was changed, revert to last known good version

## Common Issues

- symptoms: "JSON Schema validation fails with 'required property missing'"
  diagnosis: "Configuration is missing a required field defined in the schema"
  resolution: "Add the missing field to the configuration or make it optional in the schema"

- symptoms: "Lambda validator times out"
  diagnosis: "Validator function timeout too low or configuration too large to process"
  resolution: "Increase Lambda timeout or optimize validation logic"

## Output Format

```yaml
root_cause: "<description>"
evidence:
  - type: validation_error
    content: "<error details>"
severity: HIGH
mitigation:
  immediate: "<fix>"
  long_term: "<prevention>"
```
