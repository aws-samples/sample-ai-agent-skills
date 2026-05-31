---
title: "E2 — Plugin Execution Errors"
description: "Diagnose plugin runtime errors during conversations"
status: active
severity: MEDIUM
triggers:
  - "plugin action failed"
  - "plugin timeout"
  - "external service error"
owner: devops-agent
objective: "Resolve plugin execution errors"
context: "Plugin actions invoke external services during chat. Failures affect conversation flow."
---

## Phase 1 — Triage

MUST:
- Check plugin status
- Review conversation logs for plugin errors
- Verify external service availability
- Check OAuth token validity

SHOULD:
- Test external service API directly
- Check for rate limiting on external service
- Verify plugin action permissions

## Phase 2 — Remediate

MUST:
- Fix external service connectivity
- Refresh OAuth tokens
- Fix plugin configuration

SHOULD:
- Add error handling for plugin failures
- Monitor plugin success rate

## Common Issues

- symptoms: "Plugin action returns error to user"
  diagnosis: "External service API error or token expired."
  resolution: "Check external service status. Refresh OAuth tokens."

## Output Format

```yaml
root_cause: "plugin_execution — <specific_cause>"
evidence:
  - type: plugin_error
    content: "<plugin execution error>"
severity: MEDIUM
mitigation:
  immediate: "Fix plugin connectivity"
  long_term: "Monitor plugin health"
```


## Safety Ratings
- Overall Risk: MEDIUM
- Data Exposure Risk: MEDIUM — plugin execution accesses external services
- Service Disruption Risk: MEDIUM — execution failures affect app functionality
- Compliance Impact: MEDIUM — plugin actions must comply with data policies

## Escalation Conditions
- Plugin execution fails consistently
- Plugin returns incorrect or unauthorized data
- Plugin execution causes performance degradation
- Plugin credentials expire during execution
- Plugin execution bypasses access controls

## Data Sensitivity
- Plugin execution results — INTERNAL (may contain CONFIDENTIAL data)
- External service responses — CONFIDENTIAL
- Execution logs — INTERNAL
- Error details — INTERNAL
- Never expose external service data in logs

## Prohibited Actions
- Do NOT ignore plugin execution failures
- Do NOT bypass plugin access controls
- Do NOT allow plugins to access unauthorized data
- Do NOT share plugin execution results externally
- Do NOT disable plugin monitoring
- Do NOT modify plugin execution parameters without testing

## Phase 3 — Rollback
- Disable failing plugins
- Restore previous plugin configurations
- Rotate plugin credentials
- Verify app functionality after plugin changes
- Document execution issues
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
