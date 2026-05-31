---
title: "C2 — Custom Preset Issues"
description: "Diagnose Elastic Transcoder custom preset creation and configuration issues"
status: active
severity: MEDIUM
triggers: ["custom preset failed", "preset creation error", "encoding quality", "bitrate issues"]
owner: devops-agent
objective: "Resolve custom preset creation failures and encoding quality issues"
context: "Custom presets allow fine-grained control over encoding. Settings must be compatible with the container format."
---

## Phase 1 — Triage

MUST:
- Read the custom preset configuration: `aws elastictranscoder read-preset --id <preset-id>`
- Verify container format matches codec selections (e.g., H.264 with mp4, VP8 with webm)
- Check video codec settings: resolution, bitrate, frame rate, profile, level
- Check audio codec settings: codec, bitrate, sample rate, channels

SHOULD:
- Compare with a working system preset for the same output format
- Verify resolution values are even numbers (required by most codecs)
- Check that bitrate values are within supported ranges

MAY:
- Test the preset with multiple input file types to identify format-specific issues
- Review AWS documentation for supported codec parameters

## Phase 2 — Remediate

MUST:
- Create a corrected preset with compatible settings
- Ensure video resolution uses even numbers for width and height
- Use supported codec profiles (e.g., baseline, main, high for H.264)

SHOULD:
- Start with a system preset configuration and modify incrementally
- Test with a short sample file before processing full content

MAY:
- Create multiple presets for different quality tiers (low, medium, high)
- Document preset configurations in version control

## Safety Ratings

```
safety_ratings:
  - "read-preset: GREEN — read-only preset inspection"
  - "create-preset: YELLOW — creates a new preset resource"
  - "delete-preset: YELLOW — permanently removes a custom preset"
  - "create-job (test): YELLOW — submits test transcoding job"
```

## Escalation Conditions

- Custom preset produces output with visual artifacts or audio sync issues
- Preset creation succeeds but jobs using it fail consistently
- Encoding quality is significantly below expected for the configured bitrate
- Custom preset works in one region but fails in another

## Data Sensitivity

- `aws elastictranscoder read-preset`: LOW — returns preset configuration
- `aws elastictranscoder create-preset`: LOW — creates encoding configuration only
- `aws s3 ls`: LOW — returns file listing

## Prohibited Actions

- NEVER set video bitrate to 0 unless intending audio-only output
- NEVER use unsupported codec profiles or levels
- NEVER delete a custom preset without checking if active jobs reference it
- NEVER copy system preset IDs for custom preset references

## Phase 3 — Rollback

1. If a faulty custom preset was created, delete it: `aws elastictranscoder delete-preset --id <preset-id>`
2. Revert any jobs to use the previously working preset
3. If test jobs produced incorrect output, clean up output files: `aws s3 rm s3://<output-bucket>/<test-output-prefix> --recursive`
4. Restore the original preset configuration if it was documented
5. Verify production jobs are using the correct preset before resuming

## Common Issues

- symptoms: "Preset creation fails with 'Invalid codec' error"
  diagnosis: "Selected codec is not compatible with the container format."
  resolution: "Use compatible combinations: mp4 (H.264/AAC), webm (VP8/Vorbis), flac (FLAC), ts (H.264/AAC for HLS)."

- symptoms: "Output video has black bars or incorrect aspect ratio"
  diagnosis: "Preset resolution does not match input aspect ratio and sizing policy is not set correctly."
  resolution: "Set SizingPolicy to 'Fit', 'Fill', 'Stretch', or 'ShrinkToFit' based on desired behavior."

## Output Format

```yaml
root_cause: "custom_preset — <specific_cause>"
evidence:
  - type: preset_config
    content: "<codec and container details>"
severity: MEDIUM
mitigation:
  immediate: "Create corrected preset"
  long_term: "Maintain preset configuration documentation"
```

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
