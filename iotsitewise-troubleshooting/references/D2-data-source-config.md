---
title: "D2 — Data Source Configuration"
description: "Diagnose gateway data source configuration issues"
status: active
severity: MEDIUM
triggers:
  - "OPC-UA not reading"
  - "data source error"
  - "tag not found"
  - "capability config error"
owner: devops-agent
objective: "Identify and resolve gateway data source configuration issues"
context: "Data source config issues occur when: OPC-UA endpoint wrong, authentication failed, node IDs invalid, or scan rate misconfigured."
---

## Phase 1 — Triage

MUST:
- Check capability configuration: `aws iotsitewise describe-gateway-capability-configuration --gateway-id <id> --capability-namespace "iotsitewise:opcuacollector:2"`
- Verify OPC-UA server endpoint URL and port
- Check authentication credentials (anonymous, username/password, certificate)
- Verify node IDs match OPC-UA server address space

SHOULD:
- Check scan rate and dead-band settings
- Verify property aliases map to correct SiteWise properties
- Test OPC-UA connection with a standalone client

MAY:
- Browse OPC-UA server address space to verify node IDs
- Check data source selection filter configuration

## Phase 2 — Remediate

MUST:
- Fix OPC-UA endpoint URL and authentication
- Correct node IDs to match server address space
- Update capability configuration via deployment

SHOULD:
- Set appropriate scan rates for data frequency
- Configure dead-band to reduce unnecessary data

## Output Format

```yaml
root_cause: "data_source_config — <specific_cause>"
evidence:
  - type: capability_config
    content: "<data source configuration>"
severity: MEDIUM
mitigation:
  immediate: "Fix endpoint or node configuration"
  long_term: "Document data source mappings"
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
