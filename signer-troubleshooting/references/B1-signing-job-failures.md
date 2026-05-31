---
title: "B1 — Signing Job Failures"
description: "Diagnose signing job execution failures"
status: active
severity: HIGH
triggers: ["signing job failed", "StartSigningJob error", "job status failed"]
owner: devops-agent
objective: "Resolve signing job execution failures"
context: "Signing jobs are asynchronous. They read source artifacts from S3, sign them, and write signed artifacts to S3. Failures can be caused by S3 access issues, invalid artifacts, or profile problems."
---

## Phase 1 — Triage

MUST:
- Check job status: `aws signer describe-signing-job --job-id <job-id>`
- Verify the signing profile is active: `aws signer get-signing-profile --profile-name <profile-name>`
- Check S3 source artifact exists: `aws s3 ls s3://<bucket>/<key>`

SHOULD:
- Verify S3 bucket policy allows Signer service access
- Check if the source artifact format matches the platform requirements

MAY:
- Review CloudTrail for StartSigningJob errors
- Check S3 destination bucket permissions

## Phase 2 — Remediate

MUST:
- Ensure the signing profile is active (not revoked or cancelled)
- Verify S3 source bucket allows `signer.amazonaws.com` to GetObject
- Verify S3 destination bucket allows `signer.amazonaws.com` to PutObject
- Ensure the artifact format matches the platform (ZIP for Lambda, binary for IoT)

SHOULD:
- Check S3 bucket encryption — if using KMS, ensure Signer has key access
- Retry the signing job after fixing the issue

MAY:
- Enable S3 access logging to diagnose permission issues
- Set up SNS notifications for job status changes

## Common Issues

- symptoms: "Signing job fails with 'Access Denied' error"
  diagnosis: "S3 bucket policy does not allow Signer service to read source or write destination."
  resolution: "Add bucket policy allowing signer.amazonaws.com for s3:GetObject (source) and s3:PutObject (destination)."

- symptoms: "Signing job fails with 'Invalid artifact' error"
  diagnosis: "Source artifact format does not match the signing platform requirements."
  resolution: "Verify artifact format. Lambda requires ZIP. IoT requires platform-specific binary format."

## Safety Ratings

```
safety_ratings:
  - "describe-signing-job: GREEN — read-only job details"
  - "get-signing-profile: GREEN — read-only profile details"
  - "start-signing-job: YELLOW — initiates signing and writes to S3"
  - "s3 put-bucket-policy: YELLOW — modifies S3 access policy"
```

## Escalation Conditions

- Signing jobs fail consistently with internal service errors
- S3 bucket policy changes require approval from a different team
- KMS key policy prevents Signer access and key admin is unavailable
- Signing job failures block CI/CD pipeline deployments

## Data Sensitivity

- `aws signer describe-signing-job`: MEDIUM — reveals S3 paths and signing parameters
- S3 source artifacts: HIGH — contain application code
- S3 signed artifacts: HIGH — contain signed application code
- S3 bucket policies: MEDIUM — reveal access control configuration

## Prohibited Actions

- NEVER grant Signer broad S3 access (s3:*) — use specific GetObject and PutObject permissions
- NEVER store signing artifacts in public S3 buckets
- NEVER retry failed signing jobs without investigating the root cause
- NEVER modify S3 bucket policies without understanding impact on other services

## Phase 3 — Rollback

1. If S3 bucket policy was modified incorrectly, revert to previous policy
2. If a signing job produced incorrect output, delete the signed artifact from S3
3. If the signing profile was changed, revert to the previous profile
4. Verify the CI/CD pipeline can successfully sign artifacts after fixes
5. Document the failure and resolution

## Output Format

```yaml
root_cause: "signing_job_failure — <specific_cause>"
evidence:
  - type: job_error
    content: "<error details from describe-signing-job>"
severity: HIGH
mitigation:
  immediate: "Fix signing job prerequisites"
  long_term: "Automate signing job validation in CI/CD"
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
