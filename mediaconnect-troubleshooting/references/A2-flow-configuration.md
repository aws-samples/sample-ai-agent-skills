---
title: "A2 — Flow Configuration"
description: "Diagnose MediaConnect flow configuration and update issues"
status: active
severity: MEDIUM
triggers: ["flow misconfigured", "cannot update flow", "flow settings wrong", "UpdateFlow error"]
owner: devops-agent
objective: "Resolve flow configuration issues including source, output, and protocol settings"
context: "Flow configuration changes may require stopping the flow first. Some settings are immutable after creation."
---

## Phase 1 — Triage

MUST:
- Describe the flow: `aws mediaconnect describe-flow --flow-arn <flow-arn>`
- Check flow status — some updates require STANDBY: `aws mediaconnect describe-flow --flow-arn <flow-arn> --query 'Flow.Status'`
- Verify current source configuration: `aws mediaconnect describe-flow --flow-arn <flow-arn> --query 'Flow.Source'`

SHOULD:
- Compare current config against desired config
- Check if the flow has active outputs or entitlements that depend on current settings

MAY:
- Review CloudTrail for recent UpdateFlow events
- Check if maintenance mode is affecting the flow

## Phase 2 — Remediate

MUST:
- Stop the flow before making changes that require STANDBY: `aws mediaconnect stop-flow --flow-arn <flow-arn>`
- Update source settings: `aws mediaconnect update-flow-source --flow-arn <flow-arn> --source-arn <source-arn> --protocol <protocol>`
- Restart the flow after changes: `aws mediaconnect start-flow --flow-arn <flow-arn>`

SHOULD:
- Validate protocol-specific settings before applying (port ranges, CIDR blocks, encryption)
- Notify downstream consumers before stopping the flow

MAY:
- Use describe-flow to verify changes took effect
- Update flow tags for documentation: `aws mediaconnect tag-resource --resource-arn <flow-arn> --tags Key=Environment,Value=Production`

## Safety Ratings

```
safety_ratings:
  - "describe-flow: GREEN — read-only flow inspection"
  - "stop-flow: RED — interrupts live video delivery to all outputs and subscribers"
  - "update-flow-source: YELLOW — modifies source configuration, may affect ingest"
  - "start-flow: YELLOW — resumes video delivery, verify config before starting"
```

## Escalation Conditions

- Flow cannot be stopped due to ConflictException (stuck in transitional state)
- Configuration update fails with validation errors that don't match documentation
- Flow does not return to ACTIVE after restart following configuration change
- Multiple dependent flows (via entitlements) are affected by the configuration change

## Data Sensitivity

- `aws mediaconnect describe-flow`: MEDIUM — contains source IPs, encryption settings, VPC details
- `aws mediaconnect update-flow-source`: MEDIUM — modifies source configuration including IPs and encryption
- `aws mediaconnect stop-flow`: HIGH — impacts live video delivery
- `aws mediaconnect start-flow`: MEDIUM — resumes delivery with current configuration

## Prohibited Actions

- NEVER stop a production flow without confirming with the operations team
- NEVER change source protocol on a live production flow without a maintenance window
- NEVER modify encryption settings without verifying key availability first
- NEVER update source whitelist CIDR to 0.0.0.0/0 in production

## Phase 3 — Rollback

1. If configuration change caused issues, stop the flow: `aws mediaconnect stop-flow --flow-arn <flow-arn>`
2. Revert source settings to previous values: `aws mediaconnect update-flow-source --flow-arn <flow-arn> --source-arn <source-arn> --protocol <original-protocol>`
3. Restart the flow: `aws mediaconnect start-flow --flow-arn <flow-arn>`
4. Verify all outputs are delivering: `aws mediaconnect describe-flow --flow-arn <flow-arn> --query 'Flow.Outputs[*].{Name:Name,Status:Status}'`
5. Document the rollback and root cause for post-incident review

## Common Issues

- symptoms: "UpdateFlowSource returns ConflictException"
  diagnosis: "Flow is in a transitional state (STARTING or STOPPING)."
  resolution: "Wait for flow to reach ACTIVE or STANDBY before making changes. Poll status with describe-flow."

- symptoms: "Flow starts but source shows no bitrate"
  diagnosis: "Source configuration does not match the sender's protocol or network settings."
  resolution: "Verify protocol, port, and source whitelist CIDR match the upstream encoder configuration."

- symptoms: "Cannot change the availability zone of an existing flow"
  diagnosis: "AZ is immutable after flow creation."
  resolution: "Create a new flow in the desired AZ and migrate outputs and entitlements."

## Output Format

```yaml
root_cause: "flow_configuration — <specific_cause>"
evidence:
  - type: config_mismatch
    content: "<current vs desired configuration>"
severity: MEDIUM
mitigation:
  immediate: "Apply correct configuration and restart flow"
  long_term: "Maintain flow configuration in IaC with change management"
```

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
  - "NEVER suggest disabling call recording without compliance review"
  - "NEVER suggest removing authentication"
  - "NEVER expose customer PII in logs"
