---
title: "E1 — Validation Errors"
description: "Diagnose policy validation errors"
status: active
severity: MEDIUM
triggers: ["validation error", "policy check failed", "validate-policy error"]
owner: devops-agent
objective: "Resolve policy validation errors"
context: "Policy validation checks syntax, grammar, and best practices. Errors indicate structural issues in IAM policies."
---

## Phase 1 — Triage

MUST:
- Run validation: `aws accessanalyzer validate-policy --policy-document file://policy.json --policy-type IDENTITY_POLICY`
- Review validation findings (ERROR, WARNING, SUGGESTION)
- Check specific finding codes and messages

SHOULD:
- Verify policy JSON is well-formed
- Check for deprecated actions or resource formats

MAY:
- Compare with known-good policies

## Phase 2 — Remediate

MUST:
- Fix ERROR-level findings (these prevent policy from working)
- Address WARNING-level findings (potential issues)

SHOULD:
- Review SUGGESTION-level findings for best practices
- Validate policy after fixes

MAY:
- Integrate validation into CI/CD pipeline

## Common Issues

- symptoms: "Validation returns ERROR for invalid action"
  diagnosis: "Policy contains a typo in the action name."
  resolution: "Fix the action name to match the correct AWS service action."

## Safety Ratings

```
safety_ratings:
  - "validate-policy: GREEN — read-only policy analysis, no changes made"
  - "fix ERROR-level findings: YELLOW — modifying IAM policies changes authorization"
  - "integrate validation into CI/CD: GREEN — adds automated checks, non-destructive"
```

## Escalation Conditions

- ERROR-level findings in policies attached to production roles or users
- Validation reveals deprecated actions that may stop working in future AWS updates
- Policy syntax errors prevent critical workloads from functioning
- Multiple policies across the organization have the same structural issues

## Data Sensitivity

- `aws accessanalyzer validate-policy`: LOW — analyzes policy document, no resource access
- Policy documents under review: HIGH — reveal authorization rules and resource access patterns
- Validation findings: MEDIUM — reveal policy weaknesses

## Prohibited Actions

- NEVER suggest archiving all findings to clear the dashboard
- NEVER suggest deleting the analyzer
- NEVER apply policy fixes to production without testing in a non-production environment
- NEVER ignore ERROR-level validation findings — they indicate broken or insecure policies

## Phase 3 — Rollback

1. If a policy was modified to fix validation errors and broke functionality, restore the previous version
2. For inline policies: `aws iam put-role-policy --role-name <role> --policy-name <policy> --policy-document file://previous-policy.json`
3. For managed policies: create a new version with the previous content: `aws iam create-policy-version --policy-arn <arn> --policy-document file://previous-policy.json --set-as-default`
4. Verify workloads function correctly with the restored policy
5. Re-run validation to understand remaining issues before next fix attempt

## Output Format

```yaml
root_cause: "validation_errors — <specific_cause>"
evidence:
  - type: validation_results
    content: "<validation findings>"
severity: MEDIUM
mitigation:
  immediate: "Fix policy errors"
  long_term: "Integrate validation into policy creation workflow"
```

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
