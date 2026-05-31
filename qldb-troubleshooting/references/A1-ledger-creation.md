---
title: "A1 — Ledger Creation Failures"
description: "Diagnose QLDB ledger creation failures"
status: active
severity: HIGH
triggers: ["ledger creation failed", "CREATING stuck", "permissions error"]
owner: devops-agent
objective: "Identify and resolve ledger creation failures"
context: "Creation fails when: IAM permissions insufficient, KMS key invalid, account limits reached, or region not supported."
---

## Phase 1 — Triage

MUST:
- Check ledger status: `aws qldb describe-ledger --name <name>`
- Verify IAM permissions for qldb:CreateLedger
- Check KMS key if using customer-managed: `aws kms describe-key --key-id <id>`
- Verify account limits for ledgers

SHOULD:
- Check CloudTrail for creation errors
- Verify region supports QLDB
- Check if ledger name is unique

## Phase 2 — Remediate

MUST:
- Grant qldb:CreateLedger permission
- Fix KMS key permissions
- Wait for ACTIVE status

SHOULD:
- Enable deletion protection (default)
- Document ledger configuration

## Output Format

```yaml
root_cause: "ledger_creation — <specific_cause>"
evidence:
  - type: ledger_status
    content: "<ledger details>"
severity: HIGH
```


## Safety Ratings
- Overall Risk: HIGH
- Data Exposure Risk: LOW — creation failures do not expose data
- Service Disruption Risk: LOW — creation failures do not affect existing ledgers
- Compliance Impact: HIGH — ledger configuration affects data integrity and audit compliance

## Escalation Conditions
- Ledger creation fails repeatedly
- Creation blocked by IAM or SCP policies
- Encryption configuration fails during creation
- Deletion protection not enabled on production ledgers
- Service quota limits prevent ledger creation

## Data Sensitivity
- Ledger configurations — INTERNAL
- KMS key configurations — CONFIDENTIAL
- IAM role configurations — INTERNAL
- Deletion protection settings — INTERNAL
- Never expose ledger configuration details externally

## Prohibited Actions
- Do NOT create production ledgers without deletion protection
- Do NOT create ledgers without encryption
- Do NOT use overly permissive IAM roles for ledger access
- Do NOT create ledgers without audit logging
- Do NOT share ledger admin credentials
- Do NOT bypass ledger creation validation

## Phase 3 — Rollback
- Delete failed ledger resources
- Clean up orphaned IAM roles
- Revert KMS key configurations
- Document creation failure reasons before retrying
- Verify no residual resources remain
- Update ledger creation procedures

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
