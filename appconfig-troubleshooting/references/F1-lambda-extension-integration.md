---
title: "F1 — Lambda Extension Integration"
description: "Diagnose AppConfig Lambda extension issues"
status: active
severity: HIGH
triggers:
  - "lambda extension error"
  - "configuration not available in lambda"
  - "appconfig lambda layer"
owner: devops-agent
objective: "Resolve AppConfig Lambda extension integration issues"
context: "The AppConfig Lambda extension runs as a Lambda layer, providing a local HTTP endpoint for configuration retrieval with caching."
---

## Phase 1 — Triage

MUST:
- Verify the AppConfig Lambda layer is attached: `aws lambda get-function-configuration --function-name <name>`
- Check the layer ARN matches the correct region and version
- Verify the Lambda function has IAM permissions for appconfig:GetLatestConfiguration and appconfig:StartConfigurationSession
- Check Lambda function logs for AppConfig extension errors

SHOULD:
- Verify the extension port (default 2772) is not conflicting
- Check environment variables: AWS_APPCONFIG_EXTENSION_POLL_INTERVAL_SECONDS, AWS_APPCONFIG_EXTENSION_POLL_TIMEOUT_MILLIS

MAY:
- Test the extension endpoint locally: `curl http://localhost:2772/applications/<app>/environments/<env>/configurations/<profile>`

## Phase 2 — Remediate

MUST:
- Add the AppConfig Lambda layer if missing
- Fix IAM permissions for AppConfig access
- Correct the configuration path in the HTTP request

SHOULD:
- Set appropriate poll interval for the use case
- Use the latest layer version for bug fixes

MAY:
- Configure prefetch for faster cold starts

safety_ratings:
  - "Checking Lambda configuration: GREEN"
  - "Adding Lambda layer: YELLOW"
  - "Modifying IAM permissions: YELLOW"

escalation_conditions:
  - "Production Lambda functions cannot retrieve configuration"
  - "AppConfig extension causing Lambda timeouts"

data_sensitivity:
  - command: "get-function-configuration"
    sensitivity: MEDIUM
    contains: "Lambda configuration, layer ARNs, environment variables"

prohibited_actions:
  - "NEVER grant appconfig:* to Lambda functions — use least privilege"

## Phase 3 — Rollback

- If layer was changed, revert to previous layer version
- If IAM permissions were modified, restore previous policy

## Common Issues

- symptoms: "Lambda function times out after adding AppConfig extension"
  diagnosis: "Extension initialization adding to cold start time, or configuration path incorrect causing retries"
  resolution: "Verify configuration path and increase Lambda timeout to account for extension initialization"

- symptoms: "Connection refused on port 2772"
  diagnosis: "AppConfig Lambda layer not attached or wrong layer ARN"
  resolution: "Attach the correct AppConfig Lambda extension layer for the region"

## Output Format

```yaml
root_cause: "<description>"
evidence:
  - type: lambda_config
    content: "<Lambda function configuration>"
severity: HIGH
mitigation:
  immediate: "<fix>"
  long_term: "<prevention>"
```
