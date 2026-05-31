---
title: "G1 — CDN Caching Issues"
description: "Diagnose CDN caching and stale content problems"
status: active
severity: MEDIUM
triggers:
  - "stale content"
  - "cache not clearing"
  - "old version showing"
  - "CDN cache"
  - "content not updating"
owner: devops-agent
objective: "Identify and resolve CDN caching issues"
context: "CDN caching issues include: stale content after deployment, custom cache headers misconfigured, browser cache vs CDN cache confusion, or CloudFront invalidation not completing."
---

## Phase 1 — Triage

MUST:
- Verify deployment completed successfully: `aws amplify list-jobs --app-id <app-id> --branch-name <branch> --max-results 3`
- Check custom headers configuration in amplify.yml
- Test with cache-busting query parameter or incognito mode
- Check response headers for cache-control values

SHOULD:
- Verify CloudFront invalidation completed after deploy
- Check if custom cache headers are overriding Amplify defaults
- Differentiate between browser cache and CDN cache

MAY:
- Check CloudFront distribution cache behavior
- Review service worker caching if PWA

## Phase 2 — Remediate

MUST:
- If browser cache: clear browser cache or use hard refresh
- If CDN cache: trigger redeployment to force invalidation
- Fix custom cache headers if misconfigured

SHOULD:
- Set appropriate cache headers in amplify.yml customHeaders section
- Trigger manual redeploy: `aws amplify start-job --app-id <app-id> --branch-name <branch> --job-type RELEASE`

MAY:
- Implement cache-busting strategies (hashed filenames)
- Configure service worker update strategy

## Safety Ratings

```
safety_ratings:
  - "list-jobs: GREEN — read-only deployment check"
  - "check response headers: GREEN — read-only cache analysis"
  - "start-job RELEASE: YELLOW — triggers redeployment and CDN invalidation"
  - "modify custom cache headers: YELLOW — changes caching behavior"
```

## Escalation Conditions

- Stale content serving incorrect data to production users
- CDN cache invalidation not completing after deployment
- Service worker caching preventing updates for PWA users
- Custom cache headers causing security-sensitive content to be cached

## Data Sensitivity

- Cache headers: LOW — HTTP header values
- Deployment status: LOW — operational metadata
- CDN cache behavior: LOW — caching configuration

## Prohibited Actions

- NEVER suggest deleting a production branch
- NEVER suggest disabling HTTPS
- NEVER set cache-control headers to cache sensitive or authenticated content
- NEVER disable CDN caching entirely as a permanent fix

## Phase 3 — Rollback

1. If cache headers were changed incorrectly, restore previous customHeaders in amplify.yml
2. Trigger a redeployment to apply restored headers: `aws amplify start-job --app-id <id> --branch-name <branch> --job-type RELEASE`
3. If service worker was modified, deploy a version that forces cache update
4. Verify content is fresh by checking response headers in incognito mode
5. Monitor for stale content reports after rollback

## Output Format

```yaml
root_cause: "cdn_caching_issue — <specific_cause>"
evidence:
  - type: cache_headers
    content: "<response cache-control headers>"
  - type: deploy_status
    content: "<latest deployment status>"
severity: MEDIUM
mitigation:
  immediate: "Redeploy or fix cache headers"
  long_term: "Implement proper caching strategy with cache-busting"
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
