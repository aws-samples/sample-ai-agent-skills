---
title: "D1 — IDE Integration Issues"
description: "Diagnose IDE plugin installation and connectivity"
status: active
severity: HIGH
triggers:
  - "plugin not installing"
  - "IDE integration broken"
  - "plugin crash"
owner: devops-agent
objective: "Resolve IDE integration issues"
context: "Q Developer supports VS Code, JetBrains, Visual Studio, and Cloud9."
---

## Phase 1 — Triage

MUST:
- Verify IDE is supported
- Check plugin installation status
- Verify plugin version is current
- Check IDE logs for plugin errors

SHOULD:
- Check for conflicting extensions
- Verify IDE version compatibility
- Check system requirements (memory, disk)

## Phase 2 — Remediate

MUST:
- Install or update plugin
- Fix IDE compatibility issues
- Resolve extension conflicts

SHOULD:
- Keep plugin updated
- Report persistent bugs

## Common Issues

- symptoms: "Plugin fails to activate in VS Code"
  diagnosis: "VS Code version too old or conflicting extension."
  resolution: "Update VS Code. Disable conflicting extensions. Reinstall plugin."

## Output Format

```yaml
root_cause: "ide_integration — <specific_cause>"
evidence:
  - type: plugin_status
    content: "<IDE and plugin details>"
severity: HIGH
mitigation:
  immediate: "Fix plugin installation"
  long_term: "Keep IDE and plugin updated"
```


## Safety Ratings
- Overall Risk: LOW
- Data Exposure Risk: LOW — IDE integration is a development tool concern
- Service Disruption Risk: LOW — integration issues affect developer productivity
- Compliance Impact: LOW — IDE integration is an operational concern

## Escalation Conditions
- IDE integration completely fails
- Plugin crashes IDE repeatedly
- Authentication with Q Developer service fails
- Integration causes IDE performance degradation
- Multiple developers affected simultaneously

## Data Sensitivity
- IDE configurations — INTERNAL
- Authentication tokens — RESTRICTED
- Plugin settings — INTERNAL
- Usage telemetry — INTERNAL
- Never expose authentication tokens

## Prohibited Actions
- Do NOT share IDE authentication tokens between developers
- Do NOT install IDE plugins from untrusted sources
- Do NOT bypass IDE security settings
- Do NOT ignore IDE integration errors
- Do NOT disable IDE security features
- Do NOT share IDE configurations containing credentials

## Phase 3 — Rollback
- Restore previous IDE plugin version
- Revert IDE configuration changes
- Re-authenticate with Q Developer service
- Verify IDE functionality after changes
- Document integration issues
- Update IDE setup procedures

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
