---
title: "D1 — Feature Flag Configuration"
description: "Diagnose feature flag setup and configuration issues"
status: active
severity: MEDIUM
triggers:
  - "feature flag not working"
  - "flag configuration error"
  - "feature flag schema"
owner: devops-agent
objective: "Correctly configure AppConfig feature flags"
context: "Feature flags use the AWS.AppConfig.FeatureFlags configuration type with a specific JSON schema for flag definitions."
---

## Phase 1 — Triage

MUST:
- Verify the configuration profile type is AWS.AppConfig.FeatureFlags
- Get the current flag configuration: `aws appconfig get-hosted-configuration-version --application-id <app-id> --configuration-profile-id <id> --version-number <ver> outfile.json`
- Validate the flag JSON structure matches the required schema
- Check if the flag is enabled in the configuration

SHOULD:
- Verify the flag attributes and constraints are correctly defined
- Check if the flag has been deployed to the target environment

MAY:
- Review flag configuration history for recent changes

## Phase 2 — Remediate

MUST:
- Fix the flag JSON structure if schema validation fails
- Ensure flags have the required `enabled` boolean field
- Deploy the corrected configuration to the target environment

SHOULD:
- Use the AppConfig feature flag schema reference for correct structure
- Test flag evaluation locally before deploying

MAY:
- Add flag attributes for targeting rules

safety_ratings:
  - "Reading flag configuration: GREEN"
  - "Updating flag values: YELLOW"
  - "Creating new flag: GREEN"

escalation_conditions:
  - "Production feature flag stuck in wrong state"
  - "Flag schema change affects multiple environments"

data_sensitivity:
  - command: "get-hosted-configuration-version"
    sensitivity: MEDIUM
    contains: "Feature flag definitions, targeting rules"

prohibited_actions:
  - "NEVER toggle production feature flags without confirming the deployment strategy"
  - "NEVER delete feature flag definitions that are actively referenced by applications"

## Phase 3 — Rollback

- Deploy the previous configuration version to revert flag changes
- Document which flags were changed and their previous values

## Common Issues

- symptoms: "Feature flag API returns error"
  diagnosis: "Configuration profile is not of type AWS.AppConfig.FeatureFlags"
  resolution: "Create a new profile with type AWS.AppConfig.FeatureFlags"

- symptoms: "Flag value not updating in application"
  diagnosis: "New configuration not deployed, or AppConfig agent cache not refreshed"
  resolution: "Deploy the updated configuration and wait for cache TTL"

## Output Format

```yaml
root_cause: "<description>"
evidence:
  - type: flag_config
    content: "<flag definition>"
severity: MEDIUM
mitigation:
  immediate: "<fix>"
  long_term: "<prevention>"
```
