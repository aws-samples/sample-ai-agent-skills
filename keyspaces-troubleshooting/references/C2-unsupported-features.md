---
title: "C2 — Unsupported Features"
description: "Diagnose issues caused by using Cassandra features not available in Amazon Keyspaces"
status: active
severity: MEDIUM
triggers:
  - "unsupported feature"
  - "materialized view"
  - "UDF"
  - "UDA"
  - "trigger"
  - "COMPACT STORAGE"
  - "feature not available"
owner: devops-agent
objective: "Identify unsupported Cassandra features and provide Keyspaces alternatives"
context: "Keyspaces does not support several Apache Cassandra features: materialized views, UDFs, UDAs, triggers, COMPACT STORAGE, lightweight transactions (limited), ALLOW FILTERING (restricted), certain system tables, and Cassandra-native tools. Applications migrating from Cassandra must adapt."
---

## Phase 1 — Triage

MUST:
- Identify the specific unsupported feature being used
- Check if the feature is a hard limitation or has a partial alternative
- Review the application's dependency on the unsupported feature
- Verify the error message matches a known unsupported feature

SHOULD:
- Catalog all unsupported features used by the application
- Assess the impact of each unsupported feature on functionality
- Check Keyspaces release notes for recently added feature support

MAY:
- Compare full Cassandra schema with Keyspaces supported features
- Evaluate if DynamoDB or another service better fits the requirements

## Phase 2 — Remediate

MUST:
- Replace materialized views with denormalized tables or secondary indexes
- Replace UDFs/UDAs with application-side logic
- Remove triggers and implement event-driven logic (e.g., Lambda with DynamoDB Streams pattern)
- Remove COMPACT STORAGE from table definitions

SHOULD:
- Implement application-level alternatives for each unsupported feature
- Test alternatives thoroughly before production deployment
- Document workarounds for the development team

MAY:
- Consider hybrid architecture if some features are critical
- Evaluate AWS Glue or Lambda for data transformation previously done by UDFs

## Common Issues

- symptoms: "CREATE MATERIALIZED VIEW fails"
  diagnosis: "Materialized views are not supported in Keyspaces."
  resolution: "Create a separate denormalized table. Maintain consistency via application dual-writes."

- symptoms: "CREATE FUNCTION / CREATE AGGREGATE fails"
  diagnosis: "UDFs and UDAs are not supported in Keyspaces."
  resolution: "Move computation to application code or use AWS Lambda for data processing."

- symptoms: "Lightweight transaction (IF NOT EXISTS) fails or behaves differently"
  diagnosis: "LWT support is limited in Keyspaces. Some conditional operations may not work as expected."
  resolution: "Check current LWT support status. Use application-level idempotency if LWT is unavailable."

## Output Format

```yaml
root_cause: "unsupported_feature — <specific_feature>"
evidence:
  - type: feature_used
    content: "<the unsupported Cassandra feature>"
  - type: error_message
    content: "<error returned by Keyspaces>"
  - type: alternative
    content: "<recommended Keyspaces-compatible alternative>"
severity: MEDIUM
mitigation:
  immediate: "Remove unsupported feature usage and implement alternative"
  long_term: "Design for Keyspaces feature set from the start"
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
- Never drop keyspaces or tables containing production data
- Never modify capacity mode on tables during peak traffic without load testing
- Never alter TTL settings without understanding data retention requirements
- Never change encryption settings on tables with active workloads
- Never disable point-in-time recovery on production tables

## Phase 3 — Rollback
- "Document current state before any changes"
- "If fix worsens the issue, revert to documented state"
- Restore tables from point-in-time recovery if data is accidentally modified
- Revert capacity mode changes if throttling increases after modification
- Re-enable previous auto-scaling configurations if performance degrades

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
