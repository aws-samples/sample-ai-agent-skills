---
title: "A1 — Application Creation Failures"
description: "Diagnose and resolve AppConfig application creation issues"
status: active
severity: MEDIUM
triggers:
  - "create-application failed"
  - "application creation error"
owner: devops-agent
objective: "Successfully create an AppConfig application"
context: "AppConfig applications are top-level containers for environments and configuration profiles."
---

## Phase 1 — Triage

MUST:
- Check the error message from `aws appconfig create-application` call
- Verify IAM permissions include `appconfig:CreateApplication`
- Check if application name already exists: `aws appconfig list-applications`
- Review CloudTrail for the CreateApplication event

SHOULD:
- Check service quotas for AppConfig applications per account
- Verify the AWS region supports AppConfig

MAY:
- Check for SCPs blocking AppConfig actions

## Phase 2 — Remediate

MUST:
- Fix IAM permissions if access denied
- Use a unique application name if name conflict exists

SHOULD:
- Add tags during creation for organization: `aws appconfig create-application --name <name> --tags <tags>`

MAY:
- Use CloudFormation/CDK for repeatable application setup

safety_ratings:
  - "Checking application list: GREEN"
  - "Creating application: GREEN"
  - "Modifying IAM permissions: YELLOW"

escalation_conditions:
  - "SCP blocking AppConfig in the account"
  - "Service quota reached for applications"

data_sensitivity:
  - command: "list-applications"
    sensitivity: LOW
    contains: "Application names and IDs"

prohibited_actions:
  - "NEVER delete an existing application to resolve a naming conflict without confirming it's unused"

## Phase 3 — Rollback

- If application was created incorrectly, delete it: `aws appconfig delete-application --application-id <id>`
- Verify no environments or profiles exist before deletion (cascading delete)

## Common Issues

- symptoms: "AccessDeniedException on CreateApplication"
  diagnosis: "IAM policy missing appconfig:CreateApplication permission"
  resolution: "Add appconfig:CreateApplication to the IAM policy"

- symptoms: "Application name already exists"
  diagnosis: "AppConfig application names must be unique within the account/region"
  resolution: "Use a different name or check if the existing application can be reused"

## Output Format

```yaml
root_cause: "<description>"
evidence:
  - type: api_error
    content: "<error message>"
severity: MEDIUM
mitigation:
  immediate: "<fix>"
  long_term: "<prevention>"
```
