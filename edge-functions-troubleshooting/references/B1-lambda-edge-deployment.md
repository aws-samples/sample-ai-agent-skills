---
title: "B1 — Lambda@Edge Deployment"
description: "Diagnose Lambda@Edge deployment failures"
status: active
severity: HIGH
triggers: ["Lambda@Edge deploy failed", "cannot associate Lambda", "InvalidLambdaFunctionAssociation", "deployment error"]
owner: devops-agent
objective: "Resolve Lambda@Edge deployment and association issues"
context: "Lambda@Edge functions must be in us-east-1 and published as a numbered version. $LATEST cannot be used with CloudFront."
---

## Phase 1 — Triage

MUST:
- Verify function is in us-east-1: `aws lambda get-function --function-name <function-name> --region us-east-1`
- Check function has a published version (not $LATEST): `aws lambda list-versions-by-function --function-name <function-name> --region us-east-1`
- Verify execution role has the correct trust policy for edgelambda.amazonaws.com

SHOULD:
- Check function package size (1MB for viewer, 50MB for origin events)
- Verify function runtime is supported (Node.js or Python)
- Check for Lambda@Edge restrictions: no env vars, no layers, no VPC

MAY:
- Review CloudTrail for UpdateDistribution errors
- Check if the function is already associated with another distribution at the same event type

## Phase 2 — Remediate

MUST:
- Deploy function in us-east-1 if in wrong region
- Publish a numbered version: `aws lambda publish-version --function-name <function-name> --region us-east-1`
- Add edgelambda.amazonaws.com to the execution role trust policy

SHOULD:
- Remove unsupported features (env vars, layers, VPC, DLQ)
- Reduce package size if exceeding limits
- Test function locally before deploying

MAY:
- Use SAM or CDK for Lambda@Edge deployment automation
- Implement blue-green deployment with version aliases

## Common Issues

- symptoms: "InvalidLambdaFunctionAssociation — function must be in us-east-1"
  diagnosis: "Lambda function was created in a region other than us-east-1."
  resolution: "Recreate the function in us-east-1. Lambda@Edge only works from this region."

- symptoms: "InvalidLambdaFunctionAssociation — must use a function version, not $LATEST"
  diagnosis: "Function ARN references $LATEST instead of a numbered version."
  resolution: "Publish a version: aws lambda publish-version --function-name <name> --region us-east-1. Use the versioned ARN."

## Safety Ratings

```
safety_ratings:
  - "lambda get-function: GREEN — read-only"
  - "lambda publish-version: YELLOW — creates immutable version"
  - "cloudfront update-distribution: RED — modifies CDN behavior for all users"
  - "iam update-assume-role-policy: RED — modifies trust relationships"
```

## Escalation Conditions

- Function deployment causes 5xx errors on the distribution
- Replication to edge locations fails or is incomplete after 30 minutes
- Function requires features not supported by Lambda@Edge
- Cross-account Lambda@Edge deployment requires organizational changes

## Data Sensitivity

- `aws lambda get-function`: MEDIUM — reveals function configuration
- `aws lambda list-versions-by-function`: LOW — version metadata only
- `aws iam get-role`: MEDIUM — reveals trust policy and permissions
- `aws cloudfront update-distribution`: HIGH — modifies production CDN

## Prohibited Actions

- NEVER deploy Lambda@Edge functions outside us-east-1
- NEVER use $LATEST version with CloudFront — always publish a numbered version
- NEVER modify the execution role trust policy to include untrusted principals
- NEVER deploy untested Lambda@Edge functions to production distributions

## Phase 3 — Rollback

1. Update distribution to use previous Lambda@Edge version: `aws cloudfront update-distribution --id <dist-id> --distribution-config file://previous-config.json --if-match <etag>`
2. Wait for distribution to reach 'Deployed' status (5-15 minutes)
3. If urgent, remove the Lambda@Edge association entirely from the cache behavior
4. Verify production traffic is unaffected after rollback
5. Document the deployment failure and add to deployment checklist

## Output Format

```yaml
root_cause: "lambda_edge_deployment — <specific_cause>"
evidence:
  - type: deployment_error
    content: "<error details>"
severity: HIGH
mitigation:
  immediate: "Fix deployment configuration"
  long_term: "Implement automated deployment pipeline with validation"
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
