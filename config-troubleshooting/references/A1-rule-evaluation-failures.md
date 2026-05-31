---
title: "A1 — Rule Evaluation Failures"
description: "Diagnose why AWS Config rules are failing to evaluate"
status: active
severity: HIGH
triggers:
  - "rule evaluation failed"
  - "rule not evaluating"
  - "compliance not updating"
  - "config rule error"
  - "evaluation timeout"
owner: devops-agent
objective: "Identify the root cause of Config rule evaluation failures and restore proper compliance evaluation"
context: "Config rules can fail to evaluate due to recorder being stopped, IAM permission issues, Lambda errors (custom rules), resource scope mismatches, or service limits. The rule evaluation status and CloudWatch metrics provide the most diagnostic value."
---

## Phase 1 — Triage

MUST:
- Check recorder status: `aws configservice describe-configuration-recorder-status`
- List rules and their state: `aws configservice describe-config-rules`
- Check rule evaluation status: `aws configservice describe-config-rule-evaluation-status --config-rule-names <rule-name>`
- Check compliance summary: `aws configservice describe-compliance-by-config-rule`
- Verify delivery channel: `aws configservice describe-delivery-channel-status`

SHOULD:
- Check CloudWatch metrics for evaluation failures: `aws cloudwatch get-metric-statistics --namespace AWS/Config --metric-name ConfigRulesEvaluationsFailed --start-time <start> --end-time <end> --period 3600 --statistics Sum`
- Check CloudTrail for Config events: `aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventSource,AttributeValue=config.amazonaws.com --max-results 20`
- Verify the Config service role: `aws iam get-role --role-name <config-role>`

MAY:
- Force re-evaluation: `aws configservice start-config-rules-evaluation --config-rule-names <rule-name>`
- Check service quotas for Config rules: `aws service-quotas get-service-quota --service-code config --quota-code L-8E14B865`

## Phase 2 — Remediate

MUST:
- Ensure the configuration recorder is running
- Fix IAM role permissions if the service role is misconfigured
- Verify the rule scope matches existing resource types

SHOULD:
- Re-evaluate the rule after fixing the issue
- Set up CloudWatch alarms for evaluation failures
- Review rule parameters for correctness

MAY:
- Enable detailed CloudTrail logging for Config API calls
- Create an SNS notification for rule compliance changes

## Common Issues

- symptoms: "Rule shows INSUFFICIENT_DATA for all resources"
  diagnosis: "The configuration recorder is stopped or the rule scope does not match any recorded resource types."
  resolution: "Start the recorder with start-configuration-recorder and verify the rule scope includes recorded resource types."

- symptoms: "Rule evaluation status shows error with Lambda ARN"
  diagnosis: "Custom rule Lambda function is failing, throttled, or has incorrect permissions."
  resolution: "Check Lambda CloudWatch logs, verify the function exists, and ensure the Config service role can invoke it."

- symptoms: "Rule evaluates but compliance never changes from NOT_APPLICABLE"
  diagnosis: "The rule scope resource type does not match any resources in the account."
  resolution: "Verify the resource type in the rule scope exists in the account and is being recorded."

## Safety Ratings

| Phase 2 Action | Rating | Rationale |
|---|---|---|
| Start Config recorder | YELLOW | Service enablement; reversible |
| Fix IAM role permissions | YELLOW | Permission change; reversible |
| Verify rule scope | GREEN | Read-only check |
| Force re-evaluation | GREEN | Triggers evaluation; no destructive change |

## Escalation Conditions

- Recorder configuration changes
- Remediation action deployment
- IAM role modifications for Config service

## Data Sensitivity

| Triage Command | Sensitivity | Rationale |
|---|---|---|
| `describe-configuration-recorder-status` | LOW | Recorder status |
| `describe-config-rules` | LOW | Rule configuration |
| `describe-config-rule-evaluation-status` | LOW | Evaluation status |
| `describe-compliance-by-config-rule` | LOW | Compliance summary |

## Prohibited Actions

- NEVER suggest stopping Config recorder
- NEVER suggest deleting delivery channel
- NEVER suggest disabling organization rules
- NEVER delete Config rules without understanding dependencies

## Phase 3 — Rollback

| Remediation | Rollback Step |
|---|---|
| Recorder start | Stop recorder (not recommended) |
| IAM role fix | Revert IAM policy to previous version |
| Rule scope change | Revert rule scope via `put-config-rule` |

## Output Format

```yaml
root_cause: "rule_evaluation_failure — <specific_cause>"
evidence:
  - type: recorder_status
    content: "<describe-configuration-recorder-status output>"
  - type: rule_evaluation
    content: "<describe-config-rule-evaluation-status output>"
  - type: iam_role
    content: "<role permissions analysis>"
severity: HIGH
mitigation:
  immediate: "Fix the identified issue and force re-evaluation"
  long_term: "Set up monitoring for rule evaluation failures"
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
