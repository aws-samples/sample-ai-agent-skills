---
title: "H1 — Data Event Costs"
description: "Manage and optimize CloudTrail data event costs"
status: active
severity: MEDIUM
triggers:
  - "CloudTrail expensive"
  - "data event costs"
  - "high CloudTrail bill"
  - "reduce logging costs"
owner: devops-agent
objective: "Optimize CloudTrail data event costs while maintaining required logging"
context: "Data events are charged per event ($0.10 per 100,000). High-volume S3 buckets and Lambda functions can generate millions of events. Use advanced event selectors to filter specific resources and reduce costs."
---

## Phase 1 — Triage

MUST:
- Check event selectors for data events: `aws cloudtrail get-event-selectors --trail-name <trail-name>`
- Check CloudTrail usage in Cost Explorer or billing
- Identify which resources generate the most data events
- Count trails with data events enabled: `aws cloudtrail describe-trails`

SHOULD:
- Estimate event volume per resource
- Check if multiple trails log the same data events (duplicate charges)
- Review if all data event resources are necessary

MAY:
- Use Cost Explorer to analyze CloudTrail costs over time
- Compare data event costs across regions

## Phase 2 — Remediate

MUST:
- Use advanced event selectors to filter specific buckets/functions
- Remove data events for non-critical resources
- Eliminate duplicate data event logging across trails

SHOULD:
- Filter by specific S3 bucket prefixes instead of entire buckets
- Use read-only or write-only filtering where appropriate
- Monitor costs after changes

MAY:
- Consider CloudTrail Lake vs S3 trails for cost optimization
- Set up AWS Budgets alerts for CloudTrail costs

## Common Issues

- symptoms: "Unexpected high CloudTrail bill"
  diagnosis: "Data events enabled for high-volume S3 buckets."
  resolution: "Use advanced selectors to filter specific buckets. Remove unnecessary data events."

- symptoms: "Duplicate charges for same events"
  diagnosis: "Multiple trails logging the same data events."
  resolution: "Consolidate to one trail with data events. Additional copies are charged."

## Safety Ratings

| Phase 2 Action | Rating | Rationale |
|---|---|---|
| Use advanced event selectors to filter | GREEN | Reduces cost without losing critical events |
| Remove data events for non-critical resources | YELLOW | Reduces audit coverage; verify compliance first |
| Eliminate duplicate logging | GREEN | Cost optimization without coverage loss |

## Escalation Conditions

- Data event costs exceeding budget significantly — escalate to cost management team
- Removing data events that may be required for compliance — escalate to compliance team
- Duplicate data event logging across multiple trails — escalate to trail configuration owner

## Data Sensitivity

| Triage Command | Sensitivity | Rationale |
|---|---|---|
| `aws cloudtrail get-event-selectors` | HIGH | Exposes data event configuration |
| `aws cloudtrail describe-trails` | HIGH | Exposes all trail configurations |
| Cost Explorer / billing data | MEDIUM | Exposes account spending |

## Prohibited Actions

- **NEVER** suggest stopping or deleting CloudTrail trails
- **NEVER** suggest disabling log file validation
- **NEVER** suggest removing S3 bucket policy for log delivery
- **NEVER** remove data events required for compliance or security monitoring

## Phase 3 — Rollback

| Remediation Action | Rollback Step |
|---|---|
| Removed data events for specific resources | Re-add data event selectors for those resources |
| Changed to advanced event selectors | Revert to previous event selector configuration |
| Consolidated trails | Recreate removed trails if needed |

## Output Format

```yaml
root_cause: "data_event_costs — <specific_cause>"
evidence:
  - type: event_selectors
    content: "<data event configuration>"
  - type: cost_analysis
    content: "<estimated data event volume and cost>"
severity: MEDIUM
mitigation:
  immediate: "Filter data events to required resources only"
  long_term: "Implement cost monitoring and governance"
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
  - command: "lookup-events"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"
  - command: "get-trail-status"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"
  - command: "describe-trails"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"

## Prohibited Actions

prohibited_actions:
  - "NEVER suggest disabling CloudTrail logging"
  - "NEVER suggest deleting trail logs"
  - "NEVER reduce logging scope in production"
