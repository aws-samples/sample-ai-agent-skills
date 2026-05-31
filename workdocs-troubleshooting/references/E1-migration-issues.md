---
title: "E1 — Migration Issues"
description: "Diagnose WorkDocs content migration issues"
status: active
severity: HIGH
triggers: ["migration failed", "cannot migrate", "import error", "migration stuck"]
owner: devops-agent
objective: "Resolve content migration issues including data transfer, metadata preservation, and permission mapping"
context: "Migration to WorkDocs uses the WorkDocs Migration Service or API-based scripts. Large migrations require bandwidth and rate limit planning."
---

## Phase 1 — Triage

MUST:
- Verify target WorkDocs site is active and accessible
- Check target user accounts exist and are activated: `aws workdocs describe-users --organization-id <org-id>`
- Verify IAM permissions for migration operations
- Check API rate limits and throttling: review CloudTrail for throttling events

SHOULD:
- Verify source system connectivity and access
- Check migration tool version and configuration
- Estimate total data volume and expected migration duration

MAY:
- Review CloudTrail for migration-related API calls
- Check CloudWatch metrics for API throttling

## Phase 2 — Remediate

MUST:
- Fix authentication and permission issues for both source and target
- Ensure target users have sufficient storage quota for migrated content
- Retry failed migrations with smaller batch sizes if rate limited

SHOULD:
- Map source permissions to WorkDocs permission model before migration
- Preserve folder structure during migration
- Validate migrated content against source for completeness

MAY:
- Request API rate limit increases via AWS Support for large migrations
- Schedule migrations during off-peak hours to reduce throttling

## Safety Ratings

```
safety_ratings:
  - "workdocs describe-users: GREEN — read-only user listing"
  - "workdocs initiate-document-version-upload: YELLOW — uploads content to WorkDocs"
  - "workdocs create-folder: YELLOW — creates folder structure"
  - "workdocs add-resource-permissions: YELLOW — sets permissions on migrated content"
```

## Escalation Conditions

- Migration consistently fails at the same point suggesting data corruption
- API rate limits prevent migration from completing within the maintenance window
- Source system access is lost during migration
- Migrated content shows data integrity issues (corrupted files, missing metadata)

## Data Sensitivity

- `aws workdocs describe-users`: MEDIUM — reveals user identities
- Source system content: HIGH — contains business documents being migrated
- Migration logs: MEDIUM — may contain file names and user mappings
- Permission mappings: MEDIUM — reveals access control structure

## Prohibited Actions

- NEVER delete source content before verifying migration completeness
- NEVER migrate content without a rollback plan
- NEVER skip permission mapping — migrated content may be accessible to wrong users
- NEVER run migration against production without testing on a subset first

## Phase 3 — Rollback

1. If migration produced incorrect content, identify and delete migrated files: `aws workdocs delete-document --document-id <doc-id>`
2. If folder structure was created incorrectly, delete empty folders: `aws workdocs delete-folder --folder-id <folder-id>`
3. If permissions were set incorrectly, remove and reconfigure: `aws workdocs remove-resource-permissions --resource-id <id> --principal-id <principal-id>`
4. Verify source content is intact and unchanged after rollback
5. Document migration issues and adjust migration plan before retry

## Common Issues

- symptoms: "Migration fails with throttling errors"
  diagnosis: "API rate limits exceeded due to high volume of requests."
  resolution: "Reduce batch size, add delays between requests, or request rate limit increase from AWS Support."

- symptoms: "Migrated files have wrong permissions"
  diagnosis: "Permission mapping between source and WorkDocs was incorrect or missing."
  resolution: "Review permission mapping. Use add-resource-permissions to fix individual resources."

## Output Format

```yaml
root_cause: "migration — <specific_cause>"
evidence:
  - type: migration_error
    content: "<migration status, error details, data volume>"
severity: HIGH
mitigation:
  immediate: "Fix migration issue and retry"
  long_term: "Document migration runbook with tested procedures"
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
