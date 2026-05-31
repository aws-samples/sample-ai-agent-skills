---
title: "F1 — Blueprint Failures"
description: "Diagnose Lake Formation blueprint creation and execution failures"
status: active
severity: HIGH
triggers:
  - "blueprint failed"
  - "blueprint creation error"
  - "CreateBlueprint error"
  - "blueprint execution failed"
owner: devops-agent
objective: "Identify and resolve Lake Formation blueprint failures"
context: "Blueprints create Glue workflows for data ingestion into the data lake. They require specific IAM roles with Glue, S3, LF, and data source permissions. Blueprint failures often stem from role permission issues, data source connectivity, or configuration errors."
---

## Phase 1 — Triage

MUST:
- List blueprints: `aws lakeformation list-blueprints`
- Get blueprint details: `aws lakeformation describe-blueprint --name <name>`
- Check blueprint execution role permissions
- Verify data source connectivity

SHOULD:
- Check Glue workflow status: `aws glue get-workflow --name <workflow-name>`
- Verify the execution role trust policy
- Check CloudTrail for blueprint events
- Review Glue job logs for execution errors

MAY:
- Check data source credentials
- Verify S3 target location permissions

## Phase 2 — Remediate

MUST:
- Fix execution role permissions (Glue, S3, LF, data source)
- Ensure role trust policy includes glue.amazonaws.com and lakeformation.amazonaws.com
- Verify data source connectivity and credentials

SHOULD:
- Test data source connection independently
- Verify S3 target location is registered with LF
- Review and fix blueprint configuration parameters

MAY:
- Create standardized blueprint execution roles
- Set up monitoring for blueprint executions

## Common Issues

- symptoms: "Blueprint execution fails with permission error"
  diagnosis: "Execution role lacks required permissions."
  resolution: "Add Glue, S3, LF, and data source permissions to the execution role."

- symptoms: "Blueprint creates workflow but jobs fail"
  diagnosis: "Data source connectivity or credential issue."
  resolution: "Verify data source connection. Check credentials and network access."

## Output Format

```yaml
root_cause: "blueprint_failure — <specific_cause>"
evidence:
  - type: blueprint_status
    content: "<blueprint execution status>"
  - type: role_permissions
    content: "<execution role permissions>"
severity: HIGH
mitigation:
  immediate: "Fix execution role and data source configuration"
  long_term: "Standardize blueprint roles and automate validation"
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
