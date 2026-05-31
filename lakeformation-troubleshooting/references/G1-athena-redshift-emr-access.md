---
title: "G1 — Athena/Redshift/EMR Access Issues"
description: "Diagnose analytics service access issues with LF-managed tables"
status: active
severity: HIGH
triggers:
  - "Athena access denied"
  - "Redshift Spectrum LF error"
  - "EMR cannot access table"
  - "query access denied"
owner: devops-agent
objective: "Identify and resolve analytics service access issues with LF-managed tables"
context: "Athena, Redshift Spectrum, and EMR integrate with Lake Formation for fine-grained access control. Each service has specific IAM role requirements. The query execution role must have both IAM and LF permissions. Column-level and row-level security are enforced at query time."
---

## Phase 1 — Triage

MUST:
- Check LF permissions for the query role: `aws lakeformation list-permissions --principal '{"DataLakePrincipalIdentifier":"<role-arn>"}'`
- Verify IAM permissions for the query role (glue:GetTable, s3:GetObject, etc.)
- Check if IAMAllowedPrincipals is revoked (LF permissions enforced)
- Verify the table exists in Glue catalog

SHOULD:
- For Athena: check workgroup configuration and execution role
- For Redshift: check Spectrum IAM role and cluster permissions
- For EMR: check instance profile and security configuration
- Test with a simple SELECT query on the table

MAY:
- Check for column-level or row-level restrictions
- Review data cell filters affecting the query
- Check for LF-Tag-based permissions

## Phase 2 — Remediate

MUST:
- Grant LF permissions (SELECT at minimum) to the query execution role
- Ensure IAM permissions include glue:GetTable, glue:GetDatabase, s3:GetObject
- Verify the correct role is used for query execution

SHOULD:
- For Athena: configure workgroup with appropriate execution role
- For Redshift: attach LF-enabled IAM role to the cluster
- For EMR: configure LF integration in security configuration
- Test access after permission changes

MAY:
- Set up service-specific access monitoring
- Create role templates for each analytics service

## Common Issues

- symptoms: "Athena query returns access denied"
  diagnosis: "Athena execution role lacks LF SELECT permission."
  resolution: "Grant LF SELECT permission to the Athena execution role."

- symptoms: "Redshift Spectrum cannot read LF-managed table"
  diagnosis: "Redshift IAM role not configured for LF integration."
  resolution: "Enable LF integration for the Redshift IAM role. Grant LF permissions."

## Output Format

```yaml
root_cause: "analytics_access — <specific_cause>"
evidence:
  - type: lf_permissions
    content: "<LF permissions for query role>"
  - type: iam_permissions
    content: "<IAM permissions for query role>"
  - type: service_config
    content: "<analytics service LF integration config>"
severity: HIGH
mitigation:
  immediate: "Fix permissions for the analytics service role"
  long_term: "Standardize analytics service LF integration"
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
