---
title: "B2 â€” Target Resolution"
description: "Diagnose target resolution failures at experiment runtime"
status: active
severity: HIGH
triggers: ["target resolution failed", "no resources found", "target error at runtime"]
owner: devops-agent
objective: "Resolve runtime target resolution failures"
context: "Target resolution happens when the experiment starts. Resources must exist and match filters at that time."
---

## Phase 1 â€” Triage

MUST:
- Check experiment status: `aws fis get-experiment --id <experiment-id>`
- Review action states for target resolution errors
- Verify target resources exist at experiment start time

SHOULD:
- Check if resources were terminated/modified between template creation and experiment start
- Verify resource state (running, available) matches action requirements

MAY:
- Check for auto-scaling events that may have changed resource IDs

## Phase 2 â€” Remediate

MUST:
- Ensure target resources exist and match filters before starting experiment
- Use tag-based filters for dynamic environments (auto-scaling)

SHOULD:
- Validate targets before experiment: run filter queries manually

MAY:
- Use resource ARNs for static environments

## Common Issues

- symptoms: "Experiment started but action shows FAILED â€” no targets"
  diagnosis: "Resources matching filters were terminated before experiment."
  resolution: "Verify resources exist. Use tag-based filters for dynamic environments."

## Output Format

```yaml
root_cause: "target_resolution â€” <specific_cause>"
evidence:
  - type: resolution_error
    content: "<target resolution failure details>"
severity: HIGH
mitigation:
  immediate: "Verify targets exist and retry"
  long_term: "Use tag-based targeting for dynamic environments"
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
