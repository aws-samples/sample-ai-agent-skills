---
title: "E2 — Validation Best Practices"
description: "Diagnose policy best practice violations"
status: active
severity: LOW
triggers: ["policy best practices", "policy suggestions", "improve policy"]
owner: devops-agent
objective: "Improve policy quality using validation"
context: "Validation provides suggestions for policy improvement beyond syntax — including least privilege, resource scoping, and condition usage."
---

## Phase 1 — Triage

MUST:
- Run validation on target policy
- Review SUGGESTION and WARNING findings
- Prioritize by impact

SHOULD:
- Check for wildcard actions or resources
- Verify condition keys are appropriate

MAY:
- Compare with AWS managed policies for reference

## Phase 2 — Remediate

MUST:
- Address high-impact suggestions (wildcard removal, resource scoping)

SHOULD:
- Add condition keys for additional security
- Replace wildcard actions with specific actions

MAY:
- Create policy templates following best practices

## Common Issues

- symptoms: "Validation suggests removing wildcard but app needs many actions"
  diagnosis: "Policy uses * for convenience but should list specific actions."
  resolution: "Use policy generation to identify actually used actions, then list them explicitly."

## Safety Ratings

```
safety_ratings:
  - "validate-policy (suggestions): GREEN — read-only analysis"
  - "remove wildcard actions: YELLOW — restricts permissions, may break workloads"
  - "add condition keys: YELLOW — adds constraints to existing policies"
  - "create policy templates: GREEN — documentation, no runtime impact"
```

## Escalation Conditions

- Wildcard actions (Action: *) found on production roles with broad resource scope
- Policies lack condition keys for sensitive actions (iam:*, s3:DeleteBucket, ec2:TerminateInstances)
- Validation suggestions indicate policies violate organizational security standards
- Multiple teams use the same overly permissive policy template

## Data Sensitivity

- Validation suggestions: LOW — general best practice recommendations
- Policy documents under review: HIGH — reveal authorization rules
- Comparison with AWS managed policies: LOW — public reference data

## Prohibited Actions

- NEVER suggest archiving all findings to clear the dashboard
- NEVER suggest deleting the analyzer
- NEVER replace wildcard permissions on production roles without identifying all required actions first
- NEVER add restrictive condition keys without testing the impact on existing workloads

## Phase 3 — Rollback

1. If wildcard removal broke functionality, restore the previous policy version
2. For managed policies: `aws iam create-policy-version --policy-arn <arn> --policy-document file://previous-policy.json --set-as-default`
3. If condition keys were added too restrictively, remove them from the policy statement
4. Verify all dependent workloads function correctly after rollback
5. Use policy generation to identify the minimum required actions before re-attempting

## Output Format

```yaml
root_cause: "validation_best_practices — <specific_cause>"
evidence:
  - type: suggestions
    content: "<validation suggestions>"
severity: LOW
mitigation:
  immediate: "Address high-impact suggestions"
  long_term: "Enforce policy standards via CI/CD validation"
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
