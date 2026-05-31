---
title: "F1 — Kinesis Streaming Issues"
description: "Diagnose QLDB journal streaming to Kinesis failures"
status: active
severity: HIGH
triggers: ["stream failed", "Kinesis error", "stream not delivering"]
owner: devops-agent
objective: "Identify and resolve Kinesis streaming failures"
context: "Streaming fails when: Kinesis stream doesn't exist, IAM role wrong, stream impaired, or shard capacity exceeded."
---

## Phase 1 — Triage

MUST:
- Check stream status: `aws qldb describe-journal-kinesis-stream --ledger-name <name> --stream-id <id>`
- Verify Kinesis stream exists: `aws kinesis describe-stream --stream-name <name>`
- Check IAM role has kinesis:PutRecord permission
- Verify stream is in ACTIVE state (not IMPAIRED)

SHOULD:
- Check Kinesis shard capacity for throughput
- Verify IAM role trust policy allows qldb.amazonaws.com
- Check for stream errors in CloudWatch

## Phase 2 — Remediate

MUST:
- Create Kinesis stream if missing
- Fix IAM role permissions and trust policy
- Recreate QLDB stream if IMPAIRED

SHOULD:
- Size Kinesis stream for expected throughput
- Monitor stream health

## Output Format

```yaml
root_cause: "kinesis_streaming — <specific_cause>"
severity: HIGH
```


## Safety Ratings
- Overall Risk: HIGH
- Data Exposure Risk: HIGH — Kinesis streams contain ledger change data
- Service Disruption Risk: MEDIUM — streaming failures affect downstream systems
- Compliance Impact: HIGH — stream data must comply with data handling policies

## Escalation Conditions
- QLDB stream creation fails
- Stream stops delivering data to Kinesis
- Stream delivery latency exceeds thresholds
- Stream IAM role permissions revoked
- Stream data contains sensitive information

## Data Sensitivity
- Stream data — CONFIDENTIAL (contains ledger changes)
- Kinesis stream configurations — INTERNAL
- Stream IAM role configurations — INTERNAL
- Delivery metrics — INTERNAL
- Never expose stream data to unauthorized consumers

## Prohibited Actions
- Do NOT create streams to unauthorized Kinesis destinations
- Do NOT disable stream encryption
- Do NOT ignore stream delivery failures
- Do NOT grant stream roles access to unrelated Kinesis streams
- Do NOT retain stream data beyond approved periods
- Do NOT share stream configurations externally

## Phase 3 — Rollback
- Delete incorrectly configured streams
- Restore previous stream configurations
- Restore Kinesis stream permissions
- Verify stream delivery after changes
- Document streaming issues
- Update stream management procedures

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
  - "NEVER suggest overly broad IAM permissions as a fix"
  - "NEVER suggest disabling security controls for troubleshooting"
  - "NEVER suggest deleting resources without confirmation"
