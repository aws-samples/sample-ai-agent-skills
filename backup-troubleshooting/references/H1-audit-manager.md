---
title: "H1 — Backup Audit Manager Issues"
description: "Diagnose AWS Backup Audit Manager framework and control compliance issues"
status: active
severity: MEDIUM
triggers:
  - "audit manager"
  - "backup compliance"
  - "framework"
  - "control failure"
  - "non-compliant"
  - "backup audit"
owner: devops-agent
objective: "Identify and resolve Backup Audit Manager framework configuration and compliance issues"
context: "AWS Backup Audit Manager provides pre-built and custom frameworks with controls to monitor backup compliance. Controls check for backup frequency, minimum retention, vault lock, cross-region backup, cross-account backup, and more. Frameworks generate daily compliance reports stored in S3. Non-compliant resources are flagged with specific control failures."
---

## Phase 1 — Triage

MUST:
- List frameworks: `aws backup list-frameworks`
- Describe the framework: `aws backup describe-framework --framework-name <name>`
- Check framework deployment status (must be COMPLETED to generate reports)
- List report plans: `aws backup list-report-plans`
- Describe the report plan: `aws backup describe-report-plan --report-plan-name <name>`

SHOULD:
- Check the framework's controls and their parameters
- Verify the report delivery S3 bucket exists and has correct permissions
- Check the most recent report job: `aws backup list-report-jobs --by-report-plan-name <name> --max-results 5`
- Review non-compliant resources in the latest report

MAY:
- Check CloudTrail for framework modification events
- Verify the framework covers all required resource types
- Compare framework controls against organizational compliance requirements

## Phase 2 — Remediate

MUST:
- Fix non-compliant resources by updating backup plans, vault locks, or lifecycle rules
- Ensure the report delivery S3 bucket is accessible and has the correct bucket policy
- Verify framework deployment completed successfully

SHOULD:
- Create custom controls for organization-specific compliance requirements
- Set up SNS notifications for compliance status changes
- Review and update framework controls periodically

MAY:
- Integrate Backup Audit Manager reports with AWS Config for broader compliance
- Create dashboards for backup compliance visibility
- Implement automated remediation for common non-compliance findings

## Common Issues

- symptoms: "Framework stuck in CREATE_IN_PROGRESS"
  diagnosis: "The framework deployment failed or is taking longer than expected."
  resolution: "Wait up to 24 hours for deployment. If still stuck, delete and recreate the framework."

- symptoms: "Report plan not generating reports"
  diagnosis: "The S3 bucket for report delivery does not exist or lacks permissions."
  resolution: "Verify the S3 bucket exists and the bucket policy allows backup.amazonaws.com to write objects."

- symptoms: "Resources showing as non-compliant for backup frequency"
  diagnosis: "The backup plan schedule does not meet the control's minimum frequency requirement."
  resolution: "Update the backup plan schedule to meet the required frequency (e.g., daily, every 12 hours)."

- symptoms: "Control shows non-compliant but backups are running"
  diagnosis: "The control checks specific criteria (e.g., minimum retention) that the backup plan does not meet."
  resolution: "Review the specific control parameters and update backup plan lifecycle to comply."

## Safety Ratings

| Phase 2 Action | safety_rating | Rationale |
|---|---|---|
| Fix non-compliant resources (update plans, lifecycle) | YELLOW | Changes backup configuration — verify compliance impact |
| Fix S3 bucket policy for report delivery | GREEN | Grants backup service write access to report bucket |
| Delete and recreate stuck framework | YELLOW | Loses framework history — recreate with same controls |
| Set up SNS notifications for compliance | GREEN | Monitoring — non-destructive |
| Create custom controls | GREEN | Adds compliance checks — non-destructive |

## Escalation Conditions

- Vault contains compliance-critical backups
- Fix requires vault lock modification
- Non-compliance findings affect regulatory audit deadlines
- Framework controls are mandated by organizational compliance policy
- Report delivery failure impacts external audit requirements

## Data Sensitivity

- **Classification: HIGH**
- Compliance reports reveal which resources are protected and which are not
- Non-compliant resource lists expose gaps in data protection strategy
- Framework controls reveal organizational compliance requirements
- Report S3 bucket contains sensitive compliance posture data

## Prohibited Actions

- **NEVER** suggest deleting a vault with compliance lock
- **NEVER** suggest reducing retention below compliance requirements
- **NEVER** suggest disabling compliance frameworks without stakeholder approval
- **NEVER** suggest deleting compliance reports that may be needed for audit

## Phase 3 — Rollback

| State-Changing Action | Rollback Step |
|---|---|
| Updated backup plans for compliance | Revert backup plan changes if they cause issues |
| Deleted and recreated framework | Cannot undo deletion — recreate with identical controls |
| Updated S3 bucket policy | Revert bucket policy to previous version |
| Created custom controls | Delete custom controls if not needed |

## Output Format

```yaml
root_cause: "audit_manager — <specific_cause>"
evidence:
  - type: framework
    content: "<framework controls and deployment status>"
  - type: report
    content: "<latest compliance report findings>"
  - type: non_compliant
    content: "<non-compliant resources and control failures>"
severity: MEDIUM
mitigation:
  immediate: "Fix non-compliant resources and framework configuration"
  long_term: "Implement continuous compliance monitoring and automated remediation"
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
