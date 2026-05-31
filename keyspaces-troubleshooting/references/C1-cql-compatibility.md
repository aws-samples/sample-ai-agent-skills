---
title: "C1 — CQL Compatibility"
description: "Diagnose Amazon Keyspaces CQL compatibility issues"
status: active
severity: MEDIUM
triggers:
  - "CQL error"
  - "syntax not supported"
  - "CQL compatibility"
  - "query syntax error"
  - "Cassandra CQL difference"
owner: devops-agent
objective: "Identify CQL compatibility gaps and provide Keyspaces-compatible alternatives"
context: "Keyspaces supports CQL 3.x with limitations. Some CQL statements, data types, and functions available in Apache Cassandra are not supported. BATCH is limited to single partition. TRUNCATE is not supported. Some aggregate functions have restrictions."
---

## Phase 1 — Triage

MUST:
- Identify the specific CQL statement causing the error
- Check if the statement uses unsupported syntax or features
- Verify data types used are supported by Keyspaces
- Check Keyspaces CQL reference for the specific operation

SHOULD:
- Compare the CQL version used by the client driver with Keyspaces compatibility
- Check if the query uses reserved keywords as identifiers
- Verify collection types (list, set, map) usage is within Keyspaces limits

MAY:
- Review the full application CQL schema for compatibility
- Test queries in cqlsh connected to Keyspaces endpoint

## Phase 2 — Remediate

MUST:
- Replace unsupported CQL syntax with Keyspaces-compatible alternatives
- Use quoted identifiers for reserved keyword column names
- Limit BATCH statements to single partition operations

SHOULD:
- Update client driver to a version compatible with Keyspaces
- Test all CQL operations against Keyspaces before production deployment
- Document CQL differences for the development team

MAY:
- Create a CQL compatibility checklist for migration projects
- Implement application-level workarounds for unsupported features

## Common Issues

- symptoms: "BATCH statement rejected — cross-partition batch"
  diagnosis: "Keyspaces only supports single-partition BATCH statements."
  resolution: "Split cross-partition batches into individual statements or single-partition batches."

- symptoms: "TRUNCATE TABLE not supported"
  diagnosis: "Keyspaces does not support TRUNCATE. This is a known limitation."
  resolution: "Delete rows using DELETE with partition key, or drop and recreate the table."

- symptoms: "Aggregate function not working"
  diagnosis: "Some aggregate functions (COUNT, SUM, AVG) have limitations in Keyspaces."
  resolution: "Use application-side aggregation or paginated queries with client-side computation."

## Output Format

```yaml
root_cause: "cql_compatibility — <specific_cause>"
evidence:
  - type: cql_statement
    content: "<the failing CQL statement>"
  - type: error_message
    content: "<specific error returned by Keyspaces>"
  - type: compatibility_gap
    content: "<unsupported feature identified>"
severity: MEDIUM
mitigation:
  immediate: "Replace unsupported CQL with compatible alternative"
  long_term: "Audit all CQL for Keyspaces compatibility"
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
