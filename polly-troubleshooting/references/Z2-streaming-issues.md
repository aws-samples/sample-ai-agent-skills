---
title: "Z2 — Streaming Issues"
description: "Diagnose speech synthesis streaming problems"
status: active
severity: MEDIUM
triggers: ["streaming error", "chunked transfer", "audio stuttering"]
owner: devops-agent
objective: "Identify and resolve streaming synthesis issues"
context: "Streaming issues occur when: network latency high, buffer too small, connection timeout, or client not processing chunks."
---

## Phase 1 — Triage

MUST:
- Verify client is processing HTTP chunked transfer correctly
- Check network latency to Polly endpoint
- Verify client buffer size is appropriate
- Check for connection timeout settings

SHOULD:
- Test with shorter text to isolate network vs content issues
- Verify SDK version supports streaming
- Check if client is waiting for full response vs processing chunks

## Phase 2 — Remediate

MUST:
- Process audio chunks as they arrive (don't wait for full response)
- Increase client buffer size for smooth playback
- Fix connection timeout settings

SHOULD:
- Use SDK streaming APIs for efficient chunk processing
- Implement audio buffering for smooth playback

## Output Format

```yaml
root_cause: "streaming — <specific_cause>"
evidence:
  - type: network_test
    content: "<latency and throughput>"
severity: MEDIUM
```


## Safety Ratings
- Overall Risk: MEDIUM
- Data Exposure Risk: MEDIUM — streaming audio may contain sensitive content
- Service Disruption Risk: MEDIUM — streaming issues affect real-time applications
- Compliance Impact: LOW — streaming is an operational concern

## Escalation Conditions
- Audio streaming completely fails
- Streaming latency exceeds real-time requirements
- Stream interruptions cause application failures
- Streaming connection drops repeatedly
- Streaming quality degrades significantly

## Data Sensitivity
- Streaming audio content — CONFIDENTIAL (may contain sensitive data)
- Streaming configurations — INTERNAL
- Connection parameters — INTERNAL
- Latency metrics — INTERNAL
- Never expose streaming content containing sensitive data

## Prohibited Actions
- Do NOT stream audio over unencrypted connections
- Do NOT ignore streaming latency issues for real-time applications
- Do NOT bypass streaming authentication
- Do NOT expose streaming endpoints without proper access controls
- Do NOT ignore persistent streaming failures
- Do NOT stream sensitive content without encryption

## Phase 3 — Rollback
- Restore previous streaming configurations
- Revert connection parameter changes
- Restore streaming quality settings
- Verify streaming functionality after changes
- Document streaming issues and latency patterns
- Update streaming configuration guidelines

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
