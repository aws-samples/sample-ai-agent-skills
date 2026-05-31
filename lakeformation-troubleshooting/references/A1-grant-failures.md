---
title: "A1 — Permission Grant Failures"
description: "Diagnose why Lake Formation permission grants are failing"
status: active
severity: HIGH
triggers:
  - "grant failed"
  - "cannot grant permissions"
  - "GrantPermissions error"
  - "permission grant denied"
owner: devops-agent
objective: "Identify why LF permission grants are failing and resolve the issue"
context: "Permission grants can fail due to insufficient LF admin privileges, invalid principal ARN, invalid resource specification, the caller not being an LF admin or having grant option, or the resource not being registered with LF."
---

## Phase 1 — Triage

MUST:
- Verify caller is LF admin: `aws lakeformation get-data-lake-settings --query 'DataLakeSettings.DataLakeAdmins'`
- Check if the resource exists in Glue catalog: `aws glue get-table --database-name <db> --name <table>`
- Verify the principal ARN is valid
- Check CloudTrail for grant failure: `aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventName,AttributeValue=GrantPermissions --max-results 10`

SHOULD:
- Verify the data location is registered: `aws lakeformation list-resources`
- Check if the caller has grant option for the permission
- Verify the permission type is valid for the resource type
- Check for conflicting permissions

MAY:
- Verify IAM permissions for the LF API call
- Check for SCPs blocking LF operations

## Phase 2 — Remediate

MUST:
- Ensure the caller is an LF admin or has grant option
- Use correct resource specification (database, table, column)
- Verify the principal ARN format is correct

SHOULD:
- Register data locations before granting permissions
- Remove IAMAllowedPrincipals if enforcing LF permissions
- Test grants with a single principal first

MAY:
- Use LF-Tags for scalable permission management
- Automate permission grants with IaC

## Common Issues

- symptoms: "AccessDeniedException when granting"
  diagnosis: "Caller is not an LF admin and doesn't have grant option."
  resolution: "Add caller as LF admin or grant with grant option from an admin."

- symptoms: "Invalid resource error"
  diagnosis: "Database or table doesn't exist in Glue catalog."
  resolution: "Verify resource exists: aws glue get-table --database-name <db> --name <table>."

## Output Format

```yaml
root_cause: "grant_failure — <specific_cause>"
evidence:
  - type: lf_admins
    content: "<LF admin configuration>"
  - type: resource
    content: "<resource existence and registration>"
severity: HIGH
mitigation:
  immediate: "Fix the specific grant blocker"
  long_term: "Establish LF admin delegation and permission management processes"
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
