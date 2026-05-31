---
title: "B1 — Ad Insertion Failures"
description: "Diagnose server-side ad insertion failures in MediaTailor"
status: active
severity: CRITICAL
triggers: ["ads not playing", "ad break empty", "no ads inserted", "ad insertion failed", "avail not filled"]
owner: devops-agent
objective: "Resolve ad insertion failures to restore ad monetization"
context: "MediaTailor inserts ads by rewriting manifests to include ad segments from the ADS response. Failures can occur at ADS request, VAST parsing, or segment delivery stages."
---

## Phase 1 — Triage

MUST:
- Check ADS error metrics: `aws cloudwatch get-metric-statistics --namespace AWS/MediaTailor --metric-name AdDecisionServer.Errors --dimensions Name=Configuration,Value=<config-name> --start-time <start> --end-time <end> --period 300 --statistics Sum`
- Check fill rate: `aws cloudwatch get-metric-statistics --namespace AWS/MediaTailor --metric-name Avail.FillRate --dimensions Name=Configuration,Value=<config-name> --start-time <start> --end-time <end> --period 300 --statistics Average`
- Verify ADS URL is reachable and returns valid VAST: `curl -v "<ads-url>"`

SHOULD:
- Check for VAST parsing errors in CloudWatch Logs (if logging enabled)
- Verify ad segments are accessible from the ad segment URLs in VAST response

MAY:
- Test with a known-good VAST response to isolate ADS vs MediaTailor issues
- Review ad marker (SCTE-35) presence in origin manifest

## Phase 2 — Remediate

MUST:
- Fix ADS URL if returning errors or invalid VAST
- Ensure origin manifest contains proper ad markers (EXT-X-CUE-OUT/IN for HLS, Period for DASH)
- Verify ad segment URLs in VAST response are accessible

SHOULD:
- Configure slate URL for unfilled ad time
- Enable prefetch schedules to improve fill rates

MAY:
- Adjust avail suppression mode for live streams
- Configure bumpers for ad break transitions

## Safety Ratings

```
safety_ratings:
  - "cloudwatch get-metric-statistics: GREEN — read-only metrics query"
  - "curl ADS URL: GREEN — read-only HTTP request to ad server"
  - "put-playback-configuration: YELLOW — updates affect future sessions"
  - "create-prefetch-schedule: YELLOW — changes ad fetching behavior"
```

## Escalation Conditions

- Ad fill rate drops below 50% affecting revenue
- ADS is returning errors for all configurations simultaneously
- Ad segments are returning 403/404 from CDN indicating CDN misconfiguration
- SCTE-35 markers are missing from origin requiring encoder changes

## Data Sensitivity

- CloudWatch ad metrics: LOW — aggregate statistics only
- ADS URLs: HIGH — may contain API keys, session tokens, or viewer identifiers
- VAST responses: MEDIUM — contain ad creative URLs and tracking pixels
- Viewer session logs: HIGH — may contain viewer IP addresses and device information

## Prohibited Actions

- NEVER expose ADS URLs containing viewer PII or authentication tokens
- NEVER disable ad insertion on a monetized configuration without business approval
- NEVER modify SCTE-35 marker configuration on a live encoder during broadcast
- NEVER delete prefetch schedules during active ad campaigns

## Phase 3 — Rollback

1. If ADS URL change caused failures, revert to previous ADS URL in configuration
2. If prefetch schedule changes reduced fill rate, delete new schedule: `aws mediatailor delete-prefetch-schedule --PlaybackConfigurationName <name> --Name <schedule-name>`
3. If avail suppression changes caused issues, revert suppression mode
4. Monitor Avail.FillRate metric for 15 minutes after rollback to confirm recovery

## Common Issues

- symptoms: "Avail.FillRate is 0% but ADS is responding"
  diagnosis: "Origin manifest missing ad markers (SCTE-35/EXT-X-CUE-OUT)."
  resolution: "Verify origin manifest contains ad markers. Check encoder SCTE-35 configuration."

- symptoms: "AdDecisionServer.Errors count is high"
  diagnosis: "ADS returning non-200 responses or invalid VAST XML."
  resolution: "Test ADS URL directly. Check ADS server health. Validate VAST response format."

## Output Format

```yaml
root_cause: "ad_insertion — <specific_cause>"
evidence:
  - type: ad_metrics
    content: "<fill rate and error metrics>"
severity: CRITICAL
mitigation:
  immediate: "Fix ad insertion pipeline"
  long_term: "Implement ADS health monitoring and fallback"
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
