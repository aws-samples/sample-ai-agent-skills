---
title: "E1 — Governed Table Transaction Issues"
description: "Diagnose governed table transaction problems in Lake Formation"
status: active
severity: HIGH
triggers:
  - "transaction failed"
  - "TransactionCanceledException"
  - "transaction timeout"
  - "governed table error"
owner: devops-agent
objective: "Identify and resolve governed table transaction issues"
context: "Governed tables support ACID transactions with snapshot isolation. Transactions can fail due to conflicts, timeouts, or permission issues. Concurrent writes to the same data can cause TransactionCanceledException. Transactions must be explicitly committed or cancelled."
---

## Phase 1 — Triage

MUST:
- Check transaction status: `aws lakeformation describe-transaction --transaction-id <id>`
- List active transactions: `aws lakeformation list-transactions --status-filter ACTIVE`
- Verify governed table exists: `aws glue get-table --database-name <db> --name <table>`
- Check for transaction conflicts in CloudTrail

SHOULD:
- Check for long-running transactions that may be blocking
- Verify the principal has transaction permissions
- Check for concurrent write operations
- Review transaction timeout settings

MAY:
- Check governed table storage metrics
- Review compaction status

## Phase 2 — Remediate

MUST:
- Cancel stuck transactions: `aws lakeformation cancel-transaction --transaction-id <id>`
- Retry failed transactions with conflict resolution
- Ensure proper transaction lifecycle (start → operations → commit/cancel)

SHOULD:
- Implement retry logic for transaction conflicts
- Minimize transaction duration to reduce conflicts
- Use appropriate isolation level for the use case

MAY:
- Set up monitoring for transaction failures
- Implement transaction management patterns

## Common Issues

- symptoms: "TransactionCanceledException"
  diagnosis: "Concurrent write conflict on the same data."
  resolution: "Implement retry logic. Reduce concurrent writes to the same partitions."

- symptoms: "Transaction timeout"
  diagnosis: "Transaction exceeded the timeout period."
  resolution: "Reduce transaction scope. Commit more frequently."

## Output Format

```yaml
root_cause: "transaction_issue — <specific_cause>"
evidence:
  - type: transaction_status
    content: "<transaction details>"
  - type: conflicts
    content: "<concurrent transaction information>"
severity: HIGH
mitigation:
  immediate: "Cancel stuck transactions and retry"
  long_term: "Implement transaction management patterns and retry logic"
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
