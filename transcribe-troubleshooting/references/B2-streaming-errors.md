---
title: "B2 — Streaming Transcription Errors"
description: "Diagnose streaming transcription runtime errors"
status: active
severity: MEDIUM
triggers: ["streaming error", "stream disconnected", "partial results missing"]
owner: devops-agent
objective: "Identify and resolve streaming runtime errors"
context: "Streaming errors include connection drops, audio processing failures, and timeout issues. Partial results are interim; final results are definitive."
---

## Phase 1 — Triage

MUST:
- Check error message from stream event
- Verify audio is being sent continuously (no long gaps)
- Check connection duration vs idle timeout
- Verify audio chunk size and timing

SHOULD:
- Monitor for BadRequestException events in stream
- Check network stability
- Verify audio quality during stream

MAY:
- Log all stream events for debugging
- Check for concurrent stream limits

## Phase 2 — Remediate

MUST:
- Send audio continuously to avoid idle timeout
- Handle stream errors with reconnection
- Distinguish partial vs final results

SHOULD:
- Implement heartbeat/keepalive mechanism
- Buffer audio for reconnection scenarios

MAY:
- Implement stream health monitoring

## Common Issues

- symptoms: "Stream disconnects after silence"
  diagnosis: "Idle timeout reached during silence."
  resolution: "Send silence frames to keep connection alive."

- symptoms: "Partial results but no final results"
  diagnosis: "Stream closed before final results generated."
  resolution: "Wait for final result events before closing stream."

## Output Format

```yaml
root_cause: "streaming_errors — <specific_cause>"
evidence:
  - type: stream_events
    content: "<stream error events>"
severity: MEDIUM
mitigation:
  immediate: "Fix audio streaming or handle timeouts"
  long_term: "Implement robust streaming error handling"
```


## Safety Ratings

| Phase 2 Action | Rating | Rationale |
|---|---|---|
| Investigate and collect diagnostic data | GREEN | Read-only analysis; no state change |
| Modify configuration settings | YELLOW | Configuration change; reversible with backup |
| Restart or reboot components | YELLOW | Service interruption; recoverable |
| Delete or remove resources | RED | Potentially destructive; requires confirmation and backup |

## Escalation Conditions

- Production environment impact detected or imminent
- Multiple failed remediation attempts (>2 cycles without resolution)
- Security-related root cause identified (unauthorized access, data exposure)
- Issue requires cross-service or cross-account coordination
- Customer-facing service degradation exceeding SLA thresholds
- Regulatory or compliance implications discovered during investigation

## Data Sensitivity

- Audio recordings may contain private conversations
- Transcription output preserves all spoken content including PII
- Medical transcriptions contain protected health information (PHI)
- Classification: HIGHLY CONFIDENTIAL - audio content and potential PHI/PII

## Prohibited Actions

- NEVER process audio containing PII without content redaction enabled
- NEVER delete custom vocabularies used by production transcription jobs
- NEVER modify medical transcription settings without clinical team approval
- NEVER store transcription output containing sensitive data without encryption

## Phase 3 -- Rollback

MUST:
- Document all changes made during Phase 2 remediation
- Verify rollback procedures exist before applying any fix
- Confirm service health after rollback using the same diagnostic steps from Phase 1
- Notify stakeholders if rollback is initiated in production

SHOULD:
- Maintain a timeline of all actions taken for post-incident review
- Capture before/after configuration snapshots
- Test rollback in non-production environment when possible
- Update runbook with lessons learned after incident resolution

MAY:
- Automate rollback procedures for frequently occurring issues
- Create CloudWatch alarms to detect regression after rollback

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
