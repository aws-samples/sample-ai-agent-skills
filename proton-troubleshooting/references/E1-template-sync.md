---
title: "E1 — Proton Template Sync Issues"
description: "Diagnose template sync configuration and synchronization failures"
status: active
severity: MEDIUM
triggers:
  - "template sync failed"
  - "sync not working"
  - "template not updating"
  - "git sync error"
owner: devops-agent
objective: "Identify why Proton template sync fails and restore automatic template version creation"
context: "Template sync connects a Git repository to Proton templates, automatically creating new template versions when changes are pushed. Requires a CodeStar connection and proper repository structure."
---

## Phase 1 — Triage

MUST:
- Check template sync status: `aws proton get-template-sync-status --template-name <template-name> --template-type ENVIRONMENT`
- Check sync config: `aws proton get-template-sync-config --template-name <template-name> --template-type ENVIRONMENT`
- Verify CodeStar connection: `aws codestar-connections get-connection --connection-arn <connection-arn>`
- Check if connection is AVAILABLE: connection status must be AVAILABLE, not PENDING
- Check CloudTrail for sync events: `aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventName,AttributeValue=CreateTemplateSyncConfig --max-results 10`

SHOULD:
- Verify repository and branch exist and are accessible
- Check repository directory structure matches Proton template bundle format
- Verify the subdirectory path in sync config points to the correct location

MAY:
- Check recent commits to the synced branch
- Verify webhook is configured on the repository

## Phase 2 — Remediate

MUST:
- Authorize CodeStar connection if in PENDING status (requires console)
- Fix repository directory structure to match Proton template bundle format
- Correct the subdirectory path in sync configuration

SHOULD:
- Test template bundle locally before pushing to the synced branch
- Verify branch protection rules don't prevent Proton from reading
- Check repository permissions for the CodeStar connection

MAY:
- Delete and recreate sync configuration if corrupted
- Set up repository branch protection with Proton access

## Common Issues

- symptoms: "Template sync configured but no new versions created"
  diagnosis: "CodeStar connection is PENDING — not authorized."
  resolution: "Authorize the connection in the AWS Console."

- symptoms: "Sync creates version but it fails registration"
  diagnosis: "Repository template bundle structure is invalid."
  resolution: "Fix the bundle structure (schema.yaml, manifest.yaml, infrastructure/)."

- symptoms: "Sync stopped working after repository changes"
  diagnosis: "Branch renamed, deleted, or subdirectory path changed."
  resolution: "Update sync configuration with correct branch and subdirectory."

## Output Format

```yaml
root_cause: "template_sync_issue — <specific_cause>"
evidence:
  - type: sync_status
    content: "<sync status and last sync details>"
  - type: connection_status
    content: "<CodeStar connection state>"
  - type: repository
    content: "<repository and branch details>"
severity: MEDIUM
mitigation:
  immediate: "Fix connection, repository structure, or sync configuration"
  long_term: "Add template validation to repository CI before sync"
```


## Safety Ratings
- Overall Risk: MEDIUM
- Data Exposure Risk: MEDIUM — template sync accesses source repositories
- Service Disruption Risk: MEDIUM — sync failures prevent template updates
- Compliance Impact: MEDIUM — template sync ensures configuration consistency

## Escalation Conditions
- Template sync completely fails
- Sync produces incorrect template versions
- Repository access issues prevent sync
- Sync conflicts between template versions
- Sync causes unintended template updates

## Data Sensitivity
- Repository access configurations — CONFIDENTIAL
- Template source code — INTERNAL
- Sync configurations — INTERNAL
- Sync logs — INTERNAL
- Never expose repository credentials

## Prohibited Actions
- Do NOT bypass template sync validation
- Do NOT ignore sync failures
- Do NOT modify sync configurations during active sync operations
- Do NOT grant sync roles access to unauthorized repositories
- Do NOT disable template sync without alternative update mechanism
- Do NOT share sync configurations externally

## Phase 3 — Rollback
- Restore previous sync configurations
- Re-establish repository connections
- Revert template versions if sync produced incorrect results
- Verify template consistency after sync recovery
- Document sync issues and repository access
- Update template sync procedures

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
  - "NEVER suggest overly broad IAM permissions as a fix"
  - "NEVER suggest disabling security controls for troubleshooting"
  - "NEVER suggest deleting resources without confirmation"
