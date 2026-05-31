---
title: "D2 — Response Controls Issues"
description: "Diagnose response formatting and control problems"
status: active
severity: LOW
triggers:
  - "response too short"
  - "response format wrong"
  - "citations missing"
owner: devops-agent
objective: "Resolve response control issues"
context: "Response controls affect answer length, format, and citation behavior."
---

## Phase 1 — Triage

MUST:
- Check chat controls configuration
- Review response settings
- Test with queries expecting specific formats
- Verify citation configuration

SHOULD:
- Check for response length limits
- Review attribution settings

## Phase 2 — Remediate

MUST:
- Adjust response control settings
- Fix citation configuration

SHOULD:
- Test with diverse queries
- Document expected response behavior

## Common Issues

- symptoms: "Responses lack source citations"
  diagnosis: "Attribution settings not configured."
  resolution: "Enable attribution in chat controls configuration."

## Output Format

```yaml
root_cause: "response_controls — <specific_cause>"
evidence:
  - type: response_config
    content: "<response control settings>"
severity: LOW
mitigation:
  immediate: "Fix response settings"
  long_term: "Define response quality standards"
```


## Safety Ratings
- Overall Risk: HIGH
- Data Exposure Risk: HIGH — response controls prevent sensitive data exposure
- Service Disruption Risk: LOW — control issues affect response quality
- Compliance Impact: HIGH — response controls enforce data protection policies

## Escalation Conditions
- Response controls fail to filter sensitive content
- Controls too restrictive preventing useful responses
- Control configuration errors
- Controls not applied consistently
- Sensitive data exposed in responses

## Data Sensitivity
- Response control configurations — CONFIDENTIAL
- Filtering rules — CONFIDENTIAL
- Response evaluation logs — INTERNAL
- Sensitive data patterns — CONFIDENTIAL
- Never expose response control logic externally

## Prohibited Actions
- Do NOT disable response controls on production apps
- Do NOT create controls that allow sensitive data exposure
- Do NOT bypass response control review
- Do NOT ignore response control failures
- Do NOT share response control configurations
- Do NOT modify controls without security review

## Phase 3 — Rollback
- Restore previous response control configurations
- Re-enable disabled controls
- Verify control effectiveness after changes
- Restore data protection controls
- Document control changes and security impact
- Update response control procedures

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
