---
title: "H2 — Streaming Response Issues"
description: "Diagnose Bedrock streaming response handling problems"
status: active
severity: MEDIUM
triggers:
  - "streaming error"
  - "InvokeModelWithResponseStream"
  - "event stream"
  - "streaming timeout"
owner: devops-agent
objective: "Identify and resolve streaming response issues"
context: "InvokeModelWithResponseStream returns an event stream with response chunks. Clients must handle the AWS event stream protocol. Streaming reduces perceived latency but requires specific client implementation. Not all SDKs handle streaming identically."
---

## Phase 1 — Triage

MUST:
- Verify the model supports streaming
- Check client code handles event stream protocol correctly
- Verify IAM permissions include bedrock:InvokeModelWithResponseStream
- Check for connection timeout settings

SHOULD:
- Test with a simple streaming request
- Verify SDK version supports Bedrock streaming
- Check for proxy or load balancer interference with streaming
- Review error handling for stream interruptions

MAY:
- Check for WebSocket or HTTP/2 requirements
- Review network configuration for long-lived connections

## Phase 2 — Remediate

MUST:
- Use the correct SDK method for streaming (varies by language)
- Handle event stream chunks properly in client code
- Set appropriate connection and read timeouts

SHOULD:
- Implement stream error handling and reconnection
- Use SDK-provided streaming helpers
- Test streaming end-to-end before production

MAY:
- Implement streaming response monitoring
- Create streaming client templates

## Common Issues

- symptoms: "Streaming connection drops mid-response"
  diagnosis: "Timeout too short or proxy terminating long connections."
  resolution: "Increase read timeout. Configure proxy for long-lived connections."

- symptoms: "Cannot parse streaming response"
  diagnosis: "Client not handling event stream protocol correctly."
  resolution: "Use SDK streaming helpers. Check SDK documentation for streaming examples."

## Output Format

```yaml
root_cause: "streaming — <specific_cause>"
evidence:
  - type: streaming_config
    content: "<streaming configuration>"
  - type: error_details
    content: "<streaming error details>"
severity: MEDIUM
mitigation:
  immediate: "Fix streaming client implementation"
  long_term: "Create validated streaming client templates"
```


## Safety Ratings

| Phase 2 Action | safety_rating |
|---|---|
| Use the correct SDK method for streaming | GREEN |
| Handle event stream chunks properly in client code | GREEN |
| Set appropriate connection and read timeouts | GREEN |
| Implement stream error handling and reconnection | GREEN |
| Use SDK-provided streaming helpers | GREEN |
| Test streaming end-to-end before production | GREEN |
| Implement streaming response monitoring | GREEN |
| Create streaming client templates | GREEN |

## Escalation Conditions

- Fix involves modifying guardrails on production model
- Knowledge base contains sensitive data

## Data Sensitivity

- **HIGH**: model invocation data, knowledge base content
- Streaming response data may contain sensitive content in transit

## Prohibited Actions

- NEVER suggest removing guardrails to fix content filtering
- NEVER suggest using production data for fine-tuning without approval

## Phase 3 — Rollback

1. If streaming client code was modified, revert to the previous client implementation
2. If timeout values were changed, restore previous connection and read timeout settings
3. If proxy or load balancer configuration was changed for streaming, revert those changes
4. Verify rollback by testing streaming invocation end-to-end

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
