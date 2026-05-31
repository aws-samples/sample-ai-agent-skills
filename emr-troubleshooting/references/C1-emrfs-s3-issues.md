---
title: "C1 — EMRFS/S3 Issues"
description: "Diagnose EMRFS and S3 access issues on EMR"
status: active
severity: HIGH
triggers:
  - "S3 access denied from EMR"
  - "EMRFS error"
  - "S3 read/write failure"
  - "AmazonS3Exception"
owner: devops-agent
objective: "Identify and resolve EMRFS/S3 access issues on EMR"
context: "EMRFS is the EMR file system implementation for S3. Issues include access denied (IAM), throttling (request rate), encryption (KMS), and path issues. S3 provides strong consistency natively — EMRFS consistent view is deprecated."
---

## Phase 1 — Triage

MUST:
- Check instance profile S3 permissions: `aws iam simulate-principal-policy --policy-source-arn <instance-profile-role-arn> --action-names s3:GetObject s3:PutObject --resource-arns <s3-arn>`
- Verify S3 bucket exists and is accessible
- Check for S3 throttling (503 SlowDown) in application logs
- Verify KMS key permissions if bucket uses SSE-KMS

SHOULD:
- Check S3 bucket policy for deny statements
- Verify EMRFS configuration in cluster settings
- Check for S3 VPC endpoint if in private subnet
- Review S3 request metrics for throttling

MAY:
- Check for cross-account S3 access issues
- Verify S3 access point configuration

## Phase 2 — Remediate

MUST:
- Fix IAM permissions on the instance profile role
- Ensure KMS key policy allows the instance profile role
- Configure S3 VPC endpoint for private subnets

SHOULD:
- Implement S3 request retry with exponential backoff
- Use S3 prefixes to distribute request load
- Monitor S3 request metrics

MAY:
- Enable S3 request metrics for detailed monitoring
- Optimize S3 access patterns (partitioning, file sizes)

## Common Issues

- symptoms: "Access Denied when reading/writing S3"
  diagnosis: "Instance profile role lacks S3 permissions or bucket policy denies."
  resolution: "Add S3 permissions to instance profile role. Check bucket policy."

- symptoms: "503 SlowDown errors"
  diagnosis: "S3 request rate limit exceeded."
  resolution: "Add retry logic. Distribute requests across prefixes."

## Safety Ratings

| Phase 2 Action | safety_rating | Rationale |
|---|---|---|
| Fix IAM permissions on instance profile role | YELLOW | Changes role permissions — review scope |
| Ensure KMS key policy allows instance profile | YELLOW | Expands key access — review scope |
| Configure S3 VPC endpoint | GREEN | Adds network path — non-destructive |
| Implement S3 request retry with backoff | GREEN | Application-level change — non-destructive |
| Distribute requests across S3 prefixes | GREEN | Data organization optimization |

## Escalation Conditions

- Cluster runs production Spark jobs
- Fix requires cluster termination and recreation
- S3 access failure blocks data pipeline with strict SLA
- KMS key policy change affects other services using the same key

## Data Sensitivity

- **Classification: HIGH**
- S3 bucket paths reveal data lake structure and organization
- IAM permissions expose data access boundaries
- KMS key policies reveal encryption trust relationships
- S3 request metrics may reveal data access patterns and volumes

## Prohibited Actions

- **NEVER** suggest terminating a cluster with running steps without saving state
- **NEVER** suggest disabling encryption on a cluster or its storage
- **NEVER** suggest making S3 buckets publicly accessible to resolve access issues
- **NEVER** suggest using overly permissive S3 policies (s3:*) to resolve access errors

## Phase 3 — Rollback

| State-Changing Action | Rollback Step |
|---|---|
| Modified instance profile IAM permissions | Revert IAM policy to previous version |
| Updated KMS key policy | Revert KMS key policy to previous version |
| Created S3 VPC endpoint | Delete VPC endpoint if not needed |

## Output Format

```yaml
root_cause: "emrfs_s3 — <specific_cause>"
evidence:
  - type: iam_permissions
    content: "<instance profile S3 permissions>"
  - type: s3_errors
    content: "<S3 error details>"
severity: HIGH
mitigation:
  immediate: "Fix S3 access permissions"
  long_term: "Optimize S3 access patterns and implement monitoring"
```

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
  - "NEVER suggest disabling encryption"
  - "NEVER suggest public subnet placement for clusters"
  - "NEVER suggest removing Kerberos authentication"
