---
title: "C2 — Rate Control Issues"
description: "Diagnose SSM Automation rate control execution failures including concurrency and error threshold problems"
status: active
severity: HIGH
triggers: ["rate control failed", "MaxConcurrency", "MaxErrors exceeded", "multi-target automation", "too many failures"]
owner: devops-agent
objective: "Resolve rate control execution issues for multi-target automations"
context: "Rate control executes automation documents against multiple targets concurrently. MaxConcurrency controls parallelism, MaxErrors controls failure tolerance. Steps within each target run sequentially."
---

## Phase 1 — Triage

MUST:
- Get execution details: `aws ssm get-automation-execution --automation-execution-id <execution-id>`
- Check target execution status: `aws ssm describe-automation-executions --filters Key=ParentExecutionId,Values=<parent-execution-id>`
- Review MaxConcurrency and MaxErrors settings in the execution

SHOULD:
- Identify which targets failed: check child execution statuses
- Verify target list is valid (instance IDs, resource groups, tags)
- Check if MaxErrors threshold was exceeded

MAY:
- Review individual target execution failures for patterns
- Check resource group membership: `aws resource-groups list-group-resources --group-name <group-name>`

## Phase 2 — Remediate

MUST:
- Fix target-specific failures (permissions, instance state, etc.)
- Adjust MaxErrors if legitimate transient failures cause premature stopping
- Verify all targets are valid and accessible

SHOULD:
- Set MaxConcurrency to a safe value for production (e.g., 10% or small number)
- Set MaxErrors to tolerate expected transient failures
- Use tags or resource groups for dynamic target selection

MAY:
- Implement pre-validation steps to check target health before execution
- Use separate executions for different target groups

## Safety Ratings

```
safety_ratings:
  - "describe-automation-executions: GREEN — read-only execution listing"
  - "start-automation-execution with rate control: RED — modifies multiple targets concurrently"
  - "stop-automation-execution: RED — cancels multi-target execution, may leave targets in mixed state"
  - "resource-groups list-group-resources: GREEN — read-only resource listing"
```

## Escalation Conditions

- Rate control execution modifies production instances and some targets are in unknown state
- MaxErrors threshold exceeded on critical infrastructure
- Concurrency too high causing API throttling or resource contention
- Target list includes unintended resources due to tag or resource group misconfiguration

## Data Sensitivity

- `aws ssm describe-automation-executions`: LOW — returns execution metadata
- `aws ssm get-automation-execution`: MEDIUM — contains target list and parameters
- `aws resource-groups list-group-resources`: LOW — returns resource identifiers
- `aws ec2 describe-instances`: LOW — returns instance metadata

## Prohibited Actions

- NEVER set MaxConcurrency to 100% on production targets without staged rollout testing
- NEVER set MaxErrors to the total target count — this disables failure protection
- NEVER re-run a failed rate control execution without understanding which targets already changed
- NEVER use broad tag filters that may include unintended targets

## Phase 3 — Rollback

1. Stop the parent execution if still running: `aws ssm stop-automation-execution --automation-execution-id <parent-id> --type Cancel`
2. Identify which child executions completed, failed, or were cancelled
3. For completed targets, determine if changes need reverting
4. Run a targeted rollback automation against only the affected targets
5. Verify all targets are in a consistent state after rollback

## Common Issues

- symptoms: "Rate control execution stops after a few targets with MaxErrors exceeded"
  diagnosis: "More targets failed than the MaxErrors threshold allows."
  resolution: "Investigate individual target failures. Fix root cause. Increase MaxErrors if failures are transient."

- symptoms: "Execution is very slow with many targets"
  diagnosis: "MaxConcurrency is set too low for the number of targets."
  resolution: "Increase MaxConcurrency gradually. Use percentage-based values for large target sets."

- symptoms: "Some targets skipped without error"
  diagnosis: "Targets may not match the selection criteria or instances are not in running state."
  resolution: "Verify target selection (tags, resource groups, instance IDs). Ensure instances are running and have SSM agent."

## Output Format

```yaml
root_cause: "rate_control — <specific_cause>"
evidence:
  - type: rate_control_error
    content: "<MaxErrors/MaxConcurrency details and target failure summary>"
severity: HIGH
mitigation:
  immediate: "Fix target failures and adjust thresholds"
  long_term: "Implement staged rollout with pre-validation"
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
