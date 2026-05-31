---
title: "D2 — Plugin Configuration Issues"
description: "Diagnose plugin settings and configuration problems"
status: active
severity: MEDIUM
triggers:
  - "plugin settings wrong"
  - "features disabled"
  - "proxy configuration"
owner: devops-agent
objective: "Resolve plugin configuration issues"
context: "Plugin settings control features, proxy, and behavior."
---

## Phase 1 — Triage

MUST:
- Check plugin settings in IDE
- Verify proxy configuration if behind firewall
- Check feature toggles (suggestions, security scan)
- Verify authentication method

SHOULD:
- Check for organization-level policies
- Verify reference tracking settings

## Phase 2 — Remediate

MUST:
- Configure proxy settings
- Enable required features
- Fix authentication settings

SHOULD:
- Document standard configuration
- Distribute settings via IDE profiles

## Common Issues

- symptoms: "Suggestions work but security scan is disabled"
  diagnosis: "Security scan feature toggle is off."
  resolution: "Enable security scanning in plugin settings."

## Output Format

```yaml
root_cause: "plugin_config — <specific_cause>"
evidence:
  - type: settings
    content: "<plugin configuration>"
severity: MEDIUM
mitigation:
  immediate: "Fix plugin settings"
  long_term: "Standardize configuration"
```


## Safety Ratings
- Overall Risk: LOW
- Data Exposure Risk: LOW — plugin issues are development tool concerns
- Service Disruption Risk: LOW — plugin issues affect developer productivity
- Compliance Impact: LOW — plugin management is an operational concern

## Escalation Conditions
- Plugin crashes IDE repeatedly
- Plugin update causes compatibility issues
- Plugin authentication fails
- Plugin causes data loss in IDE
- Multiple developers affected by same plugin issue

## Data Sensitivity
- Plugin configurations — INTERNAL
- Plugin logs — INTERNAL
- IDE workspace data — INTERNAL
- Authentication tokens — RESTRICTED
- Never expose plugin authentication tokens

## Prohibited Actions
- Do NOT install plugins from untrusted sources
- Do NOT share plugin authentication tokens
- Do NOT ignore plugin crash patterns
- Do NOT bypass plugin update procedures
- Do NOT disable plugin security features
- Do NOT modify plugin internals

## Phase 3 — Rollback
- Restore previous plugin version
- Revert plugin configuration changes
- Clear plugin cache if corrupted
- Verify IDE functionality after plugin changes
- Document plugin issues
- Update plugin management procedures

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
