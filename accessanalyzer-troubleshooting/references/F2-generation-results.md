---
title: "F2 — Policy Generation Results"
description: "Diagnose policy generation result issues"
status: active
severity: MEDIUM
triggers: ["generated policy incomplete", "missing permissions", "generation results"]
owner: devops-agent
objective: "Resolve policy generation result issues"
context: "Generated policies are based on observed CloudTrail activity. They may be incomplete for infrequent operations."
---

## Phase 1 — Triage

MUST:
- Get generated policy: `aws accessanalyzer get-generated-policy --job-id <id>`
- Review generated policy for completeness
- Compare with current policy permissions

SHOULD:
- Check if generation period covers all usage patterns
- Verify no infrequent operations are missing

MAY:
- Test generated policy in non-production

## Phase 2 — Remediate

MUST:
- Review and supplement generated policy with known required permissions
- Test before applying to production

SHOULD:
- Add permissions for infrequent but required operations
- Document additions and rationale

MAY:
- Use generated policy as starting point, not final policy

## Common Issues

- symptoms: "Generated policy missing permissions for monthly batch job"
  diagnosis: "Generation period didn't cover the monthly execution."
  resolution: "Extend generation period to cover full usage cycle. Manually add missing permissions."

## Safety Ratings

```
safety_ratings:
  - "get-generated-policy: GREEN — read-only retrieval of generated policy"
  - "compare with current policy: GREEN — read-only analysis"
  - "apply generated policy to production: RED — changes IAM authorization"
  - "test generated policy in non-production: YELLOW — changes IAM in test environment"
```

## Escalation Conditions

- Generated policy is missing critical permissions for production workloads
- Generated policy includes unexpected permissions suggesting unauthorized API usage
- Policy generation period did not cover all usage patterns (seasonal, DR, batch)
- Generated policy needs to be applied to roles shared across multiple teams

## Data Sensitivity

- `aws accessanalyzer get-generated-policy`: HIGH — reveals minimum permissions and API usage patterns
- Policy comparison: HIGH — reveals both current and recommended authorization
- Generated policy application: HIGH — directly changes who can do what

## Prohibited Actions

- NEVER suggest archiving all findings to clear the dashboard
- NEVER suggest deleting the analyzer
- NEVER apply a generated policy to production without supplementing it with known required permissions
- NEVER assume a generated policy is complete — it only reflects observed usage during the generation period

## Phase 3 — Rollback

1. If a generated policy was applied and broke functionality, restore the previous policy immediately
2. For inline policies: `aws iam put-role-policy --role-name <role> --policy-name <policy> --policy-document file://original-policy.json`
3. For managed policies: `aws iam create-policy-version --policy-arn <arn> --policy-document file://original-policy.json --set-as-default`
4. Verify all dependent workloads are functioning after rollback
5. Re-run generation with a longer period before re-attempting policy replacement

## Output Format

```yaml
root_cause: "generation_results — <specific_cause>"
evidence:
  - type: generated_policy
    content: "<policy content and completeness assessment>"
severity: MEDIUM
mitigation:
  immediate: "Supplement generated policy"
  long_term: "Use longer generation periods for comprehensive results"
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
