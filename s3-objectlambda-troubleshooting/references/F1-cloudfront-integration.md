---
title: "F1 — CloudFront Integration"
description: "Diagnose CloudFront integration issues with Object Lambda access points"
status: active
severity: HIGH
triggers: ["CloudFront OLAP error", "OAC not working", "CloudFront 403", "origin access control Object Lambda"]
owner: devops-agent
objective: "Resolve CloudFront integration issues with S3 Object Lambda"
context: "CloudFront can use OLAPs as origins but requires OAC (not OAI). The OLAP policy must allow the CloudFront distribution."
---

## Phase 1 — Triage

MUST:
- Check CloudFront distribution origin configuration: `aws cloudfront get-distribution --id <distribution-id> --query 'Distribution.DistributionConfig.Origins'`
- Verify OAC is configured (not OAI): `aws cloudfront get-origin-access-control --id <oac-id>`
- Check OLAP policy allows CloudFront: `aws s3control get-access-point-policy-for-object-lambda --account-id <account-id> --name <olap-name>`

SHOULD:
- Verify the origin domain name uses the OLAP regional endpoint format
- Check CloudFront error rates: `aws cloudwatch get-metric-statistics --namespace AWS/CloudFront --metric-name 5xxErrorRate --dimensions Name=DistributionId,Value=<dist-id> --start-time <start> --end-time <end> --period 300 --statistics Average`
- Verify cache behavior is configured correctly for the OLAP origin

MAY:
- Test direct OLAP access (bypassing CloudFront) to isolate the issue
- Review CloudFront access logs for error patterns

## Phase 2 — Remediate

MUST:
- Create OAC if using OAI: `aws cloudfront create-origin-access-control --origin-access-control-config CallingSigningBehavior=always,ContentType=s3,Description="OLAP OAC",Name=<name>,OriginAccessControlOriginType=s3,SigningProtocol=sigv4`
- Update OLAP policy to allow CloudFront OAC principal
- Set origin domain to OLAP regional endpoint: `<olap-name>-<account-id>.s3-object-lambda.<region>.amazonaws.com`

SHOULD:
- Configure appropriate cache TTLs for transformed content
- Set up custom error pages for OLAP errors
- Enable CloudFront logging for debugging

MAY:
- Implement origin failover to a backup origin
- Use CloudFront Functions for request/response header manipulation

## Common Issues

- symptoms: "CloudFront returns 403 AccessDenied for OLAP origin"
  diagnosis: "Using OAI instead of OAC, or OLAP policy does not allow CloudFront."
  resolution: "Create an OAC and update the OLAP policy with: Condition: StringEquals: s3:authType: REST-HEADER, aws:SourceArn: <distribution-arn>."

- symptoms: "CloudFront returns 502 Bad Gateway"
  diagnosis: "Origin domain name is incorrect or OLAP is not accessible from CloudFront."
  resolution: "Verify origin domain uses the correct OLAP regional endpoint format. Check OLAP exists and is functional."

## Safety Ratings

```
safety_ratings:
  - "cloudfront get-distribution: GREEN — read-only"
  - "cloudfront get-origin-access-control: GREEN — read-only"
  - "cloudfront create-origin-access-control: YELLOW — creates new OAC resource"
  - "cloudfront update-distribution: RED — modifies CDN behavior, affects all users"
```

## Escalation Conditions

- CloudFront distribution update fails or causes widespread 5xx errors
- OAC configuration requires changes to multiple distributions
- OLAP policy changes affect other CloudFront distributions sharing the same OLAP
- Cache invalidation is needed across a large distribution with many edge locations

## Data Sensitivity

- `aws cloudfront get-distribution`: MEDIUM — reveals origin configuration and domain names
- `aws s3control get-access-point-policy-for-object-lambda`: MEDIUM — reveals access control
- `aws cloudfront get-origin-access-control`: LOW — OAC metadata only
- CloudFront access logs: HIGH — contain client IPs, request URIs, and user agents

## Prohibited Actions

- NEVER use OAI for Object Lambda origins — it is not supported and will fail
- NEVER modify a production CloudFront distribution without a change management process
- NEVER disable OAC signing — it removes authentication between CloudFront and the OLAP
- NEVER set cache TTL to 0 for all requests — this defeats the purpose of CloudFront caching

## Phase 3 — Rollback

1. If CloudFront distribution was updated, revert to previous configuration: `aws cloudfront update-distribution --id <dist-id> --distribution-config file://previous-config.json --if-match <etag>`
2. If OAC was created incorrectly, delete it: `aws cloudfront delete-origin-access-control --id <oac-id> --if-match <etag>`
3. If OLAP policy was modified, restore previous policy
4. Invalidate CloudFront cache if incorrect content was cached: `aws cloudfront create-invalidation --distribution-id <dist-id> --paths "/*"`
5. Monitor error rates after rollback to confirm resolution

## Output Format

```yaml
root_cause: "cloudfront_integration — <specific_cause>"
evidence:
  - type: cloudfront_error
    content: "<error details>"
severity: HIGH
mitigation:
  immediate: "Fix OAC and OLAP policy"
  long_term: "Implement IaC for CloudFront-OLAP integration with validation"
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
  - command: "get-bucket-policy"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"
  - command: "get-bucket-acl"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"
  - command: "get-public-access-block"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"

## Prohibited Actions

prohibited_actions:
  - "NEVER suggest disabling S3 Block Public Access"
  - "NEVER suggest Principal: * without Condition keys"
  - "NEVER suggest removing bucket encryption"
