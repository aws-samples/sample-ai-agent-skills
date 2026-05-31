---
title: "B2 — Custom Frameworks"
description: "Diagnose custom framework creation and configuration issues"
status: active
severity: MEDIUM
triggers: ["custom framework error", "create framework", "framework controls"]
owner: devops-agent
objective: "Resolve custom framework issues"
context: "Custom frameworks allow combining controls from multiple sources. They can include standard controls and custom controls."
---

## Phase 1 — Triage

MUST:
- List custom frameworks: `aws auditmanager list-assessment-frameworks --frameworkType Custom`
- Check framework details and control sets
- Verify custom controls are properly configured

SHOULD:
- Check if custom controls have valid data sources
- Verify control mapping to evidence sources

MAY:
- Compare with standard frameworks for completeness

## Phase 2 — Remediate

MUST:
- Fix control configuration issues
- Ensure custom controls have valid evidence sources

SHOULD:
- Test custom framework with a small assessment first

MAY:
- Share custom frameworks across accounts via Organizations

## Common Issues

- symptoms: "Custom control shows no evidence"
  diagnosis: "Custom control data source not configured or service not enabled."
  resolution: "Configure data source (Config rule, CloudTrail, manual) for the custom control."

## Output Format

```yaml
root_cause: "custom_frameworks — <specific_cause>"
evidence:
  - type: framework_config
    content: "<custom framework and control configuration>"
severity: MEDIUM
mitigation:
  immediate: "Fix framework configuration"
  long_term: "Validate custom frameworks before production use"
```


## Safety Ratings

| Phase 2 Action | safety_rating | Rationale |
|---|---|---|
| Fix control configuration issues | YELLOW | Modifies compliance controls — affects evidence collection |
| Ensure custom controls have valid evidence sources | YELLOW | Changes evidence collection — may affect compliance posture |
| Test custom framework with small assessment | GREEN | Non-production testing — non-destructive |
| Share custom frameworks across accounts | YELLOW | Cross-account change — review before sharing |

## Escalation Conditions

- Fix requires modifying compliance framework used by auditors
- Assessment covers production accounts with active compliance requirements
- Custom control changes affect evidence collection for active assessments
- Framework sharing affects compliance posture across multiple accounts

## Data Sensitivity

- Custom framework configuration reveals organization-specific compliance requirements
- Control data sources reveal which AWS services are monitored for compliance
- Evidence source configuration reveals audit methodology

## Prohibited Actions

- NEVER suggest deleting assessments to fix compliance gaps — this destroys audit evidence
- NEVER remove controls from custom frameworks without auditor approval
- NEVER modify custom frameworks used in active assessments without documentation

## Phase 3 — Rollback

- If custom controls were modified: restore previous control configuration
- If framework was shared incorrectly: revoke sharing from unintended accounts
- If evidence sources were changed: restore original data source configuration
- Document all framework changes for compliance audit trail

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
