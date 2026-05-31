---
name: wickr-diagnostics
version: "1.0.0"
last_updated: "2025-04-12"
description: >
  Use this skill to investigate and troubleshoot AWS Wickr
  problems by analyzing network creation, admin console access,
  user provisioning, SSO integration, message delivery, file
  sharing, retention policies, compliance logging, bot integration,
  API access, federation, and following structured runbooks.
  Activate when: network creation failures, admin console issues,
  user provisioning errors, SSO problems, message delivery failures,
  file sharing issues, retention policy errors, compliance logging
  problems, bot integration failures, API access issues, federation
  problems, or the user says something is wrong with Wickr.
compatibility: >
  Requires AWS CLI or SDK access with wickr, iam, cloudtrail,
  cloudwatch, s3, and kms permissions.
---

# AWS Wickr Diagnostics

## When to use

Any AWS Wickr investigation — network creation, admin console access, user provisioning, SSO integration, message delivery, file sharing, retention policies, compliance logging, bot integration, API access, or federation issues.

## Investigation workflow

### Step 1 — Collect and triage

```
aws wickr list-networks --query 'networks[*].{NetworkId:networkId,Name:networkName,Status:status}'
aws wickr get-network --network-id <id>
aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventSource,AttributeValue=wickr.amazonaws.com --max-results 20
```

### Step 2 — Domain deep dive

```
aws wickr list-tags-for-resource --resource-arn <arn>
aws iam get-role --role-name <wickr-service-role>
aws s3api get-bucket-policy --bucket <wickr-retention-bucket>
```

### Step 3 — Detailed investigation

```
aws cloudwatch get-metric-statistics --namespace AWS/Wickr --metric-name MessageDeliveryFailures --dimensions Name=NetworkId,Value=<id> --start-time <start> --end-time <end> --period 300 --statistics Sum
aws kms describe-key --key-id <wickr-kms-key-id>
aws logs filter-log-events --log-group-name /aws/wickr/<network-id> --filter-pattern "ERROR"
```

Read `references/guardrails.md` before concluding on any Wickr issue.

## Tool quick reference

| Tool / API | When to use |
|------------|-------------|
| `wickr list-networks` | List Wickr networks |
| `wickr get-network` | Get network details |
| `wickr create-network` | Create new Wickr network |
| `wickr delete-network` | Delete Wickr network |
| `wickr list-tags-for-resource` | List resource tags |
| `iam get-role` | Check Wickr service role |
| `s3api get-bucket-policy` | Check retention bucket policy |
| `kms describe-key` | Check encryption key status |

## Gotchas: AWS Wickr

- AWS Wickr provides end-to-end encryption. AWS cannot access message content — even for troubleshooting. Do not expect AWS Support to read messages.
- Wickr networks are separate from VPCs. A Wickr network is a logical grouping of users and settings, not a networking construct.
- Admin console access requires IAM Identity Center (SSO) or SAML federation. Direct IAM user login to the admin console is not supported.
- Message retention requires explicit configuration. By default, messages are ephemeral and not retained. Retention must be enabled with an S3 bucket and KMS key.
- Bot integration uses the Wickr Bot SDK. Bots run as separate processes and connect to the Wickr network via API. They are not Lambda functions.
- Federation between Wickr networks allows cross-organization messaging but must be explicitly configured by both network admins.
- File sharing size limits and retention policies are configured at the network level. Individual users cannot override network-level settings.

## Anti-hallucination rules

1. Always cite specific network IDs, error messages, or API responses as evidence.
2. AWS cannot decrypt or access Wickr message content. Never suggest AWS Support can read messages.
3. Wickr networks are NOT VPCs. Never confuse Wickr networking with AWS VPC networking.
4. Message retention is opt-in. Never assume messages are stored by default.
5. Spend no more than 2 minutes on any single hypothesis. Pivot if inconclusive.

## 12 runbooks

| Category | IDs | Covers |
|----------|-----|--------|
| A — Network & Admin | A1–A2 | Network creation, admin console access |
| B — Users & Auth | B1–B2 | User provisioning, SSO integration |
| C — Messaging | C1–C2 | Message delivery failures, file sharing issues |
| D — Compliance | D1–D2 | Retention policy configuration, compliance logging |
| E — Integration | E1–E2 | Bot integration, API access |
| F — Federation | F1 | Federation issues |
| Z — Catch-All | Z1 | General troubleshooting |
