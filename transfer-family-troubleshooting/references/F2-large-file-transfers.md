---
title: "F2 — Large File Transfer Issues"
description: "Diagnose failures and timeouts during large file transfers"
status: active
severity: MEDIUM
triggers:
  - "large file"
  - "file too big"
  - "transfer timeout"
  - "multipart"
  - "upload failed"
  - "incomplete transfer"
  - "5 GB limit"
owner: devops-agent
objective: "Resolve issues with transferring large files through Transfer Family"
context: "Transfer Family automatically uses S3 multipart upload for files larger than a threshold. S3 has a 5 TB maximum object size and 5 GB maximum for single PUT. For EFS, file size is limited by available storage. Large transfers can fail due to timeouts, network interruptions, or backend limits. SFTP does not natively support resume — interrupted transfers must restart."
---

## Phase 1 — Triage

MUST:
- Check for failed or incomplete transfers in structured logs: `aws logs filter-log-events --log-group-name /aws/transfer/<server-id> --filter-pattern "\"ERROR\""`
- Check S3 for incomplete multipart uploads: `aws s3api list-multipart-uploads --bucket <bucket-name>`
- Verify the IAM role has s3:PutObject and s3:AbortMultipartUpload permissions
- Check CloudWatch for transfer errors: `aws cloudwatch get-metric-statistics --namespace AWS/Transfer --metric-name FilesIn --dimensions Name=ServerId,Value=<server-id> --start-time <start> --end-time <end> --period 300 --statistics Sum`
- Check if the connection was dropped during transfer (idle timeout, network interruption)

SHOULD:
- Check S3 bucket lifecycle rules for aborting incomplete multipart uploads
- Verify no S3 bucket size limits or quotas are in effect
- Check EFS available storage if using EFS backend
- Verify the client's connection timeout settings are sufficient for large transfers

MAY:
- Check if S3 Object Lock or versioning is causing storage issues
- Monitor network stability during large transfers
- Check if the client supports SFTP extensions for resume (rare)

## Phase 2 — Remediate

MUST:
- Clean up incomplete multipart uploads: `aws s3api abort-multipart-upload --bucket <bucket> --key <key> --upload-id <id>`
- Ensure IAM role has all required S3 permissions including multipart operations
- Configure S3 lifecycle rule to automatically abort incomplete multipart uploads after N days

SHOULD:
- Increase client-side timeout settings for large file transfers
- Use a stable network connection (wired, VPN, Direct Connect) for large transfers
- Monitor transfer progress via CloudWatch BytesIn metrics

MAY:
- Consider AWS DataSync for very large or bulk file transfers
- Implement client-side file splitting for extremely large files
- Use S3 Transfer Acceleration for cross-region large file transfers

## Common Issues

- symptoms: "Large file upload fails at ~5 GB"
  diagnosis: "S3 single PUT limit is 5 GB. Transfer Family should auto-switch to multipart, but IAM role may lack multipart permissions."
  resolution: "Ensure IAM role has s3:PutObject, s3:CreateMultipartUpload, s3:UploadPart, s3:CompleteMultipartUpload, s3:AbortMultipartUpload."

- symptoms: "Transfer disconnects after 10 minutes of inactivity"
  diagnosis: "SFTP idle timeout or network device timeout closes the connection during slow transfers."
  resolution: "Configure client keep-alive. Check firewall/NAT idle timeout settings."

- symptoms: "Incomplete multipart uploads consuming S3 storage"
  diagnosis: "Failed transfers leave orphaned multipart upload parts in S3."
  resolution: "Add S3 lifecycle rule to abort incomplete multipart uploads after 7 days."

## Output Format

```yaml
root_cause: "large_file_transfer — <specific_cause>"
evidence:
  - type: transfer_logs
    content: "<structured log entries>"
  - type: multipart_uploads
    content: "<incomplete multipart upload list>"
  - type: iam_permissions
    content: "<multipart-related permissions>"
severity: MEDIUM
mitigation:
  immediate: "Fix permissions or clean up incomplete uploads"
  long_term: "Implement lifecycle rules and transfer monitoring"
```


## Safety Ratings

| Phase 2 Action | Rating | Rationale |
|---|---|---|
| Investigate and collect diagnostic data | GREEN | Read-only analysis; no state change |
| Modify configuration settings | YELLOW | Configuration change; reversible with backup |
| Restart or reboot components | YELLOW | Service interruption; recoverable |
| Delete or remove resources | RED | Potentially destructive; requires confirmation and backup |

## Escalation Conditions

- Production environment impact detected or imminent
- Multiple failed remediation attempts (>2 cycles without resolution)
- Security-related root cause identified (unauthorized access, data exposure)
- Issue requires cross-service or cross-account coordination
- Customer-facing service degradation exceeding SLA thresholds
- Regulatory or compliance implications discovered during investigation

## Data Sensitivity

- Transferred files may contain any type of sensitive data
- SSH keys and credentials enable server access
- Directory mappings reveal S3/EFS data organization
- Classification: CONFIDENTIAL - file transfer content and access credentials

## Prohibited Actions

- NEVER modify production server endpoints during active file transfers
- NEVER change authentication providers without user notification
- NEVER delete SSH host keys without coordinating with all SFTP clients
- NEVER modify directory mappings that could expose unauthorized S3/EFS paths

## Phase 3 -- Rollback

MUST:
- Document all changes made during Phase 2 remediation
- Verify rollback procedures exist before applying any fix
- Confirm service health after rollback using the same diagnostic steps from Phase 1
- Notify stakeholders if rollback is initiated in production

SHOULD:
- Maintain a timeline of all actions taken for post-incident review
- Capture before/after configuration snapshots
- Test rollback in non-production environment when possible
- Update runbook with lessons learned after incident resolution

MAY:
- Automate rollback procedures for frequently occurring issues
- Create CloudWatch alarms to detect regression after rollback

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
  - "NEVER suggest disabling SFTP encryption"
  - "NEVER suggest FTP protocol for production"
  - "NEVER suggest overly broad IAM roles for transfer users"
