---
title: "B2 — Component Conflicts"
description: "Diagnose component version and dependency conflicts"
status: active
severity: HIGH
triggers:
  - "version conflict"
  - "dependency error"
  - "incompatible components"
  - "merge conflict"
owner: devops-agent
objective: "Identify and resolve component version and dependency conflicts"
context: "Conflicts occur when: multiple deployments specify different versions, circular dependencies exist, or version constraints are incompatible."
---

## Phase 1 — Triage

MUST:
- List all deployments targeting device: `aws greengrassv2 list-effective-deployments --core-device-thing-name <name>`
- Check component versions: `aws greengrassv2 list-installed-components --core-device-thing-name <name>`
- Review deployment merge results in nucleus logs
- Check component recipe dependencies

SHOULD:
- Compare version constraints across deployments
- Verify no circular dependencies in component graph

MAY:
- List all component versions: `aws greengrassv2 list-component-versions --arn <component-arn>`

## Phase 2 — Remediate

MUST:
- Align component versions across deployments
- Fix circular dependencies by restructuring components
- Use version ranges (>=, <) instead of exact versions where possible

SHOULD:
- Consolidate deployments to reduce merge conflicts
- Document component dependency graph

## Output Format

```yaml
root_cause: "component_conflict — <specific_cause>"
evidence:
  - type: deployment_merge
    content: "<conflicting versions>"
severity: HIGH
mitigation:
  immediate: "Align component versions"
  long_term: "Implement component versioning strategy"
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
- Never remove core device provisioning without confirming all edge workloads are migrated
- Never force-deploy components to production core devices without staging validation
- Never delete component versions that are actively deployed to edge devices
- Never modify MQTT bridge configurations on devices running critical edge processing
- Never cancel OTA updates that are partially applied to production devices

## Phase 3 — Rollback
- "Document current state before any changes"
- "If fix worsens the issue, revert to documented state"
- Trigger a rollback deployment to restore previous component versions on core devices
- Restore previous MQTT bridge and topic mapping configurations if messaging breaks
- Revert nucleus configuration changes if core device connectivity is lost

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
