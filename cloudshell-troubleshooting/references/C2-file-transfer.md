---
title: "C2 — File Transfer"
description: "Diagnose CloudShell file upload and download issues"
status: active
severity: MEDIUM
triggers: ["upload failed", "download failed", "file transfer", "cannot upload", "cannot download"]
owner: devops-agent
objective: "Resolve file upload and download issues in CloudShell"
context: "CloudShell web interface supports 1 GB file upload/download. Larger transfers require S3. VPC environments may lack internet for S3 access."
---

## Phase 1 — Triage

MUST:
- Check available storage before upload: `df -h $HOME` (inside CloudShell)
- Verify file size is under 1 GB for web interface transfer
- For S3 transfers, verify connectivity: `aws s3 ls s3://bucket/ --max-items 1`

SHOULD:
- Check if browser is blocking downloads (popup blocker, security settings)
- For VPC environments, verify S3 endpoint or NAT gateway exists

MAY:
- Test S3 transfer with a small file first: `echo "test" > /tmp/test.txt && aws s3 cp /tmp/test.txt s3://bucket/test.txt`
- Check IAM permissions for S3 access: `aws iam simulate-principal-policy --policy-source-arn <user-arn> --action-names s3:GetObject s3:PutObject`

## Phase 2 — Remediate

MUST:
- For files > 1 GB: use S3 as intermediary: `aws s3 cp s3://bucket/large-file $HOME/large-file`
- Free storage space if upload fails due to quota: clean up $HOME
- For VPC environments: add S3 gateway endpoint to VPC

SHOULD:
- Use multipart upload for large S3 transfers: `aws s3 cp --expected-size <bytes> large-file s3://bucket/`
- Configure S3 Transfer Acceleration for cross-region transfers

MAY:
- Use `scp` or `rsync` for transfers to/from EC2 instances
- Set up AWS DataSync for recurring large transfers

## Safety Ratings

```
safety_ratings:
  - "aws s3 ls: GREEN — read-only bucket listing"
  - "aws s3 cp (download): GREEN — copies to local, no source modification"
  - "aws s3 cp (upload): YELLOW — writes data to S3 bucket"
  - "create-vpc-endpoint (S3): YELLOW — modifies VPC routing"
```

## Escalation Conditions

- File transfers consistently fail with timeout errors (network issue)
- S3 bucket policy blocks access from CloudShell IP ranges
- VPC environment requires S3 endpoint but VPC changes need approval
- Large dataset transfers exceed CloudShell session time limits

## Data Sensitivity

- `aws s3 ls`: LOW — bucket listing
- `aws s3 cp` (upload): HIGH — transfers potentially sensitive files to S3
- `aws s3 cp` (download): MEDIUM — downloads files to CloudShell environment
- File content inspection: varies — depends on file contents

## Prohibited Actions

- NEVER upload sensitive files (credentials, private keys) to public S3 buckets
- NEVER use --acl public-read when uploading files via CloudShell
- NEVER transfer files containing PII without encryption
- NEVER bypass file size limits by splitting and reassembling in /tmp (ephemeral)

## Phase 3 — Rollback

1. If file was uploaded to wrong S3 location: `aws s3 rm s3://bucket/wrong-path/file`
2. If VPC endpoint was created: `aws ec2 delete-vpc-endpoints --vpc-endpoint-ids <id>`
3. If large file filled storage: `rm $HOME/large-file` and re-download to correct location
4. If S3 bucket policy was modified: restore original policy from backup
5. Document transfer procedures for the team

## Common Issues

- symptoms: "Upload button is grayed out or upload fails silently"
  diagnosis: "File exceeds 1 GB limit or $HOME storage is full."
  resolution: "Check file size and available storage. For large files, upload to S3 first: aws s3 cp local-file s3://bucket/ then download in CloudShell."

## Output Format

```yaml
root_cause: "file_transfer — <specific_cause>"
evidence:
  - type: transfer_error
    content: "<error details>"
severity: MEDIUM
mitigation:
  immediate: "Complete file transfer"
  long_term: "Establish S3-based transfer workflow for large files"
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
