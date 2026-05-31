---
title: "D2 — Manifest Manipulation"
description: "Diagnose manifest manipulation and ad marker processing issues"
status: active
severity: HIGH
triggers: ["manifest error", "ad markers missing", "SCTE-35 not detected", "HLS manifest wrong", "DASH MPD error"]
owner: devops-agent
objective: "Resolve manifest manipulation issues affecting ad insertion and playback"
context: "MediaTailor manipulates HLS (.m3u8) and DASH (.mpd) manifests to insert ad segments. It relies on ad markers (SCTE-35, EXT-X-DATERANGE, EXT-X-CUE-OUT) in the origin manifest to identify ad break opportunities."
---

## Phase 1 — Triage

MUST:
- Fetch origin manifest directly: `curl -s "<origin-url>" | head -50`
- Check for ad markers in origin manifest (SCTE-35, EXT-X-CUE-OUT/IN, EXT-X-DATERANGE)
- Fetch personalized manifest from MediaTailor endpoint and compare with origin

SHOULD:
- Verify manifest format matches configuration (HLS vs DASH)
- Check for proper segment duration alignment between content and ads
- Review ad marker format and parameters

MAY:
- Compare manifest before and after MediaTailor processing
- Check for manifest manipulation errors in CloudWatch Logs

## Phase 2 — Remediate

MUST:
- Ensure origin manifest contains proper ad markers for the configured format
- For HLS: verify EXT-X-CUE-OUT/EXT-X-CUE-IN or EXT-X-DATERANGE tags
- For DASH: verify Period elements with ad signaling

SHOULD:
- Align ad segment durations with ad break durations in markers
- Configure ad marker passthrough if downstream systems need markers

MAY:
- Adjust live edge offset for live manifests
- Configure manifest segment count and window size

## Safety Ratings

```
safety_ratings:
  - "curl origin manifest: GREEN — read-only HTTP request"
  - "curl personalized manifest: GREEN — read-only HTTP request"
  - "put-playback-configuration manifest settings: YELLOW — changes manifest behavior"
  - "Encoder ad marker changes: RED — affects all downstream consumers of the stream"
```

## Escalation Conditions

- Ad markers are missing from origin requiring encoder configuration changes
- Manifest manipulation causes player compatibility issues across device types
- Live manifest timing issues cause viewer-visible glitches during ad transitions
- DASH manifest Period structure incompatible with MediaTailor requirements

## Data Sensitivity

- Origin manifests: LOW — content segment URLs and timing metadata
- Personalized manifests: MEDIUM — contain session-specific ad segment URLs
- Ad marker data: LOW — timing and signaling metadata
- SCTE-35 binary data: LOW — ad break signaling information

## Prohibited Actions

- NEVER modify encoder SCTE-35 configuration during a live broadcast
- NEVER disable ad marker processing without understanding downstream impact
- NEVER share personalized manifest URLs containing session tokens
- NEVER assume HLS and DASH manifests behave identically for ad insertion

## Phase 3 — Rollback

1. If manifest manipulation settings were changed, revert configuration to previous settings
2. If ad marker passthrough was disabled, re-enable it for downstream systems
3. If live edge offset was changed causing playback issues, revert to previous value
4. Test personalized manifest playback after rollback to confirm ad insertion works

## Common Issues

- symptoms: "No ad breaks appear in personalized manifest despite ADS returning ads"
  diagnosis: "Origin manifest missing ad markers (EXT-X-CUE-OUT/IN or SCTE-35)."
  resolution: "Verify encoder is inserting SCTE-35 markers. Check origin manifest for EXT-X-CUE-OUT tags."

- symptoms: "Ad break duration does not match expected duration"
  diagnosis: "Ad marker duration in origin does not match ADS response ad durations."
  resolution: "Verify SCTE-35 duration matches expected ad break length. Configure ADS to return ads matching break duration."

## Output Format

```yaml
root_cause: "manifest_manipulation — <specific_cause>"
evidence:
  - type: manifest_detail
    content: "<origin and personalized manifest comparison>"
severity: HIGH
mitigation:
  immediate: "Fix manifest or ad marker configuration"
  long_term: "Implement manifest validation in monitoring pipeline"
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
