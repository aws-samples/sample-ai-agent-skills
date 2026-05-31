---
title: "C1 — Related Resources"
description: "Diagnose related resource issues"
status: active
severity: LOW
triggers: ["related resources missing", "wrong resources", "resource link"]
owner: devops-agent
objective: "Resolve related resource issues"
context: "Related resources are ARN references attached to OpsItems for context. They are static references, not live links."
---

## Phase 1 — Triage

MUST:
- List related items: `aws ssm list-ops-item-related-items --ops-item-id <id>`
- Verify ARNs are valid and resources exist
- Check if resources were added at creation or later

SHOULD:
- Verify resource ARN format is correct
- Check if resources have been deleted since association

MAY:
- Query current resource state for each related resource

## Phase 2 — Remediate

MUST:
- Add missing related resources: `aws ssm associate-ops-item-related-item --ops-item-id <id> --association-type IsParentOf --resource-type AWS::SSM::OpsItem --resource-uri <arn>`
- Remove invalid references

SHOULD:
- Standardize related resource inclusion per OpsItem type

MAY:
- Automate related resource association via EventBridge

## Common Issues

- symptoms: "Related resource shows deleted instance"
  diagnosis: "Static reference — resource was deleted after association."
  resolution: "Remove stale reference. Related resources don't auto-update."

## Output Format

```yaml
root_cause: "related_resources — <specific_cause>"
evidence:
  - type: resource_list
    content: "<related resources and their current state>"
severity: LOW
mitigation:
  immediate: "Fix resource associations"
  long_term: "Automate related resource management"
```


## Safety Ratings
- Overall Risk: LOW
- Data Exposure Risk: LOW — related resources are reference associations
- Service Disruption Risk: LOW — resource associations do not affect running resources
- Compliance Impact: LOW — resource tracking is an operational concern

## Escalation Conditions
- Related resource associations fail to resolve
- Resource ARNs become invalid after resource deletion
- Cross-account resource associations fail
- Resource association limits exceeded
- Incorrect resource associations lead to wrong remediation actions

## Data Sensitivity
- Resource ARNs — INTERNAL
- Resource metadata — INTERNAL
- Cross-account resource references — INTERNAL
- Resource relationship mappings — INTERNAL
- Never expose resource ARNs in public documentation

## Prohibited Actions
- Do NOT associate resources from unauthorized accounts
- Do NOT ignore invalid resource associations
- Do NOT create circular resource associations
- Do NOT bypass resource association validation
- Do NOT associate sensitive resources without proper access controls
- Do NOT delete resource associations during active incident investigation

## Phase 3 — Rollback
- Remove incorrect resource associations
- Restore previous resource association configurations
- Re-validate resource ARNs after changes
- Verify resource associations are accurate after rollback
- Document resource association changes
- Update resource tracking procedures

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
