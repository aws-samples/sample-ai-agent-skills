---
title: "A2 — Hierarchy Issues"
description: "Diagnose asset hierarchy configuration problems"
status: active
severity: MEDIUM
triggers:
  - "hierarchy error"
  - "parent-child failed"
  - "cannot associate assets"
  - "hierarchy depth"
owner: devops-agent
objective: "Identify and resolve asset hierarchy issues"
context: "Hierarchy issues occur when: depth limit exceeded, wrong model types, circular references, or association conflicts."
---

## Phase 1 — Triage

MUST:
- Check model hierarchy definitions: `aws iotsitewise describe-asset-model --asset-model-id <id>`
- Verify child model matches hierarchy definition
- Check current hierarchy depth
- List asset associations: `aws iotsitewise list-associated-assets --asset-id <id> --hierarchy-id <hierarchy-id>`

SHOULD:
- Verify asset is not already a child in another hierarchy of the same type
- Check for circular references in hierarchy chain

MAY:
- Map out full hierarchy tree for visualization
- Check metric aggregation across hierarchy levels

## Phase 2 — Remediate

MUST:
- Fix model hierarchy definitions to reference correct child models
- Flatten hierarchy if depth limit exceeded
- Remove circular references

SHOULD:
- Document hierarchy design before implementation
- Use consistent naming for hierarchy levels

## Output Format

```yaml
root_cause: "hierarchy_issue — <specific_cause>"
evidence:
  - type: hierarchy_config
    content: "<hierarchy definition>"
severity: MEDIUM
mitigation:
  immediate: "Fix hierarchy definition"
  long_term: "Design hierarchies within depth limits"
```

## Safety Ratings
```
safety_ratings:
  - "Triage commands (describe/list/get): GREEN — read-only"
  - "Configuration changes: YELLOW — recoverable"
  - "Delete/terminate operations: RED — irreversible"
```

## Escalation Conditions
- "Resource serves production workload"
- "Fix requires deleting or recreating resources"
- "Fix involves security or encryption changes"
- "Blast radius affects multiple dependent services"

## Data Sensitivity
- HIGH: credentials, PII, security configurations
- MEDIUM: resource configurations, metrics
- LOW: status information, error messages

## Prohibited Actions
- Never delete asset models that have active assets associated with them
- Never modify measurement properties on assets receiving live industrial data
- Never remove gateway configurations without confirming data source migration
- Never alter transform or metric computations on production assets without testing
- Never delete portal or dashboard configurations used by operations teams

## Phase 3 — Rollback
- "Document current state before any changes"
- "If fix worsens the issue, revert to documented state"
- Restore previous asset model configuration if property computations break
- Revert gateway data source settings if industrial data ingestion stops
- Re-enable previous alarm configurations if monitoring coverage is lost

## Escalation Conditions

escalation_conditions:
  - "Remediation requires modifying IAM policies in a production account"
  - "Remediation requires disabling a security control even temporarily"
  - "Root cause cannot be identified after 3 hypothesis pivots"
  - "Blast radius affects more than one account or region"
  - "Issue involves potential data loss or exposure"

## Data Sensitivity

data_sensitivity:
  - command: "describe-instances"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"
  - command: "get-console-output"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"
  - command: "ssm send-command"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"

## Prohibited Actions

prohibited_actions:
  - "NEVER suggest 0.0.0.0/0 inbound security group rules as a fix"
  - "NEVER suggest disabling instance metadata service"
  - "NEVER terminate instances without confirmation"
