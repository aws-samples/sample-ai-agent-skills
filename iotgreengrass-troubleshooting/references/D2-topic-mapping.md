---
title: "D2 — Topic Mapping Issues"
description: "Diagnose MQTT topic mapping and message routing issues"
status: active
severity: MEDIUM
triggers:
  - "topic not mapped"
  - "messages lost"
  - "wrong topic"
  - "wildcard not matching"
owner: devops-agent
objective: "Identify and resolve MQTT topic mapping issues"
context: "Topic mapping issues occur when: wildcards misconfigured, direction wrong, topic filter doesn't match published topics, or mapping syntax invalid."
---

## Phase 1 — Triage

MUST:
- Review topic mapping configuration in bridge component
- Verify source and target topic patterns match expected topics
- Check direction: LocalMqtt→IotCore or IotCore→LocalMqtt
- Test with specific topic (no wildcards) to isolate issue

SHOULD:
- Verify wildcard usage: + for single level, # for multi-level
- Check if IoT Core policy allows the target topics
- Review bridge logs for mapping errors

MAY:
- Use mosquitto_pub/sub locally to test message flow
- Check message payload format compatibility

## Phase 2 — Remediate

MUST:
- Fix topic patterns to match actual published topics
- Correct direction in mapping configuration
- Update IoT Core policy to allow bridged topics

SHOULD:
- Test mappings with explicit topics before using wildcards
- Document topic naming conventions

## Output Format

```yaml
root_cause: "topic_mapping — <specific_cause>"
evidence:
  - type: mapping_config
    content: "<topic mapping details>"
severity: MEDIUM
mitigation:
  immediate: "Fix topic patterns and direction"
  long_term: "Standardize topic naming and document mappings"
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
