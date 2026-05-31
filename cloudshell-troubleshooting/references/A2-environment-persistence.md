---
title: "A2 — Environment Persistence"
description: "Diagnose CloudShell environment persistence issues"
status: active
severity: MEDIUM
triggers: ["files disappeared", "packages not installed", "environment reset", "lost my data"]
owner: devops-agent
objective: "Resolve environment persistence and data retention issues"
context: "Only $HOME has 1 GB persistent storage. System-level changes are ephemeral. Environments are deleted after 120 days of inactivity."
---

## Phase 1 — Triage

MUST:
- Check $HOME storage usage: `du -sh $HOME` (run inside CloudShell)
- Verify files are in $HOME, not /tmp or other system directories
- Check if environment was auto-deleted after 120 days of inactivity

SHOULD:
- Review .bashrc and .bash_profile for custom configurations: `cat $HOME/.bashrc`
- Check if storage quota (1 GB) has been exceeded: `df -h $HOME`

MAY:
- Review CloudTrail for DeleteEnvironment events: `aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventName,AttributeValue=DeleteEnvironment --max-results 10`

## Phase 2 — Remediate

MUST:
- Move important files to $HOME if stored elsewhere
- Set up $HOME/bin for custom binaries and add to PATH in .bashrc
- Use `pip install --user` for Python packages to persist in $HOME

SHOULD:
- Create a setup script in $HOME to reinstall system packages on session start
- Back up critical files to S3: `aws s3 sync $HOME/important s3://bucket/cloudshell-backup/`

MAY:
- Use .bashrc to auto-install frequently needed system packages on session start
- Set up dotfile management with a git repository in $HOME

## Safety Ratings

```
safety_ratings:
  - "du -sh / df -h: GREEN — read-only storage check"
  - "s3 sync backup: GREEN — copies data to S3 without modifying source"
  - "pip install --user: GREEN — installs packages in user directory"
  - "DeleteEnvironment: RED — permanently removes all persistent storage"
```

## Escalation Conditions

- Persistent storage corruption preventing environment startup
- Storage quota exceeded with no obvious large files (possible hidden files)
- Environment auto-deleted despite recent activity (potential service bug)
- Multiple users reporting simultaneous data loss

## Data Sensitivity

- `du -sh $HOME`: LOW — shows storage usage only
- `cat $HOME/.bashrc`: MEDIUM — may contain environment variables or paths
- `aws s3 sync`: MEDIUM — transfers user files to S3
- `aws cloudtrail lookup-events`: MEDIUM — reveals user activity

## Prohibited Actions

- NEVER delete $HOME contents without confirming backup exists
- NEVER suggest storing sensitive credentials in $HOME files (use AWS credentials chain)
- NEVER increase storage by symlinking to /tmp (data will be lost)
- NEVER assume system-level package installations will persist

## Phase 3 — Rollback

1. If files were accidentally deleted from $HOME, restore from S3 backup: `aws s3 sync s3://bucket/cloudshell-backup/ $HOME/important`
2. If .bashrc was corrupted, restore default: copy from /etc/skel/.bashrc
3. If storage is full, identify large files: `find $HOME -type f -size +10M -exec ls -lh {} \;`
4. If environment was deleted, create new and restore from S3 backup
5. Document what was lost and update backup procedures

## Common Issues

- symptoms: "My installed packages are gone after reopening CloudShell"
  diagnosis: "Packages installed via yum/apt are system-level and do not persist across sessions."
  resolution: "Install in $HOME using pip install --user, npm install -g with prefix set to $HOME, or download binaries to $HOME/bin."

## Output Format

```yaml
root_cause: "environment_persistence — <specific_cause>"
evidence:
  - type: persistence_error
    content: "<error details>"
severity: MEDIUM
mitigation:
  immediate: "Recover or recreate lost data"
  long_term: "Implement S3 backup strategy and $HOME-based tool installation"
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
