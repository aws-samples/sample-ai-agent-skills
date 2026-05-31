---
title: "E2 — Home Directory Configuration Issues"
description: "Diagnose physical home directory path and access configuration problems"
status: active
severity: MEDIUM
triggers:
  - "home directory"
  - "HomeDirectory"
  - "HomeDirectoryType"
  - "PATH"
  - "landing directory"
  - "user home"
owner: devops-agent
objective: "Resolve home directory configuration issues for both physical (PATH) and logical directory types"
context: "Physical home directories (HomeDirectoryType=PATH) set a single directory path without chroot. Users land in this directory on login but may navigate above it if IAM permissions allow. For S3, the path is /<bucket-name>/optional-prefix. For EFS, the path is /<fs-id>/path. The home directory must be accessible by the user's IAM role."
---

## Phase 1 — Triage

MUST:
- Check home directory configuration: `aws transfer describe-user --server-id <server-id> --user-name <username> --query 'User.{HomeDirectory:HomeDirectory,HomeDirectoryType:HomeDirectoryType}'`
- Verify the home directory path format is correct for the backend type
- For S3: verify the bucket and prefix exist: `aws s3 ls s3://<bucket>/<prefix>/`
- For EFS: verify the path exists on the file system
- Check IAM role permissions cover the home directory path

SHOULD:
- Verify HomeDirectoryType matches the intended behavior (PATH vs LOGICAL)
- Check if the user can navigate outside the home directory (PATH type does not chroot)
- For custom IdP, verify HomeDirectory is returned in the correct format
- Check if session policy restricts access to the home directory path only

MAY:
- Test user access with the Transfer Family console test feature
- Check if multiple users share the same home directory and have conflicting permissions

## Phase 2 — Remediate

MUST:
- Set HomeDirectory in the correct format: `/<bucket-name>/prefix` for S3, `/<fs-id>/path` for EFS
- Ensure the IAM role has permissions to the home directory path
- Use LOGICAL type with chroot if users should not navigate above their home directory

SHOULD:
- Use session policies to restrict PATH-type users to their home directory prefix
- Use ${transfer:UserName} variable in home directory paths for per-user isolation
- Create the home directory path in the backend before user first login

MAY:
- Implement a Lambda-based custom IdP to dynamically set home directories based on user attributes
- Set up automated home directory provisioning for new users

## Common Issues

- symptoms: "User lands in the root of the S3 bucket instead of their prefix"
  diagnosis: "HomeDirectory is set to /<bucket-name> without a user-specific prefix."
  resolution: "Set HomeDirectory to /<bucket-name>/<username> or use logical mappings."

- symptoms: "User can navigate to other users' directories"
  diagnosis: "HomeDirectoryType is PATH (no chroot) and IAM role allows access to the entire bucket."
  resolution: "Switch to LOGICAL type for chroot, or add a session policy restricting to the user's prefix."

- symptoms: "Custom IdP users always land in wrong directory"
  diagnosis: "Lambda function returns HomeDirectory without the leading slash or with incorrect bucket name."
  resolution: "Ensure HomeDirectory starts with / followed by the bucket name: /<bucket-name>/<prefix>."

## Output Format

```yaml
root_cause: "home_directory — <specific_cause>"
evidence:
  - type: home_config
    content: "<HomeDirectory and HomeDirectoryType>"
  - type: backend_path
    content: "<path existence verification>"
  - type: permissions
    content: "<IAM role and session policy>"
severity: MEDIUM
mitigation:
  immediate: "Fix home directory path or type configuration"
  long_term: "Standardize on logical mappings with chroot for user isolation"
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
