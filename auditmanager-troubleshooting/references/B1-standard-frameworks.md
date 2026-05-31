---
title: "B1 — Standard Frameworks"
description: "Diagnose standard framework issues"
status: active
severity: MEDIUM
triggers: ["framework not available", "CIS framework", "PCI framework"]
owner: devops-agent
objective: "Resolve standard framework issues"
context: "Standard frameworks (CIS, PCI DSS, SOC 2, HIPAA) are AWS-provided and read-only. They define controls for specific compliance standards."
---

## Phase 1 — Triage

MUST:
- List available frameworks: `aws auditmanager list-assessment-frameworks --frameworkType Standard`
- Verify desired framework is available in the region
- Check framework details: `aws auditmanager get-assessment-framework --framework-id <id>`

SHOULD:
- Verify framework version is current
- Check if framework controls match your compliance requirements

MAY:
- Compare framework controls with your compliance checklist

## Phase 2 — Remediate

MUST:
- Use available standard frameworks as-is (cannot modify)
- If customization needed, create custom framework based on standard

SHOULD:
- Document which standard framework maps to your compliance needs

MAY:
- Create custom framework combining controls from multiple standards

## Common Issues

- symptoms: "Standard framework missing controls I need"
  diagnosis: "Standard frameworks are fixed — cannot add controls."
  resolution: "Create a custom framework with additional controls."

## Output Format

```yaml
root_cause: "standard_frameworks — <specific_cause>"
evidence:
  - type: framework_details
    content: "<framework availability and controls>"
severity: MEDIUM
mitigation:
  immediate: "Use appropriate framework"
  long_term: "Create custom frameworks for specific needs"
```


## Safety Ratings

| Phase 2 Action | safety_rating | Rationale |
|---|---|---|
| Use available standard frameworks as-is | GREEN | Read-only — standard frameworks cannot be modified |
| Create custom framework based on standard | YELLOW | Creates new framework — review controls before use |
| Document framework mapping | GREEN | Documentation only |

## Escalation Conditions

- Fix requires modifying compliance framework used by auditors
- Assessment covers production accounts with active compliance requirements
- Framework selection affects compliance certification scope
- Custom framework creation requires auditor validation

## Data Sensitivity

- Framework selection reveals applicable compliance standards
- Control mappings reveal compliance requirements and gaps
- Framework details may indicate regulated data types (PCI cardholder data, HIPAA PHI)

## Prohibited Actions

- NEVER suggest deleting assessments to fix compliance gaps — this destroys audit evidence
- NEVER suggest ignoring standard framework controls without documented risk acceptance
- NEVER create custom frameworks that weaken standard compliance requirements

## Phase 3 — Rollback

- Standard frameworks are read-only — no rollback needed
- If custom framework was created incorrectly: delete the custom framework before it is used in assessments
- Document all framework decisions for compliance audit trail

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
