---
title: "E1 — Cost Optimization"
description: "Optimize Macie costs while maintaining scanning coverage"
status: active
severity: MEDIUM
triggers:
  - "Macie expensive"
  - "high cost"
  - "reduce Macie spend"
  - "cost optimization"
owner: devops-agent
objective: "Reduce Macie costs while maintaining adequate sensitive data detection"
context: "Macie pricing is per S3 object evaluated. Large buckets with millions of objects can be expensive. Cost optimization strategies include sampling, targeted scanning, and excluding non-sensitive buckets."
---

## Phase 1 — Triage

MUST:
- Check usage totals: `aws macie2 get-usage-totals`
- Check usage by bucket: `aws macie2 get-usage-statistics --filter-by '[{"key":"accountId"}]' --sort-by '{"key":"totalCost","orderBy":"DESC"}'`
- List active jobs and their scope: `aws macie2 list-classification-jobs --filter-criteria '{"includes":{"key":"jobStatus","values":["RUNNING","SCHEDULED"]}}'`
- Check bucket sizes: `aws macie2 describe-buckets --query 'Buckets[*].{Name:BucketName,Objects:ClassifiableObjectCount,Size:ClassifiableSizeInBytes}'`

SHOULD:
- Identify the most expensive buckets to scan
- Check sampling depth on existing jobs
- Review if non-sensitive buckets are being scanned unnecessarily

MAY:
- Compare automated discovery costs vs job-based costs
- Project costs for planned scanning changes

## Phase 2 — Remediate

MUST:
- Use sampling for large, low-risk buckets
- Exclude known non-sensitive buckets from discovery jobs
- Review and cancel unnecessary scheduled jobs

SHOULD:
- Use targeted prefix-based scanning instead of full-bucket scans
- Schedule jobs during off-peak for cost predictability
- Set up cost alerts using AWS Budgets

MAY:
- Use automated sensitive data discovery instead of full jobs for initial assessment
- Consolidate overlapping jobs

## Common Issues

- symptoms: "Unexpected high Macie bill"
  diagnosis: "Discovery job scanning large bucket with millions of objects."
  resolution: "Use sampling. Exclude non-sensitive prefixes. Cancel unnecessary jobs."

- symptoms: "Cost increasing month over month"
  diagnosis: "Bucket growth or new scheduled jobs added."
  resolution: "Review job scope. Adjust sampling. Remove redundant jobs."

## Safety Ratings

| Phase 2 Action | Rating | Rationale |
|---|---|---|
| Use sampling for large buckets | GREEN | Reduces cost while maintaining detection |
| Exclude non-sensitive buckets | YELLOW | Reduces scanning coverage |
| Cancel unnecessary jobs | YELLOW | Stops active scanning |
| Set up cost alerts | GREEN | Monitoring-only |

## Escalation Conditions

- Macie costs exceeding budget significantly — escalate to cost management team
- Reducing scanning scope may affect compliance — escalate to compliance team
- Discovery job cancellation for compliance-critical buckets — escalate to security team

## Data Sensitivity

| Triage Command | Sensitivity | Rationale |
|---|---|---|
| `aws macie2 get-usage-totals` | MEDIUM | Exposes usage and cost data |
| `aws macie2 get-usage-statistics` | MEDIUM | Exposes per-bucket cost data |
| `aws macie2 list-classification-jobs` | HIGH | Exposes job configurations |
| `aws macie2 describe-buckets` | HIGH | Exposes bucket metadata |

## Prohibited Actions

- **NEVER** suggest disabling Macie as a troubleshooting step
- **NEVER** suggest ignoring sensitive data findings without review
- **NEVER** exclude compliance-critical buckets from scanning to reduce costs
- **NEVER** cancel scheduled jobs without understanding the compliance impact

## Phase 3 — Rollback

| Remediation Action | Rollback Step |
|---|---|
| Enabled sampling | Increase sampling back to 100% |
| Excluded buckets from jobs | Re-add buckets to job scope |
| Cancelled jobs | Recreate the jobs with original configuration |

## Output Format

```yaml
root_cause: "cost — <specific_cause>"
evidence:
  - type: usage_totals
    content: "<Macie usage and cost>"
  - type: expensive_buckets
    content: "<highest-cost buckets>"
severity: MEDIUM
mitigation:
  immediate: "Reduce scanning scope and enable sampling"
  long_term: "Establish cost governance for Macie scanning"
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
