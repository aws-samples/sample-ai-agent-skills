---
title: "C1 — Database/Table Permissions"
description: "Diagnose database and table permission issues in Lake Formation"
status: active
severity: HIGH
triggers:
  - "table access denied"
  - "database permission error"
  - "cannot access table"
  - "catalog permission issue"
owner: devops-agent
objective: "Identify and resolve database/table permission issues in Lake Formation"
context: "Database and table permissions in LF control who can access catalog resources. Permissions include CREATE_DATABASE, CREATE_TABLE, ALTER, DROP, DESCRIBE, SELECT, INSERT, DELETE. Permissions can be granted at database or table level with optional grant option."
---

## Phase 1 — Triage

MUST:
- List permissions on the database: `aws lakeformation list-permissions --resource '{"Database":{"Name":"<db>"}}'`
- List permissions on the table: `aws lakeformation list-permissions --resource '{"Table":{"DatabaseName":"<db>","Name":"<table>"}}'`
- Check IAMAllowedPrincipals grants on the resource
- Verify the principal has the required permission type (SELECT, DESCRIBE, etc.)

SHOULD:
- Check effective permissions: `aws lakeformation get-effective-permissions-for-path --resource-arn <s3-arn>`
- Verify the Glue catalog resource exists
- Check for LF-Tag-based permissions on the resource
- Verify IAM permissions complement LF permissions

MAY:
- Check for column-level restrictions
- Review data cell filters on the table

## Phase 2 — Remediate

MUST:
- Grant the required permission to the principal
- Ensure IAMAllowedPrincipals is handled appropriately
- Verify both IAM and LF permissions allow access

SHOULD:
- Use least-privilege permissions (SELECT only if read-only needed)
- Consider using LF-Tags for scalable permission management
- Test access after granting permissions

MAY:
- Set up permission templates for common access patterns
- Automate permission grants for new tables

## Common Issues

- symptoms: "Access denied on table despite LF grant"
  diagnosis: "IAM permissions missing or IAMAllowedPrincipals not revoked."
  resolution: "Verify IAM permissions. If using LF model, revoke IAMAllowedPrincipals."

- symptoms: "Can describe table but cannot query data"
  diagnosis: "DESCRIBE permission granted but SELECT not granted."
  resolution: "Grant SELECT permission on the table."

## Output Format

```yaml
root_cause: "catalog_permissions — <specific_cause>"
evidence:
  - type: lf_permissions
    content: "<LF permissions on the resource>"
  - type: iam_permissions
    content: "<IAM policy evaluation>"
severity: HIGH
mitigation:
  immediate: "Grant required permissions"
  long_term: "Implement systematic permission management with LF-Tags"
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
