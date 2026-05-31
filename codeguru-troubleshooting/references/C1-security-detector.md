---
title: "C1 — Security Detector Issues"
description: "Diagnose security detector configuration and findings"
status: active
severity: MEDIUM
triggers:
  - "security findings missing"
  - "security detector not working"
  - "no security recommendations"
owner: devops-agent
objective: "Resolve security detector issues"
context: "Security detector finds vulnerabilities and secrets in code."
---

## Phase 1 — Triage

MUST:
- Verify security detector is enabled on association
- Check code review includes security findings
- Verify language is supported for security scanning
- Review recent security findings

SHOULD:
- Check for suppressed security findings
- Verify scan coverage

## Phase 2 — Remediate

MUST:
- Enable security detector if disabled
- Ensure supported languages
- Review and act on findings

SHOULD:
- Configure security finding notifications
- Track finding remediation

## Common Issues

- symptoms: "No security findings on code with known vulnerabilities"
  diagnosis: "Security detector not enabled or language not supported."
  resolution: "Enable security detector. Verify language support."

## Output Format

```yaml
root_cause: "security_detector — <specific_cause>"
evidence:
  - type: security_findings
    content: "<security detector status>"
severity: MEDIUM
mitigation:
  immediate: "Enable security detector"
  long_term: "Integrate into security workflow"
```


## Safety Ratings

| Phase 2 Action | safety_rating |
|---|---|
| Enable security detector if disabled | GREEN |
| Ensure supported languages | GREEN |
| Review and act on findings | GREEN |
| Configure security finding notifications | GREEN |
| Track finding remediation | GREEN |

## Escalation Conditions

- Fix requires enabling security scanning on production repositories
- Critical security findings require immediate remediation

## Data Sensitivity

- **HIGH**: security findings expose vulnerabilities and secrets in code
- Detector results reveal the security posture of the codebase

## Prohibited Actions

- NEVER suggest suppressing all security findings

## Phase 3 — Rollback

1. If security detector was enabled, it can be disabled if causing issues (not recommended)
2. If notification configuration was changed, restore previous settings
3. Security findings should not be rolled back — they should be triaged and addressed
4. Verify rollback by confirming security detector configuration matches expected state

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
