---
title: "F1 — Streaming Conversation Issues"
description: "Diagnose streaming conversation failures"
status: active
severity: MEDIUM
triggers:
  - "streaming not working"
  - "StartConversation error"
  - "WebSocket connection failed"
  - "audio streaming issues"
owner: devops-agent
objective: "Identify and resolve streaming conversation issues"
context: "Streaming conversations use bidirectional HTTP/2 event streams via StartConversation API. Different from synchronous RecognizeText/RecognizeUtterance. Requires specific SDK support."
---

## Phase 1 — Triage

MUST:
- Verify bot alias supports streaming: `aws lexv2-models describe-bot-alias --bot-id <bot-id> --bot-alias-id <alias-id>`
- Check SDK version supports streaming (StartConversation)
- Verify network allows HTTP/2 connections
- Check CloudWatch for Lex runtime errors

SHOULD:
- Test with synchronous API first (RecognizeText) to isolate streaming issues
- Verify audio format matches expected input (PCM 16-bit, 8kHz or 16kHz)
- Check connection timeout settings

MAY:
- Review client-side WebSocket/HTTP2 implementation
- Check for proxy/firewall blocking HTTP/2

## Phase 2 — Remediate

MUST:
- Use SDK that supports Lex V2 streaming (latest AWS SDK)
- Configure correct audio format for voice input
- Handle event stream protocol correctly

SHOULD:
- Implement reconnection logic for dropped connections
- Set appropriate idle timeout handling

MAY:
- Use Amazon Connect for managed voice integration

## Common Issues

- symptoms: "StartConversation returns connection error"
  diagnosis: "SDK doesn't support HTTP/2 streaming or network blocks it."
  resolution: "Update SDK. Check network/proxy allows HTTP/2."

- symptoms: "Audio input not recognized"
  diagnosis: "Wrong audio format (sample rate, encoding)."
  resolution: "Use PCM 16-bit signed little-endian at 8kHz or 16kHz."

## Output Format

```yaml
root_cause: "streaming — <specific_cause>"
evidence:
  - type: connection_status
    content: "<streaming connection details>"
  - type: audio_config
    content: "<audio format configuration>"
severity: MEDIUM
mitigation:
  immediate: "Fix SDK, audio format, or network configuration"
  long_term: "Implement robust streaming with reconnection logic"
```


## Safety Ratings

| Aspect | Rating | Notes |
|--------|--------|-------|
| Data Risk | Medium | Operations may affect service configurations and data access patterns |
| Service Impact | High | Incorrect changes can disrupt active workloads and dependent services |
| Recovery Complexity | Medium | Most actions are reversible but may require coordinated rollback procedures |
| Blast Radius | Medium | Impact typically scoped to the affected resource and its direct dependents |

## Escalation Conditions

- **Escalate immediately** if the issue involves potential data loss or corruption
- **Escalate immediately** if a production workload is impacted and cannot be restored within the agreed SLA
- **Escalate immediately** if the root cause is unclear after initial diagnostic steps are exhausted
- **Escalate to service team** if the issue appears to be a platform-level bug or undocumented behavior
- **Escalate to security team** if unauthorized access, policy violations, or credential exposure is suspected

## Data Sensitivity

- Verify data classification before performing any read or export operations
- Ensure all troubleshooting outputs (logs, configurations, snapshots) are handled according to the customer's data handling policy
- Mask or redact personally identifiable information (PII) in all shared diagnostics
- Do not copy production data to non-production environments without explicit authorization
- Retain troubleshooting artifacts only for the duration required and dispose of them securely

## Prohibited Actions

- **NEVER** delete or overwrite a published bot alias that is actively serving production traffic
- **NEVER** modify intent or slot configurations on a live bot version without creating a new draft version first
- **NEVER** remove Lambda fulfillment hooks from intents without verifying downstream dependencies
- **NEVER** disable conversation logs on bots subject to compliance or audit requirements
- **NEVER** migrate a V1 bot to V2 in-place without maintaining the V1 bot as a fallback

## Phase 3 � Rollback

1. **Identify rollback trigger**: Determine if the applied change has caused degradation, unexpected behavior, or failed validation checks
2. **Preserve current state**: Capture logs, configuration snapshots, and resource states before initiating rollback
3. **Execute rollback steps**: Revert configuration changes in reverse order of application, verifying each step
4. **Validate service recovery**: Confirm that the service has returned to its pre-change operational state using health checks and functional tests
5. **Document rollback**: Record the rollback trigger, steps taken, time to recovery, and any residual issues for post-incident review
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
  - "NEVER suggest overly broad IAM permissions as a fix"
  - "NEVER suggest disabling security controls for troubleshooting"
  - "NEVER suggest deleting resources without confirmation"
