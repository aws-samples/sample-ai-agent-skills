---
title: "B1 — Configuration Profile Errors"
description: "Diagnose configuration profile creation and management issues"
status: active
severity: HIGH
triggers:
  - "configuration profile error"
  - "profile creation failed"
  - "invalid configuration source"
owner: devops-agent
objective: "Resolve configuration profile issues"
context: "Configuration profiles define the source (hosted, S3, SSM Document, CodePipeline) and validators for configuration data."
---

## Phase 1 — Triage

MUST:
- List profiles: `aws appconfig list-configuration-profiles --application-id <app-id>`
- Get profile details: `aws appconfig get-configuration-profile --application-id <app-id> --configuration-profile-id <id>`
- Check the location URI for the configuration source
- Verify validators are correctly configured

SHOULD:
- Test the configuration source is accessible (S3 bucket, SSM Document, etc.)
- Check IAM permissions for accessing the configuration source

MAY:
- List hosted configuration versions if using hosted source

## Phase 2 — Remediate

MUST:
- Fix the location URI if the source is inaccessible
- Correct validator configuration (JSON Schema syntax or Lambda ARN)

SHOULD:
- Ensure the AppConfig service role can access S3/SSM sources
- Test validators independently before attaching to profiles

MAY:
- Migrate from external sources to hosted configuration for simpler management

safety_ratings:
  - "Listing profiles: GREEN"
  - "Updating profile validators: YELLOW"
  - "Changing configuration source: YELLOW"

escalation_conditions:
  - "Production profile source is inaccessible"
  - "Validator Lambda is failing for all configurations"

data_sensitivity:
  - command: "get-configuration-profile"
    sensitivity: MEDIUM
    contains: "Configuration source URIs, validator ARNs"

prohibited_actions:
  - "NEVER delete a configuration profile that has active deployments"
  - "NEVER remove validators from production profiles to bypass validation"

## Phase 3 — Rollback

- If profile was modified, restore previous validator and source settings
- Document original configuration before changes

## Common Issues

- symptoms: "Configuration profile creation fails with invalid location URI"
  diagnosis: "S3 URI format incorrect or SSM Document doesn't exist"
  resolution: "Verify the source exists and URI format is correct (s3://bucket/key or ssm-document://name)"

- symptoms: "Profile works but deployments fail validation"
  diagnosis: "Validator schema doesn't match the configuration content"
  resolution: "Update the JSON Schema validator or fix the configuration to match the schema"

## Output Format

```yaml
root_cause: "<description>"
evidence:
  - type: profile_config
    content: "<profile details>"
severity: HIGH
mitigation:
  immediate: "<fix>"
  long_term: "<prevention>"
```
