---
title: "D1 — Gateway Connectivity Issues"
description: "Diagnose SiteWise gateway connectivity failures"
status: active
severity: HIGH
triggers:
  - "gateway offline"
  - "gateway not syncing"
  - "data not flowing"
  - "gateway disconnected"
owner: devops-agent
objective: "Identify and resolve gateway connectivity failures"
context: "Gateway connectivity fails when: Greengrass core unhealthy, network blocked, OPC-UA server unreachable, or gateway component crashed."
---

## Phase 1 — Triage

MUST:
- Check gateway status: `aws iotsitewise describe-gateway --gateway-id <id>`
- Verify Greengrass core device health: `aws greengrassv2 get-core-device --core-device-thing-name <name>`
- Check gateway component logs on device
- Verify network connectivity to OPC-UA/Modbus server

SHOULD:
- Check gateway capability status for each data source
- Verify TES role has iotsitewise:BatchPutAssetPropertyValue permission
- Check firewall rules between gateway and data sources

MAY:
- Test OPC-UA connectivity manually from gateway device
- Check Greengrass deployment status for gateway components

## Phase 2 — Remediate

MUST:
- Fix Greengrass core device health first
- Restore network connectivity to data sources
- Fix TES role permissions

SHOULD:
- Configure gateway health monitoring
- Set up alerting for gateway disconnections

## Output Format

```yaml
root_cause: "gateway_connectivity — <specific_cause>"
evidence:
  - type: gateway_status
    content: "<gateway and capability status>"
severity: HIGH
mitigation:
  immediate: "Fix network or Greengrass health"
  long_term: "Implement gateway monitoring"
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
