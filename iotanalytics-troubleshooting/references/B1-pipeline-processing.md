---
title: "B1 — Pipeline Processing Errors"
description: "Diagnose pipeline message processing failures"
status: active
severity: HIGH
triggers:
  - "pipeline not processing"
  - "messages not reaching datastore"
  - "pipeline error"
owner: devops-agent
objective: "Identify and resolve pipeline processing failures"
context: "Processing fails when: activity configuration invalid, filter drops all messages, Lambda activity errors, or datastore doesn't exist."
---

## Phase 1 — Triage

MUST:
- Check pipeline configuration: `aws iotanalytics describe-pipeline --pipeline-name <name>`
- Verify pipeline has channel source and datastore sink
- Check CloudWatch pipeline activity metrics
- Test pipeline with sample data: `aws iotanalytics run-pipeline-activity --pipeline-activities '[<activity-json>]' --payloads '<base64-payload>'`

SHOULD:
- Check each activity for configuration errors
- Verify filter conditions aren't dropping all messages
- Check Lambda activity function for errors

MAY:
- Check pipeline reprocessing status if reprocessing was triggered
- Verify datastore exists and is active

## Phase 2 — Remediate

MUST:
- Fix activity configurations
- Ensure channel and datastore references are correct
- Fix Lambda function errors in Lambda activities

SHOULD:
- Test each activity individually with sample data
- Monitor pipeline processing rates

## Output Format

```yaml
root_cause: "pipeline_processing — <specific_cause>"
evidence:
  - type: pipeline_config
    content: "<pipeline activities>"
severity: HIGH
mitigation:
  immediate: "Fix activity configuration"
  long_term: "Implement pipeline monitoring"
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
