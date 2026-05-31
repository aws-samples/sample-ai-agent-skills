---
title: "A1 — Speech Synthesis Failures"
description: "Diagnose SynthesizeSpeech API failures"
status: active
severity: HIGH
triggers: ["synthesis failed", "SynthesizeSpeech error", "no audio output"]
owner: devops-agent
objective: "Identify and resolve speech synthesis failures"
context: "Synthesis fails when: text too long, voice ID invalid, engine not supported, or IAM permissions missing."
---

## Phase 1 — Triage

MUST:
- Check error message from API response
- Verify text length (max 3000 chars for text, 6000 for SSML)
- Verify voice ID: `aws polly describe-voices --language-code <code>`
- Check IAM permissions for polly:SynthesizeSpeech

SHOULD:
- Test with simple text first (no SSML)
- Verify output format is valid (mp3, ogg_vorbis, pcm)
- Check sample rate compatibility with format

MAY:
- Test: `aws polly synthesize-speech --text "Hello" --output-format mp3 --voice-id Joanna output.mp3`
- Check CloudWatch for throttling metrics

## Phase 2 — Remediate

MUST:
- Reduce text to within character limits
- Use correct voice ID (case-sensitive)
- Grant polly:SynthesizeSpeech permission

SHOULD:
- Use StartSpeechSynthesisTask for long content
- Implement retry with backoff for throttling

## Output Format

```yaml
root_cause: "synthesis_failure — <specific_cause>"
evidence:
  - type: api_error
    content: "<error message>"
severity: HIGH
```


## Safety Ratings
- Overall Risk: LOW
- Data Exposure Risk: MEDIUM — synthesis input text may contain sensitive content
- Service Disruption Risk: MEDIUM — synthesis failures affect application functionality
- Compliance Impact: LOW — synthesis is an operational concern

## Escalation Conditions
- All synthesis requests fail
- Synthesis latency exceeds acceptable thresholds
- Synthesis produces garbled or incorrect audio
- Service quota limits reached
- Synthesis fails for specific languages or voices

## Data Sensitivity
- Input text for synthesis — CONFIDENTIAL (may contain sensitive content)
- Audio output — INTERNAL
- Voice configurations — INTERNAL
- API request logs — INTERNAL
- Never log input text containing PII or sensitive data

## Prohibited Actions
- Do NOT synthesize text containing PII without proper authorization
- Do NOT ignore synthesis failure patterns
- Do NOT bypass synthesis rate limits
- Do NOT store synthesized audio containing sensitive content without encryption
- Do NOT share synthesis API credentials
- Do NOT use synthesis for generating misleading audio content

## Phase 3 — Rollback
- Restore previous synthesis configurations
- Revert voice selection changes
- Restore API rate limit configurations
- Verify synthesis functionality after changes
- Delete synthesized audio containing sensitive data if exposed
- Document synthesis issues and resolutions

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
