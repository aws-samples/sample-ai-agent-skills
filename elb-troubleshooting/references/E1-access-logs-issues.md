---
title: "E1 — Access Logs Issues"
description: "Diagnose and resolve ELB access logging configuration and delivery issues"
status: active
severity: MEDIUM
triggers:
  - "access logs not delivered"
  - "access logs missing"
  - "S3 bucket policy"
  - "ELB logging"
owner: devops-agent
objective: "Resolve ELB access log delivery issues and ensure logs are properly stored for analysis"
context: "ELB access logs capture detailed information about requests processed by the load balancer and are delivered to an S3 bucket. ALB and CLB support access logs; NLB supports connection logs. Logs are delivered every 5 minutes. Common issues include S3 bucket policy misconfiguration, incorrect bucket region, encryption settings, or logging not enabled."
---

## Phase 1 — Triage

MUST:
- Check if access logging is enabled: `aws elbv2 describe-load-balancer-attributes --load-balancer-arn <lb-arn> --query 'Attributes[?Key==\`access_logs.s3.enabled\`]'`
- Verify the S3 bucket and prefix: `aws elbv2 describe-load-balancer-attributes --load-balancer-arn <lb-arn> --query 'Attributes[?starts_with(Key, \`access_logs\`)]'`
- Check S3 bucket policy allows ELB to write: `aws s3api get-bucket-policy --bucket <bucket-name>`
- Verify the S3 bucket exists and is in the same region as the ALB: `aws s3api get-bucket-location --bucket <bucket-name>`
- Check if logs are being delivered: `aws s3 ls s3://<bucket>/<prefix>/AWSLogs/<account-id>/elasticloadbalancing/<region>/`

SHOULD:
- Verify the S3 bucket encryption settings are compatible with ELB logging
- Check if the bucket has a lifecycle policy that may be deleting logs prematurely
- Verify the ELB service account has write access (account ID varies by region)

MAY:
- Check CloudTrail for S3 PutObject failures from the ELB service
- Verify no S3 bucket policies or SCPs are blocking ELB writes
- Review the log file format and naming convention

## Phase 2 — Remediate

MUST:
- If logging not enabled: `aws elbv2 modify-load-balancer-attributes --load-balancer-arn <lb-arn> --attributes Key=access_logs.s3.enabled,Value=true Key=access_logs.s3.bucket,Value=<bucket> Key=access_logs.s3.prefix,Value=<prefix>`
- If bucket policy missing: add the ELB service account write permission to the S3 bucket policy
- If bucket in wrong region: create a new bucket in the ALB's region and update the logging configuration
- If encryption blocks writes: use SSE-S3 encryption (ELB does not support SSE-KMS for access logs)

SHOULD:
- Verify logs appear within 5-10 minutes after enabling
- Set up S3 lifecycle policies for log retention management
- Configure S3 event notifications for log processing pipelines

MAY:
- Set up Athena tables for querying access logs
- Implement automated log analysis for error patterns
- Configure cross-account log delivery if needed

## Common Issues

- symptoms: "Access logs enabled but no files in S3"
  diagnosis: "S3 bucket policy does not allow the ELB service account to write."
  resolution: "Add the correct ELB service account (region-specific) to the bucket policy with s3:PutObject permission."

- symptoms: "Enable logging returns 'Access Denied'"
  diagnosis: "S3 bucket policy or IAM permissions prevent the configuration."
  resolution: "Update the S3 bucket policy. Ensure the IAM principal has elasticloadbalancing:ModifyLoadBalancerAttributes permission."

- symptoms: "Logs delivered but encrypted and unreadable"
  diagnosis: "Bucket uses SSE-KMS encryption which is not supported by ELB access logs."
  resolution: "Change bucket encryption to SSE-S3 or disable default encryption."

- symptoms: "Log files are empty or incomplete"
  diagnosis: "Low traffic volume or logs are still being aggregated (5-minute delivery interval)."
  resolution: "Wait for the next delivery interval. Verify traffic is flowing through the LB."

## Output Format

```yaml
root_cause: "Access logs issue — <specific_cause>"
evidence:
  - type: logging_config
    content: "<access log attributes>"
  - type: bucket_policy
    content: "<S3 bucket policy summary>"
severity: MEDIUM
mitigation:
  immediate: "Fix bucket policy, encryption, or logging configuration"
  long_term: "Implement log lifecycle management and automated analysis"
```

## Safety Ratings

safety_ratings:
  - "Phase 1 triage commands (describe/get/list): GREEN — read-only"
  - "Phase 2 S3 bucket policy changes: YELLOW — affects bucket access permissions"
  - "Phase 2 enabling logging: GREEN — no impact on traffic"

## Escalation Conditions

escalation_conditions:
  - "Access logs are required for compliance and are not being delivered"
  - "S3 bucket policy changes affect other services using the same bucket"
  - "Root cause cannot be identified after 3 hypothesis pivots"
  - "Cross-account log delivery is required"
  - "Log data may have been lost during the outage period"

## Data Sensitivity

data_sensitivity:
  - command: "get-bucket-policy"
    sensitivity: MEDIUM
    contains: "S3 access permissions and principal ARNs"
  - command: "access log files"
    sensitivity: HIGH
    contains: "Client IPs, request paths, headers, and response details"
  - command: "describe-load-balancer-attributes"
    sensitivity: LOW
    contains: "Logging configuration settings"

## Prohibited Actions

prohibited_actions:
  - "NEVER make the access log S3 bucket publicly accessible"
  - "NEVER delete access log files without verifying retention requirements"
  - "NEVER disable access logging on production LBs without explicit approval"
  - "NEVER store access logs in an unencrypted bucket if compliance requires encryption"

## Phase 3 — Rollback

- Before changing S3 bucket policy, save the current policy: `aws s3api get-bucket-policy --bucket <bucket> > bucket-policy-backup.json`
- If bucket policy changes break other services, restore the previous policy
- If logging was disabled, re-enable it with the previous bucket and prefix
- If encryption changes affect other objects in the bucket, revert the encryption settings
- Verify log delivery resumes after any configuration changes
