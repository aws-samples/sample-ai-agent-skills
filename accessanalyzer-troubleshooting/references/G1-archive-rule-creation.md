---
title: "G1 — Archive Rule Creation"
description: "Diagnose archive rule creation issues"
status: active
severity: LOW
triggers: ["archive rule error", "cannot create archive rule"]
owner: devops-agent
objective: "Resolve archive rule creation issues"
context: "Archive rules automatically archive findings matching specified criteria. They help manage finding noise for known acceptable access patterns."
---

## Phase 1 — Triage

MUST:
- List existing rules: `aws accessanalyzer list-archive-rules --analyzer-name <name>`
- Verify filter criteria syntax
- Check IAM permissions for access-analyzer:CreateArchiveRule

SHOULD:
- Verify filter matches intended findings
- Check rule doesn't over-archive important findings

MAY:
- Test filter criteria against existing findings

## Phase 2 — Remediate

MUST:
- Fix filter criteria syntax
- Ensure IAM permissions are sufficient
- Create rule: `aws accessanalyzer create-archive-rule --analyzer-name <name> --rule-name <name> --filter <filter>`

SHOULD:
- Start with narrow filters and expand as needed
- Document archive rule rationale

MAY:
- Apply rule to existing findings retroactively

## Common Issues

- symptoms: "Archive rule created but findings not archived"
  diagnosis: "Filter criteria don't match the findings."
  resolution: "Review and fix filter criteria to match target findings."

## Safety Ratings

```
safety_ratings:
  - "list-archive-rules: GREEN — read-only enumeration"
  - "create-archive-rule: YELLOW — may auto-archive future findings including critical ones"
  - "apply rule retroactively: YELLOW — archives existing findings, reversible"
```

## Escalation Conditions

- Archive rule filter is too broad and may suppress critical security findings
- Archive rule creation fails due to IAM permission issues in a production account
- Multiple conflicting archive rules create unpredictable finding suppression
- Archive rules needed for compliance but risk hiding genuine security issues

## Data Sensitivity

- `aws accessanalyzer list-archive-rules`: MEDIUM — reveals what access patterns are being accepted
- Archive rule filters: MEDIUM — reveals organizational risk acceptance criteria
- Retroactive application: MEDIUM — changes visibility of existing findings

## Prohibited Actions

- NEVER suggest archiving all findings to clear the dashboard
- NEVER suggest deleting the analyzer
- NEVER create archive rules with overly broad filters (e.g., matching all findings for a resource type)
- NEVER create archive rules without documenting the business justification

## Phase 3 — Rollback

1. Delete the problematic archive rule: `aws accessanalyzer delete-archive-rule --analyzer-name <name> --rule-name <rule>`
2. Unarchive findings that were incorrectly archived by the rule: `aws accessanalyzer update-findings --analyzer-arn <arn> --ids <finding-ids> --status ACTIVE`
3. Create a narrower replacement rule if needed
4. Verify all critical findings are visible in the active findings list
5. Document the rule change and notify the security team

## Output Format

```yaml
root_cause: "archive_rule_creation — <specific_cause>"
evidence:
  - type: rule_config
    content: "<archive rule filter and status>"
severity: LOW
mitigation:
  immediate: "Fix rule configuration"
  long_term: "Document and review archive rules regularly"
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
