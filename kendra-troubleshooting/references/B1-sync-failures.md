---
title: "B1 — Data Source Sync Failures"
description: "Diagnose data source synchronization errors"
status: active
severity: HIGH
triggers:
  - "sync failed"
  - "data source sync error"
  - "documents not indexed"
owner: devops-agent
objective: "Resolve data source sync failures"
context: "Sync crawls data sources, processes documents, and indexes them. Failures can be partial or complete."
---

## Phase 1 — Triage

MUST:
- Check sync history: `aws kendra list-data-source-sync-jobs --id <ds-id> --index-id <index-id>`
- Review sync job metrics (documents added, modified, deleted, failed)
- Check data source configuration: `aws kendra describe-data-source --id <ds-id> --index-id <index-id>`
- Verify IAM role can access the data source

SHOULD:
- Check CloudWatch Logs for sync errors
- Verify data source credentials are valid
- Check for document format issues

MAY:
- Test with a small subset of documents
- Check for network connectivity to data source

## Phase 2 — Remediate

MUST:
- Fix IAM permissions for data source access
- Update expired credentials
- Re-trigger sync after fixes

SHOULD:
- Configure sync schedule for regular updates
- Monitor sync metrics

## Common Issues

- symptoms: "Sync completes but 0 documents added"
  diagnosis: "Crawl scope too restrictive or no matching documents."
  resolution: "Check inclusion/exclusion patterns. Verify documents exist in source."

- symptoms: "Sync fails with AccessDenied"
  diagnosis: "IAM role cannot access data source."
  resolution: "Update IAM role with required permissions for the data source type."

## Output Format

```yaml
root_cause: "sync_failure — <specific_cause>"
evidence:
  - type: sync_status
    content: "<sync job metrics>"
severity: HIGH
mitigation:
  immediate: "Fix sync blockers and re-sync"
  long_term: "Monitor sync health"
```

## Safety Ratings
```
safety_ratings:
  - "Triage commands (describe/list/get): GREEN — read-only"
  - "Configuration changes: YELLOW — recoverable"
  - "Delete/terminate operations: RED — irreversible"
```

## Escalation Conditions
- "Resource serves production workload"
- "Fix requires deleting or recreating resources"
- "Fix involves security or encryption changes"
- "Blast radius affects multiple dependent services"

## Data Sensitivity
- HIGH: credentials, PII, security configurations
- MEDIUM: resource configurations, metrics
- LOW: status information, error messages

## Prohibited Actions
- Never delete indexes that serve production search applications
- Never modify data source connectors during active sync operations
- Never remove FAQ entries without confirming impact on search quality
- Never alter access control configurations without security review
- Never delete experience builder applications used by end users

## Phase 3 — Rollback
- "Document current state before any changes"
- "If fix worsens the issue, revert to documented state"
- Restore previous data source connector configuration if sync operations fail
- Revert index relevance tuning settings if search quality degrades
- Re-enable previous ACL configurations if access control breaks

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
