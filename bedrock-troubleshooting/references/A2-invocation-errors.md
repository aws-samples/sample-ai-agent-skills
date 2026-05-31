---
title: "A2 — Model Invocation Errors"
description: "Diagnose model invocation failures in Bedrock"
status: active
severity: HIGH
triggers:
  - "invocation error"
  - "InvokeModel failed"
  - "ValidationException"
  - "model response error"
owner: devops-agent
objective: "Identify and resolve model invocation errors"
context: "Invocation errors stem from incorrect request format, invalid parameters, model-specific requirements, content policy violations, or service errors. Each model has a unique request/response format. ValidationException indicates request format issues."
---

## Phase 1 — Triage

MUST:
- Check the exact error message and error code
- Verify request body format matches the model's requirements
- Check model-specific parameters (each provider has different formats)
- Verify the Content-Type header is application/json

SHOULD:
- Test with a minimal request body
- Check CloudTrail for InvokeModel events with error details
- Verify the model ID includes the correct version
- Check for content policy violations in the response

MAY:
- Check model invocation logging for request/response details
- Review CloudWatch InvocationErrors metric

## Phase 2 — Remediate

MUST:
- Use the correct request format for the specific model provider
- For Claude: use anthropic_version, messages array, max_tokens
- For Titan: use inputText, textGenerationConfig
- Validate request body JSON syntax

SHOULD:
- Start with the simplest possible request and add parameters
- Check model documentation for required vs optional parameters
- Handle error responses gracefully in application code

MAY:
- Enable model invocation logging for debugging
- Create request templates for each model

## Common Issues

- symptoms: "ValidationException — invalid request body"
  diagnosis: "Request format doesn't match model requirements."
  resolution: "Check model-specific request format. Claude needs anthropic_version and messages."

- symptoms: "ModelErrorException"
  diagnosis: "Model encountered an internal error processing the request."
  resolution: "Retry the request. If persistent, try a different model or contact support."

## Output Format

```yaml
root_cause: "invocation_error — <specific_cause>"
evidence:
  - type: error_message
    content: "<exact error message>"
  - type: request_body
    content: "<request format analysis>"
severity: HIGH
mitigation:
  immediate: "Fix request format for the specific model"
  long_term: "Create validated request templates and implement error handling"
```


## Safety Ratings

| Phase 2 Action | safety_rating |
|---|---|
| Use the correct request format for the specific model provider | GREEN |
| For Claude: use anthropic_version, messages array, max_tokens | GREEN |
| For Titan: use inputText, textGenerationConfig | GREEN |
| Validate request body JSON syntax | GREEN |
| Start with the simplest possible request and add parameters | GREEN |
| Check model documentation for required vs optional parameters | GREEN |
| Handle error responses gracefully in application code | GREEN |
| Enable model invocation logging for debugging | YELLOW |
| Create request templates for each model | GREEN |

## Escalation Conditions

- Fix involves modifying guardrails on production model
- Knowledge base contains sensitive data

## Data Sensitivity

- **HIGH**: model invocation data, knowledge base content
- Request bodies and response payloads may contain sensitive user data or PII

## Prohibited Actions

- NEVER suggest removing guardrails to fix content filtering
- NEVER suggest using production data for fine-tuning without approval

## Phase 3 — Rollback

1. If request format changes caused application errors, revert application code to the previous version
2. If model invocation logging was enabled, disable it if it captures sensitive data: update logging configuration
3. If model parameters were changed, restore previous parameter values in application configuration
4. Verify rollback by testing invocation with a minimal safe request body

## Safety Ratings

safety_ratings:
  - "Phase 1 triage commands (describe/get/list): GREEN — read-only"
  - "Phase 2 configuration changes: YELLOW — state-changing but recoverable"
  - "Phase 2 resource deletion or security changes: RED — destructive or irreversible"

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
  - "NEVER suggest removing guardrails from production models"
  - "NEVER suggest disabling content filtering"
  - "NEVER suggest overly broad model access permissions"
