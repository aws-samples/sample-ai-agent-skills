---
title: "H2 — Compliance Reporting Issues"
description: "Diagnose issues with AWS Backup compliance reports and reporting configuration"
status: active
severity: MEDIUM
triggers:
  - "compliance report"
  - "report not generated"
  - "backup report"
  - "report plan"
  - "compliance status"
owner: devops-agent
objective: "Identify and resolve issues with backup compliance reporting and report delivery"
context: "AWS Backup report plans generate compliance reports on a schedule or on-demand. Reports are delivered to an S3 bucket and can include backup job status, copy job status, and compliance with framework controls. Report plans require an S3 bucket with appropriate permissions and a valid report template."
---

## Phase 1 — Triage

MUST:
- List report plans: `aws backup list-report-plans`
- Describe the report plan: `aws backup describe-report-plan --report-plan-name <name>`
- Check recent report jobs: `aws backup list-report-jobs --by-report-plan-name <name> --max-results 10`
- Describe a failed report job: `aws backup describe-report-job --report-job-id <job-id>`
- Verify the S3 delivery bucket exists and is accessible

SHOULD:
- Check the S3 bucket policy allows backup.amazonaws.com to put objects
- Verify the report plan's ReportTemplate and ReportDeliveryChannel configuration
- Check if the report plan covers the correct accounts and regions
- Verify the report schedule is configured correctly

MAY:
- Check CloudTrail for report plan modification events
- Verify the report format (CSV or JSON) is as expected
- Check if cross-account reporting is configured correctly

## Phase 2 — Remediate

MUST:
- Ensure the S3 bucket policy grants backup.amazonaws.com s3:PutObject permission
- Fix the report plan configuration (template, delivery channel, schedule)
- Retry report generation: `aws backup start-report-job --report-plan-name <name>`

SHOULD:
- Configure report plans for all required compliance dimensions (backup, copy, restore)
- Set up S3 lifecycle rules on the report bucket for cost management
- Enable S3 versioning on the report bucket for audit trail

MAY:
- Create custom report templates for specific compliance requirements
- Integrate reports with AWS QuickSight or third-party tools for visualization
- Set up SNS notifications for report generation completion

## Common Issues

- symptoms: "Report job fails with S3 access denied"
  diagnosis: "The S3 bucket policy does not allow backup.amazonaws.com to write objects."
  resolution: "Add a bucket policy statement allowing s3:PutObject from backup.amazonaws.com."

- symptoms: "Report is empty or missing expected data"
  diagnosis: "The report plan's scope (accounts, regions, resource types) does not cover the expected resources."
  resolution: "Update the report plan to include the correct accounts, regions, and frameworks."

- symptoms: "Report plan not generating on schedule"
  diagnosis: "The report plan schedule is misconfigured or the plan is in a failed state."
  resolution: "Verify the schedule configuration. Delete and recreate the plan if in a failed state."

- symptoms: "Cross-account report data is incomplete"
  diagnosis: "Cross-account monitoring is not enabled or member accounts are not included in the report scope."
  resolution: "Enable cross-account monitoring and include all target accounts in the report plan."

## Safety Ratings

| Phase 2 Action | safety_rating | Rationale |
|---|---|---|
| Fix S3 bucket policy for report delivery | GREEN | Grants backup service write access |
| Fix report plan configuration | GREEN | Configuration change — non-destructive |
| Retry report generation | GREEN | Non-destructive — generates new report |
| Delete and recreate failed report plan | YELLOW | Loses plan history — recreate with same config |
| Enable S3 versioning on report bucket | GREEN | Adds audit trail — non-destructive |

## Escalation Conditions

- Vault contains compliance-critical backups
- Fix requires vault lock modification
- Report delivery failure impacts external audit deadlines
- Cross-account reporting gaps affect organizational compliance posture
- Report data is required for regulatory submission

## Data Sensitivity

- **Classification: HIGH**
- Compliance reports contain backup status for all protected resources
- Report plans reveal which accounts and regions are monitored
- S3 report bucket contains historical compliance posture data
- Cross-account report scope reveals organizational backup strategy

## Prohibited Actions

- **NEVER** suggest deleting a vault with compliance lock
- **NEVER** suggest reducing retention below compliance requirements
- **NEVER** suggest deleting the S3 report bucket or its contents without confirming audit retention requirements
- **NEVER** suggest disabling compliance reporting without stakeholder approval

## Phase 3 — Rollback

| State-Changing Action | Rollback Step |
|---|---|
| Updated S3 bucket policy | Revert bucket policy to previous version |
| Deleted and recreated report plan | Cannot undo deletion — recreate with identical configuration |
| Updated report plan scope | Revert scope to previous accounts/regions |
| Enabled S3 versioning | Suspend versioning if not needed (existing versions retained) |

## Output Format

```yaml
root_cause: "compliance_reporting — <specific_cause>"
evidence:
  - type: report_plan
    content: "<report plan configuration>"
  - type: report_job
    content: "<report job status and error>"
  - type: s3_bucket
    content: "<S3 bucket policy and accessibility>"
severity: MEDIUM
mitigation:
  immediate: "Fix report plan configuration and S3 permissions"
  long_term: "Implement comprehensive compliance reporting across all accounts"
```

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
  - "NEVER suggest disabling vault lock"
  - "NEVER suggest deleting backup plans without review"
  - "NEVER suggest removing cross-account backup copies"
