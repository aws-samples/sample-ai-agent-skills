---
title: "E2 — Batch Operation Issues"
description: "Diagnose batch get and put operation failures"
status: active
severity: MEDIUM
triggers:
  - "batch get failed"
  - "partial failure"
  - "entry limit exceeded"
  - "batch timeout"
owner: devops-agent
objective: "Identify and resolve batch operation failures"
context: "Batch operation issues occur when: entry limits exceeded, invalid asset/property IDs, time range too large, or response too large."
---

## Phase 1 — Triage

MUST:
- Verify entry count within limits (10 entries for put, varies for get)
- Check asset and property IDs are valid
- Verify time range for history queries
- Check error responses for specific entry failures

SHOULD:
- Check for partial success/failure in batch responses
- Verify data types match for put operations
- Check response size for large history queries

MAY:
- Test with single entry to isolate failures
- Check pagination for large result sets

## Phase 2 — Remediate

MUST:
- Reduce entries per batch to within limits
- Fix invalid asset/property IDs
- Narrow time ranges for history queries

SHOULD:
- Implement pagination for large result sets
- Handle partial failures in batch responses

## Output Format

```yaml
root_cause: "batch_operation — <specific_cause>"
evidence:
  - type: batch_response
    content: "<error entries>"
severity: MEDIUM
mitigation:
  immediate: "Fix entry count or IDs"
  long_term: "Implement robust batch handling"
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
- Never delete asset models that have active assets associated with them
- Never modify measurement properties on assets receiving live industrial data
- Never remove gateway configurations without confirming data source migration
- Never alter transform or metric computations on production assets without testing
- Never delete portal or dashboard configurations used by operations teams

## Phase 3 — Rollback
- "Document current state before any changes"
- "If fix worsens the issue, revert to documented state"
- Restore previous asset model configuration if property computations break
- Revert gateway data source settings if industrial data ingestion stops
- Re-enable previous alarm configurations if monitoring coverage is lost

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
