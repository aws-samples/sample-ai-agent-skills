---
title: "E1 — Recorder Configuration"
description: "Diagnose AWS Config recorder configuration and status issues"
status: active
severity: CRITICAL
triggers:
  - "recorder stopped"
  - "recorder not recording"
  - "configuration recorder"
  - "recorder error"
  - "recording disabled"
owner: devops-agent
objective: "Identify and resolve Config recorder issues to restore configuration item recording"
context: "The configuration recorder is the foundation of AWS Config. Without a running recorder, rules cannot evaluate, configuration items are not captured, and compliance data becomes stale. Recorder issues are the most common root cause of Config problems."
---

## Phase 1 — Triage

MUST:
- Check recorder status: `aws configservice describe-configuration-recorder-status`
- Describe recorder configuration: `aws configservice describe-configuration-recorders`
- Verify the recorder's IAM role: `aws iam get-role --role-name <recorder-role>`
- Check delivery channel status: `aws configservice describe-delivery-channel-status`
- Verify the S3 bucket exists and is accessible: `aws s3api head-bucket --bucket <config-bucket>`

SHOULD:
- Check CloudTrail for recorder stop/start events: `aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventName,AttributeValue=StopConfigurationRecorder --max-results 10`
- Verify the IAM role has the AWS managed policy: `aws iam list-attached-role-policies --role-name <recorder-role>`
- Check for recent configuration changes to the recorder

MAY:
- Check if an SCP is blocking Config recorder operations
- Verify the recorder role trust policy allows config.amazonaws.com

## Phase 2 — Remediate

MUST:
- Start the recorder if stopped: `aws configservice start-configuration-recorder --configuration-recorder-name <name>`
- Fix IAM role permissions if the role is misconfigured
- Ensure the delivery channel is valid and the S3 bucket is accessible

SHOULD:
- Investigate why the recorder was stopped (manual action, automation, SCP)
- Set up CloudWatch alarms for recorder status changes
- Verify recording settings match requirements (all resources vs specific types)

MAY:
- Enable recording of global resources if needed
- Set up AWS Config rules to monitor the recorder status itself

## Common Issues

- symptoms: "Recorder status shows recording: false"
  diagnosis: "The recorder was explicitly stopped by a user, automation, or SCP enforcement."
  resolution: "Start the recorder. Check CloudTrail to identify who/what stopped it and prevent recurrence."

- symptoms: "Recorder is running but no new configuration items appear"
  diagnosis: "The delivery channel is misconfigured or the S3 bucket is inaccessible."
  resolution: "Verify delivery channel status and S3 bucket accessibility. Check bucket policy allows Config."

- symptoms: "Recorder fails to start with AccessDenied"
  diagnosis: "The recorder's IAM role lacks required permissions or the trust policy is incorrect."
  resolution: "Attach the AWS_ConfigRole managed policy and verify the trust policy allows config.amazonaws.com."

## Safety Ratings

| Phase 2 Action | Rating | Rationale |
|---|---|---|
| Start recorder | YELLOW | Service enablement; reversible |
| Fix IAM role | YELLOW | Permission change; reversible |
| Fix delivery channel | YELLOW | Configuration change; reversible |

## Escalation Conditions

- Recorder configuration changes
- IAM role modifications for Config service
- Delivery channel changes

## Data Sensitivity

| Triage Command | Sensitivity | Rationale |
|---|---|---|
| `describe-configuration-recorder-status` | LOW | Recorder status |
| `describe-configuration-recorders` | LOW | Recorder configuration |
| `describe-delivery-channel-status` | LOW | Delivery status |
| CloudTrail events | MEDIUM | Configuration change history |

## Prohibited Actions

- NEVER suggest stopping Config recorder
- NEVER suggest deleting delivery channel
- NEVER suggest disabling organization rules

## Phase 3 — Rollback

| Remediation | Rollback Step |
|---|---|
| Recorder start | Stop recorder (not recommended) |
| IAM role fix | Revert IAM policy to previous version |
| Delivery channel fix | Revert delivery channel configuration |

## Output Format

```yaml
root_cause: "recorder_configuration — <specific_cause>"
evidence:
  - type: recorder_status
    content: "<describe-configuration-recorder-status output>"
  - type: recorder_config
    content: "<describe-configuration-recorders output>"
  - type: delivery_channel
    content: "<delivery channel status>"
severity: CRITICAL
mitigation:
  immediate: "Start the recorder and fix any configuration issues"
  long_term: "Monitor recorder status and prevent unauthorized stops"
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
  - "NEVER suggest disabling Config rules to fix compliance"
  - "NEVER suggest deleting the configuration recorder"
  - "NEVER remove remediation actions without review"
