---
title: "F2 — Workflow Issues"
description: "Diagnose Lake Formation blueprint workflow execution problems"
status: active
severity: MEDIUM
triggers:
  - "workflow failed"
  - "workflow stuck"
  - "Glue workflow error"
  - "ingestion workflow"
owner: devops-agent
objective: "Identify and resolve blueprint workflow execution issues"
context: "Blueprints create Glue workflows with crawlers and ETL jobs. Workflow failures can stem from Glue job errors, crawler failures, S3 access issues, or data format problems. Workflows execute as Glue resources with the blueprint's execution role."
---

## Phase 1 — Triage

MUST:
- Check workflow status: `aws glue get-workflow-run --name <workflow-name> --run-id <run-id>`
- List workflow runs: `aws glue get-workflow-runs --name <workflow-name>`
- Check individual job runs: `aws glue get-job-runs --job-name <job-name>`
- Check crawler status: `aws glue get-crawler --name <crawler-name>`

SHOULD:
- Review Glue job error logs in CloudWatch
- Check S3 target location accessibility
- Verify data format matches crawler/job expectations
- Check for Glue DPU capacity issues

MAY:
- Review Glue job metrics
- Check for concurrent workflow execution conflicts

## Phase 2 — Remediate

MUST:
- Fix the specific failing component (job, crawler, or trigger)
- Verify S3 permissions for the workflow role
- Ensure data source is accessible

SHOULD:
- Retry failed workflow runs after fixing issues
- Monitor workflow execution metrics
- Set up CloudWatch alarms for workflow failures

MAY:
- Optimize Glue job configurations
- Create workflow execution dashboards

## Common Issues

- symptoms: "Crawler fails to classify data"
  diagnosis: "Data format not recognized or S3 path empty."
  resolution: "Verify data exists at the S3 path. Check data format compatibility."

- symptoms: "ETL job fails with out of memory"
  diagnosis: "Insufficient DPU allocation for the data volume."
  resolution: "Increase DPU allocation in the Glue job configuration."

## Output Format

```yaml
root_cause: "workflow_issue — <specific_cause>"
evidence:
  - type: workflow_status
    content: "<workflow run status>"
  - type: job_errors
    content: "<Glue job error details>"
severity: MEDIUM
mitigation:
  immediate: "Fix failing workflow component and retry"
  long_term: "Implement workflow monitoring and automated retry"
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
