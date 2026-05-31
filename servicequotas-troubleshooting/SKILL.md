---
name: servicequotas-diagnostics
version: "1.0.0"
last_updated: "2025-04-12"
description: >
  Use this skill to investigate and troubleshoot AWS Service Quotas
  problems by analyzing quota lookups, quota increase requests, applied
  vs default quotas, CloudWatch quota alarms, utilization monitoring,
  organization quotas, quota templates, common service-specific quota
  issues (EC2, Lambda, VPC), quota request automation, and the Service
  Quotas API. Activate when: hitting service limits, quota increase
  request issues, quota alarm configuration, utilization monitoring,
  organization-wide quota management, EC2 instance limits, Lambda
  concurrency limits, VPC resource limits, automated quota management,
  or the user says something is wrong with service quotas without
  naming specific symptoms.
compatibility: >
  Requires AWS CLI or SDK access with ServiceQuotas, CloudWatch,
  EC2, Lambda, and optionally Organizations permissions.
---

# AWS Service Quotas Diagnostics

## When to use

Any AWS Service Quotas investigation where the console alone is insufficient — quota lookups, increase requests, utilization monitoring, organization quotas, service-specific limits, or automated quota management.

## Investigation workflow

### Step 1 — Collect and triage

```
aws service-quotas list-services
aws service-quotas get-service-quota --service-code <service> --quota-code <quota>
aws service-quotas list-service-quotas --service-code <service>
aws service-quotas get-aws-default-service-quota --service-code <service> --quota-code <quota>
```

### Step 2 — Domain deep dive

```
aws service-quotas list-requested-service-quota-change-history --service-code <service>
aws service-quotas get-requested-service-quota-change --request-id <id>
aws cloudwatch get-metric-statistics --namespace AWS/Usage --metric-name ResourceCount --dimensions Name=Type,Value=Resource Name=Resource,Value=<resource> Name=Service,Value=<service> Name=Class,Value=None --start-time <start> --end-time <end> --period 3600 --statistics Maximum
aws service-quotas list-service-quota-increase-requests-in-template
```

### Step 3 — Detailed investigation

```
aws service-quotas get-association-for-service-quota-template
aws service-quotas list-tags-for-resource --resource-arn <quota-arn>
aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventSource,AttributeValue=servicequotas.amazonaws.com --max-results 20
```

Read `references/servicequotas-guardrails.md` before concluding on any Service Quotas issue.

## Tool quick reference

| Tool / API | When to use |
|------------|-------------|
| `get-service-quota` | Get current applied quota value |
| `get-aws-default-service-quota` | Get default quota value |
| `list-service-quotas` | List all quotas for a service |
| `request-service-quota-increase` | Request a quota increase |
| `list-requested-service-quota-change-history` | Check increase request status |
| `list-service-quota-increase-requests-in-template` | Check quota templates |
| CloudWatch AWS/Usage metrics | Monitor quota utilization |
| `get-association-for-service-quota-template` | Check org template association |

## Gotchas: AWS Service Quotas

- Not all quotas are adjustable. Some quotas are hard limits that cannot be increased. Check the Adjustable field in get-service-quota. Non-adjustable quotas require architectural changes.
- Applied quota may differ from default. If a quota increase was previously granted, the applied value differs from the default. Always check get-service-quota for the current applied value, not get-aws-default-service-quota.
- Quota increase requests can take days. Requests are reviewed by AWS service teams. Some increases are automatic (within certain limits), others require manual review. Check request status regularly.
- CloudWatch Usage metrics are not available for all quotas. Only some services publish usage metrics to CloudWatch. Check if the specific quota has a UsageMetric in the quota details.
- Organization quota templates apply to new accounts only. Templates set default quotas for new accounts joining the organization. They do not retroactively change quotas in existing accounts.
- Some quotas are regional, others are global. EC2 instance limits are per-region. IAM user limits are global. Always check the correct region when looking up quotas.
- Service Quotas API may not cover all services. Some older services manage quotas through their own APIs or support tickets. Check if the service is listed in list-services.
- Quota utilization can spike temporarily. A brief spike above quota may succeed if the service allows burst. Sustained usage above quota will be throttled or denied.
- EC2 On-Demand instance limits are per instance family. vCPU-based limits apply per instance family (Standard, Accelerated, etc.), not per instance type. Running 100 t3.micro uses 200 vCPUs from the Standard family quota.

## Anti-hallucination rules

1. Always cite specific quota codes, service codes, or API responses as evidence.
2. Applied quota may differ from default. Never assume default values are current.
3. Not all quotas are adjustable. Never promise an increase for non-adjustable quotas.
4. Quota increase requests are not instant. Never claim immediate approval.
5. Organization templates affect new accounts only. Never claim retroactive application.
6. Spend no more than 2 minutes on any single hypothesis. Pivot if inconclusive.

## 16 runbooks

| Category | IDs | Covers |
|----------|-----|--------|
| A — Quotas | A1–A3 | Quota lookup, quota increase requests, applied vs default |
| B — Alarms | B1–B2 | CloudWatch quota alarms, utilization monitoring |
| C — Organization | C1–C2 | Organization quotas, quota templates |
| D — Service-Specific | D1–D3 | EC2 limits, Lambda limits, VPC limits |
| E — Automation | E1–E2 | Quota request automation, Service Quotas API |
| Z — Catch-All | Z1 | General troubleshooting |
