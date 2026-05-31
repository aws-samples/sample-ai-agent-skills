---
title: "E2 — Sampling"
description: "Configure and troubleshoot Macie sampling settings"
status: active
severity: LOW
triggers:
  - "sampling configuration"
  - "sample depth"
  - "partial scanning"
  - "sampling percentage"
owner: devops-agent
objective: "Configure appropriate sampling depth for cost-effective scanning"
context: "Sampling controls what percentage of objects in a bucket are scanned. Lower sampling reduces cost but increases risk of missing sensitive data. Sampling is configured per discovery job. The trade-off between cost and detection completeness must be balanced."
---

## Phase 1 — Triage

MUST:
- Check job sampling configuration: `aws macie2 describe-classification-job --job-id <job-id>`
- Check current costs: `aws macie2 get-usage-totals`
- Review bucket object counts: `aws macie2 describe-buckets --criteria '{"bucketName":{"eq":["<bucket>"]}}'`

SHOULD:
- Compare findings from sampled vs full scans
- Assess risk tolerance for each bucket
- Check if compliance requirements mandate full scanning

MAY:
- Calculate cost difference between sampling levels
- Review sampling effectiveness over multiple job runs

## Phase 2 — Remediate

MUST:
- Set 100% sampling for compliance-critical buckets
- Use appropriate sampling for development/test buckets
- Document sampling decisions and risk acceptance

SHOULD:
- Create separate jobs with different sampling for different risk levels
- Review sampling effectiveness periodically
- Adjust sampling based on finding patterns

MAY:
- Use automated discovery for initial assessment before full scanning
- Implement tiered scanning strategy (sample first, full scan if findings detected)

## Common Issues

- symptoms: "Sampling set too low — missing sensitive data"
  diagnosis: "Low sampling percentage missing objects with sensitive data."
  resolution: "Increase sampling depth. Use 100% for critical buckets."

- symptoms: "Full scanning too expensive"
  diagnosis: "100% sampling on large buckets driving high costs."
  resolution: "Use sampling for non-critical buckets. Target specific prefixes."

## Safety Ratings

| Phase 2 Action | Rating | Rationale |
|---|---|---|
| Set 100% sampling for critical buckets | YELLOW | Significant cost increase |
| Adjust sampling for non-critical buckets | GREEN | Cost optimization |
| Document sampling decisions | GREEN | Documentation only |

## Escalation Conditions

- Compliance requires 100% scanning but cost is prohibitive — escalate to compliance and cost management teams
- Sampling missing known sensitive data — escalate to security team
- Sampling configuration changes for organization-wide jobs — escalate to delegated admin

## Data Sensitivity

| Triage Command | Sensitivity | Rationale |
|---|---|---|
| `aws macie2 describe-classification-job` | HIGH | Exposes job configuration including sampling |
| `aws macie2 get-usage-totals` | MEDIUM | Exposes cost data |
| `aws macie2 describe-buckets` | HIGH | Exposes bucket metadata |

## Prohibited Actions

- **NEVER** suggest disabling Macie as a troubleshooting step
- **NEVER** suggest ignoring sensitive data findings without review
- **NEVER** set sampling below acceptable thresholds for compliance-critical buckets
- **NEVER** reduce sampling without documenting the risk acceptance

## Phase 3 — Rollback

| Remediation Action | Rollback Step |
|---|---|
| Changed sampling depth | Restore previous sampling configuration |
| Created separate jobs | Cancel or delete the new jobs |

## Output Format

```yaml
root_cause: "sampling — <specific_cause>"
evidence:
  - type: sampling_config
    content: "<current sampling settings>"
  - type: cost_impact
    content: "<cost at current sampling level>"
severity: LOW
mitigation:
  immediate: "Adjust sampling to balance cost and coverage"
  long_term: "Implement tiered scanning strategy"
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
  - "NEVER suggest disabling Macie to reduce costs without review"
  - "NEVER suggest ignoring sensitive data findings"
  - "NEVER suggest removing S3 bucket monitoring"
