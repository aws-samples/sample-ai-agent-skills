---
title: "D1 — Unused Access Analysis"
description: "Diagnose unused access analysis issues"
status: active
severity: MEDIUM
triggers: ["unused access not working", "unused roles", "unused permissions"]
owner: devops-agent
objective: "Resolve unused access analysis issues"
context: "Unused access analysis identifies IAM roles, users, access keys, and permissions not used within a specified period."
---

## Phase 1 — Triage

MUST:
- Check if unused access analyzer exists: `aws accessanalyzer list-analyzers --query 'analyzers[?type==ACCOUNT_UNUSED_ACCESS || type==ORGANIZATION_UNUSED_ACCESS]'`
- Verify analyzer is ACTIVE
- Check unused access tracking period configuration

SHOULD:
- Verify CloudTrail is logging IAM activity
- Check if findings are being generated

MAY:
- Review unused access findings: `aws accessanalyzer list-findings-v2 --analyzer-arn <arn>`

## Phase 2 — Remediate

MUST:
- Create unused access analyzer if not exists
- Ensure CloudTrail is enabled for IAM activity tracking

SHOULD:
- Set appropriate tracking period (30-180 days)
- Review and act on unused access findings

MAY:
- Automate unused role cleanup with Lambda

## Common Issues

- symptoms: "No unused access findings despite known unused roles"
  diagnosis: "Unused access analyzer not created or tracking period too short."
  resolution: "Create unused access analyzer with appropriate tracking period."

## Safety Ratings

```
safety_ratings:
  - "list-analyzers (unused access type): GREEN — read-only enumeration"
  - "list-findings-v2: GREEN — read-only finding retrieval"
  - "create unused access analyzer: YELLOW — creates new resource with ongoing cost"
  - "automate unused role cleanup with Lambda: RED — deletes IAM resources automatically"
```

## Escalation Conditions

- Unused access analyzer reveals service-linked roles or break-glass roles as unused
- Large number of unused roles suggests systemic IAM hygiene issues requiring org-wide remediation
- Unused access findings conflict with compliance requirements (e.g., SOC2 access reviews)
- CloudTrail logging gaps may cause false positives in unused access detection

## Data Sensitivity

- `aws accessanalyzer list-findings-v2`: MEDIUM — reveals IAM roles, users, and their usage patterns
- Unused access tracking period: LOW — configuration metadata
- CloudTrail activity correlation: HIGH — reveals who accessed what and when

## Prohibited Actions

- NEVER suggest archiving all findings to clear the dashboard
- NEVER suggest deleting the analyzer
- NEVER delete unused IAM roles without verifying they are not used for break-glass or seasonal operations
- NEVER automate IAM role deletion without a review and approval workflow

## Phase 3 — Rollback

1. If an unused access analyzer was created in error, delete it: `aws accessanalyzer delete-analyzer --analyzer-name <name>`
2. If IAM roles were deleted based on unused access findings, recreate them from IaC or backup
3. If permissions were removed from a role that is actually used seasonally, restore the policy: `aws iam attach-role-policy --role-name <role> --policy-arn <policy-arn>`
4. Verify CloudTrail is logging all IAM activity to prevent future false positives
5. Document any roles that were incorrectly flagged as unused for tracking period adjustment

## Output Format

```yaml
root_cause: "unused_access_analysis — <specific_cause>"
evidence:
  - type: analyzer_config
    content: "<unused access analyzer status>"
severity: MEDIUM
mitigation:
  immediate: "Configure unused access analysis"
  long_term: "Regular review and cleanup of unused access"
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
