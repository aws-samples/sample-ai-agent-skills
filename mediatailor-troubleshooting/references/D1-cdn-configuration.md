---
title: "D1 — CDN Configuration"
description: "Diagnose CDN integration and prefix configuration issues"
status: active
severity: HIGH
triggers: ["CDN error", "403 on segments", "404 on ads", "CloudFront integration", "segment URL wrong"]
owner: devops-agent
objective: "Resolve CDN configuration issues affecting content and ad segment delivery"
context: "MediaTailor rewrites manifest segment URLs using configured CDN prefixes. Incorrect prefixes cause segment fetch failures at the player. Both content and ad segments need proper CDN routing."
---

## Phase 1 — Triage

MUST:
- Get CDN configuration: `aws mediatailor get-playback-configuration --Name <config-name> --query '{AdSegmentUrlPrefix:CdnConfiguration.AdSegmentUrlPrefix,ContentSegmentUrlPrefix:CdnConfiguration.ContentSegmentUrlPrefix}'`
- Verify CDN distribution is active: `aws cloudfront get-distribution --id <dist-id> --query 'Distribution.Status'`
- Test a content segment URL through CDN: `curl -I "https://<cdn-domain>/<segment-path>"`

SHOULD:
- Compare CDN prefix with actual CloudFront distribution domain
- Check CloudFront origin configuration points to correct origin
- Verify CloudFront behaviors and cache policies

MAY:
- Check CloudFront access logs for 4xx/5xx errors
- Review CloudFront origin access identity or origin access control settings

## Phase 2 — Remediate

MUST:
- Fix AdSegmentUrlPrefix to match ad CDN distribution domain
- Fix ContentSegmentUrlPrefix to match content CDN distribution domain
- Ensure CloudFront origins are correctly configured for both content and ad servers

SHOULD:
- Configure CloudFront cache behaviors for manifest vs segment paths
- Set appropriate TTLs for manifests (low) and segments (high)

MAY:
- Configure CloudFront signed URLs/cookies for content protection
- Set up CloudFront real-time logs for debugging

## Safety Ratings

```
safety_ratings:
  - "get-playback-configuration: GREEN — read-only configuration retrieval"
  - "cloudfront get-distribution: GREEN — read-only distribution details"
  - "put-playback-configuration CDN update: YELLOW — changes segment URLs in manifests"
  - "cloudfront update-distribution: YELLOW — changes CDN behavior affecting all viewers"
```

## Escalation Conditions

- CDN misconfiguration causes widespread playback failures across all viewers
- CloudFront distribution changes require approval from CDN operations team
- Ad CDN and content CDN require different authentication mechanisms
- CDN prefix change needed during live event with active viewers

## Data Sensitivity

- CDN configuration: LOW — domain names and path prefixes
- CloudFront distribution config: MEDIUM — reveals origin servers and security settings
- CloudFront access logs: HIGH — contain viewer IP addresses and request details
- Signed URL keys: CRITICAL — CloudFront private keys must never be exposed

## Prohibited Actions

- NEVER expose CloudFront private keys used for signed URLs
- NEVER disable CloudFront distribution during active viewer sessions
- NEVER change CDN prefixes during a live event without a tested rollback plan
- NEVER configure CDN to bypass origin authentication for protected content

## Phase 3 — Rollback

1. If CDN prefix change broke playback, revert prefixes: `aws mediatailor put-playback-configuration` with original CDN configuration
2. If CloudFront distribution was modified, revert distribution config and wait for deployment (5-15 minutes)
3. If CloudFront origin was changed, revert origin settings and invalidate cache: `aws cloudfront create-invalidation --distribution-id <id> --paths "/*"`
4. Monitor segment fetch success rate for 10 minutes after rollback

## Common Issues

- symptoms: "Player receives 403 on content segments but manifest loads fine"
  diagnosis: "ContentSegmentUrlPrefix does not match CloudFront distribution domain or path."
  resolution: "Update ContentSegmentUrlPrefix to match CloudFront domain. Verify CloudFront origin allows segment access."

- symptoms: "Ads play but content segments fail"
  diagnosis: "AdSegmentUrlPrefix is correct but ContentSegmentUrlPrefix is wrong or content CDN is misconfigured."
  resolution: "Fix ContentSegmentUrlPrefix. Verify content origin is accessible through CloudFront."

## Output Format

```yaml
root_cause: "cdn_configuration — <specific_cause>"
evidence:
  - type: cdn_detail
    content: "<CDN prefix configuration and test results>"
severity: HIGH
mitigation:
  immediate: "Fix CDN prefix configuration"
  long_term: "Implement CDN health monitoring and automated prefix validation"
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
