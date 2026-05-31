---
title: "B2 — Connector Configuration Issues"
description: "Diagnose data source connector setup and configuration"
status: active
severity: MEDIUM
triggers:
  - "connector creation failed"
  - "data source config error"
  - "cannot connect to source"
owner: devops-agent
objective: "Resolve connector configuration issues"
context: "Each connector type has specific requirements for credentials, endpoints, and scope."
---

## Phase 1 — Triage

MUST:
- Check data source config: `aws qbusiness get-data-source --application-id <app-id> --index-id <idx-id> --data-source-id <ds-id>`
- Verify connector type and configuration
- Check Secrets Manager for credentials
- Verify IAM role permissions

SHOULD:
- Test connectivity to source system
- Verify crawl scope settings
- Check for connector-specific requirements

## Phase 2 — Remediate

MUST:
- Fix connector configuration
- Update credentials
- Correct endpoint URLs

SHOULD:
- Test with minimal scope first
- Document connector requirements

## Common Issues

- symptoms: "Data source creation fails with InvalidConfiguration"
  diagnosis: "Missing or invalid connector parameters."
  resolution: "Check connector-specific documentation. Provide all required fields."

## Output Format

```yaml
root_cause: "connector_config — <specific_cause>"
evidence:
  - type: connector_details
    content: "<connector configuration>"
severity: MEDIUM
mitigation:
  immediate: "Fix connector configuration"
  long_term: "Template connector configs"
```


## Safety Ratings
- Overall Risk: MEDIUM
- Data Exposure Risk: HIGH — connector configuration controls data source access
- Service Disruption Risk: LOW — configuration issues do not affect existing synced data
- Compliance Impact: HIGH — connector scope must comply with data access policies

## Escalation Conditions
- Connector configuration prevents data sync
- Configuration changes expose unauthorized data
- Connector scope too broad or too narrow
- Configuration validation fails
- Multiple connector configurations fail

## Data Sensitivity
- Data source connection details — RESTRICTED
- Connector scope configurations — CONFIDENTIAL
- Authentication settings — RESTRICTED
- Data filtering rules — INTERNAL
- Never expose data source credentials in configurations

## Prohibited Actions
- Do NOT configure connectors with overly broad data access
- Do NOT include credentials in connector configurations
- Do NOT bypass connector configuration review
- Do NOT ignore configuration validation errors
- Do NOT share connector configurations externally
- Do NOT configure connectors without data governance approval

## Phase 3 — Rollback
- Restore previous connector configurations
- Revert scope changes
- Restore authentication settings
- Verify connector functionality after changes
- Document configuration issues
- Update connector configuration procedures

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
