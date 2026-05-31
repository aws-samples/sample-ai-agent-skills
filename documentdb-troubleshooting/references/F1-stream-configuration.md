---
title: "F1 — Change Stream Configuration Issues"
description: "Diagnose change stream setup and configuration problems on DocumentDB"
status: active
severity: MEDIUM
triggers:
  - "change stream"
  - "change stream not working"
  - "stream configuration"
  - "change stream enable"
  - "watch collection"
owner: devops-agent
objective: "Identify and resolve change stream configuration issues on DocumentDB"
context: "DocumentDB change streams must be enabled via the cluster parameter group (change_stream_log_retention_duration). Streams capture insert, update, replace, and delete operations. Retention is 24 hours by default, configurable up to 7 days."
---

## Phase 1 — Triage

MUST:
- Check change stream parameter: `aws docdb describe-db-cluster-parameters --db-cluster-parameter-group-name <param-group> --filters Name=ParameterName,Values=change_stream_log_retention_duration`
- Verify the parameter is set to a value > 0 (0 = disabled)
- Check cluster engine version supports change streams (3.6+)
- Test change stream: `mongo --tls --host <endpoint> --eval "db.collection.watch()"`

SHOULD:
- Verify the collection has write activity to generate events
- Check if the change stream is opened on the correct collection/database
- Review change stream options (fullDocument, pipeline filters)
- Check storage impact of change stream logs

MAY:
- Monitor change stream log storage consumption
- Test resume token functionality
- Verify change stream works across failover

## Phase 2 — Remediate

MUST:
- Enable change streams by setting change_stream_log_retention_duration > 0 in parameter group
- Apply the parameter group change (may require cluster restart for static parameters)
- Verify change stream events are being generated

SHOULD:
- Set retention duration based on consumer processing speed
- Implement resume token persistence for reliable consumption
- Monitor change stream storage impact

MAY:
- Use pipeline filters to reduce change stream volume
- Implement dead letter handling for failed event processing

## Common Issues

- symptoms: "Change stream returns no events"
  diagnosis: "Change streams are disabled (change_stream_log_retention_duration = 0)."
  resolution: "Set change_stream_log_retention_duration to desired hours (1-168) in the parameter group."

- symptoms: "Change stream misses events after consumer restart"
  diagnosis: "Consumer did not persist the resume token and events expired from retention."
  resolution: "Persist resume tokens. Use resumeAfter with the last processed token on restart."

- symptoms: "Change stream causes storage growth"
  diagnosis: "Change stream logs consume storage proportional to write volume × retention duration."
  resolution: "Reduce retention duration or optimize write patterns. Monitor VolumeBytesUsed."

## Output Format

```yaml
root_cause: "stream_configuration — <specific_cause>"
evidence:
  - type: parameter
    content: "<change_stream_log_retention_duration setting>"
  - type: stream_test
    content: "<change stream test results>"
  - type: storage
    content: "<change stream storage impact>"
severity: MEDIUM
mitigation:
  immediate: "Enable change streams and configure retention"
  long_term: "Implement reliable change stream consumption with resume tokens"
```


---

## Safety Ratings

| Dimension | Rating |
|-----------|--------|
| Data Sensitivity | HIGH — Change streams capture all database mutations |
| Blast Radius | MEDIUM — Enabling/disabling streams affects all consumers |
| Rollback Complexity | LOW — Parameter group changes can be reverted |
| Compliance Scope | HIGH — Change streams may be required for audit logging or replication |

## Escalation Conditions

- Escalate to all stream consumers before changing retention duration
- Escalate to compliance if change streams are required for audit logging
- Escalate to operations if change stream storage is growing excessively

## Data Sensitivity

- **Change stream events** contain all database mutations — treat as production data
- **Retention duration** affects how long mutation history is available

## PROHIBITED Actions

- Do NOT disable change streams without notifying all consumers
- Do NOT reduce retention below the time consumers need to process events
- Do NOT enable change streams without understanding the storage cost impact

## Phase 3 — Rollback

- Restore previous change_stream_log_retention_duration in the parameter group
- Apply the parameter group change to the cluster
- Verify change stream consumers can resume processing

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
