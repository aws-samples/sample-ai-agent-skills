---
title: "C1 — Preset Errors"
description: "Diagnose Elastic Transcoder preset errors"
status: active
severity: MEDIUM
triggers: ["preset error", "preset not found", "invalid preset", "preset validation"]
owner: devops-agent
objective: "Resolve preset-related errors including missing presets, invalid configurations, and compatibility issues"
context: "Presets define encoding settings. System presets are immutable. Custom presets can be created and deleted."
---

## Phase 1 — Triage

MUST:
- List available presets: `aws elastictranscoder list-presets`
- Read the specific preset: `aws elastictranscoder read-preset --id <preset-id>`
- Verify the preset ID referenced in the job is correct and exists
- Check if the preset is a system preset (ID starts with "1351620000001-") or custom

SHOULD:
- Verify preset container format matches expected output format
- Check codec settings are compatible with the container format
- Compare preset settings with a known-working preset

MAY:
- Review CloudTrail for CreatePreset or DeletePreset events
- Check if the preset was recently deleted

## Phase 2 — Remediate

MUST:
- Use a valid preset ID in job submissions
- For custom presets with errors, create a new corrected preset: `aws elastictranscoder create-preset --name <name> --container <container> --video <video-params> --audio <audio-params>`
- Verify codec-container compatibility before creating presets

SHOULD:
- Test new presets with a small sample file before production use
- Document custom preset configurations for team reference

MAY:
- Create preset templates for common encoding profiles
- Use system presets as a starting point for custom preset configuration

## Safety Ratings

```
safety_ratings:
  - "list-presets: GREEN — read-only preset enumeration"
  - "read-preset: GREEN — read-only preset inspection"
  - "create-preset: YELLOW — creates a new preset resource"
  - "delete-preset: YELLOW — removes a custom preset permanently"
```

## Escalation Conditions

- System presets return errors or are unavailable in the region
- Custom preset creation fails with internal service errors despite valid configuration
- Preset works for some input files but fails for others with the same format
- Preset encoding quality is significantly different from expected output

## Data Sensitivity

- `aws elastictranscoder list-presets`: LOW — returns preset metadata
- `aws elastictranscoder read-preset`: LOW — returns preset configuration
- `aws cloudtrail lookup-events`: MEDIUM — reveals API call history

## Prohibited Actions

- NEVER attempt to modify or delete system presets — they are immutable
- NEVER delete custom presets that are referenced by active or pending jobs
- NEVER create presets with incompatible codec-container combinations
- NEVER assume a preset ID without verifying it exists in the current region

## Phase 3 — Rollback

1. If a custom preset was created with incorrect settings, delete it: `aws elastictranscoder delete-preset --id <preset-id>`
2. If jobs were submitted with the incorrect preset, cancel pending jobs: `aws elastictranscoder cancel-job --id <job-id>`
3. Resubmit jobs with the correct preset after verification
4. If a working custom preset was accidentally deleted, recreate it with the documented configuration
5. Verify all pipeline job templates reference valid preset IDs

## Common Issues

- symptoms: "Job fails with 'Preset not found' error"
  diagnosis: "Preset ID is incorrect or preset was deleted."
  resolution: "List presets to find the correct ID. Recreate the preset if it was deleted."

- symptoms: "CreatePreset returns ValidationException"
  diagnosis: "Codec and container format are incompatible (e.g., VP8 with MP4)."
  resolution: "Use compatible codec-container combinations: H.264/AAC with MP4, VP8/Vorbis with WebM."

## Output Format

```yaml
root_cause: "preset_error — <specific_cause>"
evidence:
  - type: preset_issue
    content: "<preset configuration details>"
severity: MEDIUM
mitigation:
  immediate: "Fix or replace preset"
  long_term: "Document and version custom presets"
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
