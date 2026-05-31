---
title: "G1 — Cache Behavior Association"
description: "Diagnose function-to-cache-behavior association issues"
status: active
severity: MEDIUM
triggers: ["function not running", "wrong cache behavior", "function not associated", "path pattern mismatch"]
owner: devops-agent
objective: "Resolve function association with CloudFront cache behaviors"
context: "Functions are associated per cache behavior, not per distribution. Each cache behavior can have one CF Function and one Lambda@Edge per event type."
---

## Phase 1 — Triage

MUST:
- Get distribution config with all cache behaviors: `aws cloudfront get-distribution-config --id <distribution-id>`
- Verify the function is associated with the correct cache behavior (path pattern)
- Check if the request URL matches the intended cache behavior's path pattern

SHOULD:
- List all cache behaviors and their function associations
- Verify path pattern ordering (CloudFront evaluates in order, first match wins)
- Check if the default cache behavior has the function (catches unmatched paths)

MAY:
- Test with specific URLs to verify which cache behavior handles them
- Review CloudFront access logs to see which cache behavior processed the request

## Phase 2 — Remediate

MUST:
- Associate the function with the correct cache behavior
- Fix path pattern to match the intended URLs
- Ensure path pattern ordering is correct (more specific patterns first)

SHOULD:
- Verify both CF Function and Lambda@Edge associations per event type
- Test the association with representative URLs
- Document the cache behavior to function mapping

MAY:
- Consolidate cache behaviors if too many exist
- Use CloudFront continuous deployment to test association changes

## Common Issues

- symptoms: "Function runs for some URLs but not others"
  diagnosis: "Function is associated with a specific cache behavior, not the default. Some URLs match a different behavior."
  resolution: "Check path patterns. Associate the function with all relevant cache behaviors or the default behavior."

- symptoms: "Both CF Function and Lambda@Edge run on the same event"
  diagnosis: "A cache behavior can have both a CF Function and Lambda@Edge for the same event type."
  resolution: "This is expected behavior. CF Function runs first, then Lambda@Edge. Remove one if not intended."

## Safety Ratings

```
safety_ratings:
  - "cloudfront get-distribution-config: GREEN — read-only"
  - "cloudfront update-distribution: RED — modifies cache behavior and function associations"
  - "cloudfront test-function: GREEN — no production impact"
  - "cloudfront create-invalidation: YELLOW — clears cached content"
```

## Escalation Conditions

- Cache behavior changes affect other functions or origins
- Path pattern conflicts cannot be resolved without restructuring
- Association changes require distribution update during peak traffic
- Multiple teams manage different cache behaviors on the same distribution

## Data Sensitivity

- `aws cloudfront get-distribution-config`: MEDIUM — reveals all origins, behaviors, and functions
- Cache behavior configuration: MEDIUM — reveals URL routing logic
- Path patterns: LOW — URL patterns only
- Function associations: LOW — function ARNs and names

## Prohibited Actions

- NEVER reorder cache behaviors without understanding the impact on all URL patterns
- NEVER remove function associations from cache behaviors without verifying no traffic depends on them
- NEVER add functions to the default cache behavior without testing all URL patterns
- NEVER modify cache behaviors during peak traffic without a rollback plan

## Phase 3 — Rollback

1. Restore previous distribution configuration: `aws cloudfront update-distribution --id <dist-id> --distribution-config file://previous-config.json --if-match <etag>`
2. Wait for distribution to reach 'Deployed' status
3. Verify function associations are correct after rollback
4. Test representative URLs to confirm correct behavior
5. Document the cache behavior mapping

## Output Format

```yaml
root_cause: "cache_behavior_association — <specific_cause>"
evidence:
  - type: association_error
    content: "<cache behavior and path pattern details>"
severity: MEDIUM
mitigation:
  immediate: "Fix function association"
  long_term: "Document and automate cache behavior management"
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
