---
title: "A2 — Playback Configuration"
description: "Diagnose playback configuration issues affecting stream delivery"
status: active
severity: HIGH
triggers: ["playback not working", "stream errors", "configuration mismatch", "personalization failed"]
owner: devops-agent
objective: "Resolve playback configuration issues affecting viewer experience"
context: "Playback configurations control how MediaTailor personalizes streams including ad insertion behavior, CDN routing, and manifest manipulation settings."
---

## Phase 1 — Triage

MUST:
- Get configuration details: `aws mediatailor get-playback-configuration --Name <config-name>`
- Verify the playback URL format: `https://<mediatailor-endpoint>/v1/master/<hashed-account-id>/<config-name>/<origin-path>`
- Check CloudWatch metrics: `aws cloudwatch get-metric-statistics --namespace AWS/MediaTailor --metric-name Avail.FillRate --dimensions Name=Configuration,Value=<config-name> --start-time <start> --end-time <end> --period 300 --statistics Average`

SHOULD:
- Test origin URL directly to verify it returns valid manifests
- Verify CDN prefix configuration matches actual CDN distribution

MAY:
- Check session initialization logs if logging is enabled
- Compare configuration with a known working configuration

## Phase 2 — Remediate

MUST:
- Fix VideoContentSourceUrl if origin is unreachable
- Correct CDN prefix URLs to match actual CDN distribution domains
- Ensure personalization threshold is appropriate (default 0 means no threshold)

SHOULD:
- Update DASH/HLS-specific settings to match content format
- Configure slate URL for unfilled ad time

MAY:
- Adjust avail suppression settings for live streams
- Configure ad marker passthrough settings

## Safety Ratings

```
safety_ratings:
  - "get-playback-configuration: GREEN — read-only configuration retrieval"
  - "put-playback-configuration: YELLOW — updates affect future sessions, not active ones"
  - "cloudwatch get-metric-statistics: GREEN — read-only metrics query"
  - "Testing playback URL: GREEN — read-only stream request"
```

## Escalation Conditions

- Playback failures affect more than 10% of active viewers
- Configuration changes do not take effect after 10 minutes
- Origin server is returning intermittent errors causing viewer-facing issues
- CDN configuration requires changes to CloudFront distribution settings

## Data Sensitivity

- `aws mediatailor get-playback-configuration`: MEDIUM — reveals ADS URLs and origin URLs
- Playback session URLs: MEDIUM — contain hashed account identifiers
- CloudWatch metrics: LOW — aggregate statistics only
- Viewer session data: HIGH — may contain viewer IP addresses in logs

## Prohibited Actions

- NEVER share playback session URLs containing viewer-specific tokens
- NEVER modify configuration during peak viewing hours without a rollback plan
- NEVER disable personalization on a live configuration without stakeholder approval
- NEVER expose origin server URLs that bypass CDN authentication

## Phase 3 — Rollback

1. If configuration update caused playback issues, revert to previous settings: `aws mediatailor put-playback-configuration --Name <name>` with original parameters
2. If CDN prefix change broke playback, revert CDN prefixes and wait 2-5 minutes for propagation
3. If personalization threshold change caused ad insertion issues, reset to previous value
4. Monitor CloudWatch Avail.FillRate and AdDecisionServer.Errors metrics after rollback

## Common Issues

- symptoms: "Playback URL returns 403 Forbidden"
  diagnosis: "CDN prefix mismatch or origin authentication failure."
  resolution: "Verify ContentSegmentUrlPrefix matches CDN domain. Check origin server access permissions."

- symptoms: "Ads not appearing in stream despite valid configuration"
  diagnosis: "ADS returning empty VAST or personalization threshold blocking insertion."
  resolution: "Check AdDecisionServer.Errors metric. Test ADS URL directly. Review personalization threshold."

## Output Format

```yaml
root_cause: "playback_configuration — <specific_cause>"
evidence:
  - type: configuration_detail
    content: "<configuration JSON>"
severity: HIGH
mitigation:
  immediate: "Fix configuration parameter"
  long_term: "Implement configuration validation in CI/CD"
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
