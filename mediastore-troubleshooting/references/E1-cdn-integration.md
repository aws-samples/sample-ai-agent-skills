---
title: "E1 — CDN Integration"
description: "Diagnose MediaStore integration with CloudFront CDN"
status: active
severity: HIGH
triggers: ["cloudfront mediastore error", "cdn not working", "origin error", "cloudfront 502", "cdn integration"]
owner: devops-agent
objective: "Resolve CloudFront CDN integration issues with MediaStore origins"
context: "CloudFront uses the MediaStore container endpoint as a custom origin. OAI/OAC are not used — container policies control access."
---

## Phase 1 — Triage

MUST:
- Check CloudFront distribution origin: `aws cloudfront get-distribution --id <dist-id> --query 'Distribution.DistributionConfig.Origins.Items[*].{Id:Id,DomainName:DomainName,Protocol:CustomOriginConfig.OriginProtocolPolicy}'`
- Verify MediaStore container endpoint: `aws mediastore describe-container --container-name <name> --query 'Container.Endpoint'`
- Check CloudFront error rates: `aws cloudwatch get-metric-statistics --namespace AWS/CloudFront --metric-name 5xxErrorRate --dimensions Name=DistributionId,Value=<dist-id> --start-time <start> --end-time <end> --period 300 --statistics Average`

SHOULD:
- Verify the origin domain matches the MediaStore endpoint (without https://)
- Check container policy allows access from CloudFront
- Verify origin protocol policy is set to HTTPS-only

MAY:
- Test direct access to MediaStore endpoint to isolate CDN vs origin issues
- Check CloudFront access logs for specific error details

## Phase 2 — Remediate

MUST:
- Configure origin with MediaStore endpoint domain (e.g., `<hash>.data.mediastore.<region>.amazonaws.com`)
- Set origin protocol policy to HTTPS-only
- Update container policy to allow read access

SHOULD:
- Configure cache behaviors for media content (TTL, query string forwarding)
- Set up origin request policy to forward required headers

MAY:
- Configure CloudFront functions or Lambda@Edge for URL rewriting

## Safety Ratings

```
safety_ratings:
  - "cloudfront get-distribution: GREEN — read-only distribution query"
  - "mediastore describe-container: GREEN — read-only container query"
  - "cloudfront update-distribution: YELLOW — changes CDN configuration"
  - "cloudwatch get-metric-statistics: GREEN — read-only metric query"
```

## Escalation Conditions

- CloudFront returns 502/504 errors from MediaStore origin consistently
- Cache invalidation is needed for live streaming content
- CloudFront distribution update is stuck in InProgress state
- Need to restrict MediaStore access to CloudFront only (no direct access)

## Data Sensitivity

- CloudFront distribution configuration: MEDIUM — reveals CDN architecture
- MediaStore container endpoint: LOW — required for origin configuration
- CloudFront access logs: HIGH — contain viewer IPs and request details
- Error rate metrics: LOW — numeric performance data

## Prohibited Actions

- NEVER use HTTP (non-SSL) origin protocol for MediaStore — always use HTTPS
- NEVER configure OAI or OAC for MediaStore origins — they are S3-specific
- NEVER set cache TTL too high for live streaming content — it causes stale segments
- NEVER delete the CloudFront distribution without removing DNS records first

## Phase 3 — Rollback

1. Revert CloudFront distribution to previous configuration: `aws cloudfront update-distribution --id <dist-id> --distribution-config file://previous-config.json --if-match <etag>`
2. If container policy was changed, restore previous policy
3. Wait for distribution deployment to complete (may take 15-30 minutes)
4. Verify content delivery is restored through CloudFront
5. Invalidate CloudFront cache if stale content was served: `aws cloudfront create-invalidation --distribution-id <dist-id> --paths "/*"`

## Common Issues

- symptoms: "CloudFront returns 502 Bad Gateway for MediaStore content"
  diagnosis: "Origin domain is incorrect or origin protocol policy is not HTTPS-only."
  resolution: "Set origin domain to the MediaStore endpoint (without https://). Set protocol to HTTPS-only."

- symptoms: "CloudFront returns 403 for MediaStore objects"
  diagnosis: "Container policy does not allow access from CloudFront."
  resolution: "Update container policy to allow mediastore:GetObject for the CloudFront service or all principals."

## Output Format

```yaml
root_cause: "cdn_integration — <specific_cause>"
evidence:
  - type: cdn_error
    content: "<error details>"
severity: HIGH
mitigation:
  immediate: "Fix origin configuration or container policy"
  long_term: "Automate CDN configuration in IaC"
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
