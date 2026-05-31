---
title: "Z1 — General Greengrass Troubleshooting"
description: "Catch-all runbook for unclassified Greengrass issues"
status: active
severity: MEDIUM
triggers:
  - "Greengrass not working"
  - "something wrong with Greengrass"
  - "Greengrass error"
owner: devops-agent
objective: "Systematically triage unclassified Greengrass issues"
context: "Use this runbook when the issue doesn't clearly match a specific category."
---

## Phase 1 — Triage

MUST:
- Check core device status: `aws greengrassv2 get-core-device --core-device-thing-name <name>`
- List installed components: `aws greengrassv2 list-installed-components --core-device-thing-name <name>`
- Check effective deployments: `aws greengrassv2 list-effective-deployments --core-device-thing-name <name>`
- Review nucleus logs: `tail -200 /greengrass/v2/logs/greengrass.log`

SHOULD:
- Check CloudWatch for Greengrass metrics
- Verify IoT Core connectivity
- Check device system resources (CPU, memory, disk)

MAY:
- Review CloudTrail for recent Greengrass API calls: `aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventSource,AttributeValue=greengrass.amazonaws.com --max-results 20`
- Check component-specific logs in /greengrass/v2/logs/

## Phase 2 — Route to specific runbook

Based on triage findings, route to:
- A1-A3 for core device issues
- B1-B3 for deployment issues
- C1-C2 for Lambda component issues
- D1-D2 for MQTT bridge issues
- E1-E2 for stream manager issues
- F1-F2 for shadow issues
- G1-G2 for OTA update issues

## Output Format

```yaml
root_cause: "general — <routed_to_specific_runbook>"
evidence:
  - type: triage_summary
    content: "<initial findings>"
severity: MEDIUM
mitigation:
  immediate: "Follow specific runbook"
  long_term: "Implement comprehensive monitoring"
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
- Never remove core device provisioning without confirming all edge workloads are migrated
- Never force-deploy components to production core devices without staging validation
- Never delete component versions that are actively deployed to edge devices
- Never modify MQTT bridge configurations on devices running critical edge processing
- Never cancel OTA updates that are partially applied to production devices

## Phase 3 — Rollback
- "Document current state before any changes"
- "If fix worsens the issue, revert to documented state"
- Trigger a rollback deployment to restore previous component versions on core devices
- Restore previous MQTT bridge and topic mapping configurations if messaging breaks
- Revert nucleus configuration changes if core device connectivity is lost

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
