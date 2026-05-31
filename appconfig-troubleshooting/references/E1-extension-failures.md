---
title: "E1 — Extension Failures"
description: "Diagnose AppConfig extension execution failures"
status: active
severity: MEDIUM
triggers:
  - "extension failed"
  - "extension timeout"
  - "action point error"
owner: devops-agent
objective: "Resolve extension execution failures"
context: "Extensions run at specific action points during AppConfig operations. They execute synchronously and can block operations."
---

## Phase 1 — Triage

MUST:
- List extensions: `aws appconfig list-extensions`
- Get extension details: `aws appconfig get-extension --extension-identifier <id>`
- Check which action points the extension is registered for
- If Lambda-backed: check Lambda function logs for errors

SHOULD:
- Check extension association: `aws appconfig list-extension-associations --resource-identifier <arn>`
- Verify the extension's Lambda function timeout is appropriate

MAY:
- Review CloudTrail for extension-related events

## Phase 2 — Remediate

MUST:
- Fix the Lambda function if it's returning errors
- Increase Lambda timeout if extension is timing out
- Verify the extension IAM role has required permissions

SHOULD:
- Add error handling to the extension Lambda function
- Test extensions in a non-production environment first

MAY:
- Add CloudWatch alarms for extension failures

safety_ratings:
  - "Listing extensions: GREEN"
  - "Updating extension Lambda: YELLOW"
  - "Removing extension association: YELLOW"

escalation_conditions:
  - "Extension blocking all deployments in production"
  - "Extension Lambda function is broken with no recent working version"

data_sensitivity:
  - command: "get-extension"
    sensitivity: LOW
    contains: "Extension name, action points, Lambda ARN"

prohibited_actions:
  - "NEVER remove a security-related extension to unblock deployments"

## Phase 3 — Rollback

- If extension Lambda was updated, deploy previous Lambda version
- If extension association was removed, re-associate it

## Common Issues

- symptoms: "Deployment blocked by extension timeout"
  diagnosis: "Extension Lambda function exceeding timeout"
  resolution: "Increase Lambda timeout or optimize extension logic"

- symptoms: "Extension returns error on every invocation"
  diagnosis: "Extension Lambda code bug or missing permissions"
  resolution: "Check Lambda logs, fix code, and verify IAM permissions"

## Output Format

```yaml
root_cause: "<description>"
evidence:
  - type: extension_error
    content: "<error details>"
severity: MEDIUM
mitigation:
  immediate: "<fix>"
  long_term: "<prevention>"
```
