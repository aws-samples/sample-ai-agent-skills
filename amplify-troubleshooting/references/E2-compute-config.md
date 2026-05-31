---
title: "E2 — Compute Configuration Issues"
description: "Diagnose SSR compute configuration and performance problems"
status: active
severity: MEDIUM
triggers:
  - "compute timeout"
  - "memory error"
  - "cold start slow"
  - "compute configuration"
  - "Lambda@Edge error"
owner: devops-agent
objective: "Identify and resolve compute configuration issues for SSR"
context: "Compute configuration issues affect SSR performance: memory too low causing OOM, timeout too short for complex pages, cold starts affecting response time, or compute settings not optimized for the workload."
---

## Phase 1 — Triage

MUST:
- Check app platform and framework: `aws amplify get-app --app-id <app-id> --query 'app.{Platform:platform,Framework:productionBranch.framework}'`
- Check CloudWatch metrics for Lambda@Edge: memory usage, duration, errors
- Review SSR response times and error rates
- Check for timeout errors in CloudWatch logs

SHOULD:
- Verify compute memory allocation is sufficient
- Check cold start frequency and duration
- Review page complexity and data fetching patterns

MAY:
- Compare SSR performance across different pages
- Check if static generation could replace SSR for some pages

## Phase 2 — Remediate

MUST:
- Increase memory if OOM errors occur
- Increase timeout if pages are timing out
- Optimize server-side code for faster execution

SHOULD:
- Reduce bundle size to minimize cold starts
- Implement caching for frequently accessed data
- Use static generation for pages that don't need real-time data

MAY:
- Implement edge caching for SSR responses
- Use ISR for balance between static and dynamic

## Safety Ratings

```
safety_ratings:
  - "get-app (platform/framework): GREEN — read-only check"
  - "check CloudWatch Lambda metrics: GREEN — read-only metric check"
  - "increase compute memory: YELLOW — changes compute resource allocation"
  - "increase compute timeout: YELLOW — changes request timeout behavior"
```

## Escalation Conditions

- OOM errors on production SSR pages
- Cold start latency exceeding acceptable thresholds
- Compute timeout affecting critical user-facing pages
- Memory or timeout changes not resolving performance issues

## Data Sensitivity

- Compute metrics: LOW — memory usage, duration, error counts
- Timeout errors: LOW — operational metadata
- Page complexity analysis: MEDIUM — reveals application architecture

## Prohibited Actions

- NEVER suggest deleting a production branch
- NEVER suggest disabling HTTPS
- NEVER reduce compute memory below the minimum required for the framework
- NEVER set timeout to maximum as a permanent fix without optimizing the code

## Phase 3 — Rollback

1. If memory was changed and caused issues, restore previous memory allocation
2. If timeout was changed, restore previous timeout value
3. If code optimizations broke functionality, revert code changes and redeploy
4. Verify SSR response times are acceptable after rollback
5. Monitor CloudWatch metrics for the next 24 hours

## Output Format

```yaml
root_cause: "compute_config_issue — <specific_cause>"
evidence:
  - type: compute_metrics
    content: "<memory, duration, error metrics>"
  - type: timeout_errors
    content: "<timeout occurrences>"
severity: MEDIUM
mitigation:
  immediate: "Adjust compute memory and timeout settings"
  long_term: "Optimize SSR code and implement caching"
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
