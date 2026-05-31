---
title: "Z1 — General IoT Analytics Troubleshooting"
description: "Catch-all runbook for unclassified IoT Analytics issues"
status: active
severity: MEDIUM
triggers:
  - "IoT Analytics not working"
  - "something wrong with IoT Analytics"
owner: devops-agent
objective: "Systematically triage unclassified IoT Analytics issues"
---

## Phase 1 — Triage

MUST:
- List channels: `aws iotanalytics list-channels`
- List pipelines: `aws iotanalytics list-pipelines`
- List datastores: `aws iotanalytics list-datastores`
- List datasets: `aws iotanalytics list-datasets`

SHOULD:
- Check CloudWatch IoT Analytics metrics
- Review CloudTrail for recent errors
- Verify IAM permissions

MAY:
- Check service health in the region
- Review Service Quotas usage

## Phase 2 — Route to specific runbook

Based on findings, route to A1-A2, B1-B3, C1-C2, D1-D3, or E1-E2.

## Output Format

```yaml
root_cause: "general — <routed_to_specific_runbook>"
evidence:
  - type: triage_summary
    content: "<initial findings>"
severity: MEDIUM
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
