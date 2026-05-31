---
title: "C2 â€” Action Execution"
description: "Diagnose action execution failures"
status: active
severity: HIGH
triggers: ["action failed", "action not executing", "action error"]
owner: devops-agent
objective: "Resolve action execution failures"
context: "Actions may fail due to IAM permissions, resource state, or service-specific issues."
---

## Phase 1 â€” Triage

MUST:
- Check experiment action states: `aws fis get-experiment --id <id> --query 'experiment.actions'`
- Review action failure reason
- Verify IAM role has permissions for the specific action

SHOULD:
- Check target resource state (is it in a state that allows the action?)
- Review CloudTrail for the underlying API call failures

MAY:
- Test the underlying API call manually to verify permissions

## Phase 2 â€” Remediate

MUST:
- Add missing IAM permissions to the FIS role
- Ensure target resources are in valid state for the action

SHOULD:
- Test with a single resource before scaling up
- Add specific resource-level permissions

MAY:
- Use IAM policy simulator to validate permissions

## Common Issues

- symptoms: "Action FAILED with AccessDenied"
  diagnosis: "FIS role missing permission for the underlying API call."
  resolution: "Add the required permission (e.g., ec2:StopInstances) to the FIS role."

## Output Format

```yaml
root_cause: "action_execution â€” <specific_cause>"
evidence:
  - type: action_state
    content: "<action failure reason>"
severity: HIGH
mitigation:
  immediate: "Fix IAM or resource state issue"
  long_term: "Maintain comprehensive FIS role permissions"
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
