---
title: "B2 — Data Events"
description: "Configure and troubleshoot CloudTrail data event logging"
status: active
severity: MEDIUM
triggers:
  - "data events not logging"
  - "S3 object logging"
  - "Lambda invocation logging"
  - "DynamoDB data events"
owner: devops-agent
objective: "Configure data event logging correctly for required resources"
context: "Data events log data plane operations (S3 GetObject/PutObject, Lambda Invoke, DynamoDB GetItem). They must be explicitly enabled and are always paid. Use advanced event selectors to filter specific resources and reduce cost."
---

## Phase 1 — Triage

MUST:
- Check event selectors: `aws cloudtrail get-event-selectors --trail-name <trail-name>`
- Verify data event types enabled
- Check if advanced or basic event selectors are in use
- Estimate data event volume for cost awareness

SHOULD:
- Check specific resource ARNs in event selectors
- Verify read/write event filtering
- Review cost implications of current configuration

MAY:
- Check CloudTrail Lake for data event ingestion
- Compare data event volume across resources

## Phase 2 — Remediate

MUST:
- Enable data events with advanced selectors: `aws cloudtrail put-event-selectors --trail-name <trail-name> --advanced-event-selectors '[{"Name":"S3DataEvents","FieldSelectors":[{"Field":"eventCategory","Equals":["Data"]},{"Field":"resources.type","Equals":["AWS::S3::Object"]},{"Field":"resources.ARN","StartsWith":["arn:aws:s3:::<bucket>/"]}]}]'`
- Filter to specific resources to control cost

SHOULD:
- Use advanced event selectors for granular filtering
- Monitor data event volume after enabling
- Set up cost alerts for data event charges

MAY:
- Enable data events for specific operations only (read-only or write-only)
- Use CloudTrail Lake for data event analysis

## Common Issues

- symptoms: "S3 data events enabled but too expensive"
  diagnosis: "Logging all S3 objects across all buckets."
  resolution: "Use advanced selectors to filter specific buckets or prefixes."

- symptoms: "Lambda invocations not logged"
  diagnosis: "Lambda data events not enabled in event selectors."
  resolution: "Add Lambda data event selector for specific functions or all functions."

## Safety Ratings

| Phase 2 Action | Rating | Rationale |
|---|---|---|
| Enable data events with advanced selectors | YELLOW | Increases cost; filter carefully |
| Filter to specific resources | GREEN | Reduces scope and cost |
| Monitor data event volume | GREEN | Read-only monitoring |

## Escalation Conditions

- Data event costs exceeding budget — escalate to cost management team
- Data events required for compliance but not enabled — escalate to compliance team
- High-volume data event configuration change — escalate to operations team for cost review

## Data Sensitivity

| Triage Command | Sensitivity | Rationale |
|---|---|---|
| `aws cloudtrail get-event-selectors` | HIGH | Exposes what data events are being logged |
| Data event logs | HIGH | Contain S3 object access, Lambda invocations, DynamoDB operations |

## Prohibited Actions

- **NEVER** suggest stopping or deleting CloudTrail trails
- **NEVER** suggest disabling log file validation
- **NEVER** suggest removing S3 bucket policy for log delivery
- **NEVER** enable data events for all S3 buckets without cost analysis

## Phase 3 — Rollback

| Remediation Action | Rollback Step |
|---|---|
| Enabled data events | Remove data event selectors: `aws cloudtrail put-event-selectors` with original config |
| Changed to advanced event selectors | Revert to basic event selectors if previously used |

## Output Format

```yaml
root_cause: "data_events — <specific_cause>"
evidence:
  - type: event_selectors
    content: "<data event configuration>"
  - type: cost_estimate
    content: "<estimated data event volume>"
severity: MEDIUM
mitigation:
  immediate: "Configure data events for required resources"
  long_term: "Use advanced selectors and monitor costs"
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
