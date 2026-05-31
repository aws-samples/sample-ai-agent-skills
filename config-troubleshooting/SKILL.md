---
name: config-diagnostics
version: "1.0.0"
last_updated: "2025-04-12"
description: >
  Use this skill to investigate and troubleshoot AWS Config problems
  by analyzing rule evaluations, conformance packs, aggregators,
  remediation actions, recorder configuration, and following structured
  runbooks. Activate when: rule evaluation failures, custom rule Lambda
  errors, managed rule issues, conformance pack deployment failures,
  organization conformance problems, aggregator authorization issues,
  cross-account aggregation failures, auto-remediation failures, SSM
  automation errors, recorder configuration problems, delivery channel
  issues, resource type recording gaps, advanced query errors, query
  performance issues, organization rule deployment, delegated admin
  setup, Security Hub integration, S3 delivery failures, or the user
  says something is wrong with AWS Config without naming specific symptoms.
compatibility: >
  Requires AWS CLI or SDK access with AWS Config, Lambda, SSM, S3, SNS,
  IAM, CloudWatch, CloudTrail, Organizations, KMS, and optionally
  Security Hub permissions.
---

# AWS Config Diagnostics

## When to use

Any AWS Config investigation where the console alone is insufficient — rule evaluation debugging, conformance pack troubleshooting, aggregator configuration, remediation automation failures, recorder issues, delivery channel problems, advanced query errors, multi-account setup, or integration issues.

## Investigation workflow

### Step 1 — Collect and triage

```
aws configservice describe-config-rules
aws configservice describe-compliance-by-config-rule
aws configservice get-compliance-details-by-config-rule --config-rule-name <rule-name>
aws configservice describe-configuration-recorders
aws configservice describe-configuration-recorder-status
aws configservice describe-delivery-channels
aws configservice describe-delivery-channel-status
aws cloudwatch get-metric-statistics --namespace AWS/Config --metric-name ConfigRulesEvaluationsFailed --start-time <start> --end-time <end> --period 3600 --statistics Sum
```

### Step 2 — Domain deep dive

```
aws configservice describe-config-rule-evaluation-status --config-rule-names <rule-name>
aws configservice describe-conformance-packs
aws configservice describe-conformance-pack-status
aws configservice describe-configuration-aggregators
aws configservice describe-aggregation-authorizations
aws configservice describe-remediation-configurations --config-rule-names <rule-name>
aws configservice describe-remediation-execution-status --config-rule-name <rule-name>
```

### Step 3 — Detailed investigation

```
aws configservice select-resource-config --expression "SELECT resourceId, resourceType WHERE resourceType = '<type>'"
aws configservice get-resource-config-history --resource-type <type> --resource-id <id>
aws configservice describe-organization-config-rules
aws configservice describe-organization-conformance-packs
aws lambda get-function --function-name <custom-rule-function>
aws ssm describe-automation-executions --filters Key=ExecutionId,Values=<execution-id>
aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventSource,AttributeValue=config.amazonaws.com --max-results 20
```

Read `references/config-guardrails.md` before concluding on any AWS Config issue.

## Tool quick reference

| Tool / API | When to use |
|------------|-------------|
| `describe-config-rules` | List all Config rules and their state |
| `describe-compliance-by-config-rule` | Compliance summary per rule |
| `get-compliance-details-by-config-rule` | Detailed compliance per resource |
| `describe-configuration-recorders` | Recorder configuration and status |
| `describe-delivery-channels` | Delivery channel configuration |
| `describe-config-rule-evaluation-status` | Rule evaluation timing and errors |
| `describe-conformance-packs` | Conformance pack details |
| `describe-configuration-aggregators` | Aggregator configuration |
| `describe-aggregation-authorizations` | Cross-account authorizations |
| `describe-remediation-configurations` | Remediation setup per rule |
| `describe-remediation-execution-status` | Remediation execution results |
| `select-resource-config` | Advanced SQL queries on config data |
| `get-resource-config-history` | Resource configuration timeline |
| `describe-organization-config-rules` | Organization-wide rules |
| `describe-organization-conformance-packs` | Organization conformance packs |

## Gotchas: AWS Config

- Config recorder must be ON for rules to evaluate. If the recorder is stopped, rules will not receive configuration change events and periodic rules will not trigger. Always verify recorder status first.
- Delivery channel is required for recording. Without a valid delivery channel (S3 bucket + optional SNS topic), the recorder cannot persist configuration items. A misconfigured delivery channel silently prevents recording.
- Not all resource types are recorded by default. When "Record all resources" is selected, it covers most but not all types. Some resource types require explicit inclusion. Third-party resources require custom schemas.
- Custom rules use Lambda and inherit Lambda limits. Custom rules invoke Lambda functions — they are subject to Lambda concurrency limits, timeout (max 15 min), memory limits, and cold starts. Lambda throttling causes rule evaluation failures.
- Conformance packs are CloudFormation-based. They deploy as CloudFormation stacks. Stack failures, parameter errors, and template size limits (50 rules max per pack) apply. Organization conformance packs deploy to all member accounts.
- Remediation uses SSM Automation documents. Auto-remediation executes SSM Automation runbooks. The Config service role must have ssm:StartAutomationExecution permission. Remediation failures appear in SSM Automation execution history, not Config console.
- Aggregator requires authorization from source accounts. Multi-account aggregators need explicit authorization from each source account (or use Organizations). Without authorization, the aggregator shows no data from that account.
- Advanced queries use SQL-like syntax against configuration items. Queries run against the current configuration snapshot, not historical data. Complex queries may time out. The SELECT clause supports specific fields only — not all resource attributes are queryable.
- Config costs per configuration item recorded and rule evaluation. Each configuration item recorded costs money. Each rule evaluation costs money. High-frequency changes on many resources can lead to unexpected costs. Use resource type filtering to control costs.

## Anti-hallucination rules

1. Always cite specific rule names, evaluation results, or API responses as evidence.
2. Config recorder must be running for rules to evaluate. Never assume rules work without verifying recorder status.
3. Delivery channel must be valid for recording to persist. Never skip delivery channel verification.
4. Custom rules are Lambda functions with Lambda limits. Never ignore Lambda-side errors.
5. Conformance packs have a 50-rule limit per pack. Never suggest exceeding this.
6. Spend no more than 2 minutes on any single hypothesis. Pivot if inconclusive.

## 26 runbooks

| Category | IDs | Covers |
|----------|-----|--------|
| A — Rules | A1-A3 | Rule evaluation failures, custom rule Lambda errors, managed rule issues |
| B — Conformance | B1-B2 | Conformance pack failures, organization conformance |
| C — Aggregator | C1-C2 | Aggregator issues, cross-account authorization |
| D — Remediation | D1-D2 | Auto-remediation failures, SSM automation errors |
| E — Recorder | E1-E3 | Recorder configuration, delivery channel, resource types |
| F — Advanced Query | F1-F2 | Query errors, query performance |
| G — Multi-Account | G1-G2 | Organization rules, delegated administrator |
| H — Integration | H1-H2 | Security Hub integration, S3 delivery |
| Z — Catch-All | Z1 | General troubleshooting |
