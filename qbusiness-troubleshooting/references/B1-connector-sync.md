---
title: "B1 — Data Source Connector Sync Failures"
description: "Diagnose data source sync errors and incomplete syncs"
status: active
severity: HIGH
triggers:
  - "sync failed"
  - "documents not syncing"
  - "connector error"
owner: devops-agent
objective: "Resolve data source sync failures"
context: "Sync crawls source systems, processes documents, and indexes them for retrieval."
---

## Phase 1 — Triage

MUST:
- Check sync jobs: `aws qbusiness list-data-source-sync-jobs --application-id <app-id> --index-id <idx-id> --data-source-id <ds-id>`
- Review sync metrics (added, modified, deleted, failed)
- Check data source configuration
- Verify IAM role and credentials

SHOULD:
- Check CloudWatch Logs for sync errors
- Verify source system accessibility
- Check for document format issues

MAY:
- Test with smaller document set
- Check network connectivity to source

## Phase 2 — Remediate

MUST:
- Fix credentials and permissions
- Correct data source configuration
- Re-trigger sync

SHOULD:
- Configure sync schedule
- Monitor sync health

## Common Issues

- symptoms: "Sync fails with authentication error"
  diagnosis: "Connector credentials expired."
  resolution: "Update credentials in Secrets Manager. Re-trigger sync."

## Output Format

```yaml
root_cause: "connector_sync — <specific_cause>"
evidence:
  - type: sync_status
    content: "<sync job details>"
severity: HIGH
mitigation:
  immediate: "Fix sync blockers"
  long_term: "Monitor sync health"
```


## Safety Ratings
- Overall Risk: HIGH
- Data Exposure Risk: HIGH — connectors access enterprise data sources
- Service Disruption Risk: MEDIUM — sync failures affect data freshness
- Compliance Impact: HIGH — data sync must comply with data governance policies

## Escalation Conditions
- Connector sync completely fails
- Sync ingests unauthorized data
- Sync performance degrades significantly
- Connector credentials expire
- Sync causes data source performance issues

## Data Sensitivity
- Data source credentials — RESTRICTED
- Synced document content — CONFIDENTIAL
- Connector configurations — INTERNAL
- Sync logs — INTERNAL (may contain CONFIDENTIAL data)
- Never expose data source credentials

## Prohibited Actions
- Do NOT sync data from unauthorized sources
- Do NOT store connector credentials in plaintext
- Do NOT ignore sync failures
- Do NOT sync data without data governance approval
- Do NOT grant connectors access to unauthorized data
- Do NOT bypass connector security configurations

## Phase 3 — Rollback
- Restore previous connector configurations
- Rotate connector credentials if compromised
- Re-establish data source connections
- Verify sync functionality after changes
- Document sync issues and data governance compliance
- Update connector management procedures

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
