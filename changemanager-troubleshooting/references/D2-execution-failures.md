---
title: "D2 — Execution Failures"
description: "Diagnose change execution failures and recovery"
status: active
severity: CRITICAL
triggers: ["change failed", "partial change", "need rollback"]
owner: devops-agent
objective: "Recover from change execution failures"
context: "Failed changes may leave infrastructure in partial state. Recovery requires understanding what completed and what failed."
---

## Phase 1 — Triage

MUST:
- Check which steps completed vs failed: `aws ssm describe-automation-step-executions --automation-execution-id <id>`
- Assess current infrastructure state
- Determine if partial changes were applied

SHOULD:
- Check if rollback steps exist in the runbook
- Verify affected resources are still accessible

MAY:
- Compare current state with pre-change baseline

## Phase 2 — Remediate

MUST:
- If rollback steps exist: execute them manually if automation failed
- If no rollback: manually revert completed steps
- Document what was changed and what needs reverting

SHOULD:
- Add rollback steps to the runbook for future use
- Create incident record for the failed change

MAY:
- Implement automated rollback triggers

## Common Issues

- symptoms: "Change partially applied — some resources updated, others not"
  diagnosis: "Runbook failed mid-execution without rollback."
  resolution: "Manually complete or revert the change. Add rollback logic to runbook."

## Output Format

```yaml
root_cause: "execution_failure — <specific_cause>"
evidence:
  - type: execution_state
    content: "<completed vs failed steps>"
severity: CRITICAL
mitigation:
  immediate: "Recover from partial change"
  long_term: "Add rollback logic to all change runbooks"
```


## Safety Ratings

| Phase 2 Action | safety_rating |
|---|---|
| If rollback steps exist: execute them manually if automation failed | RED |
| If no rollback: manually revert completed steps | RED |
| Document what was changed and what needs reverting | GREEN |
| Add rollback steps to the runbook for future use | GREEN |
| Create incident record for the failed change | GREEN |
| Implement automated rollback triggers | YELLOW |

## Escalation Conditions

- Fix requires bypassing approval workflow
- Change affects production infrastructure

## Data Sensitivity

- **MEDIUM**: change request details, approval workflows
- Execution failure details may reveal infrastructure state and partial configuration changes

## Prohibited Actions

- NEVER suggest bypassing approval workflows

## Phase 3 — Rollback

1. Execute rollback steps in reverse order of the completed automation steps
2. If manual rollback is required, document each step taken for audit compliance
3. Verify each resource returns to its pre-change state before proceeding to the next
4. If rollback fails, escalate immediately and freeze further changes to affected resources
5. Create a post-incident review documenting the failure and rollback process

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
