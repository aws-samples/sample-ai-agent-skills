---
title: "F3 — Function Association Issues"
description: "Diagnose issues with associating Lambda@Edge or CloudFront Functions to behaviors"
status: active
severity: MEDIUM
triggers:
  - "function association"
  - "cannot associate"
  - "function not triggering"
  - "wrong function"
owner: devops-agent
objective: "Resolve function association configuration issues"
context: "Each cache behavior can have up to 4 function triggers: viewer request, viewer response, origin request, origin response. CloudFront Functions only support viewer events. Lambda@Edge supports all 4 events. You cannot associate both a CloudFront Function and Lambda@Edge to the same event on the same behavior."
---

## Phase 1 — Triage

MUST:
- Check behavior function associations: `aws cloudfront get-distribution-config --id <dist-id> --query 'DistributionConfig.DefaultCacheBehavior.{Lambda:LambdaFunctionAssociations,Functions:FunctionAssociations}'`
- Check all behaviors (not just default): `aws cloudfront get-distribution-config --id <dist-id> --query 'DistributionConfig.CacheBehaviors.Items[*].{Path:PathPattern,Lambda:LambdaFunctionAssociations,Functions:FunctionAssociations}'`
- Verify the function is associated to the correct event type
- Verify the request matches the correct cache behavior path pattern

SHOULD:
- Check if the function is published (LIVE stage for CF Functions, published version for Lambda@Edge)
- Verify no conflicts between CloudFront Functions and Lambda@Edge on the same event
- Check distribution deployment status (changes take time to propagate)

MAY:
- Check CloudTrail for distribution update events
- Verify function permissions (Lambda@Edge needs cloudfront.amazonaws.com invoke permission)

## Phase 2 — Remediate

MUST:
- Associate function to the correct event type and behavior
- Use published versions for Lambda@Edge (not $LATEST)
- Publish CloudFront Functions to LIVE stage before associating
- Wait for distribution deployment to complete

SHOULD:
- Use CloudFront Functions for viewer events when possible (faster, cheaper)
- Use Lambda@Edge only when network access or origin events are needed
- Document function associations per behavior

MAY:
- Automate function association updates in CI/CD
- Use infrastructure as code (CloudFormation/Terraform) for consistent configuration

## Common Issues

- symptoms: "Function not triggering on requests"
  diagnosis: "Request matches a different cache behavior that doesn't have the function."
  resolution: "Check which behavior the request matches. Associate function to correct behavior."

- symptoms: "Cannot associate CloudFront Function to origin request"
  diagnosis: "CloudFront Functions only support viewer request and viewer response events."
  resolution: "Use Lambda@Edge for origin request/response events."

- symptoms: "Error associating function — conflict"
  diagnosis: "Both CloudFront Function and Lambda@Edge associated to the same event type."
  resolution: "Remove one. Only one function type per event per behavior."


## Safety Ratings

```
safety_ratings:
  - "Inspect/Describe resources: GREEN - Read-only API calls, no state change"
  - "Deploy changes: YELLOW - Affects live traffic, use canary/staged deployment"
  - "Delete/Remove resources: RED - Potentially irreversible, requires confirmation"
  - "Modify IAM/permissions: RED - Security-sensitive, may break access patterns"
  - "Network configuration changes: YELLOW - May affect connectivity"
  - "Cache invalidation: YELLOW - Temporarily increases origin load"
```

## Escalation Conditions

- Distribution serves a production website or application
- Fix requires modifying origin configuration or cache behaviors
- Resolution involves certificate changes or HTTPS configuration
- Issue affects multiple distributions or is account-level
- Lambda@Edge or CloudFront Functions changes are required on production

## Data Sensitivity

MEDIUM - Signed URL private keys and key pairs control content access. Origin configurations may expose internal infrastructure (S3 bucket names, ALB endpoints). Access logs contain client IPs, request URIs, and query strings. Field-level encryption configurations protect sensitive form data.

## Prohibited Actions

- NEVER suggest deleting a CloudFront distribution that is serving live traffic
- NEVER suggest disabling HTTPS or downgrading the security policy on a production distribution
- NEVER recommend removing all cache behaviors - this breaks content routing
- NEVER suggest invalidating '/*' repeatedly as a fix - address the root caching issue instead
- NEVER recommend removing origin access control/identity from S3 origins without alternative access controls

## Phase 3 - Rollback

1. If distribution configuration was changed, update with previous settings: `aws cloudfront update-distribution --id <id> --distribution-config <previous> --if-match <etag>`
2. If cache policy or TTL was changed, restore original cache behavior settings and allow caches to repopulate
3. If edge function was changed, update the distribution to use the previous function version/ARN
4. If origin configuration was changed, restore original origin settings including timeouts and protocols
## Output Format

```yaml
root_cause: "function_association — <specific_cause>"
evidence:
  - type: behavior_associations
    content: "<function associations per behavior>"
  - type: function_status
    content: "<function stage/version>"
  - type: distribution_status
    content: "<deployment status>"
severity: MEDIUM
mitigation:
  immediate: "Fix function association configuration"
  long_term: "Use infrastructure as code for consistent function management"
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
  - "NEVER suggest disabling HTTPS requirements"
  - "NEVER suggest removing WAF association to fix access"
  - "NEVER suggest wildcard CORS origins in production"
