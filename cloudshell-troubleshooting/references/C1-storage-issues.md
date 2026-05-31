---
title: "C1 — Storage Issues"
description: "Diagnose CloudShell storage quota and corruption issues"
status: active
severity: MEDIUM
triggers: ["disk full", "no space left", "storage quota", "cannot save file", "1 GB limit"]
owner: devops-agent
objective: "Resolve CloudShell persistent storage issues"
context: "CloudShell provides 1 GB persistent storage in $HOME per region. Exceeding quota prevents file creation. Storage is deleted after 120 days of inactivity."
---

## Phase 1 — Triage

MUST:
- Check storage usage: `du -sh $HOME` (inside CloudShell)
- Check available space: `df -h $HOME` (inside CloudShell)
- Identify large files: `find $HOME -type f -size +10M -exec ls -lh {} \;` (inside CloudShell)

SHOULD:
- Check hidden files and directories: `du -sh $HOME/.* 2>/dev/null | sort -rh | head -20` (inside CloudShell)
- Verify .npm, .cache, .local directories for bloat: `du -sh $HOME/.npm $HOME/.cache $HOME/.local 2>/dev/null`

MAY:
- Check if environment was recently recreated (data loss): `aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventName,AttributeValue=CreateEnvironment --max-results 5`

## Phase 2 — Remediate

MUST:
- Clean up unnecessary files: remove old downloads, logs, and cache files
- Clear package caches: `rm -rf $HOME/.npm/_cacache $HOME/.cache/pip` (inside CloudShell)
- Move large files to S3: `aws s3 cp $HOME/large-file s3://bucket/path/`

SHOULD:
- Set up regular cleanup in .bashrc for cache directories
- Use S3 as primary storage and download files on demand

MAY:
- Compress infrequently accessed files: `tar -czf $HOME/archive.tar.gz $HOME/old-files/`
- Use git sparse checkout for large repositories

## Safety Ratings

```
safety_ratings:
  - "du / df / find: GREEN — read-only storage inspection"
  - "rm -rf cache directories: YELLOW — removes cached data, may slow reinstalls"
  - "aws s3 cp: GREEN — copies files to S3 without modifying source"
  - "delete-environment: RED — permanently removes all persistent storage"
```

## Escalation Conditions

- Storage appears full but du shows less than 1 GB used (filesystem corruption)
- Environment fails to start due to storage issues
- Critical files cannot be backed up due to storage quota
- Multiple environments in different regions all showing storage issues

## Data Sensitivity

- `du -sh / df -h`: LOW — storage usage statistics
- `find $HOME -type f`: MEDIUM — reveals file names and paths
- `aws s3 cp`: MEDIUM — transfers user files to S3
- Cache cleanup: LOW — removes temporary data

## Prohibited Actions

- NEVER delete $HOME/.ssh without confirming keys are backed up
- NEVER delete $HOME/.aws without confirming configuration is documented
- NEVER use rm -rf $HOME/* without explicit user confirmation
- NEVER suggest expanding storage beyond 1 GB (not possible)

## Phase 3 — Rollback

1. If important files were accidentally deleted: restore from S3 backup if available
2. If .bashrc was removed: copy default from /etc/skel/.bashrc
3. If SSH keys were deleted: regenerate and update authorized_keys on remote servers
4. If AWS config was removed: reconfigure with `aws configure`
5. Document what was cleaned up and establish backup procedures

## Common Issues

- symptoms: "bash: cannot create temp file for here-document: No space left on device"
  diagnosis: "$HOME storage has reached the 1 GB limit."
  resolution: "Clean up large files: find $HOME -type f -size +10M -exec ls -lh {} \\;. Remove caches: rm -rf $HOME/.npm/_cacache $HOME/.cache."

## Output Format

```yaml
root_cause: "storage — <specific_cause>"
evidence:
  - type: storage_error
    content: "<error details>"
severity: MEDIUM
mitigation:
  immediate: "Free up storage space"
  long_term: "Implement S3-based storage strategy and regular cache cleanup"
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
