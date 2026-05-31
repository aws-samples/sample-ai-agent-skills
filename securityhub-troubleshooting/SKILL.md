---
name: securityhub-diagnostics
version: "1.0.0"
last_updated: "2025-04-12"
description: >
  Use this skill to investigate and troubleshoot AWS Security Hub problems
  by analyzing security standards, findings, controls, integrations,
  multi-account aggregation, automation, and following structured runbooks.
  Activate when: CIS benchmark failures, PCI DSS compliance issues,
  NIST 800-53 control failures, finding aggregation problems, ASFF
  format errors, finding workflow issues, control evaluation failures,
  disabled controls, custom control problems, GuardDuty integration
  gaps, Config integration issues, third-party integration failures,
  cross-account aggregation problems, delegated administrator setup,
  custom action errors, automated remediation failures, cross-region
  aggregation issues, finding replication problems, compliance scoring
  gaps, reporting issues, or the user says something is wrong with
  Security Hub without naming specific symptoms.
compatibility: >
  Requires AWS CLI or SDK access with Security Hub, Config, GuardDuty,
  IAM, CloudWatch, CloudTrail, Organizations, EventBridge, Lambda,
  and optionally SNS and S3 permissions.
---

# AWS Security Hub Diagnostics

## When to use

Any Security Hub investigation where the console alone is insufficient — security standard compliance debugging, finding management, control evaluation troubleshooting, integration configuration, multi-account and cross-region aggregation, automated response setup, or compliance reporting issues.

## Investigation workflow

### Step 1 — Collect and triage

```
aws securityhub describe-hub
aws securityhub get-enabled-standards
aws securityhub describe-standards-controls --standards-subscription-arn <standard-arn>
aws securityhub get-findings --filters '{"ComplianceStatus":[{"Value":"FAILED","Comparison":"EQUALS"}]}' --max-results 20
aws securityhub get-insight-results --insight-arn <insight-arn>
aws securityhub describe-products
aws cloudwatch get-metric-statistics --namespace AWS/SecurityHub --metric-name SecurityChecks --start-time <start> --end-time <end> --period 3600 --statistics Sum
```

### Step 2 — Domain deep dive

```
aws securityhub batch-get-standards-control-associations --standards-control-association-ids <ids>
aws securityhub list-security-control-definitions --standards-arn <standard-arn>
aws securityhub list-standards-control-associations --security-control-id <control-id>
aws securityhub list-enabled-products-for-import
aws securityhub list-members
aws securityhub get-administrator-account
aws securityhub list-finding-aggregators
aws securityhub get-finding-aggregator --finding-aggregator-arn <aggregator-arn>
```

### Step 3 — Detailed investigation

```
aws securityhub list-automation-rules
aws securityhub batch-get-automation-rules --automation-rules-arns <rule-arns>
aws securityhub describe-action-targets
aws configservice describe-configuration-recorder-status
aws configservice describe-config-rules
aws guardduty list-detectors
aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventSource,AttributeValue=securityhub.amazonaws.com --max-results 20
```

Read `references/securityhub-guardrails.md` before concluding on any Security Hub issue.

## Tool quick reference

| Tool / API | When to use |
|------------|-------------|
| `describe-hub` | Security Hub configuration and status |
| `get-enabled-standards` | List enabled security standards |
| `describe-standards-controls` | Controls within a standard |
| `get-findings` | Query findings with filters |
| `get-insight-results` | Managed and custom insight results |
| `list-enabled-products-for-import` | Integrated product sources |
| `list-members` | Member accounts |
| `get-administrator-account` | Admin account for this member |
| `list-finding-aggregators` | Cross-region aggregation config |
| `list-automation-rules` | Automated finding processing rules |
| `describe-action-targets` | Custom action targets |
| `list-security-control-definitions` | Available security controls |
| `list-standards-control-associations` | Control-to-standard mappings |
| `batch-get-standards-control-associations` | Batch control association details |

## Gotchas: AWS Security Hub

- Security Hub requires Config to be enabled for most controls. Most Security Hub controls evaluate AWS Config rules under the hood. Without Config recorder running and recording the required resource types, controls show NO_DATA or UNKNOWN status. Always verify Config is enabled first.
- Controls evaluate Config rules — they are not independent checks. Security Hub controls are backed by Config rules. If the underlying Config rule fails or is not evaluating, the Security Hub control will not produce results. Control failures often trace back to Config issues.
- Finding deduplication uses ProductArn + Id. Each finding is uniquely identified by the combination of ProductArn and Id fields. Duplicate findings from the same product with the same Id are merged. Different products can have findings with the same Id without conflict.
- ASFF (AWS Security Finding Format) is the standard format. All findings in Security Hub must conform to ASFF. Third-party integrations must transform their findings to ASFF. Invalid ASFF format causes finding import failures. Required fields include SchemaVersion, Id, ProductArn, GeneratorId, AwsAccountId, Types, CreatedAt, UpdatedAt, Severity, Title, and Resources.
- Cross-region aggregation requires explicit setup. Findings from other regions are not automatically visible. A finding aggregator must be created in the aggregation region. The aggregator can link specific regions or all regions. Cross-region aggregation is separate from cross-account aggregation.
- Compliance score is the percentage of passed controls. The security score for each standard is calculated as (passed controls / total enabled controls) × 100. Disabled controls are excluded from the calculation. Controls with NO_DATA status are counted as not passed.
- Disabling a control does not delete existing findings. When a control is disabled, it stops generating new findings but existing findings remain. Findings from disabled controls must be manually archived or will age out. Re-enabling a control triggers fresh evaluation.
- Custom actions trigger EventBridge events. Custom actions allow manual or automated response to findings. When triggered, they send events to EventBridge with the selected findings. EventBridge rules can then invoke Lambda, SNS, or other targets. Custom actions require EventBridge rules to be useful.
- Security Hub has per-region pricing based on finding count and security checks. Pricing is based on the number of security checks (control evaluations) and finding ingestion events per month. Cross-region aggregation does not incur additional finding charges. Enabling many standards with many controls increases costs.

## Anti-hallucination rules

1. Always cite specific control IDs, finding IDs, or API responses as evidence.
2. Security Hub requires Config for most controls. Never assume controls work without Config.
3. Controls are backed by Config rules. Never troubleshoot controls without checking Config.
4. ASFF is required for all findings. Never suggest non-ASFF finding formats.
5. Cross-region aggregation requires explicit setup. Never assume findings are automatically global.
6. Spend no more than 2 minutes on any single hypothesis. Pivot if inconclusive.

## 26 runbooks

| Category | IDs | Covers |
|----------|-----|--------|
| A — Standards | A1-A3 | CIS benchmark failures, PCI DSS, NIST 800-53 |
| B — Findings | B1-B3 | Finding aggregation, ASFF format, finding workflow |
| C — Controls | C1-C3 | Control failures, disabled controls, custom controls |
| D — Integration | D1-D3 | GuardDuty integration, Config integration, third-party |
| E — Multi-Account | E1-E2 | Cross-account aggregation, delegated administrator |
| F — Automation | F1-F2 | Custom actions, automated remediation |
| G — Cross-Region | G1-G2 | Cross-region aggregation, finding replication |
| H — Compliance | H1-H2 | Compliance scoring, reporting |
| Z — Catch-All | Z1 | General troubleshooting |
