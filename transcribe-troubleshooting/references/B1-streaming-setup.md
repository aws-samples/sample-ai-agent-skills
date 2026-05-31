---
title: "B1 — Streaming Transcription Setup"
description: "Diagnose streaming transcription configuration issues"
status: active
severity: HIGH
triggers:
  - "streaming not working"
  - "WebSocket connection failed"
  - "streaming setup error"
owner: devops-agent
objective: "Identify and resolve streaming transcription setup issues"
context: "Streaming transcription uses WebSocket or HTTP/2. Requires PCM 16-bit audio. Has specific SDK requirements. Connection has idle timeout."
---

## Phase 1 — Triage

MUST:
- Verify SDK supports streaming transcription
- Check audio encoding (PCM 16-bit signed little-endian)
- Verify sample rate (8kHz or 16kHz typical)
- Check network allows WebSocket/HTTP2 connections

SHOULD:
- Test with AWS SDK streaming example
- Verify IAM permissions for transcribe:StartStreamTranscription
- Check for proxy/firewall blocking WebSocket

MAY:
- Test with pre-recorded audio streamed as chunks
- Check connection timeout settings

## Phase 2 — Remediate

MUST:
- Use SDK that supports streaming (latest AWS SDK)
- Send PCM 16-bit audio at correct sample rate
- Configure proper IAM permissions

SHOULD:
- Implement reconnection logic
- Handle partial results in stream

MAY:
- Use Amazon Transcribe Streaming SDK for simplified integration

## Common Issues

- symptoms: "WebSocket connection rejected"
  diagnosis: "IAM permissions missing or presigned URL incorrect."
  resolution: "Add transcribe:StartStreamTranscription permission. Check URL signing."

- symptoms: "Streaming returns garbled results"
  diagnosis: "Wrong audio encoding or sample rate."
  resolution: "Use PCM 16-bit signed little-endian. Match sample rate parameter."

## Output Format

```yaml
root_cause: "streaming_setup — <specific_cause>"
evidence:
  - type: connection_status
    content: "<streaming connection details>"
severity: HIGH
mitigation:
  immediate: "Fix audio encoding or connection configuration"
  long_term: "Implement robust streaming with reconnection"
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
