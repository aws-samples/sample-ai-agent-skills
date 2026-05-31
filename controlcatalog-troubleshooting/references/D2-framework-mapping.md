---
title: "D2 — Framework Mapping"
description: "Diagnose compliance framework mapping issues"
status: active
severity: MEDIUM
triggers: ["framework mapping", "compliance mapping", "which controls for PCI"]
owner: devops-agent
objective: "Map controls to compliance frameworks"
context: "Common controls map to compliance frameworks (CIS, PCI DSS, SOC 2, etc.). Mapping helps identify relevant controls for compliance."
---

## Phase 1 — Triage

MUST:
- List common controls and their framework mappings
- Identify which frameworks are relevant to your compliance needs
- Check if all required framework controls are available

SHOULD:
- Compare catalog mappings with framework requirements
- Identify gaps in catalog coverage

MAY:
- Cross-reference with Audit Manager frameworks

## Phase 2 — Remediate

MUST:
- Map available controls to framework requirements
- Identify controls not in catalog that need manual implementation

SHOULD:
- Document mapping between catalog controls and framework requirements
- Plan implementation for mapped controls

MAY:
- Use Audit Manager for comprehensive framework assessment

## Common Issues

- symptoms: "Not all PCI DSS requirements have mapped controls"
  diagnosis: "Catalog may not cover all framework requirements."
  resolution: "Use catalog for available mappings. Supplement with Config rules and manual controls for gaps."

## Output Format

```yaml
root_cause: "framework_mapping — <specific_cause>"
evidence:
  - type: mapping_analysis
    content: "<framework mapping coverage>"
severity: MEDIUM
mitigation:
  immediate: "Complete framework mapping"
  long_term: "Maintain compliance mapping documentation"
```


---

## Safety Ratings

| Dimension | Rating |
|-----------|--------|
| Data Sensitivity | LOW — Framework mappings are public reference data |
| Blast Radius | LOW — Read-only analysis |
| Rollback Complexity | LOW — No changes to roll back |
| Compliance Scope | CRITICAL — Framework mapping directly affects compliance posture |

## Escalation Conditions

- Escalate IMMEDIATELY to compliance if framework mapping gaps are discovered during an active audit
- Escalate to security architects if mapping gaps require manual control implementation
- Escalate to management if compliance gaps require budget for additional tooling

## Data Sensitivity

- **Framework mapping analysis** may reveal compliance gaps — treat as confidential
- **Gap analysis results** should be shared only with authorized compliance personnel

## PROHIBITED Actions

- Do NOT present catalog mappings as complete compliance coverage to auditors without gap analysis
- Do NOT ignore unmapped framework requirements — they need manual controls
- Do NOT share compliance gap analysis outside the authorized compliance team

## Phase 3 — Rollback

- No rollback needed — read-only analysis
- If compliance implementations were based on incorrect mappings, reassess and remediate gaps
- Update compliance documentation to reflect corrected mappings

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
