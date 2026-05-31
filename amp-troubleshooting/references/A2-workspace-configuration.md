---
title: "A2 — Workspace Configuration Issues"
description: "Diagnose and resolve AMP workspace configuration problems"
status: active
severity: MEDIUM
triggers:
  - "workspace configuration"
  - "workspace settings"
  - "logging configuration"
  - "workspace tags"
owner: devops-agent
objective: "Identify and fix AMP workspace configuration issues"
context: "AMP workspace configuration includes logging settings, tags, and alias. Configuration issues can affect observability of the workspace itself and organizational compliance."
---

## Phase 1 — Triage

MUST:
- Describe workspace configuration: `aws amp describe-workspace --workspace-id <workspace-id>`
- Check workspace logging configuration: `aws amp describe-logging-configuration --workspace-id <workspace-id>`
- List workspace tags: `aws amp list-tags-for-resource --resource-arn arn:aws:aps:<region>:<account>:workspace/<workspace-id>`
- Verify workspace status is ACTIVE: check `status.statusCode` in describe output

SHOULD:
- Check CloudTrail for recent UpdateWorkspaceAlias or TagResource events
- Verify logging destination (CloudWatch log group) exists and has correct permissions
- Compare configuration against organizational standards

MAY:
- Check if workspace alias conflicts with naming conventions
- Review tag compliance against organization tag policies

## Phase 2 — Remediate

MUST:
- If logging not configured: `aws amp create-logging-configuration --workspace-id <workspace-id> --log-group-arn arn:aws:logs:<region>:<account>:log-group:<log-group>`
- If logging destination missing: create the CloudWatch log group first
- If tags incorrect: `aws amp tag-resource --resource-arn <workspace-arn> --tags <key>=<value>`

SHOULD:
- Enable logging for production workspaces to capture AMP operational events
- Ensure the log group has appropriate retention settings
- Apply consistent tags across all workspaces

MAY:
- Update workspace alias for clarity: `aws amp update-workspace-alias --workspace-id <workspace-id> --alias <new-alias>`
- Set up CloudWatch alarms on AMP log group for error detection

## Common Issues

- symptoms: "Logging configuration returns 'ResourceNotFoundException'"
  diagnosis: "No logging configuration exists for this workspace."
  resolution: "Create logging configuration: aws amp create-logging-configuration --workspace-id <id> --log-group-arn <arn>"

- symptoms: "Logging configuration shows UPDATE_FAILED status"
  diagnosis: "CloudWatch log group does not exist or AMP lacks permission to write to it."
  resolution: "Verify log group exists. Ensure AMP service has logs:CreateLogStream and logs:PutLogEvents permissions via resource policy."

- symptoms: "Tags not appearing on workspace"
  diagnosis: "TagResource call failed silently or tags were overwritten."
  resolution: "Re-apply tags with aws amp tag-resource. Check CloudTrail for TagResource/UntagResource events."

## Safety Ratings

```
safety_ratings:
  - "describe-workspace: GREEN — read-only status check"
  - "describe-logging-configuration: GREEN — read-only check"
  - "list-tags-for-resource: GREEN — read-only tag check"
  - "create-logging-configuration: YELLOW — creates CloudWatch log group integration"
  - "update-workspace-alias: GREEN — metadata change only"
```

## Escalation Conditions

- Logging configuration fails due to CloudWatch log group permissions
- Workspace configuration changes needed in a production environment
- Tag compliance violations require organizational policy changes
- Logging destination missing and needs cross-team CloudWatch setup

## Data Sensitivity

- `aws amp describe-workspace`: LOW — workspace metadata
- `aws amp describe-logging-configuration`: LOW — logging configuration details
- CloudWatch log group contents: MEDIUM — contains AMP operational events

## Prohibited Actions

- NEVER suggest deleting a workspace with active collectors writing to it
- NEVER suggest basic auth (only SigV4)
- NEVER delete logging configuration on production workspaces
- NEVER modify workspace tags without checking organizational tag policies

## Phase 3 — Rollback

1. If logging configuration was created incorrectly, delete it: `aws amp delete-logging-configuration --workspace-id <workspace-id>`
2. If tags were changed incorrectly, restore original tags: `aws amp tag-resource --resource-arn <arn> --tags <original-tags>`
3. If workspace alias was changed, restore it: `aws amp update-workspace-alias --workspace-id <id> --alias <original-alias>`
4. Verify logging is functioning correctly after any changes
5. Document configuration changes for audit trail

## Output Format

```yaml
root_cause: "Workspace configuration — <specific_cause>"
evidence:
  - type: workspace_config
    content: "<workspace configuration details>"
  - type: logging_config
    content: "<logging configuration status>"
severity: MEDIUM
mitigation:
  immediate: "Fix the identified configuration issue"
  long_term: "Use IaC to manage workspace configuration consistently"
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
