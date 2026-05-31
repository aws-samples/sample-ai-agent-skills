---
name: guardduty-diagnostics
version: "1.0.0"
last_updated: "2025-04-12"
description: >
  Use this skill to investigate and troubleshoot Amazon GuardDuty problems
  by analyzing findings, detector configuration, protection features,
  malware scanning, multi-account management, and following structured
  runbooks. Activate when: finding analysis issues, false positive
  management, suppression rule problems, detector configuration errors,
  data source issues, feature enablement failures, S3 protection gaps,
  EKS protection issues, Lambda protection problems, RDS protection
  errors, malware scan failures, delegated administrator setup,
  member account management, custom threat list issues, IP reputation
  problems, Security Hub export failures, EventBridge automation errors,
  runtime monitoring issues, agent deployment failures, or the user
  says something is wrong with GuardDuty without naming specific symptoms.
compatibility: >
  Requires AWS CLI or SDK access with GuardDuty, EC2, EKS, Lambda, RDS,
  S3, IAM, CloudWatch, CloudTrail, Organizations, Security Hub,
  EventBridge, and optionally SNS and SSM permissions.
---

# Amazon GuardDuty Diagnostics

## When to use

Any GuardDuty investigation where the console alone is insufficient — finding analysis, false positive management, detector configuration debugging, protection feature troubleshooting, malware scanning issues, multi-account management, threat intelligence configuration, integration problems, or runtime monitoring failures.

## Investigation workflow

### Step 1 — Collect and triage

```
aws guardduty list-detectors
aws guardduty get-detector --detector-id <detector-id>
aws guardduty list-findings --detector-id <detector-id> --finding-criteria '{"Criterion":{"severity":{"Gte":7}}}' --max-results 20
aws guardduty get-findings --detector-id <detector-id> --finding-ids <finding-id-1> <finding-id-2>
aws guardduty get-findings-statistics --detector-id <detector-id> --finding-statistic-types COUNT_BY_SEVERITY
aws guardduty list-filters --detector-id <detector-id>
aws cloudwatch get-metric-statistics --namespace AWS/GuardDuty --metric-name FindingsCount --start-time <start> --end-time <end> --period 3600 --statistics Sum
```

### Step 2 — Domain deep dive

```
aws guardduty get-detector --detector-id <detector-id>
aws guardduty list-members --detector-id <detector-id>
aws guardduty get-member-detectors --detector-id <detector-id> --account-ids <account-id>
aws guardduty list-publishing-destinations --detector-id <detector-id>
aws guardduty describe-publishing-destination --detector-id <detector-id> --destination-id <dest-id>
aws guardduty list-threat-intel-sets --detector-id <detector-id>
aws guardduty get-threat-intel-set --detector-id <detector-id> --threat-intel-set-id <set-id>
aws guardduty list-ip-sets --detector-id <detector-id>
```

### Step 3 — Detailed investigation

```
aws guardduty describe-malware-scans --detector-id <detector-id> --filter-criteria '{"FilterCriterion":[{"CriterionKey":"SCAN_STATUS","FilterCondition":{"EqualsValue":"COMPLETED"}}]}'
aws guardduty get-coverage-statistics --detector-id <detector-id> --statistics-type COUNT_BY_COVERAGE_STATUS
aws guardduty list-coverage --detector-id <detector-id>
aws guardduty get-organization-statistics --detector-id <detector-id>
aws guardduty describe-organization-configuration --detector-id <detector-id>
aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventSource,AttributeValue=guardduty.amazonaws.com --max-results 20
```

Read `references/guardduty-guardrails.md` before concluding on any GuardDuty issue.

## Tool quick reference

