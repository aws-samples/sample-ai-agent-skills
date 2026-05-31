---
title: "H2 — Lake Pricing"
description: "Manage CloudTrail Lake costs"
status: active
severity: MEDIUM
triggers:
  - "Lake expensive"
  - "Lake pricing"
  - "event data store cost"
  - "query costs"
owner: devops-agent
objective: "Optimize CloudTrail Lake costs"
context: "CloudTrail Lake charges per GB ingested into event data stores and per GB scanned by queries. Long retention periods increase storage costs. Unoptimized queries scan more data than necessary."
---

## Phase 1 — Triage

MUST:
- List event data stores: `aws cloudtrail list-event-data-stores`
- Check retention periods: `aws cloudtrail get-event-data-store --event-data-store <eds-arn>`
- Review query history for scan volumes: `aws cloudtrail list-queries --event-data-store <eds-arn>`
- Check billing for CloudTrail Lake charges

SHOULD:
- Estimate data ingestion volume
- Review query patterns for optimization opportunities
- Check if multiple data stores have overlapping data

MAY:
- Compare Lake costs with S3-based analysis costs
- Project future costs based on growth

## Phase 2 — Remediate

MUST:
- Set appropriate retention periods (reduce if not needed)
- Optimize queries with time-based filters to reduce scanned data
- Remove unused event data stores

SHOULD:
- Use specific WHERE clauses in queries
- Avoid SELECT * on large time ranges
- Consider one-time pricing option for predictable workloads

MAY:
- Use S3 trails for long-term archival (cheaper storage)
- Set up cost alerts for Lake charges

## Common Issues

- symptoms: "High Lake ingestion costs"
  diagnosis: "Data events or high-volume management events ingested."
  resolution: "Filter event types in data store. Use advanced selectors."

- symptoms: "High query costs"
  diagnosis: "Queries scanning large time ranges without filters."
  resolution: "Add time-based WHERE clauses. Use specific event filters."

## Safety Ratings

| Phase 2 Action | Rating | Rationale |
|---|---|---|
| Set appropriate retention periods | YELLOW | Reducing retention deletes data after the new period |
| Optimize queries with filters | GREEN | Query optimization, no state change |
| Remove unused event data stores | RED | Permanent data deletion |

## Escalation Conditions

- Lake costs exceeding budget — escalate to cost management team
- Retention period reduction affecting compliance — escalate to compliance team
- Event data store deletion — escalate to security team for data retention review

## Data Sensitivity

| Triage Command | Sensitivity | Rationale |
|---|---|---|
| `aws cloudtrail list-event-data-stores` | HIGH | Exposes data store configuration |
| `aws cloudtrail get-event-data-store` | HIGH | Exposes retention and event type configuration |
| `aws cloudtrail list-queries` | HIGH | Exposes query history and scan volumes |

## Prohibited Actions

- **NEVER** suggest stopping or deleting CloudTrail trails
- **NEVER** suggest disabling log file validation
- **NEVER** suggest removing S3 bucket policy for log delivery
- **NEVER** delete event data stores without confirming data retention requirements are met
- **NEVER** reduce retention below compliance minimums

## Phase 3 — Rollback

| Remediation Action | Rollback Step |
|---|---|
| Reduced retention period | Increase retention period (data already deleted cannot be recovered) |
| Deleted event data store | Cannot undo — data is permanently lost |
| Changed event selectors on data store | Restore previous advanced event selectors |

## Output Format

```yaml
root_cause: "lake_pricing — <specific_cause>"
evidence:
  - type: data_stores
    content: "<data store configuration and retention>"
  - type: query_volume
    content: "<query scan volumes>"
severity: MEDIUM
mitigation:
  immediate: "Optimize retention and query patterns"
  long_term: "Implement Lake cost governance"
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
