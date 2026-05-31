---
title: "C2 — OS Patching Visibility"
description: "Diagnose OS patching visibility issues in Fleet Manager"
status: active
severity: MEDIUM
triggers: ["patch status not showing", "patching visibility missing", "no patch compliance data"]
owner: devops-agent
objective: "Resolve OS patching visibility issues in Fleet Manager"
context: "Fleet Manager shows patch compliance data from Patch Manager. Visibility requires at least one patch scan or install operation. Patch baselines define what patches are tracked."
---

## Phase 1 — Triage

MUST:
- Check patch compliance: `aws ssm describe-instance-patches --instance-id <instance-id> --query 'Patches[*].{KB:KBId,State:State,Severity:Severity}'`
- Verify patch baseline association: `aws ssm describe-effective-patches-for-patch-baseline --baseline-id <baseline-id>`
- Check if a patch scan has been run: `aws ssm list-commands --instance-id <instance-id> --filters Key=DocumentName,Values=AWS-RunPatchBaseline`

SHOULD:
- Verify the patch baseline is appropriate for the OS
- Check Patch Manager maintenance window configuration

MAY:
- Review patch operation logs on the instance
- Check Patch Manager compliance summary: `aws ssm list-compliance-summaries --filters Key=ComplianceType,Values=Patch`

## Phase 2 — Remediate

MUST:
- Run a patch scan if none has been executed: `aws ssm send-command --instance-ids <instance-id> --document-name AWS-RunPatchBaseline --parameters Operation=Scan`
- Ensure a patch baseline is associated with the instance (via patch group tag or default baseline)
- Verify SSM Agent can execute patch operations

SHOULD:
- Configure a maintenance window for regular patch scanning
- Tag instances with appropriate patch group tags

MAY:
- Create custom patch baselines for specific OS requirements
- Configure patch compliance reporting to S3

## Common Issues

- symptoms: "Fleet Manager shows no patch data for an instance"
  diagnosis: "No patch scan has been run on the instance."
  resolution: "Run a patch scan: aws ssm send-command --document-name AWS-RunPatchBaseline --parameters Operation=Scan."

- symptoms: "Patch compliance shows all patches as 'NOT_APPLICABLE'"
  diagnosis: "Patch baseline does not match the instance OS or version."
  resolution: "Verify patch baseline OS matches instance OS. Create a custom baseline if needed."

## Safety Ratings

```
safety_ratings:
  - "describe-instance-patches: GREEN — read-only patch status query"
  - "send-command Scan: GREEN — scans for patches without installing"
  - "send-command Install: RED — installs patches, may require reboot"
  - "list-compliance-summaries: GREEN — read-only compliance data"
```

## Escalation Conditions

- Patch scan fails repeatedly with errors on specific instances
- Patch compliance data is inconsistent between Fleet Manager and Patch Manager
- Custom patch baselines are not being applied correctly
- Compliance reporting is required for audit and data is missing

## Data Sensitivity

- `aws ssm describe-instance-patches`: LOW — patch metadata and compliance status
- `aws ssm list-commands`: MEDIUM — reveals command history and parameters
- `aws ssm send-command Scan`: LOW — read-only scan operation
- `aws ssm send-command Install`: HIGH — modifies system by installing patches

## Prohibited Actions

- NEVER run patch Install operations on production instances without change management approval
- NEVER modify default patch baselines without understanding the impact on all associated instances
- NEVER skip patch scanning to assume compliance
- NEVER disable patch compliance reporting to hide non-compliant instances

## Phase 3 — Rollback

1. If a patch scan produced incorrect results, re-run with correct baseline
2. If patch baseline was changed incorrectly, revert to previous baseline ID
3. If maintenance window was misconfigured, update the schedule
4. Verify patch compliance data is accurate after remediation
5. Document any baseline or maintenance window changes

## Output Format

```yaml
root_cause: "patching_visibility — <specific_cause>"
evidence:
  - type: patch_error
    content: "<error details>"
severity: MEDIUM
mitigation:
  immediate: "Run patch scan to populate data"
  long_term: "Configure regular patch scanning via maintenance window"
```

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
  - "NEVER suggest overly broad SSM document permissions"
  - "NEVER suggest disabling session logging"
  - "NEVER suggest sharing SSM documents publicly"