| Tool / API | When to use |
|------------|-------------|
| `list-detectors` | List detector IDs in the region |
| `get-detector` | Detector configuration and feature status |
| `list-findings` | List findings with optional criteria |
| `get-findings` | Full finding details |
| `get-findings-statistics` | Finding count by severity |
| `list-filters` | Suppression and filter rules |
| `list-members` | Member accounts in multi-account setup |
| `get-member-detectors` | Member account detector features |
| `list-publishing-destinations` | Finding export destinations |
| `list-threat-intel-sets` | Custom threat intelligence lists |
| `list-ip-sets` | Trusted IP lists |
| `describe-malware-scans` | Malware scan history and results |
| `get-coverage-statistics` | Runtime monitoring coverage |
| `list-coverage` | Per-resource runtime coverage |
| `describe-organization-configuration` | Organization-wide settings |

## Gotchas: Amazon GuardDuty

- GuardDuty analyzes VPC Flow Logs, DNS logs, and CloudTrail WITHOUT requiring you to enable them separately. GuardDuty uses independent data streams — you do not need to enable VPC Flow Logs or CloudTrail logging for GuardDuty to work. However, enabling them independently is still recommended for your own analysis.
- Findings are regional — there is no global view without aggregation. Each region has its own detector and findings. To get a centralized view, use Security Hub cross-region aggregation or export findings to a central S3 bucket.
- Suppression rules hide findings but do not delete them. Suppressed findings are auto-archived and do not appear in the default findings list, but they still exist and count toward finding limits. Suppression rules use filter criteria to match findings.
- S3 protection analyzes CloudTrail S3 data events. It detects suspicious S3 API activity by monitoring CloudTrail S3 data events. This is a separate data source from the default CloudTrail management events. S3 protection has its own pricing.
- EKS protection requires EKS audit logs. EKS Audit Log Monitoring analyzes Kubernetes audit logs from EKS clusters. EKS Runtime Monitoring requires the GuardDuty agent deployed on EKS nodes. These are separate features with separate pricing.
- Runtime monitoring requires an agent deployed on nodes. For EC2, EKS, and ECS Fargate, runtime monitoring requires the GuardDuty security agent. The agent can be auto-managed or manually deployed. Without the agent, runtime findings are not generated.
- Malware scanning creates EBS snapshots which incur cost. When GuardDuty detects suspicious activity, it can trigger malware scanning by creating EBS volume snapshots. These snapshots incur standard EBS snapshot costs. Scans have size limits and timeout constraints.
- Delegated admin manages all member accounts. The delegated administrator account can enable/disable features, manage findings, and configure settings for all member accounts. Member accounts cannot override delegated admin settings.
- Findings auto-archive after 90 days. Active findings that are not updated for 90 days are automatically archived. Archived findings are retained for the same period. Use finding export to S3 for long-term retention beyond the archive period.

## Anti-hallucination rules

1. Always cite specific finding IDs, detector IDs, or API responses as evidence.
2. GuardDuty does NOT require VPC Flow Logs or CloudTrail to be enabled. Never suggest enabling them as a prerequisite for GuardDuty.
3. Findings are regional. Never claim GuardDuty provides a global view without aggregation.
4. Suppression rules hide but do not delete findings. Never suggest suppression as deletion.
5. Runtime monitoring requires agent deployment. Never assume runtime findings work without the agent.
6. Spend no more than 2 minutes on any single hypothesis. Pivot if inconclusive.

## 26 runbooks

| Category | IDs | Covers |
|----------|-----|--------|
| A — Findings | A1-A3 | Finding analysis, false positives, suppression rules |
| B — Detector | B1-B3 | Detector configuration, data sources, feature enablement |
| C — Protection | C1-C4 | S3 protection, EKS protection, Lambda protection, RDS protection |
| D — Malware | D1-D2 | Malware scanning, scan failures |
| E — Multi-Account | E1-E2 | Delegated administrator, member accounts |
| F — Threat Intel | F1-F2 | Custom threat lists, IP reputation |
| G — Integration | G1-G2 | Security Hub export, EventBridge automation |
| H — Runtime | H1-H2 | Runtime monitoring, agent deployment |
| Z — Catch-All | Z1 | General troubleshooting |
