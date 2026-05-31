---
title: "B3 — Transform Activity Issues"
description: "Diagnose data transformation activity problems"
status: active
severity: MEDIUM
triggers:
  - "transform not working"
  - "attribute missing after transform"
  - "data format wrong"
owner: devops-agent
objective: "Identify and resolve transform activity issues"
context: "Transform issues occur when: addAttributes expression wrong, removeAttributes removes needed fields, selectAttributes misses fields, or math expression invalid."
---

## Phase 1 — Triage

MUST:
- Review transform activity configuration in pipeline
- Test transform with sample data using run-pipeline-activity
- Verify attribute names in addAttributes/removeAttributes/selectAttributes
- Check math expression syntax for math activities

SHOULD:
- Verify input message has expected attributes
- Check if previous activities modified the message structure
- Verify output matches expected format for next activity

MAY:
- Compare input and output of transform activity
- Check for attribute name case sensitivity issues

## Phase 2 — Remediate

MUST:
- Fix attribute names in transform activities
- Correct math expression syntax
- Ensure selectAttributes includes all needed fields

SHOULD:
- Document expected message format at each pipeline stage
- Test transforms with edge case data

## Output Format

```yaml
root_cause: "transform_activity — <specific_cause>"
evidence:
  - type: transform_test
    content: "<input vs output>"
severity: MEDIUM
mitigation:
  immediate: "Fix transform configuration"
  long_term: "Document pipeline data flow"
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
- Never delete channels or datastores that are actively receiving data
- Never modify pipeline activities on production pipelines without testing
- Never change data retention settings without understanding downstream impact
- Never delete datasets that feed dashboards or reporting systems
- Never alter SQL query definitions in scheduled datasets without validation

## Phase 3 — Rollback
- "Document current state before any changes"
- "If fix worsens the issue, revert to documented state"
- Restore previous pipeline activity configuration if data processing breaks
- Revert channel or datastore settings if data ingestion stops
- Re-enable previous dataset scheduling if automated analysis is disrupted

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
