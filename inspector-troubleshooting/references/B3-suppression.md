---
title: "B3 — Suppression"
description: "Manage finding suppression rules in Inspector"
status: active
severity: LOW
triggers:
  - "suppress finding"
  - "suppression rule"
  - "filter findings"
  - "hide finding"
owner: devops-agent
objective: "Configure and manage suppression rules for accepted-risk findings"
context: "Suppression rules filter findings from the active view. They use filter criteria to match findings by severity, CVE, resource type, title, etc. Suppressed findings still exist and can be viewed. Suppression is for accepted risks after proper assessment."
---

## Phase 1 — Triage

MUST:
- List existing suppression rules: `aws inspector2 list-filters --query 'Filters[*].{Name:Name,Action:Action,Criteria:FilterCriteria}'`
- Check suppressed findings count: `aws inspector2 list-findings --filter-criteria '{"findingStatus":[{"comparison":"EQUALS","value":"SUPPRESSED"}]}' --max-results 5`
- Verify suppression rule criteria match intended findings

SHOULD:
- Review if suppression rules are too broad (suppressing valid findings)
- Check when rules were created and by whom
- Verify suppressed findings have documented justification

MAY:
- Audit suppression rules periodically
- Check if suppressed findings have been remediated

## Phase 2 — Remediate

MUST:
- Create targeted suppression rules: `aws inspector2 create-filter --name <name> --action SUPPRESS --filter-criteria '<criteria-json>' --description '<justification>'`
- Use specific criteria to avoid over-suppression
- Document the risk acceptance reason in the description

SHOULD:
- Review and clean up stale suppression rules
- Delete rules for remediated vulnerabilities: `aws inspector2 delete-filter --arn <filter-arn>`
- Set up periodic review of suppressed findings

MAY:
- Create organization-level suppression rules via delegated admin
- Export suppressed findings for compliance reporting

## Common Issues

- symptoms: "Suppression rule not working — findings still active"
  diagnosis: "Filter criteria don't match the finding attributes."
  resolution: "Check criteria syntax. Use exact match values from the finding."

- symptoms: "Too many findings suppressed"
  diagnosis: "Suppression rule criteria too broad."
  resolution: "Narrow criteria. Use specific CVE IDs, resource IDs, or package names."

## Safety Ratings

| Phase 2 Action | Rating | Rationale |
|---|---|---|
| Create targeted suppression rule | YELLOW | Hides findings from active view |
| Delete stale suppression rules | GREEN | Restores finding visibility |
| Review suppressed findings | GREEN | Read-only analysis |

## Escalation Conditions

- Suppression rules hiding critical findings — escalate to security team
- Organization-wide suppression rules — escalate to delegated admin and security team
- Suppression rules created without documentation — escalate to security operations for review

## Data Sensitivity

| Triage Command | Sensitivity | Rationale |
|---|---|---|
| `aws inspector2 list-filters` | MEDIUM | Exposes suppression rule criteria |
| `aws inspector2 list-findings` (suppressed) | MEDIUM | Exposes suppressed vulnerability details |

## Prohibited Actions

- **NEVER** suggest disabling Inspector scanning as a troubleshooting step
- **NEVER** suggest suppressing all findings without individual review
- **NEVER** create suppression rules without documented risk acceptance
- **NEVER** suppress findings based solely on severity without reviewing exploitability

## Phase 3 — Rollback

| Remediation Action | Rollback Step |
|---|---|
| Created suppression rule | Delete the rule: `aws inspector2 delete-filter --arn <filter-arn>` |
| Deleted suppression rule | Recreate the rule if still needed |

## Output Format

```yaml
root_cause: "suppression — <specific_cause>"
evidence:
  - type: suppression_rules
    content: "<existing rules and criteria>"
  - type: suppressed_count
    content: "<number of suppressed findings>"
severity: LOW
mitigation:
  immediate: "Fix suppression rule criteria"
  long_term: "Establish suppression governance process"
```

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
  - "NEVER suggest disabling scanning to reduce findings"
  - "NEVER suggest ignoring critical vulnerabilities"
  - "NEVER suggest removing SSM agent to avoid scans"
