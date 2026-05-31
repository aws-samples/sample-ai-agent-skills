---
title: "A2 — Suggestion Quality Issues"
description: "Diagnose poor quality or irrelevant code suggestions"
status: active
severity: LOW
triggers:
  - "bad suggestions"
  - "irrelevant code"
  - "wrong suggestions"
owner: devops-agent
objective: "Improve code suggestion quality"
context: "Suggestion quality depends on context, language, and code patterns."
---

## Phase 1 — Triage

MUST:
- Check language support level for the file type
- Verify sufficient code context (imports, comments, function signatures)
- Check if workspace indexing is enabled
- Review suggestion acceptance rate

SHOULD:
- Add descriptive comments before requesting suggestions
- Open related files for better context
- Check if customization is available

MAY:
- Compare suggestions across different contexts
- Check for language-specific best practices

## Phase 2 — Remediate

MUST:
- Add context (comments, type hints, docstrings)
- Ensure related files are open in IDE

SHOULD:
- Enable workspace indexing
- Consider customization for domain-specific code

## Common Issues

- symptoms: "Suggestions are generic and not project-specific"
  diagnosis: "Insufficient project context."
  resolution: "Enable workspace indexing. Open related files. Add descriptive comments."

## Output Format

```yaml
root_cause: "suggestion_quality — <specific_cause>"
evidence:
  - type: context_analysis
    content: "<code context evaluation>"
severity: LOW
mitigation:
  immediate: "Improve code context"
  long_term: "Enable customization"
```


## Safety Ratings
- Overall Risk: LOW
- Data Exposure Risk: MEDIUM — quality analysis involves code context
- Service Disruption Risk: LOW — quality issues affect developer productivity
- Compliance Impact: MEDIUM — suggestion quality affects code security

## Escalation Conditions
- Suggestion quality drops significantly
- Suggestions contain security anti-patterns
- Suggestions not relevant to code context
- Suggestion acceptance rate drops significantly
- Suggestions produce non-functional code

## Data Sensitivity
- Code context — CONFIDENTIAL
- Quality metrics — INTERNAL
- Suggestion patterns — INTERNAL
- User feedback data — INTERNAL
- Never expose code context externally

## Prohibited Actions
- Do NOT accept low-quality suggestions without review
- Do NOT ignore security issues in suggestions
- Do NOT share suggestion quality data externally
- Do NOT bypass code review for AI suggestions
- Do NOT disable suggestion quality monitoring
- Do NOT use suggestions without understanding the code

## Phase 3 — Rollback
- Restore previous suggestion configurations
- Revert customization changes affecting quality
- Verify suggestion quality after changes
- Restore quality monitoring configurations
- Document quality issues and patterns
- Update suggestion usage guidelines

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
