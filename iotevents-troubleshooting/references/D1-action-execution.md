---
title: "D1 — Action Execution Failures"
description: "Diagnose detector and alarm action execution failures"
status: active
severity: HIGH
triggers:
  - "action failed"
  - "SNS not sent"
  - "SQS not received"
  - "action error"
owner: devops-agent
objective: "Identify and resolve action execution failures"
context: "Action failures occur when: IAM role lacks permissions, target resource doesn't exist, payload too large, or target service throttling."
---

## Phase 1 — Triage

MUST:
- Check IAM role permissions: `aws iam list-attached-role-policies --role-name <role-name>`
- Verify target resource exists (SNS topic, SQS queue, Lambda function)
- Check CloudWatch for ActionExecutionError metrics
- Verify IAM role trust policy allows iotevents.amazonaws.com

SHOULD:
- Check action configuration in detector model
- Verify payload template syntax if using custom payloads
- Check target service CloudWatch logs

MAY:
- Test target resource manually (publish to SNS, send to SQS)
- Check Service Quotas for target service

## Phase 2 — Remediate

MUST:
- Add required permissions to IAM role (sns:Publish, sqs:SendMessage, lambda:InvokeFunction)
- Fix target resource ARN in action configuration
- Ensure IAM role trust policy includes iotevents.amazonaws.com

SHOULD:
- Configure DLQ on target services for failed deliveries
- Monitor action execution success rates

## Output Format

```yaml
root_cause: "action_execution — <specific_cause>"
evidence:
  - type: iam_role
    content: "<role permissions>"
severity: HIGH
mitigation:
  immediate: "Fix IAM permissions or target resource"
  long_term: "Implement action monitoring and DLQs"
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
- Never delete detector models that are actively monitoring production inputs
- Never modify alarm models during active alarm states without understanding impact
- Never change input routing configurations without verifying downstream detectors
- Never alter state transition logic on production detector models without testing
- Never remove action targets (Lambda, SNS, etc.) from active detector models

## Phase 3 — Rollback
- "Document current state before any changes"
- "If fix worsens the issue, revert to documented state"
- Republish previous detector model version if event processing breaks
- Restore previous alarm model configuration if alarm evaluation fails
- Revert input routing changes if detectors stop receiving data

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
