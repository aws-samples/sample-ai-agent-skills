---
title: "Z1 — General AWS Backup Troubleshooting (Catch-All)"
description: "Fallback SOP for AWS Backup issues that do not match any specific runbook"
status: active
severity: MEDIUM
triggers:
  - ".*"
owner: devops-agent
objective: "Systematically investigate an unknown AWS Backup issue, classify the failure domain, and match to an existing SOP or escalate"
context: "This SOP is invoked when symptoms don't match any of the specific runbooks. It provides a broad, methodical investigation that narrows the failure domain step by step."
---

## Phase 1 — Triage

MUST:
- List recent backup jobs (all states): `aws backup list-backup-jobs --max-results 20`
- List failed backup jobs: `aws backup list-backup-jobs --by-state FAILED --max-results 10`
- List failed restore jobs: `aws backup list-restore-jobs --by-status FAILED --max-results 10`
- List backup vaults: `aws backup list-backup-vaults`
- Check region settings: `aws backup describe-region-settings`
- Check CloudWatch backup metrics: `aws cloudwatch get-metric-statistics --namespace AWS/Backup --metric-name NumberOfBackupJobsFailed --start-time <start> --end-time <end> --period 3600 --statistics Sum`

SHOULD:
- List backup plans: `aws backup list-backup-plans`
- Check recent CloudTrail events: `aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventSource,AttributeValue=backup.amazonaws.com --max-results 20`
- List copy jobs: `aws backup list-copy-jobs --max-results 10`
- Check for any frameworks: `aws backup list-frameworks`

## Phase 2 — Classify

Based on triage results, classify into a failure domain:
- Backup job failures / errors → Backup Jobs (A1-A4)
- Restore operation failures → Restore (B1-B3)
- Plan or schedule issues → Backup Plans (C1-C3)
- Vault access or lock issues → Vault (D1-D3)
- Cross-account problems → Cross-Account (E1-E2)
- Service-specific failures → Service-Specific (F1-F3)
- KMS or IAM errors → Security (G1-G2)
- Compliance or audit issues → Compliance (H1-H2)

If classified: switch to the specific SOP immediately.
If unclassified: continue to Phase 3.

## Phase 3 — Deep Investigation

MUST:
- Describe specific failed jobs for detailed error messages
- Check IAM roles used by backup operations: `aws iam get-role --role-name AWSBackupDefaultServiceRole`
- Verify KMS key status for backup vaults
- Check for recent configuration changes in CloudTrail

SHOULD:
- Check AWS Health Dashboard for AWS Backup service events
- Verify supported services list against the resources being backed up
- Check for vault lock configurations that may restrict operations
- Review backup plan selections for resource targeting issues

MAY:
- Check Organizations backup policies if using centralized management
- Verify network connectivity for Backup Gateway (VMware) operations
- Review service quotas for backup-related limits

## Phase 4 — Report

MUST:
- State the investigation path taken
- State root cause if identified, or "unclassified" with best hypothesis
- List all evidence collected
- Recommend next steps

## Safety Ratings

| Phase 2 Action | safety_rating | Rationale |
|---|---|---|
| Describe and list backup/restore jobs | GREEN | Read-only investigation |
| Check CloudWatch metrics and CloudTrail events | GREEN | Read-only investigation |
| List backup plans and vaults | GREEN | Read-only investigation |
| Pivot to specific SOP | YELLOW | Specific SOP may include state-changing actions |
| Check Organizations backup policies | GREEN | Read-only investigation |

## Escalation Conditions

- Vault contains compliance-critical backups
- Fix requires vault lock modification
- Issue affects multiple backup vaults or plans simultaneously
- Root cause cannot be classified after systematic investigation
- Issue impacts compliance reporting or audit requirements

## Data Sensitivity

- **Classification: HIGH**
- Backup job listings reveal protected resource inventory
- Vault configurations expose encryption and access policies
- CloudTrail events may contain sensitive resource identifiers
- Region settings reveal organizational backup strategy

## Prohibited Actions

- **NEVER** suggest deleting a vault with compliance lock
- **NEVER** suggest reducing retention below compliance requirements
- **NEVER** suggest bulk-deleting recovery points during investigation
- **NEVER** suggest modifying vault lock policies without understanding compliance implications

## Phase 3 — Rollback

| State-Changing Action | Rollback Step |
|---|---|
| General triage is read-only | No rollback needed for Phase 1 investigation |
| Pivot to specific SOP | Follow rollback steps in the specific SOP |

## Output Format

```yaml
root_cause: "<identified_cause OR unclassified>"
failure_domain: "<backup_jobs|restore|plans|vault|cross_account|service_specific|security|compliance|unknown>"
investigation_path: "backup jobs → restore jobs → vaults → plans → <domain_classification>"
evidence:
  - type: backup_jobs
    content: "<backup job status summary>"
  - type: restore_jobs
    content: "<restore job status summary>"
  - type: vaults
    content: "<vault configuration summary>"
  - type: cloudtrail
    content: "<relevant events>"
severity: MEDIUM
mitigation:
  immediate: "<specific action if root cause found, or escalate>"
  long_term: "Implement monitoring for the identified failure pattern"
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
