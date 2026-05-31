---
title: "C2 — Migration-Specific Issues"
description: "Diagnose issues with specific migration paths"
status: active
severity: MEDIUM
triggers:
  - "Java upgrade failed"
  - "migration incomplete"
  - "dependency conflicts after migration"
owner: devops-agent
objective: "Resolve migration-specific issues"
context: "Each migration path has specific requirements and known limitations."
---

## Phase 1 — Triage

MUST:
- Verify source and target versions
- Check for deprecated API usage
- Review dependency compatibility
- Check build after transformation

SHOULD:
- Run test suite after migration
- Check for breaking changes in target version
- Review migration summary

## Phase 2 — Remediate

MUST:
- Fix compilation errors after migration
- Update incompatible dependencies
- Address deprecated API usage

SHOULD:
- Run full test suite
- Review all changed files

## Common Issues

- symptoms: "Build fails after Java 8 to 17 migration"
  diagnosis: "Removed APIs or incompatible dependencies."
  resolution: "Update dependencies. Replace removed APIs with alternatives."

## Output Format

```yaml
root_cause: "migration — <specific_cause>"
evidence:
  - type: migration_status
    content: "<migration results>"
severity: MEDIUM
mitigation:
  immediate: "Fix migration issues"
  long_term: "Test thoroughly before deploying"
```


## Safety Ratings
- Overall Risk: HIGH
- Data Exposure Risk: MEDIUM — migration involves source code transformation
- Service Disruption Risk: HIGH — migration failures can break application functionality
- Compliance Impact: MEDIUM — migrated code must meet security standards

## Escalation Conditions
- Migration produces non-functional code
- Migration introduces security vulnerabilities
- Migration fails to complete
- Migration causes build failures
- Migration changes break existing tests

## Data Sensitivity
- Source code — CONFIDENTIAL
- Migration configurations — INTERNAL
- Transformation logs — INTERNAL
- Build artifacts — INTERNAL
- Never expose source code externally

## Prohibited Actions
- Do NOT apply migration changes without code review
- Do NOT migrate production code without backup
- Do NOT ignore migration validation failures
- Do NOT bypass testing after migration
- Do NOT accept migration results without security review
- Do NOT migrate without version control

## Phase 3 — Rollback
- Revert to pre-migration code from version control
- Restore previous build configurations
- Re-run tests to verify rollback
- Document migration issues and failures
- Update migration procedures
- Plan re-migration with corrected approach

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
