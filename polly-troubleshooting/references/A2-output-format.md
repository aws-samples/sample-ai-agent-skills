---
title: "A2 — Output Format Issues"
description: "Diagnose audio output format and encoding problems"
status: active
severity: MEDIUM
triggers: ["audio corrupt", "wrong format", "sample rate error", "encoding issue"]
owner: devops-agent
objective: "Identify and resolve output format issues"
context: "Format issues occur when: sample rate incompatible, format not supported, PCM missing headers, or encoding mismatch."
---

## Phase 1 — Triage

MUST:
- Verify output format is supported (mp3, ogg_vorbis, pcm, json)
- Check sample rate is valid for format (8000, 16000, 22050, 24000)
- Verify audio player supports the output format
- Check if PCM output needs WAV header wrapping

SHOULD:
- Test with default sample rate first
- Verify neural voices use 24000 Hz default
- Check if json format was accidentally used (returns speech marks, not audio)

## Phase 2 — Remediate

MUST:
- Use compatible sample rate for output format
- Add WAV headers to PCM output if needed
- Use correct format for use case (mp3 for playback, pcm for processing)

SHOULD:
- Use 24000 Hz for neural voices
- Document format requirements for downstream consumers

## Output Format

```yaml
root_cause: "output_format — <specific_cause>"
evidence:
  - type: format_config
    content: "<format and sample rate>"
severity: MEDIUM
```


## Safety Ratings
- Overall Risk: LOW
- Data Exposure Risk: LOW — output format is a configuration concern
- Service Disruption Risk: LOW — format issues affect audio quality, not availability
- Compliance Impact: LOW — output format is an operational concern

## Escalation Conditions
- Output format incompatible with target application
- Audio quality degradation after format changes
- Format conversion failures
- Unsupported format requested
- Output file size exceeds limits

## Data Sensitivity
- Audio output files — INTERNAL
- Format configurations — INTERNAL
- Sample rate settings — INTERNAL
- Codec configurations — INTERNAL
- Never expose audio content containing sensitive information

## Prohibited Actions
- Do NOT use uncompressed formats for large-scale production without cost analysis
- Do NOT ignore format compatibility warnings
- Do NOT bypass format validation
- Do NOT store audio output in publicly accessible locations
- Do NOT use formats that degrade audio quality below acceptable thresholds
- Do NOT ignore output file size limits

## Phase 3 — Rollback
- Restore previous output format configurations
- Revert sample rate changes
- Restore codec settings
- Verify audio quality after format changes
- Document format issues and compatibility requirements
- Update format selection guidelines

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
