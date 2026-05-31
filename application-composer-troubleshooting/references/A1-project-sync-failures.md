---
title: "A1 — Project Sync Failures"
description: "Diagnose Application Composer project sync failures"
status: active
severity: HIGH
triggers: ["project sync not working", "sync failed", "changes not saving to files"]
owner: devops-agent
objective: "Resolve project sync failures between Application Composer and local files"
context: "Project sync writes canvas changes to local files in real-time. Failures can cause data loss or stale templates."
---

## Phase 1 — Triage

MUST:
- Verify the local project directory exists and is writable
- Check if the template file is valid YAML/JSON: `python3 -c "import yaml; yaml.safe_load(open('template.yaml'))"`
- Confirm no file locks from other editors or processes: `lsof template.yaml`

SHOULD:
- Check if git has uncommitted changes that may conflict: `git status`
- Verify disk space is available: `df -h .`

MAY:
- Review browser console logs for Application Composer errors
- Check if the IDE extension version is current

## Phase 2 — Remediate

MUST:
- Commit or stash local changes before re-enabling sync: `git stash`
- Ensure the template file is not read-only: `chmod 644 template.yaml`
- Restart the sync connection from Application Composer UI

SHOULD:
- Close other editors that may hold file locks
- Validate the template after sync resumes: `sam validate --template-file template.yaml --lint`

MAY:
- Re-clone the project if file system corruption is suspected
- Switch to a clean branch for testing sync

## Safety Ratings

```
safety_ratings:
  - "python3 yaml.safe_load: GREEN — read-only YAML validation"
  - "lsof: GREEN — read-only file lock inspection"
  - "git stash: YELLOW — modifies working directory state"
  - "chmod 644: YELLOW — modifies file permissions"
  - "sam validate: GREEN — read-only template validation"
```

## Escalation Conditions

- Sync fails repeatedly after clearing file locks and permissions
- Template file becomes corrupted or empty after sync attempt
- Sync causes data loss in Lambda function code files
- Multiple team members experience sync failures simultaneously

## Data Sensitivity

- `lsof template.yaml`: LOW — reveals process IDs holding file locks
- `git status`: LOW — reveals uncommitted file changes
- `sam validate`: LOW — validates template structure without exposing secrets

## Prohibited Actions

- NEVER force-overwrite local files without confirming the user has backups
- NEVER disable version control to work around sync issues
- NEVER suggest deleting the local project directory as a fix
- NEVER modify template files manually while sync is active

## Phase 3 — Rollback

1. If sync overwrote local changes, recover from git: `git checkout -- template.yaml` or `git stash pop`
2. If the template was corrupted, restore from last known good commit: `git checkout HEAD~1 -- template.yaml`
3. Disable project sync in Application Composer before making manual template edits
4. Verify the restored template is valid: `sam validate --template-file template.yaml --lint`
5. Document the sync failure and conditions that caused it

## Common Issues

- symptoms: "Changes made in Application Composer are not reflected in local files"
  diagnosis: "Project sync may be disconnected or the file path has changed."
  resolution: "Re-enable project sync and verify the correct project directory is selected."

## Output Format

```yaml
root_cause: "project_sync — <specific_cause>"
evidence:
  - type: sync_error
    content: "<error details>"
severity: HIGH
mitigation:
  immediate: "Restore sync connection"
  long_term: "Commit changes before sync sessions"
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
