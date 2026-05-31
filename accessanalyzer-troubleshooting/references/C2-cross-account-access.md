---
title: "C2 — Cross-Account Access"
description: "Diagnose cross-account access finding issues"
status: active
severity: HIGH
triggers: ["cross-account finding", "shared with other account", "external principal"]
owner: devops-agent
objective: "Resolve cross-account access finding issues"
context: "Cross-account access findings indicate resources shared with other AWS accounts. For organization analyzers, sharing within the org is not flagged."
---

## Phase 1 — Triage

MUST:
- Get finding details: `aws accessanalyzer get-finding --analyzer-arn <arn> --id <id>`
- Check principal field to identify the external account
- Verify if the cross-account access is intentional

SHOULD:
- Check if the external account is within your organization
- Review the condition context for access restrictions

MAY:
- Verify the external account identity

## Phase 2 — Remediate

MUST:
- If unintentional: remove external principal from resource policy
- If intentional: archive the finding

SHOULD:
- Use organization analyzer to avoid findings for intra-org sharing
- Document intentional cross-account access

MAY:
- Create archive rules for known cross-account relationships

## Common Issues

- symptoms: "Finding for cross-account access that's intentional"
  diagnosis: "Account analyzer flags all external access including intentional sharing."
  resolution: "Use organization analyzer (intra-org not flagged) or archive the finding."

## Safety Ratings

```
safety_ratings:
  - "get-finding: GREEN — read-only finding detail retrieval"
  - "archive finding: YELLOW — hides finding, reversible"
  - "remove external principal from policy: RED — changes resource access control"
  - "create archive rule for cross-account: YELLOW — may suppress future legitimate findings"
```

## Escalation Conditions

- Cross-account access detected to resources containing PII, financial data, or secrets
- External account is unknown or cannot be verified as a trusted partner
- Cross-account access grants write or admin permissions to the external principal
- Multiple resources share access with the same unrecognized external account

## Data Sensitivity

- `aws accessanalyzer get-finding`: HIGH — reveals external principal ARN and access conditions
- Cross-account principal identification: HIGH — reveals trust relationships between accounts
- Resource policy review: HIGH — reveals full access control configuration

## Prohibited Actions

- NEVER suggest archiving all findings to clear the dashboard
- NEVER suggest deleting the analyzer
- NEVER archive cross-account findings without verifying the external account identity
- NEVER remove cross-account access from production resources without change management approval

## Phase 3 — Rollback

1. If cross-account access was removed incorrectly, restore the resource policy with the external principal
2. For S3: `aws s3api put-bucket-policy --bucket <bucket> --policy file://policy-with-cross-account.json`
3. For IAM roles: `aws iam update-assume-role-policy --role-name <role> --policy-document file://trust-with-cross-account.json`
4. If an archive rule was created too broadly, delete it: `aws accessanalyzer delete-archive-rule --analyzer-name <name> --rule-name <rule>`
5. Verify the cross-account access is restored and the external party confirms functionality

## Output Format

```yaml
root_cause: "cross_account_access — <specific_cause>"
evidence:
  - type: finding_details
    content: "<external principal and access details>"
severity: HIGH
mitigation:
  immediate: "Verify and resolve finding"
  long_term: "Use organization analyzer and archive rules"
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
