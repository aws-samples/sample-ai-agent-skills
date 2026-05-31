---
title: "B2 — Finding Resolution"
description: "Diagnose finding resolution issues"
status: active
severity: HIGH
triggers: ["finding not resolving", "still showing after fix", "resolved but active"]
owner: devops-agent
objective: "Resolve finding resolution issues"
context: "Findings auto-resolve when the underlying issue is fixed. Resolution may take up to 24 hours after the fix."
---

## Phase 1 — Triage

MUST:
- Get finding details: `aws accessanalyzer get-finding --analyzer-arn <arn> --id <id>`
- Verify the underlying resource policy was actually fixed
- Check finding status and last update time

SHOULD:
- Verify the fix addresses the specific condition in the finding
- Check if finding has conditions that still apply

MAY:
- Manually trigger re-analysis: `aws accessanalyzer start-resource-scan --analyzer-arn <arn> --resource-arn <resource-arn>`

## Phase 2 — Remediate

MUST:
- Fix the underlying resource policy or configuration
- Wait up to 24 hours for auto-resolution

SHOULD:
- Use start-resource-scan to trigger immediate re-analysis
- Verify fix addresses all conditions in the finding

MAY:
- Archive finding if it represents accepted risk

## Common Issues

- symptoms: "Fixed S3 bucket policy but finding still ACTIVE"
  diagnosis: "Re-analysis hasn't run yet (up to 24 hours)."
  resolution: "Trigger re-analysis: aws accessanalyzer start-resource-scan --analyzer-arn <arn> --resource-arn <s3-arn>"

## Safety Ratings

```
safety_ratings:
  - "get-finding: GREEN — read-only finding detail retrieval"
  - "start-resource-scan: GREEN — triggers re-analysis, non-destructive"
  - "modify resource policy: RED — changes access control on the resource"
  - "archive finding: YELLOW — hides finding, reversible but may mask risk"
```

## Escalation Conditions

- Finding indicates public access to a resource containing sensitive data
- Resource policy change requires cross-team coordination
- Finding persists after policy fix, suggesting a deeper misconfiguration
- Multiple findings on the same resource indicate systemic access control issues

## Data Sensitivity

- `aws accessanalyzer get-finding`: HIGH — reveals full resource policy and external principal details
- `aws accessanalyzer start-resource-scan`: LOW — triggers analysis only
- Resource policy modifications: HIGH — directly affects who can access the resource

## Prohibited Actions

- NEVER suggest archiving all findings to clear the dashboard
- NEVER suggest deleting the analyzer
- NEVER modify resource policies on production resources without change management approval
- NEVER archive a finding as a substitute for actually fixing the underlying access issue

## Phase 3 — Rollback

1. If a resource policy was modified incorrectly, restore the previous policy from version history or backup
2. For S3 bucket policies: `aws s3api put-bucket-policy --bucket <bucket> --policy file://previous-policy.json`
3. For IAM role trust policies: `aws iam update-assume-role-policy --role-name <role> --policy-document file://previous-trust.json`
4. Trigger re-analysis after rollback: `aws accessanalyzer start-resource-scan --analyzer-arn <arn> --resource-arn <resource-arn>`
5. Verify the finding status reflects the current resource state

## Output Format

```yaml
root_cause: "finding_resolution — <specific_cause>"
evidence:
  - type: finding_details
    content: "<finding status and resource policy>"
severity: HIGH
mitigation:
  immediate: "Fix resource policy and trigger re-analysis"
  long_term: "Prevent external access with SCPs and preventive controls"
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
