---
title: "F2 — Web Experience Customization"
description: "Diagnose web experience branding and customization issues"
status: active
severity: LOW
triggers:
  - "branding not applied"
  - "customization error"
  - "web experience appearance"
owner: devops-agent
objective: "Resolve web experience customization issues"
context: "Web experience supports title, subtitle, and welcome message customization."
---

## Phase 1 — Triage

MUST:
- Check web experience configuration
- Verify customization settings applied
- Test in browser after changes

SHOULD:
- Check for caching issues
- Verify customization API call succeeded

## Phase 2 — Remediate

MUST:
- Apply correct customization settings
- Clear browser cache after changes

SHOULD:
- Document branding requirements
- Test across browsers

## Common Issues

- symptoms: "Custom title not showing"
  diagnosis: "Browser caching old version."
  resolution: "Clear browser cache. Verify customization was saved."

## Output Format

```yaml
root_cause: "web_customization — <specific_cause>"
evidence:
  - type: customization
    content: "<customization settings>"
severity: LOW
mitigation:
  immediate: "Fix customization"
  long_term: "Document branding standards"
```


## Safety Ratings
- Overall Risk: LOW
- Data Exposure Risk: LOW — customization affects appearance, not data access
- Service Disruption Risk: LOW — customization issues affect user experience
- Compliance Impact: LOW — customization is an operational concern

## Escalation Conditions
- Customization changes break web experience functionality
- Customization causes rendering issues
- Customization changes not applied
- Multiple customization issues simultaneously
- Customization conflicts with accessibility requirements

## Data Sensitivity
- Customization configurations — INTERNAL
- Branding assets — INTERNAL
- UI configurations — INTERNAL
- User experience metrics — INTERNAL
- Never expose internal customization details

## Prohibited Actions
- Do NOT deploy untested customizations to production
- Do NOT bypass customization validation
- Do NOT create customizations that violate accessibility standards
- Do NOT ignore customization rendering issues
- Do NOT modify customizations during peak usage
- Do NOT share customization configurations externally

## Phase 3 — Rollback
- Restore previous customization settings
- Revert branding changes
- Verify web experience rendering after rollback
- Restore accessibility configurations
- Document customization issues
- Update customization procedures

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
