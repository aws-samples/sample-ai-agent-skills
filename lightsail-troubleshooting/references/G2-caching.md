---
title: "G2 — CDN Caching Issues"
description: "Diagnose CDN caching behavior and cache invalidation problems"
status: active
severity: MEDIUM
triggers:
  - "caching not working"
  - "stale content"
  - "cache invalidation"
  - "content not updating"
owner: devops-agent
objective: "Identify and resolve CDN caching behavior issues"
context: "Lightsail distributions cache content at edge locations. Cache behavior can be configured per path pattern. Default TTL, cache headers, and forwarded headers affect caching. Cache invalidation (reset) clears cached content. Stale content is the most common complaint."
---

## Phase 1 — Triage

MUST:
- Check distribution cache behavior: `aws lightsail get-distributions --query 'distributions[*].{Name:name,DefaultCache:defaultCacheBehavior,CacheSettings:cacheBehaviorSettings}'`
- Check if content is being cached (check response headers: X-Cache, Age)
- Verify cache behavior settings match content type
- Check if origin is sending appropriate cache headers

SHOULD:
- Test with cache-busting query strings
- Check forwarded headers configuration
- Review origin Cache-Control and Expires headers

MAY:
- Check cache hit ratio metrics
- Review path-specific cache behaviors

## Phase 2 — Remediate

MUST:
- Reset cache (invalidation): `aws lightsail reset-distribution-cache --distribution-name <name>`
- Configure appropriate cache behavior for content type
- Ensure origin sends correct Cache-Control headers

SHOULD:
- Update cache settings: `aws lightsail update-distribution --distribution-name <name> --default-cache-behavior behavior=cache --cache-behavior-settings <settings>`
- Use versioned URLs for static assets to avoid stale content

MAY:
- Configure different cache behaviors for different path patterns
- Implement cache-busting strategies in the application

## Common Issues

- symptoms: "Updated content not showing through CDN"
  diagnosis: "Old content cached at edge locations."
  resolution: "Reset distribution cache. Use versioned URLs for assets."

- symptoms: "Dynamic content being cached"
  diagnosis: "Cache behavior set to cache all content."
  resolution: "Configure dont-cache behavior for dynamic paths. Forward appropriate headers."

- symptoms: "Low cache hit ratio"
  diagnosis: "Too many unique URLs or short TTL."
  resolution: "Optimize cache key. Increase TTL. Reduce query string variations."

## Output Format

```yaml
root_cause: "caching — <specific_cause>"
evidence:
  - type: cache_behavior
    content: "<cache behavior configuration>"
  - type: cache_headers
    content: "<origin cache headers>"
  - type: cache_metrics
    content: "<hit/miss ratio>"
severity: MEDIUM
mitigation:
  immediate: "Reset cache and fix cache behavior configuration"
  long_term: "Implement proper caching strategy with versioned assets"
```


## Safety Ratings

| Aspect | Rating | Notes |
|--------|--------|-------|
| Data Risk | Medium | Operations may affect service configurations and data access patterns |
| Service Impact | High | Incorrect changes can disrupt active workloads and dependent services |
| Recovery Complexity | Medium | Most actions are reversible but may require coordinated rollback procedures |
| Blast Radius | Medium | Impact typically scoped to the affected resource and its direct dependents |

## Escalation Conditions

- **Escalate immediately** if the issue involves potential data loss or corruption
- **Escalate immediately** if a production workload is impacted and cannot be restored within the agreed SLA
- **Escalate immediately** if the root cause is unclear after initial diagnostic steps are exhausted
- **Escalate to service team** if the issue appears to be a platform-level bug or undocumented behavior
- **Escalate to security team** if unauthorized access, policy violations, or credential exposure is suspected

## Data Sensitivity

- Verify data classification before performing any read or export operations
- Ensure all troubleshooting outputs (logs, configurations, snapshots) are handled according to the customer's data handling policy
- Mask or redact personally identifiable information (PII) in all shared diagnostics
- Do not copy production data to non-production environments without explicit authorization
- Retain troubleshooting artifacts only for the duration required and dispose of them securely

## Prohibited Actions

- **NEVER** delete a static IP address attached to a production instance without reassignment planning
- **NEVER** modify firewall rules to allow unrestricted inbound access (0.0.0.0/0) on sensitive ports
- **NEVER** delete snapshots that serve as the only backup for production instances or databases
- **NEVER** force-stop a managed database instance without verifying active connections and transactions
- **NEVER** detach a disk from a running instance without first unmounting it from the operating system

## Phase 3 � Rollback

1. **Identify rollback trigger**: Determine if the applied change has caused degradation, unexpected behavior, or failed validation checks
2. **Preserve current state**: Capture logs, configuration snapshots, and resource states before initiating rollback
3. **Execute rollback steps**: Revert configuration changes in reverse order of application, verifying each step
4. **Validate service recovery**: Confirm that the service has returned to its pre-change operational state using health checks and functional tests
5. **Document rollback**: Record the rollback trigger, steps taken, time to recovery, and any residual issues for post-incident review
## Safety Ratings

safety_ratings:
  - "Phase 1 triage commands (describe/get/list): GREEN — read-only"
  - "Phase 2 configuration changes: YELLOW — state-changing but recoverable"
  - "Phase 2 resource deletion or security changes: RED — destructive or irreversible"

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
