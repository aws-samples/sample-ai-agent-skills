---
title: "B2 — S3 Origin Access Issues"
description: "Diagnose access denied and configuration issues with S3 origins"
status: active
severity: HIGH
triggers:
  - "S3 origin"
  - "AccessDenied"
  - "403 from S3"
  - "OAC"
  - "OAI"
  - "origin access"
owner: devops-agent
objective: "Resolve S3 origin access issues including OAC/OAI configuration and bucket policy"
context: "CloudFront accesses S3 origins using Origin Access Control (OAC, recommended) or Origin Access Identity (OAI, legacy). The S3 bucket policy must grant access to the CloudFront distribution. Common issues include missing bucket policy, wrong OAC/OAI configuration, SSE-KMS without OAC, and using S3 website endpoint vs REST endpoint."
---

## Phase 1 — Triage

MUST:
- Check origin configuration: `aws cloudfront get-distribution-config --id <dist-id> --query 'DistributionConfig.Origins.Items[*].{Id:Id,Domain:DomainName,OAC:OriginAccessControlId,S3:S3OriginConfig}'`
- Check if OAC is configured: `aws cloudfront get-origin-access-control --id <oac-id>`
- Check S3 bucket policy: `aws s3api get-bucket-policy --bucket <bucket>`
- Check if bucket uses SSE-KMS: `aws s3api get-bucket-encryption --bucket <bucket>`
- Verify the origin domain format (REST API endpoint vs website endpoint)

SHOULD:
- Check S3 Block Public Access: `aws s3api get-public-access-block --bucket <bucket>`
- Verify KMS key policy allows CloudFront if using SSE-KMS
- Check if the bucket is in an opt-in region (OAI does not support opt-in regions)

MAY:
- List OAIs if using legacy configuration: `aws cloudfront list-cloud-front-origin-access-identities`
- Check CloudTrail for S3 access denied events

## Phase 2 — Remediate

MUST:
- For new distributions, use OAC (not OAI)
- Update S3 bucket policy to allow the CloudFront distribution:
  ```json
  {
    "Statement": [{
      "Effect": "Allow",
      "Principal": {"Service": "cloudfront.amazonaws.com"},
      "Action": "s3:GetObject",
      "Resource": "arn:aws:s3:::<bucket>/*",
      "Condition": {"StringEquals": {"AWS:SourceArn": "arn:aws:cloudfront::<account>:distribution/<dist-id>"}}
    }]
  }
  ```
- For SSE-KMS, use OAC and add CloudFront to the KMS key policy

SHOULD:
- Migrate from OAI to OAC for existing distributions
- Use S3 REST API endpoint (bucket.s3.region.amazonaws.com), not website endpoint, with OAC
- Restrict bucket policy to specific distribution ARN

MAY:
- Enable S3 server access logging to debug access patterns
- Use S3 Object Lambda with OAC for dynamic content transformation

## Common Issues

- symptoms: "403 AccessDenied from S3 origin"
  diagnosis: "S3 bucket policy does not grant access to CloudFront OAC/OAI."
  resolution: "Update bucket policy with CloudFront service principal and distribution condition."

- symptoms: "403 on SSE-KMS encrypted objects"
  diagnosis: "OAI does not support SSE-KMS. Must use OAC."
  resolution: "Migrate to OAC and update KMS key policy to allow cloudfront.amazonaws.com."

- symptoms: "S3 website features (redirects, index documents) not working"
  diagnosis: "Using S3 REST endpoint with OAC. Website features require the website endpoint."
  resolution: "Use S3 website endpoint as custom origin (no OAC) or use CloudFront Functions for redirects."


## Safety Ratings

```
safety_ratings:
  - "Inspect/Describe resources: GREEN - Read-only API calls, no state change"
  - "Modify IAM/permissions: RED - Security-sensitive, may break access patterns"
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
2. If edge function was changed, update the distribution to use the previous function version/ARN
3. If access restriction settings were changed, restore original trusted key groups or signers
4. If origin configuration was changed, restore original origin settings including timeouts and protocols
5. If logging configuration was changed, restore original log bucket and prefix settings
## Output Format

```yaml
root_cause: "s3_origin — <specific_cause>"
evidence:
  - type: origin_config
    content: "<OAC/OAI configuration>"
  - type: bucket_policy
    content: "<S3 bucket policy>"
  - type: encryption
    content: "<bucket encryption settings>"
severity: HIGH
mitigation:
  immediate: "Fix bucket policy or OAC configuration"
  long_term: "Migrate to OAC and implement least-privilege bucket policies"
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
