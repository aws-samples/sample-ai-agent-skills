---
name: inspector-diagnostics
description: >
  Use this skill to investigate and troubleshoot Amazon Inspector problems
  by analyzing EC2 scanning failures, ECR image scanning, Lambda function
  scanning, finding analysis, false positives, suppression rules, coverage
  gaps, SSM agent requirements, Security Hub integration, EventBridge
  notifications, multi-account management, scan frequency, and exclusion
  rules. Activate when: EC2 instances not being scanned, ECR images not
  scanned, Lambda functions not assessed, unexpected findings, false
  positive management, finding suppression issues, coverage gaps in
  accounts, SSM agent not reporting, Security Hub findings not appearing,
  EventBridge rules not triggering, delegated admin issues, member account
  problems, scan frequency configuration, exclusion rules not working,
  or the user says something is wrong with Inspector without naming
  specific symptoms.
compatibility: >
  Requires AWS CLI or SDK access with Inspector2, SSM, EC2, ECR, Lambda,
  SecurityHub, EventBridge, and IAM permissions. Multi-account features
  require Organizations access.
---

# Amazon Inspector Diagnostics

## When to use

Any Amazon Inspector investigation where the console alone is insufficient — scanning failures, coverage gaps, finding analysis, multi-account management, integration issues, or configuration troubleshooting.

## Investigation workflow

### Step 1 — Collect and triage

```
aws inspector2 list-coverage --filter-criteria '{"resourceType":[{"comparison":"EQUALS","value":"AWS_EC2_INSTANCE"}]}'
aws inspector2 get-findings-report-status
aws inspector2 list-findings --filter-criteria '{"findingStatus":[{"comparison":"EQUALS","value":"ACTIVE"}]}' --max-results 10
aws inspector2 batch-get-account-status
```

### Step 2 — Domain deep dive

```
aws inspector2 list-coverage-statistics
aws inspector2 list-account-permissions
aws inspector2 list-filters
aws inspector2 list-members --only-associated
```

### Step 3 — Detailed investigation

```
aws inspector2 get-member --account-id <account-id>
aws inspector2 get-delegated-admin-account
aws ssm describe-instance-information --query 'InstanceInformationList[*].{Id:InstanceId,PingStatus:PingStatus,AgentVersion:AgentVersion}'
aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventSource,AttributeValue=inspector2.amazonaws.com --max-results 20
```

Read `references/inspector-guardrails.md` before concluding on any Inspector issue.

## Tool quick reference

| Tool / API | When to use |
|------------|-------------|
| `batch-get-account-status` | Check Inspector activation status per scan type |
| `list-coverage` | Check which resources are being scanned |
| `list-coverage-statistics` | Get coverage summary by resource type |
| `list-findings` | List vulnerability findings with filters |
| `list-filters` | Check suppression and finding filters |
| `list-members` | Check member account status in multi-account |
| `get-delegated-admin-account` | Check delegated admin configuration |
| `ssm describe-instance-information` | Check SSM agent status for EC2 scanning |

## Gotchas: Amazon Inspector

- Inspector v2 is a different service from Inspector Classic. Inspector v2 uses inspector2 API namespace. Classic used inspector. They have different features, pricing, and scanning methods. Do not confuse the two.
- EC2 scanning requires SSM agent running and managed instance status. Without SSM agent, Inspector cannot scan EC2 instances for software vulnerabilities. The instance must appear in SSM Fleet Manager.
- ECR scanning is automatic for pushed images when enabled. Inspector scans images on push and rescans when new CVEs are published. Scanning is per-repository — check if the repository has scanning enabled.
- Lambda scanning covers both code and standard scanning. Code scanning analyzes Lambda function code for vulnerabilities. Standard scanning checks Lambda layers and runtimes. Both must be enabled separately.
- Findings are regional. Inspector findings exist in the region where the resource is located. Multi-region visibility requires enabling Inspector in each region or using Security Hub aggregation.
- Suppression rules filter findings from view but don't delete them. Suppressed findings still exist and count toward quotas. They can be unsuppressed. Use suppression for accepted risks, not for hiding problems.
- Inspector pricing is per-scan, not per-finding. EC2 scanning is priced per instance per month. ECR scanning is per image scan. Lambda scanning is per function per month. High finding counts don't increase cost.
- Coverage gaps mean resources exist but aren't being scanned. Check SSM agent status for EC2, repository scanning settings for ECR, and function configuration for Lambda.
- Delegated admin manages Inspector for the entire organization. Only one delegated admin per organization. The admin can enable/disable scanning for member accounts.

## Anti-hallucination rules

1. Always cite specific finding ARNs, instance IDs, or API responses as evidence.
2. Inspector v2 and Inspector Classic are different services. Never confuse them.
3. EC2 scanning requires SSM agent. Never assume instances are automatically scanned.
4. Suppression hides findings, it doesn't fix vulnerabilities. Never suggest suppression as remediation.
5. Findings are regional. Never assume cross-region visibility without aggregation.
6. Spend no more than 2 minutes on any single hypothesis. Pivot if inconclusive.

## 20 runbooks

| Category | IDs | Covers |
|----------|-----|--------|
| A — Scanning | A1–A3 | EC2 scanning failures, ECR scanning, Lambda scanning |
| B — Findings | B1–B3 | Finding analysis, false positives, suppression |
| C — Coverage | C1–C2 | Coverage gaps, SSM agent requirements |
| D — Integration | D1–D2 | Security Hub, EventBridge |
| E — Multi-Account | E1–E2 | Delegated admin, member accounts |
| F — Configuration | F1–F2 | Scan frequency, exclusion rules |
| Z — Catch-All | Z1 | General troubleshooting |
