---
title: "B3 — SQL Query Issues"
description: "Diagnose IoT rule SQL query syntax and matching problems"
status: active
severity: MEDIUM
triggers:
  - "SQL query not matching"
  - "rule not filtering"
  - "wrong data selected"
  - "topic filter mismatch"
  - "SQL syntax error"
owner: devops-agent
objective: "Identify and resolve rule SQL query issues"
context: "SQL query issues include: topic filter not matching published topics, WHERE clause filtering all messages, SELECT not extracting correct fields, function usage errors, or nested JSON access problems."
---

## Phase 1 — Triage

MUST:
- Check rule SQL: `aws iot get-topic-rule --rule-name <rule-name> --query 'rule.sql'`
- Verify topic filter in FROM clause matches published topics
- Check WHERE clause conditions against actual message payloads
- Verify SELECT fields exist in the message payload

SHOULD:
- Test SQL with IoT console MQTT test client
- Check for case sensitivity in topic names
- Verify wildcard usage (+ for single level, # for multi-level)
- Check IoT-specific function usage (topic(), timestamp(), etc.)

MAY:
- Compare expected vs actual message payload structure
- Check for JSON parsing issues in nested objects

## Phase 2 — Remediate

MUST:
- Fix topic filter to match published topics (remember: topics are case-sensitive)
- Fix SQL syntax using IoT SQL reference
- Ensure FROM clause uses single quotes around topic filter

SHOULD:
- Use IoT console test feature to validate SQL before deploying
- Example correct SQL: `SELECT temperature, humidity, topic(2) as deviceId FROM 'sensors/+/data' WHERE temperature > 30`

MAY:
- Use SQL functions for data transformation
- Implement multiple rules for different message types

## Output Format

```yaml
root_cause: "sql_query_issue — <specific_cause>"
evidence:
  - type: rule_sql
    content: "<current SQL query>"
  - type: topic_mismatch
    content: "<expected vs actual topics>"
severity: MEDIUM
mitigation:
  immediate: "Fix SQL syntax or topic filter"
  long_term: "Implement SQL testing in deployment pipeline"
```

## Safety Ratings
```
safety_ratings:
  - "Triage commands (describe/list/get): GREEN — read-only"
  - "Configuration changes: YELLOW — recoverable"
  - "Delete/terminate operations: RED — irreversible"
```

## Escalation Conditions
- "Resource serves production workload"
- "Fix requires deleting or recreating resources"
- "Fix involves security or encryption changes"
- "Blast radius affects multiple dependent services"

## Data Sensitivity
- HIGH: credentials, PII, security configurations
- MEDIUM: resource configurations, metrics
- LOW: status information, error messages

## Prohibited Actions
- Never revoke or delete device certificates without confirming device decommission
- Never modify IoT policies on production devices without testing in a staging environment
- Never delete thing groups that contain active production devices
- Never alter MQTT topic rules that route critical telemetry data
- Never disable custom authorizers serving active device connections

## Phase 3 — Rollback
- "Document current state before any changes"
- "If fix worsens the issue, revert to documented state"
- Reattach previous certificates if device connectivity is lost after changes
- Restore previous IoT policy versions if devices lose permissions
- Revert rule engine action configurations if message routing breaks

## Escalation Conditions

escalation_conditions:
  - "Remediation requires modifying IAM policies in a production account"
  - "Remediation requires disabling a security control even temporarily"
  - "Root cause cannot be identified after 3 hypothesis pivots"
  - "Blast radius affects more than one account or region"
  - "Issue involves potential data loss or exposure"

## Data Sensitivity

data_sensitivity:
  - command: "describe-instances"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"
  - command: "get-console-output"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"
  - command: "ssm send-command"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"

## Prohibited Actions

prohibited_actions:
  - "NEVER suggest 0.0.0.0/0 inbound security group rules as a fix"
  - "NEVER suggest disabling instance metadata service"
  - "NEVER terminate instances without confirmation"
