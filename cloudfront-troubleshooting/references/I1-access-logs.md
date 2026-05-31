---
title: "I1 — Access Log Issues"
description: "Diagnose CloudFront standard access log delivery and configuration issues"
status: active
severity: MEDIUM
triggers:
  - "access logs"
  - "standard logs"
  - "logs not appearing"
  - "log delivery"
  - "missing logs"
owner: devops-agent
objective: "Resolve CloudFront standard access log delivery issues"
context: "CloudFront standard access logs are delivered to an S3 bucket. Logs are delivered on a best-effort basis with typical delay of a few minutes to a few hours. Log files use a specific naming format. The S3 bucket must have the correct ACL or bucket policy to allow CloudFront to write logs. Logs are not guaranteed to be complete — some records may be missing."
---

## Phase 1 — Triage

MUST:
- Check logging configuration: `aws cloudfront get-distribution-config --id <dist-id> --query 'DistributionConfig.Logging'`
- Verify the S3 bucket exists and is in the correct region
- Check S3 bucket ACL/policy allows CloudFront log delivery
- Check if logs exist with expected prefix: `aws s3 ls s3://<log-bucket>/<prefix>/ --recursive | tail -20`

SHOULD:
- Verify bucket ownership controls allow ACLs (BucketOwnerPreferred or ObjectWriter)
- Check if S3 Block Public Access is blocking the log delivery ACL
- Verify the log prefix is correct

MAY:
- Check CloudTrail for PutObject events from CloudFront to the log bucket
- Compare log timestamps with expected delivery delay

## Phase 2 — Remediate

MUST:
- Enable logging in distribution config with correct bucket and prefix
- Grant CloudFront write access to the S3 bucket:
  - Option 1: Bucket ACL with `awslogsdelivery` full control
  - Option 2: Bucket policy allowing `s3:PutObject` from CloudFront
- Ensure bucket ownership controls support ACLs if using ACL method

SHOULD:
- Use a dedicated logging bucket separate from content buckets
- Set up lifecycle rules to manage log retention
- Use a consistent prefix per distribution for organization

MAY:
- Set up Athena for log analysis
- Configure S3 event notifications for log processing pipelines

## Common Issues

- symptoms: "No log files appearing in S3 bucket"
  diagnosis: "Logging not enabled or S3 bucket permissions incorrect."
  resolution: "Enable logging and grant CloudFront write access to the bucket."

- symptoms: "Logs delayed by several hours"
  diagnosis: "Standard logs are best-effort with variable delay."
  resolution: "Expected behavior. Use real-time logs for near-instant delivery."

- symptoms: "Log delivery stopped after enabling BucketOwnerEnforced"
  diagnosis: "BucketOwnerEnforced disables ACLs, breaking CloudFront log delivery via ACL."
  resolution: "Use bucket policy instead of ACL, or switch to BucketOwnerPreferred."


## Safety Ratings

```
safety_ratings:
  - "Inspect/Describe resources: GREEN - Read-only API calls, no state change"
  - "Review logs and metrics: GREEN - Read-only observability data access"
  - "Delete/Remove resources: RED - Potentially irreversible, requires confirmation"
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
2. If logging configuration was changed, restore original log bucket and prefix settings
## Output Format

```yaml
root_cause: "access_logs — <specific_cause>"
evidence:
  - type: logging_config
    content: "<logging configuration>"
  - type: bucket_permissions
    content: "<S3 bucket ACL/policy>"
  - type: log_files
    content: "<recent log file listing>"
severity: MEDIUM
mitigation:
  immediate: "Fix logging configuration or bucket permissions"
  long_term: "Implement log analysis pipeline with Athena"
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
