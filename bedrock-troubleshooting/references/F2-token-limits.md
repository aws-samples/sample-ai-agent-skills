---
title: "F2 — Token Limit Issues"
description: "Diagnose token limit exceeded errors in Bedrock"
status: active
severity: MEDIUM
triggers:
  - "token limit exceeded"
  - "max tokens"
  - "context window"
  - "input too long"
owner: devops-agent
objective: "Identify and resolve token limit issues"
context: "Each model has a maximum context window (input + output tokens). Exceeding the limit returns an error. Different models have different limits (e.g., Claude 3 supports 200K tokens). The max_tokens parameter controls output length but cannot exceed the model's limit."
---

## Phase 1 — Triage

MUST:
- Check the error message for token limit details
- Verify the model's maximum context window
- Estimate input token count (roughly 4 chars per token for English)
- Check max_tokens parameter in the request

SHOULD:
- Compare input size with model's context window
- Check if system prompt consumes significant tokens
- Verify output max_tokens is reasonable
- Check for unnecessarily large context

MAY:
- Use tokenizer to get exact token counts
- Compare token limits across models

## Phase 2 — Remediate

MUST:
- Reduce input size to fit within the model's context window
- Set max_tokens appropriately (input + max_tokens ≤ context window)
- Use a model with a larger context window if needed

SHOULD:
- Implement input truncation or summarization
- Use RAG (knowledge bases) instead of stuffing context
- Monitor token usage per request
- Implement token counting before invocation

MAY:
- Create token budget management
- Implement automatic context compression

## Common Issues

- symptoms: "ValidationException — input exceeds maximum"
  diagnosis: "Input tokens exceed the model's context window."
  resolution: "Reduce input size. Use a model with larger context window. Implement RAG."

- symptoms: "Response truncated unexpectedly"
  diagnosis: "max_tokens set too low or remaining context window insufficient."
  resolution: "Increase max_tokens. Reduce input to leave room for output."

## Output Format

```yaml
root_cause: "token_limits — <specific_cause>"
evidence:
  - type: token_counts
    content: "<input/output token analysis>"
  - type: model_limits
    content: "<model context window limits>"
severity: MEDIUM
mitigation:
  immediate: "Reduce input size or increase model context window"
  long_term: "Implement token management and RAG for large contexts"
```


## Safety Ratings

| Phase 2 Action | safety_rating |
|---|---|
| Reduce input size to fit within the model's context window | GREEN |
| Set max_tokens appropriately | GREEN |
| Use a model with a larger context window if needed | YELLOW |
| Implement input truncation or summarization | GREEN |
| Use RAG (knowledge bases) instead of stuffing context | GREEN |
| Monitor token usage per request | GREEN |
| Implement token counting before invocation | GREEN |
| Create token budget management | GREEN |
| Implement automatic context compression | GREEN |

## Escalation Conditions

- Fix involves modifying guardrails on production model
- Knowledge base contains sensitive data

## Data Sensitivity

- **HIGH**: model invocation data, knowledge base content
- Token usage data may reveal the nature and volume of content being processed

## Prohibited Actions

- NEVER suggest removing guardrails to fix content filtering
- NEVER suggest using production data for fine-tuning without approval

## Phase 3 — Rollback

1. If the model was switched to a larger context window model, revert to the previous model ID
2. If input truncation logic was added, remove or disable it if it causes data loss
3. If max_tokens was changed, restore the previous value in application configuration
4. Verify rollback by testing with the original input sizes and confirming expected behavior

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
