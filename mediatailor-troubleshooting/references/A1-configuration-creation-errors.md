---
title: "A1 — Configuration Creation Errors"
description: "Diagnose MediaTailor playback configuration creation failures"
status: active
severity: HIGH
triggers: ["cannot create configuration", "put-playback-configuration failed", "configuration creation error"]
owner: devops-agent
objective: "Resolve playback configuration creation issues"
context: "Playback configurations define SSAI behavior including ADS URL, origin URL, CDN settings, and personalization thresholds."
---

## Phase 1 — Triage

MUST:
- Check existing configurations: `aws mediatailor list-playback-configurations`
- Verify IAM permissions for mediatailor:PutPlaybackConfiguration
- Validate the VideoContentSourceUrl is a reachable HLS/DASH origin

SHOULD:
- Verify the AdDecisionServerUrl format includes required player parameters
- Check if a configuration with the same name already exists

MAY:
- Review CloudTrail for PutPlaybackConfiguration errors: `aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventName,AttributeValue=PutPlaybackConfiguration --max-results 10`

## Phase 2 — Remediate

MUST:
- Fix IAM permissions to include mediatailor:PutPlaybackConfiguration
- Ensure VideoContentSourceUrl returns valid HLS (.m3u8) or DASH (.mpd) manifests
- Verify AdDecisionServerUrl is a valid HTTPS URL

SHOULD:
- Configure CDN prefixes (AdSegmentUrlPrefix, ContentSegmentUrlPrefix) for production
- Set appropriate personalization threshold percentage

MAY:
- Enable log configuration with initial 5% sampling
- Configure bumper and slate URLs if needed

## Safety Ratings

```
safety_ratings:
  - "list-playback-configurations: GREEN — read-only enumeration"
  - "put-playback-configuration: YELLOW — creates or updates configuration, affects future playback sessions"
  - "delete-playback-configuration: RED — immediately breaks active viewer sessions"
  - "CloudTrail lookup: GREEN — read-only audit log query"
```

## Escalation Conditions

- Configuration creation fails with internal service errors after fixing all parameters
- Configuration requires cross-account S3 access that needs bucket policy changes
- ADS URL requires VPC connectivity not available to MediaTailor
- Multiple configurations need simultaneous updates during a live event

## Data Sensitivity

- `aws mediatailor list-playback-configurations`: LOW — returns configuration metadata
- `aws mediatailor get-playback-configuration`: MEDIUM — may reveal ADS URLs with API keys in query parameters
- `aws cloudtrail lookup-events`: MEDIUM — reveals IAM principal identities and actions
- ADS URLs: HIGH — may contain authentication tokens or API keys as query parameters

## Prohibited Actions

- NEVER delete a playback configuration during active viewer sessions
- NEVER expose ADS URLs containing API keys or authentication tokens in logs or responses
- NEVER set log percentage to 100% on high-traffic production configurations without cost analysis
- NEVER modify a live event configuration without a rollback plan

## Phase 3 — Rollback

1. If a configuration was created with wrong parameters, update it: `aws mediatailor put-playback-configuration --Name <name> --AdDecisionServerUrl <correct-url> --VideoContentSourceUrl <correct-url>`
2. If a configuration must be deleted, first verify no active sessions: check CloudWatch metrics for active session count
3. If CDN prefixes were misconfigured, update the configuration and allow 2-5 minutes for propagation
4. Document the configuration change and notify the streaming operations team

## Common Issues

- symptoms: "PutPlaybackConfiguration returns ValidationException"
  diagnosis: "Required parameters missing or invalid URL format."
  resolution: "Verify VideoContentSourceUrl and AdDecisionServerUrl are valid HTTPS URLs. Check all required parameters."

- symptoms: "Configuration created but playback returns 404"
  diagnosis: "VideoContentSourceUrl is unreachable or returns invalid manifest."
  resolution: "Test origin URL directly with curl. Verify manifest format (HLS/DASH) and accessibility."

## Output Format

```yaml
root_cause: "configuration_creation — <specific_cause>"
evidence:
  - type: creation_error
    content: "<error details>"
severity: HIGH
mitigation:
  immediate: "Fix configuration parameters"
  long_term: "Use IaC templates for consistent configuration"
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
