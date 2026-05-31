---
title: "D2 — Dedup Configuration"
description: "Diagnose deduplication configuration issues"
status: active
severity: MEDIUM
triggers: ["dedup config", "dedup string", "configure deduplication"]
owner: devops-agent
objective: "Configure proper deduplication"
context: "Dedup strings must be set in operational data. Different sources (EventBridge, API, manual) require different configuration approaches."
---

## Phase 1 — Triage

MUST:
- Review how OpsItems are created (manual, EventBridge, API)
- Check if dedup strings are included in creation calls
- Verify dedup string format and consistency

SHOULD:
- Review EventBridge rule input transformers for dedup configuration
- Check API calls for operational data inclusion

MAY:
- Audit existing OpsItems for dedup string patterns

## Phase 2 — Remediate

MUST:
- For EventBridge: add /aws/dedup in input transformer
- For API: include operational data with dedup key in CreateOpsItem calls

SHOULD:
- Use deterministic dedup strings (e.g., resource-id + issue-type)
- Document dedup string conventions

MAY:
- Create helper Lambda for consistent dedup string generation

## Common Issues

- symptoms: "Dedup works for some OpsItems but not others"
  diagnosis: "Inconsistent dedup string format across sources."
  resolution: "Standardize dedup string format across all OpsItem creation sources."

## Output Format

```yaml
root_cause: "dedup_configuration — <specific_cause>"
evidence:
  - type: dedup_setup
    content: "<dedup configuration across sources>"
severity: MEDIUM
mitigation:
  immediate: "Fix dedup configuration"
  long_term: "Standardize dedup across all sources"
```


## Safety Ratings
- Overall Risk: LOW
- Data Exposure Risk: LOW — configuration changes are operational
- Service Disruption Risk: LOW — configuration changes do not affect resources
- Compliance Impact: LOW — dedup configuration is an operational concern

## Escalation Conditions
- Configuration changes cause deduplication to stop working
- Configuration update causes incorrect OpsItem merging
- Configuration syntax errors prevent rule evaluation
- Multiple configuration changes conflict with each other
- Configuration rollback fails

## Data Sensitivity
- Deduplication configuration rules — INTERNAL
- Matching criteria and keys — INTERNAL
- Configuration change history — INTERNAL
- Rule evaluation logs — INTERNAL
- Never expose configuration details that reveal infrastructure patterns

## Prohibited Actions
- Do NOT apply untested deduplication configurations to production
- Do NOT remove all deduplication rules simultaneously
- Do NOT create conflicting deduplication rules
- Do NOT modify configurations during high-volume incident periods
- Do NOT bypass configuration validation
- Do NOT ignore configuration change audit logs

## Phase 3 — Rollback
- Restore previous deduplication configuration
- Revert rule changes that caused incorrect behavior
- Re-validate configuration after rollback
- Verify deduplication functions correctly with restored configuration
- Document configuration changes and their impact
- Update configuration management procedures

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
