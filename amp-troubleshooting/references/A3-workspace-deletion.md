---
title: "A3 — Workspace Deletion and Recovery"
description: "Handle AMP workspace deletion issues and understand recovery limitations"
status: active
severity: CRITICAL
triggers:
  - "workspace deletion"
  - "delete workspace"
  - "recover workspace"
  - "accidental deletion"
  - "workspace missing"
owner: devops-agent
objective: "Manage workspace deletion safely and understand that deletion is permanent"
context: "AMP workspace deletion is permanent and irreversible. All metrics data, alert manager definitions, and ruler configurations are destroyed immediately. There is no recycle bin, snapshot, or recovery mechanism."
---

## Phase 1 — Triage

MUST:
- Verify workspace exists: `aws amp describe-workspace --workspace-id <workspace-id>`
- Check workspace status — if DELETING, it cannot be stopped: look at `status.statusCode`
- Check CloudTrail for DeleteWorkspace events: `aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventName,AttributeValue=DeleteWorkspace --max-results 10`
- Identify who initiated the deletion from CloudTrail event details

SHOULD:
- Check if the workspace was deleted by automation (Terraform destroy, CloudFormation stack deletion)
- Verify no other workspaces are affected
- Check if alert manager and ruler configurations were backed up

MAY:
- Review IAM policies to identify who has `aps:DeleteWorkspace` permission
- Check for organization SCPs that should prevent workspace deletion

## Phase 2 — Remediate

MUST:
- If workspace was accidentally deleted: it CANNOT be recovered. Create a new workspace.
- If deletion is stuck (status DELETING for extended period): contact AWS Support
- If preventing future accidental deletion: restrict `aps:DeleteWorkspace` via IAM policies

SHOULD:
- Export alert manager definition before deletion: `aws amp describe-alert-manager-definition --workspace-id <id>`
- Export ruler configurations before deletion: `aws amp list-rule-groups-namespaces --workspace-id <id>` then describe each
- Implement IAM policies that require MFA for workspace deletion
- Use resource tags and SCPs to protect production workspaces

MAY:
- Set up CloudTrail alarms for DeleteWorkspace API calls
- Implement a backup pipeline for ruler and alert manager configurations
- Use AWS Config rules to detect workspace deletion

## Common Issues

- symptoms: "Workspace not found — ResourceNotFoundException"
  diagnosis: "Workspace was deleted or never existed."
  resolution: "Check CloudTrail for DeleteWorkspace events. If deleted, create a new workspace — data cannot be recovered."

- symptoms: "Workspace stuck in DELETING status"
  diagnosis: "Backend cleanup is taking longer than expected."
  resolution: "Wait up to 10 minutes. If still DELETING, contact AWS Support."

- symptoms: "Accidental deletion by Terraform destroy"
  diagnosis: "Terraform lifecycle configuration did not prevent destruction."
  resolution: "Add lifecycle { prevent_destroy = true } to the Terraform resource. Create a new workspace and reconfigure collectors."

- symptoms: "Need to recover metrics data from deleted workspace"
  diagnosis: "Data recovery is NOT possible after workspace deletion."
  resolution: "Accept data loss. Implement proactive backups: export critical data to S3, back up ruler/alertmanager configs in version control."

## Safety Ratings

```
safety_ratings:
  - "describe-workspace: GREEN — read-only status check"
  - "describe-alert-manager-definition: GREEN — read-only config backup"
  - "list-rule-groups-namespaces: GREEN — read-only config backup"
  - "delete-workspace: RED — permanent, irreversible data destruction"
  - "restrict aps:DeleteWorkspace via IAM: YELLOW — changes IAM authorization"
```

## Escalation Conditions

- Workspace was accidentally deleted and data recovery is needed (not possible)
- Workspace deletion was initiated by unauthorized automation
- Production workspace is at risk of deletion due to IaC misconfiguration
- Workspace stuck in DELETING status for more than 10 minutes

## Data Sensitivity

- `aws amp describe-workspace`: LOW — workspace metadata
- Alert manager and ruler configurations: MEDIUM — contain alerting logic and SNS topic ARNs
- Workspace metrics data: HIGH — all ingested time series data (unrecoverable after deletion)
- CloudTrail DeleteWorkspace events: HIGH — reveals who deleted the workspace

## Prohibited Actions

- NEVER suggest deleting a workspace with active collectors writing to it
- NEVER suggest basic auth (only SigV4)
- NEVER delete a workspace without first exporting alert manager and ruler configurations
- NEVER assume workspace data can be recovered after deletion

## Phase 3 — Rollback

1. Workspace deletion is PERMANENT and IRREVERSIBLE — there is no rollback for data
2. If deletion was accidental, create a new workspace: `aws amp create-workspace --alias <alias>`
3. Restore alert manager definition from backup: `aws amp create-alert-manager-definition --workspace-id <new-id> --data file://alertmanager.yaml`
4. Restore ruler configurations from backup: `aws amp put-rule-groups-namespace --workspace-id <new-id> --name <namespace> --data file://rules.yaml`
5. Reconfigure all collectors to point to the new workspace ID
6. Add lifecycle protection: Terraform `prevent_destroy = true`, IAM deny policy on `aps:DeleteWorkspace`

## Output Format

```yaml
root_cause: "Workspace deletion — <specific_cause>"
evidence:
  - type: cloudtrail_event
    content: "<DeleteWorkspace event details>"
  - type: workspace_status
    content: "<current workspace status or not found>"
severity: CRITICAL
mitigation:
  immediate: "Create new workspace if needed; data from deleted workspace is unrecoverable"
  long_term: "Restrict aps:DeleteWorkspace via IAM, implement config backups in version control"
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
  - "NEVER suggest overly broad IAM permissions as a fix"
  - "NEVER suggest disabling security controls for troubleshooting"
  - "NEVER suggest deleting resources without confirmation"
