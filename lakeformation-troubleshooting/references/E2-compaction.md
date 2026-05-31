---
title: "E2 — Governed Table Compaction"
description: "Diagnose governed table compaction issues"
status: active
severity: MEDIUM
triggers:
  - "compaction failed"
  - "too many small files"
  - "compaction not running"
  - "governed table performance"
owner: devops-agent
objective: "Identify and resolve governed table compaction issues"
context: "Governed tables create small files during transactions. Compaction merges these files to optimize query performance and reduce storage costs. Compaction can be automatic or manual. Failed compaction leads to degraded query performance."
---

## Phase 1 — Triage

MUST:
- Check table storage metrics via Glue: `aws glue get-table --database-name <db> --name <table>`
- List recent transactions: `aws lakeformation list-transactions --status-filter COMMITTED`
- Check for compaction-related CloudTrail events
- Verify LF permissions for compaction operations

SHOULD:
- Check the number of files in the table's S3 location
- Review query performance metrics
- Verify compaction role permissions
- Check for failed compaction attempts

MAY:
- Review storage costs for the governed table
- Check for partition-level compaction needs

## Phase 2 — Remediate

MUST:
- Trigger compaction if not running automatically
- Verify compaction role has S3 read/write permissions
- Monitor compaction progress

SHOULD:
- Schedule regular compaction for frequently updated tables
- Set up alerts for file count thresholds
- Optimize transaction batch sizes to reduce small files

MAY:
- Implement automated compaction scheduling
- Create compaction monitoring dashboards

## Common Issues

- symptoms: "Query performance degraded on governed table"
  diagnosis: "Too many small files from transactions, compaction needed."
  resolution: "Run compaction. Schedule regular compaction for active tables."

- symptoms: "Compaction fails with permission error"
  diagnosis: "Compaction role lacks S3 or LF permissions."
  resolution: "Verify compaction role has S3 read/write and LF permissions."

## Output Format

```yaml
root_cause: "compaction — <specific_cause>"
evidence:
  - type: file_count
    content: "<number of files in table location>"
  - type: compaction_status
    content: "<compaction execution status>"
severity: MEDIUM
mitigation:
  immediate: "Run compaction to merge small files"
  long_term: "Schedule automated compaction and optimize transaction patterns"
```


## Safety Ratings

| Aspect | Rating | Notes |
|--------|--------|-------|
| Data Risk | Medium | Operations may affect service configurations and data access patterns |
| Service Impact | High | Incorrect changes can disrupt active workloads and dependent services |
| Recovery Complexity | Medium | Most actions are reversible but may require coordinated rollback procedures |
| Blast Radius | Medium | Impact typically scoped to the affected resource and its direct dependents |

## Escalation Conditions

- **Escalate immediately** if the issue involves potential data loss or corruption
- **Escalate immediately** if a production workload is impacted and cannot be restored within the agreed SLA
- **Escalate immediately** if the root cause is unclear after initial diagnostic steps are exhausted
- **Escalate to service team** if the issue appears to be a platform-level bug or undocumented behavior
- **Escalate to security team** if unauthorized access, policy violations, or credential exposure is suspected

## Data Sensitivity

- Verify data classification before performing any read or export operations
- Ensure all troubleshooting outputs (logs, configurations, snapshots) are handled according to the customer's data handling policy
- Mask or redact personally identifiable information (PII) in all shared diagnostics
- Do not copy production data to non-production environments without explicit authorization
- Retain troubleshooting artifacts only for the duration required and dispose of them securely

## Prohibited Actions

- **NEVER** revoke Lake Formation permissions on production databases without explicit customer approval
- **NEVER** modify IAMAllowedPrincipals settings without understanding the full permission inheritance chain
- **NEVER** deregister S3 data locations that are actively referenced by Glue tables or crawlers
- **NEVER** disable hybrid access mode on databases with active IAM-based consumers
- **NEVER** alter cross-account resource shares (RAM) without coordinating with all participating accounts

## Phase 3 � Rollback

1. **Identify rollback trigger**: Determine if the applied change has caused degradation, unexpected behavior, or failed validation checks
2. **Preserve current state**: Capture logs, configuration snapshots, and resource states before initiating rollback
3. **Execute rollback steps**: Revert configuration changes in reverse order of application, verifying each step
4. **Validate service recovery**: Confirm that the service has returned to its pre-change operational state using health checks and functional tests
5. **Document rollback**: Record the rollback trigger, steps taken, time to recovery, and any residual issues for post-incident review
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
