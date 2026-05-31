---
title: "G2 — Glue Integration Issues"
description: "Diagnose Lake Formation and Glue integration problems"
status: active
severity: MEDIUM
triggers:
  - "Glue job LF error"
  - "Glue crawler permission"
  - "catalog access from Glue"
  - "Glue LF integration"
owner: devops-agent
objective: "Identify and resolve Glue integration issues with Lake Formation"
context: "Glue crawlers, jobs, and the Data Catalog are tightly integrated with Lake Formation. Glue roles need both IAM and LF permissions. Crawlers need CREATE_TABLE and ALTER permissions. ETL jobs need SELECT and INSERT permissions. The Glue service role must be configured for LF."
---

## Phase 1 — Triage

MUST:
- Check Glue role LF permissions: `aws lakeformation list-permissions --principal '{"DataLakePrincipalIdentifier":"<glue-role-arn>"}'`
- Verify Glue role IAM permissions
- Check Glue job/crawler status and error logs
- Verify the Glue catalog resource exists

SHOULD:
- Check if the Glue role is registered as an LF principal
- Verify S3 access for the Glue role
- Check for LF permission conflicts
- Review Glue job CloudWatch logs

MAY:
- Check Glue connection configurations
- Verify Glue security configurations

## Phase 2 — Remediate

MUST:
- Grant LF permissions to the Glue role (CREATE_TABLE, ALTER, SELECT, INSERT as needed)
- Ensure Glue role IAM policy includes glue:*, s3:*, lakeformation:GetDataAccess
- Verify the Glue role trust policy includes glue.amazonaws.com

SHOULD:
- Use separate roles for crawlers and ETL jobs with least-privilege
- Test Glue operations after permission changes
- Monitor Glue job execution metrics

MAY:
- Create standardized Glue roles for LF integration
- Automate Glue role permission management

## Common Issues

- symptoms: "Glue crawler fails with access denied"
  diagnosis: "Crawler role lacks LF CREATE_TABLE or ALTER permission."
  resolution: "Grant CREATE_TABLE and ALTER LF permissions to the crawler role."

- symptoms: "Glue ETL job cannot read source table"
  diagnosis: "ETL role lacks LF SELECT permission on the source table."
  resolution: "Grant LF SELECT permission to the ETL role."

## Output Format

```yaml
root_cause: "glue_integration — <specific_cause>"
evidence:
  - type: glue_role_permissions
    content: "<Glue role LF and IAM permissions>"
  - type: job_errors
    content: "<Glue job/crawler error details>"
severity: MEDIUM
mitigation:
  immediate: "Fix Glue role permissions"
  long_term: "Standardize Glue-LF integration patterns"
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
