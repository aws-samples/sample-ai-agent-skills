---
title: "F1 — Bumper and Slate Configuration"
description: "Diagnose bumper and slate configuration and playback issues"
status: active
severity: MEDIUM
triggers: ["bumper not playing", "slate not showing", "unfilled ad time", "bumper error", "slate URL error"]
owner: devops-agent
objective: "Resolve bumper and slate configuration issues for seamless ad break experience"
context: "Bumpers are short clips before/after ad breaks. Slates fill unfilled ad time when ADS returns fewer ads than break duration. Both require valid MP4 files accessible via HTTPS."
---

## Phase 1 — Triage

MUST:
- Check bumper configuration: `aws mediatailor get-playback-configuration --Name <config-name> --query '{Bumper:Bumper,SlateAdUrl:SlateAdUrl}'`
- Verify bumper/slate URLs are accessible: `curl -I "<bumper-or-slate-url>"`
- Check if bumper/slate MP4 files are valid video files

SHOULD:
- Verify MP4 encoding matches content stream profiles (codec, resolution, bitrate)
- Check CloudWatch metrics for slate usage frequency
- Test playback with intentionally empty ADS response to trigger slate

MAY:
- Review bumper duration against ad break timing
- Check if bumper URLs use HTTPS (required)

## Phase 2 — Remediate

MUST:
- Fix bumper/slate URLs to point to accessible HTTPS endpoints
- Ensure MP4 files are properly encoded and not corrupted
- Verify MP4 codec compatibility with content stream (H.264/AAC recommended)

SHOULD:
- Encode bumper/slate at multiple bitrates matching content ABR ladder
- Keep bumper duration short (3-5 seconds recommended)

MAY:
- Configure separate start and end bumpers for different ad break experiences
- Set up CDN caching for bumper/slate files to reduce origin load

## Safety Ratings

```
safety_ratings:
  - "get-playback-configuration: GREEN — read-only configuration retrieval"
  - "curl bumper/slate URL: GREEN — read-only HTTP request"
  - "put-playback-configuration bumper update: YELLOW — changes ad break experience"
  - "S3 upload of new bumper/slate: YELLOW — replaces media files"
```

## Escalation Conditions

- Slate plays continuously indicating complete ADS failure
- Bumper encoding incompatible with content requiring re-encoding
- Bumper/slate S3 bucket access requires cross-account permissions
- Slate URL returns 403 during live event requiring immediate CDN fix

## Data Sensitivity

- Bumper/slate URLs: LOW — public media file URLs
- Bumper/slate content: LOW — generic transition clips
- Configuration details: LOW — URL references only
- Playback metrics: LOW — aggregate usage statistics

## Prohibited Actions

- NEVER use copyrighted content as bumper/slate without proper licensing
- NEVER remove slate configuration from a production configuration without ensuring 100% ad fill
- NEVER replace bumper/slate files at the same URL during active sessions (use versioned URLs)
- NEVER configure bumper duration longer than 10 seconds without business approval

## Phase 3 — Rollback

1. If bumper was removed or changed, restore previous bumper URL in configuration
2. If slate was removed causing blank screen during unfilled time, restore slate URL
3. If new bumper/slate MP4 is corrupted, revert to previous version at the URL
4. Test playback with empty ADS response to verify slate displays correctly

## Common Issues

- symptoms: "Slate not playing during unfilled ad time, black screen instead"
  diagnosis: "SlateAdUrl not configured or URL returns 404."
  resolution: "Configure SlateAdUrl with valid HTTPS URL to MP4 file. Verify URL accessibility."

- symptoms: "Bumper plays but causes video glitch at ad break transition"
  diagnosis: "Bumper MP4 encoding does not match content stream codec/resolution profile."
  resolution: "Re-encode bumper to match content profiles (same codec, compatible resolution, matching frame rate)."

## Output Format

```yaml
root_cause: "bumper_slate — <specific_cause>"
evidence:
  - type: bumper_slate_config
    content: "<bumper and slate configuration details>"
severity: MEDIUM
mitigation:
  immediate: "Fix bumper/slate URL or encoding"
  long_term: "Implement bumper/slate validation in deployment pipeline"
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
