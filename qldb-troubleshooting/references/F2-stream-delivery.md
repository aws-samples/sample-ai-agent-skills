---
title: "F2 — Stream Delivery Issues"
description: "Diagnose stream data delivery and processing problems"
status: active
severity: MEDIUM
triggers: ["missing records", "delayed delivery", "duplicate records"]
owner: devops-agent
objective: "Identify and resolve stream delivery issues"
---

## Phase 1 — Triage

MUST:
- Check stream status and error count
- Verify Kinesis consumer is processing records
- Check for BLOCK_SUMMARY and REVISION_DETAILS record types
- Verify stream start time includes expected data

SHOULD:
- Check Kinesis iterator age for delivery delay
- Verify consumer checkpoint handling
- Check for duplicate record handling

## Phase 2 — Remediate

MUST:
- Fix consumer processing errors
- Handle both BLOCK_SUMMARY and REVISION_DETAILS records
- Implement idempotent processing for duplicates

SHOULD:
- Monitor iterator age for delivery latency
- Implement dead letter handling for failed records

## Output Format

```yaml
root_cause: "stream_delivery — <specific_cause>"
severity: MEDIUM
```


## Safety Ratings
- Overall Risk: HIGH
- Data Exposure Risk: HIGH — stream delivery contains ledger change data
- Service Disruption Risk: MEDIUM — delivery failures affect downstream systems
- Compliance Impact: HIGH — stream data must comply with data handling policies

## Escalation Conditions
- Stream delivery completely stops
- Stream delivery latency exceeds thresholds
- Stream data contains sensitive information
- Kinesis stream permissions revoked
- Stream delivery causes downstream processing failures

## Data Sensitivity
- Stream data — CONFIDENTIAL (contains ledger changes)
- Kinesis stream configurations — INTERNAL
- Delivery metrics — INTERNAL
- Stream IAM configurations — INTERNAL
- Never expose stream data to unauthorized consumers

## Prohibited Actions
- Do NOT deliver stream data to unauthorized Kinesis streams
- Do NOT disable stream encryption
- Do NOT ignore stream delivery failures
- Do NOT grant broad Kinesis access to stream roles
- Do NOT retain stream data beyond approved periods
- Do NOT share stream configurations externally

## Phase 3 — Rollback
- Restore previous stream configurations
- Re-enable stream delivery if disabled
- Restore Kinesis stream permissions
- Verify stream delivery after changes
- Document stream delivery issues
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
