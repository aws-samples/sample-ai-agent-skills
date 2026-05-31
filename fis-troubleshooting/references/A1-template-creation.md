---
title: "A1 â€” Template Creation"
description: "Diagnose experiment template creation failures"
status: active
severity: HIGH
triggers: ["template creation failed", "cannot create template", "template error"]
owner: devops-agent
objective: "Resolve experiment template creation issues"
context: "Experiment templates define actions, targets, stop conditions, and IAM role. Creation failures are typically due to invalid action IDs, target configurations, or IAM issues."
---

## Phase 1 â€” Triage

MUST:
- Verify action IDs are valid: `aws fis list-actions --query 'actions[*].id'`
- Check IAM role exists and has correct trust policy for fis.amazonaws.com
- Verify target resource types match action requirements

SHOULD:
- Validate stop condition alarm ARNs exist
- Check service quotas for template limits

MAY:
- Review existing templates for reference: `aws fis list-experiment-templates`

## Phase 2 â€” Remediate

MUST:
- Use valid action IDs from the list-actions output
- Create IAM role with fis.amazonaws.com trust policy
- Ensure target configuration matches action requirements

SHOULD:
- Start with simple single-action templates
- Add stop conditions for safety

MAY:
- Use CloudFormation for template management

## Common Issues

- symptoms: "ValidationException on create-experiment-template"
  diagnosis: "Invalid action ID or target configuration."
  resolution: "Verify action ID exists and target resourceType matches action requirements."

## Output Format

```yaml
root_cause: "template_creation â€” <specific_cause>"
evidence:
  - type: template_config
    content: "<template configuration and error>"
severity: HIGH
mitigation:
  immediate: "Fix template configuration"
  long_term: "Use IaC for template management"
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
