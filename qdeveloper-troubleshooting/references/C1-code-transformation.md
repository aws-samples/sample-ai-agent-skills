---
title: "C1 — Code Transformation Failures"
description: "Diagnose code transformation and migration errors"
status: active
severity: MEDIUM
triggers:
  - "transformation failed"
  - "migration error"
  - "upgrade failed"
owner: devops-agent
objective: "Resolve code transformation failures"
context: "Code transformation supports specific language/framework migrations. Results need review."
---

## Phase 1 — Triage

MUST:
- Verify transformation type is supported
- Check source code compatibility
- Review transformation error messages
- Verify project structure requirements

SHOULD:
- Check for unsupported code patterns
- Verify build system compatibility
- Review transformation logs

## Phase 2 — Remediate

MUST:
- Fix unsupported code patterns
- Ensure project meets transformation requirements
- Retry transformation

SHOULD:
- Review all transformed code manually
- Run tests after transformation

## Common Issues

- symptoms: "Transformation fails with unsupported pattern"
  diagnosis: "Code uses patterns not supported by transformation."
  resolution: "Manually update unsupported patterns. Retry transformation."

## Output Format

```yaml
root_cause: "code_transformation — <specific_cause>"
evidence:
  - type: transform_status
    content: "<transformation error details>"
severity: MEDIUM
mitigation:
  immediate: "Fix unsupported patterns"
  long_term: "Review transformation results thoroughly"
```


## Safety Ratings
- Overall Risk: HIGH
- Data Exposure Risk: MEDIUM — code transformation processes source code
- Service Disruption Risk: HIGH — transformation failures can break applications
- Compliance Impact: MEDIUM — transformed code must meet security standards

## Escalation Conditions
- Code transformation produces non-functional code
- Transformation introduces security vulnerabilities
- Transformation fails to complete
- Transformation changes break existing functionality
- Transformation causes build failures

## Data Sensitivity
- Source code — CONFIDENTIAL
- Transformation configurations — INTERNAL
- Transformation logs — INTERNAL
- Build artifacts — INTERNAL
- Never expose source code externally

## Prohibited Actions
- Do NOT apply transformations without code review
- Do NOT transform production code without backup
- Do NOT ignore transformation validation failures
- Do NOT bypass testing after transformation
- Do NOT accept transformation results without security review
- Do NOT transform without version control

## Phase 3 — Rollback
- Revert to pre-transformation code from version control
- Restore previous build configurations
- Re-run tests to verify rollback
- Document transformation issues
- Update transformation procedures
- Plan re-transformation with corrected approach

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
