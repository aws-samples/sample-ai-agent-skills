---
title: "C1 — Scene Loading Issues"
description: "Diagnose scene loading and display failures"
status: active
severity: MEDIUM
triggers: ["scene not loading", "3D model missing", "scene error"]
owner: devops-agent
objective: "Identify and resolve scene loading failures"
context: "Scene loading fails when: scene JSON invalid, 3D models missing from S3, S3 permissions wrong, or browser incompatible."
---

## Phase 1 — Triage

MUST:
- Check scene: `aws iottwinmaker get-scene --workspace-id <id> --scene-id <scene-id>`
- Verify scene content URL is accessible
- Check 3D model files exist in S3 bucket
- Verify workspace IAM role has S3 read access

SHOULD:
- Check browser console for JavaScript errors
- Verify 3D model format (glTF/GLB)
- Check model file sizes for performance

## Phase 2 — Remediate

MUST:
- Upload missing 3D models to workspace S3 bucket
- Fix scene JSON references to models
- Fix S3 permissions

SHOULD:
- Optimize 3D models for web rendering
- Test scenes in supported browsers

## Output Format

```yaml
root_cause: "scene_loading — <specific_cause>"
evidence:
  - type: scene_config
    content: "<scene details>"
severity: MEDIUM
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
