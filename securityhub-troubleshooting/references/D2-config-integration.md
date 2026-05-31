---
title: "D2 — Config Integration"
description: "Diagnose Security Hub integration with AWS Config"
status: active
severity: CRITICAL
triggers:
  - "config integration"
  - "config not working with security hub"
  - "config rules security hub"
  - "config dependency"
  - "controls need config"
owner: devops-agent
objective: "Identify and resolve AWS Config integration issues that affect Security Hub control evaluation"
context: "AWS Config is the most critical integration for Security Hub. Most security controls evaluate Config rules. Without Config properly configured, the majority of Security Hub controls show NO_DATA. This is the most common root cause of Security Hub issues."
---

## Phase 1 — Triage

MUST:
- Verify Config recorder is running: `aws configservice describe-configuration-recorder-status`
- Check Config delivery channel: `aws configservice describe-delivery-channel-status`
- Verify Config records all supported resource types: `aws configservice describe-configuration-recorders`
- Check Config rules created by Security Hub: `aws configservice describe-config-rules`
- Verify Config rule evaluation status: `aws configservice describe-config-rule-evaluation-status`

SHOULD:
- Check Security Hub controls for NO_DATA status: `aws securityhub describe-standards-controls --standards-subscription-arn <standard-arn>`
- Verify the Config service-linked role exists
- Check Config compliance summary: `aws configservice describe-compliance-by-config-rule`

MAY:
- Check CloudTrail for Config configuration changes
- Verify Config is enabled in all regions where Security Hub is active

## Phase 2 — Remediate

MUST:
- Start Config recorder if stopped: `aws configservice start-configuration-recorder --configuration-recorder-name <name>`
- Enable "Record all supported resource types" for comprehensive Security Hub coverage
- Enable global resource recording for IAM-related controls
- Fix delivery channel if misconfigured

SHOULD:
- Wait for initial Config evaluation after enabling (up to 24 hours)
- Verify Security Hub controls transition from NO_DATA to PASSED/FAILED

MAY:
- Set up monitoring for Config recorder status
- Create alerts for Config delivery failures

## Common Issues

- symptoms: "Most Security Hub controls show NO_DATA"
  diagnosis: "Config recorder is not running or not recording the required resource types."
  resolution: "Enable Config with 'Record all supported resource types' and global resources. Start the recorder."

- symptoms: "Some controls show NO_DATA while others work"
  diagnosis: "Config is recording some resource types but not all. The failing controls need resource types not being recorded."
  resolution: "Enable recording for all supported resource types, or add the specific missing types."

- symptoms: "Config is enabled but Security Hub controls still show NO_DATA"
  diagnosis: "Config rules may not have completed initial evaluation, or the delivery channel is failing."
  resolution: "Check delivery channel status. Wait for initial evaluation. Force re-evaluation if needed."

## Safety Ratings

| Phase 2 Action | Rating | Rationale |
|---|---|---|
| Start Config recorder | YELLOW | Service enablement; reversible |
| Enable all resource types | YELLOW | Configuration change; reversible |
| Fix delivery channel | YELLOW | Configuration change; reversible |

## Escalation Conditions

- Standard enablement/disablement
- Config recorder changes
- Cross-account configuration changes

## Data Sensitivity

| Triage Command | Sensitivity | Rationale |
|---|---|---|
| `describe-configuration-recorder-status` | LOW | Recorder status |
| `describe-delivery-channel-status` | LOW | Delivery status |
| `describe-standards-controls` | LOW | Control status |

## Prohibited Actions

- NEVER suggest disabling Security Hub
- NEVER suggest disabling all security standards
- NEVER suggest auto-archiving all findings

## Phase 3 — Rollback

| Remediation | Rollback Step |
|---|---|
| Config recorder start | Stop recorder (not recommended) |
| Resource type enablement | Revert to previous resource type list |
| Delivery channel fix | Revert delivery channel configuration |

## Output Format

```yaml
root_cause: "config_integration — <specific_cause>"
evidence:
  - type: config_recorder
    content: "<recorder status and configuration>"
  - type: delivery_channel
    content: "<delivery channel status>"
  - type: controls
    content: "<Security Hub control status>"
severity: CRITICAL
mitigation:
  immediate: "Enable Config with all resource types and start the recorder"
  long_term: "Monitor Config status and ensure it stays running"
```

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
  - "NEVER suggest disabling security standards"
  - "NEVER suppress findings without investigation"
  - "NEVER remove security integrations"
