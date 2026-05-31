---
name: macie-diagnostics
version: "1.0.0"
last_updated: "2025-04-12"
description: >
  Use this skill to investigate and troubleshoot Amazon Macie problems
  by analyzing discovery job failures, classification errors, custom data
  identifiers, finding analysis, false positives, S3 bucket inventory,
  sensitive data discovery, multi-account management, cost optimization,
  sampling configuration, Security Hub integration, and EventBridge
  notifications. Activate when: discovery jobs failing, classification
  inaccurate, custom identifiers not matching, unexpected findings,
  false positive management, S3 bucket inventory incomplete, sensitive
  data not detected, delegated admin issues, member account problems,
  high Macie costs, sampling configuration, Security Hub findings
  missing, EventBridge rules not triggering, or the user says something
  is wrong with Macie without naming specific symptoms.
compatibility: >
  Requires AWS CLI or SDK access with Macie2, S3, SecurityHub,
  EventBridge, IAM, and Organizations permissions. Multi-account
  features require Organizations access.
---

# Amazon Macie Diagnostics

## When to use

Any Amazon Macie investigation where the console alone is insufficient — discovery job failures, classification issues, finding analysis, multi-account management, cost optimization, or integration troubleshooting.

## Investigation workflow

### Step 1 — Collect and triage

```
aws macie2 get-macie-session
aws macie2 list-classification-jobs --filter-criteria '{"includes":{"key":"jobStatus","values":["RUNNING","PAUSED","USER_PAUSED"]}}'
aws macie2 list-findings --finding-criteria '{"criterion":{"severity.description":{"eq":["High"]}}}' --max-results 10
aws macie2 describe-buckets --query 'Buckets[*].{Name:BucketName,Classifiable:ClassifiableObjectCount,Sensitive:SensitiveDataOccurrences}'
version: "1.0.0"
last_updated: "2025-04-12"
```

### Step 2 — Domain deep dive

```
aws macie2 describe-classification-job --job-id <job-id>
aws macie2 get-finding-statistics --finding-criteria '{}' --group-by '{"key":"severity.description"}'
aws macie2 list-custom-data-identifiers
aws macie2 get-sensitive-data-occurrences --finding-id <finding-id>
```

### Step 3 — Detailed investigation

```
aws macie2 get-administrator-account
aws macie2 list-members
aws macie2 get-usage-totals
aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventSource,AttributeValue=macie2.amazonaws.com --max-results 20
```

Read `references/macie-guardrails.md` before concluding on any Macie issue.

## Tool quick reference

| Tool / API | When to use |
|------------|-------------|
| `get-macie-session` | Check Macie activation status |
| `list-classification-jobs` | List discovery jobs and their status |
| `describe-classification-job` | Get job details and configuration |
| `describe-buckets` | Check S3 bucket inventory and classification |
| `list-findings` | List sensitive data findings |
| `list-custom-data-identifiers` | Check custom data identifier configuration |
| `get-usage-totals` | Check Macie usage and costs |
| `list-members` | Check member accounts in multi-account |

## Gotchas: Amazon Macie

- Macie pricing is based on S3 objects evaluated, not findings generated. Scanning large buckets with many objects can be expensive. Use sampling to control costs. Check get-usage-totals regularly.
- Discovery jobs scan S3 objects for sensitive data. They are not real-time — they run on a schedule or one-time. New objects added after a job starts are not scanned until the next run.
- Custom data identifiers use regex patterns. Complex regex can cause timeouts or false matches. Test patterns thoroughly before deploying. Maximum regex length is 512 characters.
- Macie classifies data using managed and custom identifiers. Managed identifiers detect common PII (SSN, credit cards, etc.). Custom identifiers extend detection. Both contribute to findings.
- Findings are regional. Macie findings exist in the region where the S3 bucket is located. Use Security Hub aggregation for cross-region visibility.
- Bucket inventory shows all S3 buckets but only scans those included in jobs. Having buckets in inventory doesn't mean they are being scanned. Create discovery jobs to scan specific buckets.
- Sampling reduces cost by scanning a percentage of objects. Sampling may miss sensitive data in unsampled objects. Use 100% sampling for compliance-critical buckets.
- Macie must be enabled per region. It is not a global service. Enable in each region where S3 buckets contain potentially sensitive data.
- Suppression rules hide findings but don't delete them. Similar to Inspector, suppression is for accepted risks, not remediation.

## Anti-hallucination rules

1. Always cite specific job IDs, finding IDs, or API responses as evidence.
2. Macie scans S3 only. Never suggest Macie for non-S3 data stores.
3. Discovery jobs are not real-time. Never claim Macie provides continuous monitoring without scheduled jobs.
4. Pricing is per object evaluated. Never ignore cost implications of large-scale scanning.
5. Custom identifiers use regex. Never assume they support arbitrary pattern languages.
6. Spend no more than 2 minutes on any single hypothesis. Pivot if inconclusive.

## 20 runbooks

| Category | IDs | Covers |
|----------|-----|--------|
| A — Discovery | A1–A3 | Discovery job failures, classification errors, custom data identifiers |
| B — Findings | B1–B2 | Finding analysis, false positives |
| C — S3 | C1–C2 | Bucket inventory, sensitive data discovery |
| D — Multi-Account | D1–D2 | Delegated admin, member accounts |
| E — Cost | E1–E2 | Cost optimization, sampling |
| F — Integration | F1–F2 | Security Hub, EventBridge |
| Z — Catch-All | Z1 | General troubleshooting |
