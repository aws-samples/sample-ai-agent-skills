---
title: "G1 — Log File Validation"
description: "Diagnose log file integrity validation issues"
status: active
severity: HIGH
triggers:
  - "log validation failed"
  - "digest files missing"
  - "log integrity"
  - "tamper detection"
owner: devops-agent
objective: "Ensure CloudTrail log file integrity validation is working correctly"
context: "Log file validation creates SHA-256 digest files hourly. These digests verify log files haven't been modified or deleted. Validation must be enabled on the trail and checked using validate-logs CLI command."
---

## Phase 1 — Triage

MUST:
- Check if validation is enabled: `aws cloudtrail get-trail --name <trail-name> --query 'Trail.LogFileValidationEnabled'`
- Run validation: `aws cloudtrail validate-logs --trail-arn <trail-arn> --start-time <start> --end-time <end>`
- Check for digest files in S3: `aws s3 ls s3://<bucket>/AWSLogs/<account-id>/CloudTrail-Digest/`

SHOULD:
- Review validation results for invalid or missing files
- Check S3 bucket versioning (helps recover deleted files)
- Verify digest file delivery timing

MAY:
- Check if lifecycle rules are deleting digest files
- Review S3 access logs for unauthorized modifications

## Phase 2 — Remediate

MUST:
- Enable validation if disabled: `aws cloudtrail update-trail --name <trail-name> --enable-log-file-validation`
- Investigate any validation failures (potential tampering)
- Enable S3 bucket versioning to protect against deletion

SHOULD:
- Set up regular automated validation checks
- Configure S3 Object Lock for immutable log storage
- Alert on validation failures

MAY:
- Enable MFA Delete on the S3 bucket
- Create separate read-only IAM policies for log access

## Common Issues

- symptoms: "Validation shows INVALID files"
  diagnosis: "Log files were modified after delivery."
  resolution: "Investigate potential tampering. Check S3 access logs. Enable Object Lock."

- symptoms: "Digest files missing"
  diagnosis: "Validation not enabled or digest files deleted."
  resolution: "Enable validation. Check S3 lifecycle rules. Enable versioning."

## Safety Ratings

| Phase 2 Action | Rating | Rationale |
|---|---|---|
| Enable log file validation | GREEN | Additive security feature |
| Enable S3 bucket versioning | GREEN | Additive protection |
| Investigate validation failures | GREEN | Read-only analysis |
| Configure S3 Object Lock | YELLOW | Makes objects immutable; cannot be undone |

## Escalation Conditions

- Log file validation shows INVALID files — escalate immediately to security team (potential tampering)
- Digest files missing or deleted — escalate to security team for investigation
- Unauthorized S3 access to CloudTrail logs detected — escalate to security team
- Compliance requirement for immutable log storage — escalate to compliance team

## Data Sensitivity

| Triage Command | Sensitivity | Rationale |
|---|---|---|
| `aws cloudtrail validate-logs` | HIGH | Exposes log integrity status |
| `aws cloudtrail get-trail` (validation config) | HIGH | Exposes validation configuration |
| `aws s3 ls` (digest files) | HIGH | Exposes digest file locations |

## Prohibited Actions

- **NEVER** suggest stopping or deleting CloudTrail trails
- **NEVER** suggest disabling log file validation
- **NEVER** suggest removing S3 bucket policy for log delivery
- **NEVER** delete digest files — they are required for log integrity verification
- **NEVER** dismiss validation failures without thorough investigation

## Phase 3 — Rollback

| Remediation Action | Rollback Step |
|---|---|
| Enabled log file validation | Disable validation (NOT recommended): `aws cloudtrail update-trail --no-enable-log-file-validation` |
| Enabled S3 bucket versioning | Cannot disable versioning (can only suspend) |
| Configured S3 Object Lock | Cannot be removed once enabled on a bucket |

## Output Format

```yaml
root_cause: "log_validation — <specific_cause>"
evidence:
  - type: validation_status
    content: "<validation enabled/disabled>"
  - type: validation_results
    content: "<validate-logs output>"
severity: HIGH
mitigation:
  immediate: "Enable validation and investigate failures"
  long_term: "Automate validation and enable immutable storage"
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
  - command: "lookup-events"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"
  - command: "get-trail-status"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"
  - command: "describe-trails"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"

## Prohibited Actions

prohibited_actions:
  - "NEVER suggest disabling CloudTrail logging"
  - "NEVER suggest deleting trail logs"
  - "NEVER reduce logging scope in production"
