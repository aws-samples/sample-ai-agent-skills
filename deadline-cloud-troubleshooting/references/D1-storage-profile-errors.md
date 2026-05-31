---
title: "D1 — Storage Profile Errors"
description: "Diagnose storage profile configuration and path mapping errors"
status: active
severity: HIGH
triggers: ["storage profile error", "path mapping failed", "file not found on worker", "storage configuration wrong"]
owner: devops-agent
objective: "Resolve storage profile configuration issues for file access"
context: "Storage profiles define how file paths are mapped between the submitter's file system and worker file systems. They are queue-specific and OS-specific. Incorrect profiles cause jobs to fail finding input files."
---

## Phase 1 — Triage

MUST:
- List storage profiles for the queue: `aws deadline list-storage-profiles-for-queue --farm-id <farm-id> --queue-id <queue-id>`
- Get storage profile details: `aws deadline get-storage-profile-for-queue --farm-id <farm-id> --queue-id <queue-id> --storage-profile-id <id>`
- Verify file system locations match actual worker file system paths

SHOULD:
- Check OS-specific path mappings (Linux vs Windows)
- Verify the submitter's storage profile matches the worker's profile
- Test file accessibility on a worker instance

MAY:
- Review job attachment configuration for the queue
- Check S3 bucket permissions for job attachment storage

## Phase 2 — Remediate

MUST:
- Fix file system location paths to match actual worker file system
- Create OS-specific storage profiles for mixed-OS fleets
- Ensure storage profile is associated with the correct queue

SHOULD:
- Configure both Linux and Windows path mappings if using mixed fleets
- Verify S3 bucket for job attachments has correct permissions

MAY:
- Set up shared file systems (EFS/FSx) and configure mount points in profiles
- Configure storage profile for multiple project directories

## Safety Ratings

```
safety_ratings:
  - "list-storage-profiles-for-queue: GREEN — read-only profile enumeration"
  - "get-storage-profile-for-queue: GREEN — read-only profile details"
  - "create-storage-profile-for-queue: YELLOW — creates new path mapping"
  - "update-storage-profile-for-queue: YELLOW — changes path mappings affecting job file access"
  - "delete-storage-profile-for-queue: RED — removes path mapping, breaks file access for jobs"
```

## Escalation Conditions

- Storage profile changes affect multiple teams' render pipelines
- Shared file system (EFS/FSx) mount issues require infrastructure team
- Cross-platform path mapping requires coordination between Linux and Windows teams
- Storage profile deletion needed but referenced by active jobs

## Data Sensitivity

- Storage profile paths: MEDIUM — reveals file system structure and project organization
- S3 bucket configuration: MEDIUM — reveals attachment storage location
- File system contents: HIGH — may contain proprietary creative assets
- Mount point configuration: MEDIUM — reveals infrastructure setup

## Prohibited Actions

- NEVER delete a storage profile used by active or pending jobs
- NEVER modify storage profiles during active render sessions
- NEVER configure storage profiles pointing to system directories
- NEVER expose S3 job attachment bucket publicly

## Phase 3 — Rollback

1. If storage profile was modified incorrectly, revert path mappings
2. If storage profile was deleted, recreate with original configuration
3. If new profile caused job failures, revert to previous profile and resubmit jobs
4. Verify file access on workers after profile changes

## Common Issues

- symptoms: "Job fails with file not found on worker"
  diagnosis: "Storage profile path mapping does not match actual file location on worker."
  resolution: "Update storage profile file system locations to match worker file system. Verify paths exist on workers."

- symptoms: "Windows workers cannot find files but Linux workers can"
  diagnosis: "Storage profile only has Linux path mappings, missing Windows paths."
  resolution: "Create Windows-specific file system locations in the storage profile (use backslash paths)."

## Output Format

```yaml
root_cause: "storage_profile — <specific_cause>"
evidence:
  - type: profile_config
    content: "<storage profile path mappings>"
severity: HIGH
mitigation:
  immediate: "Fix path mappings"
  long_term: "Standardize storage profiles across projects"
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
