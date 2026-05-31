---
title: "A2 — Application Configuration Issues"
description: "Diagnose application settings and configuration problems"
status: active
severity: MEDIUM
triggers:
  - "app configuration error"
  - "cannot update application"
  - "settings not applied"
owner: devops-agent
objective: "Resolve application configuration issues"
context: "Application configuration includes identity, encryption, and chat controls."
---

## Phase 1 — Triage

MUST:
- Get app config: `aws qbusiness get-application --application-id <app-id>`
- Check chat controls: `aws qbusiness get-chat-controls-configuration --application-id <app-id>`
- Verify IAM role permissions
- Check for configuration validation errors

SHOULD:
- Review encryption settings
- Check application status
- Verify index configuration

## Phase 2 — Remediate

MUST:
- Fix configuration validation errors
- Update IAM role if permissions changed
- Apply correct settings

SHOULD:
- Document configuration changes
- Test after configuration updates

## Common Issues

- symptoms: "UpdateApplication returns ValidationException"
  diagnosis: "Invalid configuration parameter."
  resolution: "Check parameter values against API documentation."

## Output Format

```yaml
root_cause: "app_config — <specific_cause>"
evidence:
  - type: config_details
    content: "<application configuration>"
severity: MEDIUM
mitigation:
  immediate: "Fix configuration"
  long_term: "Version control configurations"
```


## Safety Ratings
- Overall Risk: MEDIUM
- Data Exposure Risk: HIGH — configuration changes affect data access controls
- Service Disruption Risk: MEDIUM — misconfigurations can disrupt app functionality
- Compliance Impact: HIGH — configuration affects data privacy and access compliance

## Escalation Conditions
- Configuration changes break app functionality
- Data access controls modified without authorization
- Encryption settings changed
- Configuration drift from approved baseline
- Multiple configuration issues simultaneously

## Data Sensitivity
- App configuration parameters — INTERNAL
- Data access control settings — CONFIDENTIAL
- Encryption configurations — CONFIDENTIAL
- User access settings — CONFIDENTIAL
- Never expose app configuration details externally

## Prohibited Actions
- Do NOT modify data access controls without authorization
- Do NOT disable encryption
- Do NOT change Identity Center settings without testing
- Do NOT bypass configuration change management
- Do NOT remove data access restrictions
- Do NOT modify configurations during active usage without approval

## Phase 3 — Rollback
- Restore previous app configurations
- Revert data access control changes
- Restore encryption settings
- Verify app functionality after rollback
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
