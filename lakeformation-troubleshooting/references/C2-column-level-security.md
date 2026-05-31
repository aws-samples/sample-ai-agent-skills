---
title: "C2 — Column-Level Security"
description: "Diagnose column-level permission issues in Lake Formation"
status: active
severity: MEDIUM
triggers:
  - "column access denied"
  - "column not visible"
  - "column-level permissions"
  - "restricted columns"
owner: devops-agent
objective: "Identify and resolve column-level security issues in Lake Formation"
context: "Column-level permissions restrict which columns a principal can access. When granted, only specified columns are visible. Queries referencing non-permitted columns fail. Column permissions use include or exclude lists and work with Athena, Redshift Spectrum, and EMR."
---

## Phase 1 — Triage

MUST:
- Check column permissions: `aws lakeformation list-permissions --resource '{"TableWithColumns":{"DatabaseName":"<db>","Name":"<table>","ColumnNames":["<col>"]}}'`
- List all permissions on the table including column restrictions
- Verify which columns the principal can access
- Check if the query references non-permitted columns

SHOULD:
- Verify column names match exactly (case-sensitive)
- Check for include vs exclude column lists
- Verify data cell filters that may restrict columns
- Test with a simple SELECT on permitted columns

MAY:
- Check for LF-Tag-based column permissions
- Review column permission inheritance

## Phase 2 — Remediate

MUST:
- Grant column-level permissions with correct column names
- Use include list (specific columns) or exclude list (all except)
- Verify queries only reference permitted columns

SHOULD:
- Document column access patterns per role
- Use exclude lists when most columns should be accessible
- Test column permissions with sample queries

MAY:
- Implement data cell filters for row+column filtering
- Automate column permission management

## Common Issues

- symptoms: "Query fails with column access denied"
  diagnosis: "Query references columns not in the principal's permission grant."
  resolution: "Add missing columns to the permission grant or modify the query."

- symptoms: "SELECT * fails but specific columns work"
  diagnosis: "Column-level permissions restrict visible columns."
  resolution: "Use explicit column list in queries or grant access to all needed columns."

## Output Format

```yaml
root_cause: "column_security — <specific_cause>"
evidence:
  - type: column_permissions
    content: "<column-level permission grants>"
  - type: query_columns
    content: "<columns referenced in failing query>"
severity: MEDIUM
mitigation:
  immediate: "Fix column permission grants"
  long_term: "Document column access patterns and automate management"
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
