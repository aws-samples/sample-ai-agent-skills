---
title: "F1 â€” Experiment Failures"
description: "Diagnose experiment execution failures"
status: active
severity: HIGH
triggers: ["experiment failed", "experiment error", "experiment not completing"]
owner: devops-agent
objective: "Resolve experiment execution failures"
context: "Experiments can fail due to target resolution, action execution, IAM, or stop condition triggers. Check experiment state and action states for details."
---

## Phase 1 â€” Triage

MUST:
- Get experiment details: `aws fis get-experiment --id <id>`
- Check experiment state (initiating, running, completed, stopped, failed)
- Review individual action states and failure reasons

SHOULD:
- Check if stopped by stop condition vs actual failure
- Review CloudTrail events during experiment timeframe

MAY:
- Compare with previous successful experiments

## Phase 2 â€” Remediate

MUST:
- Address specific failure reason (IAM, targets, actions)
- Fix template configuration if needed
- Retry experiment after fixes

SHOULD:
- Start with simpler experiment to isolate the issue
- Verify all prerequisites before retry

MAY:
- Add logging/monitoring for experiment execution

## Common Issues

- symptoms: "Experiment state: failed, all actions: failed"
  diagnosis: "Likely IAM role issue or target resolution failure."
  resolution: "Check role permissions and target filters. Fix and retry."

## Output Format

```yaml
root_cause: "experiment_failure â€” <specific_cause>"
evidence:
  - type: experiment_state
    content: "<experiment and action states>"
severity: HIGH
mitigation:
  immediate: "Fix failure cause and retry"
  long_term: "Implement pre-flight checks before experiments"
```

---

## Safety Ratings

| Aspect | Rating |
|--------|--------|
| Service Impact | CRITICAL — FIS intentionally injects failures into AWS resources |
| Data Risk | HIGH — Experiments can cause data loss if stop conditions are misconfigured |
| Recovery Complexity | VARIES — Depends on the action type; some actions auto-recover, others require manual intervention |
| Blast Radius | CRITICAL — Misconfigured targets or missing stop conditions can affect production workloads |

## Escalation Conditions

- Experiment running without stop conditions in a production environment
- Stop condition alarm not triggering despite service degradation
- Experiment affecting more resources than intended due to target filter misconfiguration
- Action causing data loss or corruption (e.g., EBS volume actions, RDS failover)
- Experiment stuck in running state and cannot be stopped
- IAM role granting broader permissions than required for the experiment scope

## Data Sensitivity

- Experiment templates reveal infrastructure architecture and failure testing strategy
- Target resource filters expose production resource tagging and grouping patterns
- Stop condition configurations reveal operational thresholds and alarm boundaries
- Experiment logs may contain resource identifiers and failure injection details
- IAM role policies reveal which services and actions are in scope for chaos testing

## PROHIBITED Actions

- NEVER suggest running chaos experiments in production without stop conditions configured
- NEVER create experiment templates targeting all resources without specific filters
- NEVER use overly broad IAM roles that grant permissions beyond experiment requirements
- NEVER run experiments during maintenance windows or known high-traffic periods without approval
- NEVER disable or remove stop conditions from running experiments
- NEVER target stateful resources (databases, file systems) without verified backup and recovery procedures

## Phase 3 — Rollback

- If experiment caused unintended impact: stop experiment immediately via ws fis stop-experiment
- If stop condition failed to trigger: manually stop experiment and investigate alarm configuration
- If resources were terminated: verify auto-scaling recovery or manually replace affected instances
- If network disruption persists after experiment: verify security group and NACL rules are restored
- If database failover caused issues: verify primary instance health and replication status
- Document all rollback actions with experiment ID, affected resources, and recovery verification steps
## Safety Ratings

safety_ratings:
  - "Phase 1 triage commands (describe/get/list): GREEN â€” read-only"
  - "Phase 2 configuration changes: YELLOW â€” state-changing but recoverable"
  - "Phase 2 resource deletion or security changes: RED â€” destructive or irreversible"

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
