---
title: "B2 — Pipeline Activity Failures"
description: "Diagnose individual pipeline activity failures"
status: active
severity: MEDIUM
triggers:
  - "Lambda activity failed"
  - "math activity error"
  - "enrichment failed"
  - "filter error"
owner: devops-agent
objective: "Identify and resolve individual pipeline activity failures"
context: "Activity failures occur when: Lambda function errors, math expression invalid, enrichment permissions missing, or attribute names wrong."
---

## Phase 1 — Triage

MUST:
- Identify failing activity in pipeline definition
- Test activity with sample data: `aws iotanalytics run-pipeline-activity --pipeline-activities '[<activity>]' --payloads '<base64>'`
- For Lambda: check function logs and permissions
- For enrichment: verify IoT Core permissions

SHOULD:
- Check attribute names match message payload
- Verify math expressions for syntax errors
- Check filter conditions for logic errors

MAY:
- Test with minimal payload to isolate issue
- Check activity-specific CloudWatch metrics

## Phase 2 — Remediate

MUST:
- Fix Lambda function errors or permissions
- Correct math expression syntax
- Grant IoT Core permissions for enrichment activities

SHOULD:
- Test activities individually before combining in pipeline
- Add error logging in Lambda activities

## Output Format

```yaml
root_cause: "activity_failure — <specific_cause>"
evidence:
  - type: activity_test
    content: "<test result>"
severity: MEDIUM
mitigation:
  immediate: "Fix activity configuration"
  long_term: "Test activities before deployment"
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
