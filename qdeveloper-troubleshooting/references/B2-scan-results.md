---
title: "B2 — Security Scan Results Issues"
description: "Diagnose scan result interpretation and false positives"
status: active
severity: LOW
triggers:
  - "false positive"
  - "scan result unclear"
  - "how to fix finding"
owner: devops-agent
objective: "Resolve scan result interpretation issues"
context: "Scan results include severity, description, and remediation guidance."
---

## Phase 1 — Triage

MUST:
- Review finding details and severity
- Check if finding is a true or false positive
- Verify the code pattern flagged
- Check remediation guidance

SHOULD:
- Cross-reference with other security tools
- Check if finding is in test code vs production code

## Phase 2 — Remediate

MUST:
- Fix true positive findings
- Document false positive rationale

SHOULD:
- Prioritize by severity
- Track finding trends over time

## Common Issues

- symptoms: "Many false positives in test files"
  diagnosis: "Scanner flags test patterns as vulnerabilities."
  resolution: "Review findings in context. Exclude test files if appropriate."

## Output Format

```yaml
root_cause: "scan_results — <specific_cause>"
evidence:
  - type: findings
    content: "<scan finding details>"
severity: LOW
mitigation:
  immediate: "Triage findings"
  long_term: "Establish finding review process"
```


## Safety Ratings
- Overall Risk: MEDIUM
- Data Exposure Risk: MEDIUM — scan results contain vulnerability details
- Service Disruption Risk: LOW — scan result issues do not affect development
- Compliance Impact: HIGH — scan results are required for security compliance

## Escalation Conditions
- Scan results not being generated
- Results contain false positives affecting development velocity
- Critical vulnerabilities found in production code
- Scan results not accessible to authorized users
- Results inconsistent between scans

## Data Sensitivity
- Vulnerability details — CONFIDENTIAL
- Affected code locations — CONFIDENTIAL
- Remediation recommendations — INTERNAL
- Scan history — INTERNAL
- Never expose vulnerability details to unauthorized parties

## Prohibited Actions
- Do NOT ignore critical scan findings
- Do NOT share scan results with unauthorized parties
- Do NOT dismiss findings without proper review
- Do NOT deploy code with unresolved critical findings
- Do NOT delete scan results without retention compliance
- Do NOT bypass scan result review processes

## Phase 3 — Rollback
- Restore previous scan configurations
- Re-run scans if results were lost
- Verify scan result accuracy after changes
- Restore scan result access configurations
- Document scan result issues
- Update scan result management procedures

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
