---
title: "C2 — 3D Rendering Issues"
description: "Diagnose 3D model rendering and performance problems"
status: active
severity: LOW
triggers: ["model not rendering", "slow scene", "texture missing"]
owner: devops-agent
objective: "Identify and resolve 3D rendering issues"
context: "Rendering issues occur when: model format unsupported, textures missing, model too large, or WebGL not supported."
---

## Phase 1 — Triage

MUST:
- Verify model format is glTF or GLB
- Check model file exists in S3 and is accessible
- Verify browser supports WebGL
- Check model file size

SHOULD:
- Check for missing textures or materials
- Verify model references in scene JSON
- Test with a simple model first

## Phase 2 — Remediate

MUST:
- Convert models to glTF/GLB format
- Upload missing textures to S3
- Use WebGL-compatible browser

SHOULD:
- Optimize models (reduce polygon count, compress textures)
- Test scenes with different model complexities

## Output Format

```yaml
root_cause: "3d_rendering — <specific_cause>"
evidence:
  - type: model_details
    content: "<model format and size>"
severity: LOW
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
- Never delete workspaces containing active digital twin entities
- Never modify component types that are referenced by production entities
- Never remove data connectors without confirming alternative data sources
- Never alter scene configurations used by active monitoring dashboards
- Never delete entity hierarchies without understanding downstream visualization impact

## Phase 3 — Rollback
- "Document current state before any changes"
- "If fix worsens the issue, revert to documented state"
- Restore previous component type definitions if entity data binding breaks
- Revert scene configuration changes if 3D visualization stops rendering
- Re-enable previous data connector settings if twin data synchronization fails

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
