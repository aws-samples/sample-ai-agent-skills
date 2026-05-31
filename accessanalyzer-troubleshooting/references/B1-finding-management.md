---
title: "B1 — Finding Management"
description: "Diagnose finding management issues"
status: active
severity: MEDIUM
triggers: ["too many findings", "finding not appearing", "finding management"]
owner: devops-agent
objective: "Resolve finding management issues"
context: "Findings can be ACTIVE, ARCHIVED, or RESOLVED. Management includes filtering, archiving, and resolving findings."
---

## Phase 1 — Triage

MUST:
- List active findings: `aws accessanalyzer list-findings --analyzer-arn <arn> --filter '{"status":{"eq":["ACTIVE"]}}'`
- Check finding count and types
- Verify findings are being generated (analyzer is ACTIVE)

SHOULD:
- Filter by resource type or status
- Check archive rules for over-archiving

MAY:
- Export findings for analysis

## Phase 2 — Remediate

MUST:
- Review and triage active findings
- Archive accepted findings
- Resolve findings by fixing the underlying issue

SHOULD:
- Create archive rules for recurring acceptable findings
- Prioritize findings by resource type and risk

MAY:
- Integrate with Security Hub for centralized finding management

## Common Issues

- symptoms: "No findings but known public S3 bucket exists"
  diagnosis: "Analyzer not in ACTIVE status or analysis not yet complete."
  resolution: "Verify analyzer status. Wait up to 24 hours for initial analysis."

## Safety Ratings

```
safety_ratings:
  - "list-findings: GREEN — read-only enumeration of findings"
  - "archive finding: YELLOW — hides finding from active view, reversible"
  - "create archive rule: YELLOW — may auto-archive future findings including critical ones"
  - "Security Hub integration: GREEN — read-only data sharing"
```

## Escalation Conditions

- More than 100 active findings indicating widespread external access
- Findings indicate public access to sensitive resources (S3, KMS, IAM roles)
- Archive rules are suppressing findings that should be investigated
- No findings generated despite known external access patterns

## Data Sensitivity

- `aws accessanalyzer list-findings`: MEDIUM — reveals external access patterns and resource ARNs
- Finding details: HIGH — contains resource policies, principal identifiers, and access conditions
- Export of findings: HIGH — aggregated security posture data

## Prohibited Actions

- NEVER suggest archiving all findings to clear the dashboard
- NEVER suggest deleting the analyzer
- NEVER archive findings without reviewing them individually or via a well-scoped archive rule
- NEVER dismiss findings for public S3 buckets or IAM roles without security team approval

## Phase 3 — Rollback

1. Unarchive incorrectly archived findings: `aws accessanalyzer update-findings --analyzer-arn <arn> --ids <finding-ids> --status ACTIVE`
2. Delete overly broad archive rules: `aws accessanalyzer delete-archive-rule --analyzer-name <name> --rule-name <rule>`
3. Re-trigger analysis for resources that may have been missed: `aws accessanalyzer start-resource-scan --analyzer-arn <arn> --resource-arn <resource-arn>`
4. Verify all critical findings are visible in the active findings list
5. Review Security Hub integration to ensure findings are flowing correctly

## Output Format

```yaml
root_cause: "finding_management — <specific_cause>"
evidence:
  - type: finding_summary
    content: "<finding counts and types>"
severity: MEDIUM
mitigation:
  immediate: "Triage findings"
  long_term: "Implement finding management workflow"
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
