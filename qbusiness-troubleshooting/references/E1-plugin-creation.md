---
title: "E1 — Plugin Creation Issues"
description: "Diagnose plugin creation and configuration failures"
status: active
severity: MEDIUM
triggers:
  - "plugin creation failed"
  - "cannot add plugin"
  - "plugin configuration error"
owner: devops-agent
objective: "Resolve plugin creation issues"
context: "Plugins extend Q Business with external service actions. Each requires OAuth/API configuration."
---

## Phase 1 — Triage

MUST:
- List plugins: `aws qbusiness list-plugins --application-id <app-id>`
- Check plugin status and type
- Verify OAuth credentials in Secrets Manager
- Check plugin API endpoint

SHOULD:
- Verify OAuth scopes
- Test external service connectivity
- Check IAM permissions for plugin

## Phase 2 — Remediate

MUST:
- Fix OAuth credentials
- Correct API endpoint configuration
- Ensure proper scopes

SHOULD:
- Test plugin independently
- Document plugin requirements

## Common Issues

- symptoms: "Plugin stuck in CREATION_FAILED"
  diagnosis: "OAuth credentials invalid or endpoint unreachable."
  resolution: "Verify OAuth app in external service. Update credentials."

## Output Format

```yaml
root_cause: "plugin_creation — <specific_cause>"
evidence:
  - type: plugin_status
    content: "<plugin configuration>"
severity: MEDIUM
mitigation:
  immediate: "Fix plugin configuration"
  long_term: "Document plugin setup procedures"
```


## Safety Ratings
- Overall Risk: MEDIUM
- Data Exposure Risk: MEDIUM — plugins may access external services
- Service Disruption Risk: LOW — plugin creation does not affect existing functionality
- Compliance Impact: MEDIUM — plugins must comply with data access policies

## Escalation Conditions
- Plugin creation fails with validation errors
- Plugin accesses unauthorized external services
- Plugin authentication configuration fails
- Plugin causes app instability
- Multiple plugin creations fail

## Data Sensitivity
- Plugin configurations — INTERNAL
- External service credentials — RESTRICTED
- API endpoint details — INTERNAL
- Plugin execution logs — INTERNAL
- Never expose plugin credentials

## Prohibited Actions
- Do NOT create plugins that access unauthorized services
- Do NOT store plugin credentials in plaintext
- Do NOT bypass plugin validation
- Do NOT create plugins without security review
- Do NOT share plugin configurations externally
- Do NOT deploy untested plugins to production

## Phase 3 — Rollback
- Delete incorrectly created plugins
- Restore previous plugin configurations
- Rotate plugin credentials if compromised
- Verify app functionality after plugin changes
- Document plugin issues
- Update plugin creation procedures

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
