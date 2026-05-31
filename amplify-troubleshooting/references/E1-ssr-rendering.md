---
title: "E1 — SSR Rendering Issues"
description: "Diagnose server-side rendering errors and failures"
status: active
severity: HIGH
triggers:
  - "SSR not working"
  - "server-side rendering error"
  - "500 error on page"
  - "SSR timeout"
  - "page not rendering"
owner: devops-agent
objective: "Identify and resolve SSR rendering issues"
context: "SSR issues include: framework not supported, build output missing server artifacts, Lambda@Edge errors, runtime errors in server code, environment variables not available, or compute timeout."
---

## Phase 1 — Triage

MUST:
- Check app platform: `aws amplify get-app --app-id <app-id> --query 'app.platform'` (must be WEB_COMPUTE)
- Check recent deployment status: `aws amplify list-jobs --app-id <app-id> --branch-name <branch> --max-results 5`
- Check CloudWatch logs for SSR errors: `aws logs filter-log-events --log-group-name /aws/amplify/<app-id>/<branch> --start-time <epoch>`
- Verify framework is supported for SSR on Amplify

SHOULD:
- Check if build output includes server-side artifacts
- Verify environment variables are set for server-side code
- Check compute configuration (memory, timeout)
- Review framework-specific SSR requirements

MAY:
- Test SSR locally to isolate Amplify-specific issues
- Check Lambda@Edge function logs in us-east-1

## Phase 2 — Remediate

MUST:
- Ensure platform is WEB_COMPUTE (not WEB)
- Fix server-side code errors identified in logs
- Verify build output includes server artifacts

SHOULD:
- Update platform if needed: `aws amplify update-app --app-id <app-id> --platform WEB_COMPUTE`
- Set server-side environment variables
- Increase compute timeout for slow SSR pages

MAY:
- Optimize SSR performance with caching strategies
- Implement ISR (Incremental Static Regeneration) where supported

## Safety Ratings

```
safety_ratings:
  - "get-app (platform): GREEN — read-only platform check"
  - "list-jobs: GREEN — read-only job status check"
  - "filter-log-events: GREEN — read-only log review"
  - "update-app (platform WEB_COMPUTE): YELLOW — changes app platform type"
  - "increase compute timeout: YELLOW — changes compute configuration"
```

## Escalation Conditions

- SSR errors causing 500 errors on production pages
- Framework not supported for SSR on Amplify
- Lambda@Edge errors requiring investigation in us-east-1
- SSR timeout affecting critical user-facing pages

## Data Sensitivity

- SSR error logs: MEDIUM — may contain application data and stack traces
- Platform configuration: LOW — app platform metadata
- Lambda@Edge logs: MEDIUM — may contain request/response details

## Prohibited Actions

- NEVER suggest deleting a production branch
- NEVER suggest disabling HTTPS
- NEVER change platform from WEB_COMPUTE to WEB on a production app (breaks SSR)
- NEVER ignore 500 errors — they indicate server-side failures visible to users

## Phase 3 — Rollback

1. If platform was changed incorrectly, restore it: `aws amplify update-app --app-id <id> --platform WEB_COMPUTE`
2. If server-side code changes caused errors, revert the code and redeploy
3. If compute configuration was changed, restore previous settings
4. Trigger a redeployment of the last known working version
5. Verify SSR pages render correctly after rollback

## Output Format

```yaml
root_cause: "ssr_rendering_issue — <specific_cause>"
evidence:
  - type: platform
    content: "<app platform>"
  - type: ssr_logs
    content: "<server-side error logs>"
severity: HIGH
mitigation:
  immediate: "Fix SSR code errors or platform configuration"
  long_term: "Implement SSR monitoring and performance optimization"
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
