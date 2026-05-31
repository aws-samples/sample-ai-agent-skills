---
title: "F2 — Change Stream Consumer Issues"
description: "Diagnose change stream consumer lag, failures, and processing problems"
status: active
severity: MEDIUM
triggers:
  - "consumer lag"
  - "change stream consumer"
  - "resume token"
  - "events lost"
  - "consumer behind"
owner: devops-agent
objective: "Identify and resolve change stream consumer processing issues"
context: "Change stream consumers must process events within the retention window (24h-7d). Consumer issues include lag, lost resume tokens, processing failures, and event ordering. Consumers must handle resume tokens for reliable processing across restarts."
---

## Phase 1 — Triage

MUST:
- Verify change stream retention setting: `aws docdb describe-db-cluster-parameters --db-cluster-parameter-group-name <param-group> --filters Name=ParameterName,Values=change_stream_log_retention_duration`
- Check if consumer is connected and processing events
- Verify resume token is valid and within retention window
- Check consumer application logs for errors

SHOULD:
- Measure consumer processing rate vs event generation rate
- Check for consumer connection issues (TLS, network)
- Verify consumer handles all event types (insert, update, delete, replace)
- Check for consumer resource constraints (CPU, memory)

MAY:
- Monitor consumer lag metrics
- Check for event ordering issues in multi-shard scenarios
- Review consumer error handling and retry logic

## Phase 2 — Remediate

MUST:
- Fix consumer processing errors
- Ensure resume token is persisted after each successfully processed event
- Verify consumer can keep up with event generation rate

SHOULD:
- Implement batch processing for high-volume streams
- Add error handling with dead letter queue for failed events
- Scale consumer processing capacity if lagging

MAY:
- Implement consumer health monitoring
- Use multiple consumers with partitioned processing
- Set up alerts for consumer lag thresholds

## Common Issues

- symptoms: "InvalidResumeToken error when restarting consumer"
  diagnosis: "Resume token has expired because events were purged from retention."
  resolution: "Start a new change stream without resumeAfter. Implement faster token persistence."

- symptoms: "Consumer falls behind and cannot catch up"
  diagnosis: "Event generation rate exceeds consumer processing rate."
  resolution: "Optimize consumer processing. Use batch processing. Scale consumer resources."

- symptoms: "Duplicate events processed after consumer restart"
  diagnosis: "Resume token was not persisted for the last processed event."
  resolution: "Persist resume token after each event or batch. Implement idempotent processing."

## Output Format

```yaml
root_cause: "consumer_issue — <specific_cause>"
evidence:
  - type: consumer
    content: "<consumer status and processing rate>"
  - type: resume_token
    content: "<resume token validity>"
  - type: retention
    content: "<change stream retention configuration>"
severity: MEDIUM
mitigation:
  immediate: "Fix consumer errors and resume token management"
  long_term: "Implement robust consumer with monitoring and dead letter handling"
```


---

## Safety Ratings

| Dimension | Rating |
|-----------|--------|
| Data Sensitivity | HIGH — Change stream events contain database mutation data |
| Blast Radius | MEDIUM — Consumer failures cause event processing lag or data loss |
| Rollback Complexity | MEDIUM — Lost resume tokens require starting from current position |
| Compliance Scope | MEDIUM — Event processing may be required for audit or replication |

## Escalation Conditions

- Escalate to operations if consumer lag exceeds retention window (events will be lost)
- Escalate to application team if consumer processing errors affect downstream systems
- Escalate to data team if event loss affects data consistency

## Data Sensitivity

- **Change stream events** contain database mutations — treat as production data
- **Resume tokens** are operational state — persist securely

## PROHIBITED Actions

- Do NOT ignore consumer lag — events expire after the retention window
- Do NOT process events without idempotency — duplicates are possible after restarts
- Do NOT store resume tokens in volatile storage

## Phase 3 — Rollback

- If resume token is lost, start a new change stream from the current position
- Fix consumer processing errors and restart
- Reconcile any data gaps caused by missed events
- Increase retention duration if consumers need more processing time

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
  - "NEVER suggest disabling TLS"
  - "NEVER suggest making clusters publicly accessible"
  - "NEVER suggest disabling audit logging"
