---
title: "I3 — CloudFront + S3 Issues"
description: "Diagnose CloudFront distribution issues when using S3 as origin"
status: active
severity: MEDIUM
triggers:
  - "CloudFront S3 403"
  - "OAC"
  - "OAI"
  - "CloudFront origin access"
  - "S3 REST endpoint vs website endpoint"
owner: devops-agent
objective: "Fix CloudFront-to-S3 integration issues including OAC/OAI and endpoint configuration"
context: "CloudFront can use S3 as an origin with Origin Access Control (OAC, recommended) or Origin Access Identity (OAI, legacy). OAC supports SSE-KMS, all S3 regions, and S3 Object Lambda. The S3 origin must use the REST API endpoint (bucket.s3.region.amazonaws.com), not the website endpoint, for OAC/OAI. Using the website endpoint treats S3 as a custom origin."
---

## Phase 1 — Triage

MUST:
- Check CloudFront distribution origin configuration: `aws cloudfront get-distribution --id <dist-id> --query 'Distribution.DistributionConfig.Origins'`
- Verify OAC or OAI is configured: `aws cloudfront list-origin-access-controls`
- Check the S3 bucket policy for CloudFront access
- Verify the S3 origin uses the REST endpoint (not website endpoint)

SHOULD:
- Check if the bucket policy allows the CloudFront distribution
- Verify the origin path is correct
- Check CloudFront error pages configuration for custom error responses

MAY:
- Check CloudFront access logs for error details
- Verify SSL/TLS settings if using custom domain

## Phase 2 — Remediate

MUST:
- Use OAC (recommended over OAI): `aws cloudfront create-origin-access-control --origin-access-control-config '{"Name":"<name>","SigningProtocol":"sigv4","SigningBehavior":"always","OriginAccessControlOriginType":"s3"}'`
- Update bucket policy to allow CloudFront OAC:
  ```json
  {
    "Effect": "Allow",
    "Principal": {"Service": "cloudfront.amazonaws.com"},
    "Action": "s3:GetObject",
    "Resource": "arn:aws:s3:::<bucket>/*",
    "Condition": {"StringEquals": {"AWS:SourceArn": "arn:aws:cloudfront::<account>:distribution/<dist-id>"}}
  }
  ```
- Use the S3 REST API endpoint as the origin (not the website endpoint)

SHOULD:
- Migrate from OAI to OAC for better security and feature support
- Configure CloudFront default root object to match S3 index document
- Set up custom error pages in CloudFront for 403/404 responses

MAY:
- Enable CloudFront logging for troubleshooting
- Use CloudFront Functions or Lambda@Edge for URL rewriting

## Common Issues

- symptoms: "403 AccessDenied through CloudFront but direct S3 access works"
  diagnosis: "OAC/OAI not configured or bucket policy does not allow CloudFront."
  resolution: "Configure OAC and update bucket policy to allow cloudfront.amazonaws.com."

- symptoms: "Subdirectory index.html not served (403 on /path/)"
  diagnosis: "S3 REST endpoint does not serve index documents for subdirectories. Only the website endpoint does."
  resolution: "Use CloudFront Functions to append index.html to directory requests."

- symptoms: "SSE-KMS objects return 403 through CloudFront with OAI"
  diagnosis: "OAI does not support SSE-KMS. Only OAC supports KMS-encrypted objects."
  resolution: "Migrate from OAI to OAC and grant CloudFront kms:Decrypt on the KMS key."

## Output Format

```yaml
root_cause: "cloudfront_s3 — <specific_cause>"
evidence:
  - type: distribution_config
    content: "<origin and OAC/OAI configuration>"
  - type: bucket_policy
    content: "<CloudFront access policy>"
severity: MEDIUM
mitigation:
  immediate: "Fix OAC configuration and bucket policy"
  long_term: "Migrate to OAC and configure proper error handling"
```


## Safety Ratings

| Rating | Justification |
|--------|--------------|
| 🟡 YELLOW | Involves modifying bucket policies for CloudFront OAC/OAI access and creating origin access controls. Bucket policy changes are state-changing but recoverable. Uses get-distribution and get-bucket-policy for diagnosis. |

## Escalation Conditions
- Remediation requires modifying bucket policy in a production account
- Fix involves changing Block Public Access settings
- Remediation affects cross-account access
- Change impacts encryption configuration
- CloudFront origin changes affect content delivery for production applications

## Rollback
- Pre-change: "Save current bucket policy/ACL/CORS before modification"
- Verification: "Test access with the specific operation after change"
- Revert: "Restore previous configuration if change causes unintended access"

## Data Sensitivity
- HIGH: "Bucket policies reveal all authorized principals"
- HIGH: "ACLs expose cross-account grants"
- MEDIUM: "CloudFront distribution configuration reveals origin architecture"
- LOW: "Bucket metrics and storage class distribution"

## Prohibited Actions
- NEVER suggest disabling S3 Block Public Access as a remediation
- NEVER suggest `"Principal": "*"` without restrictive Condition keys
- NEVER suggest removing bucket encryption
- NEVER suggest `s3:*` in any policy fix
- NEVER suggest deleting a bucket to resolve configuration issues

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

## Phase 3 � Rollback

- Document the current state before making any changes
- If remediation worsens the issue, revert to the documented pre-change state
- Remove any temporary resources created during investigation
- Verify the system returns to its previous functional state after rollback
