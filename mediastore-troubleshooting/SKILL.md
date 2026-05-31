---
name: mediastore-diagnostics
version: "1.0.0"
last_updated: "2025-04-12"
description: >
  Use this skill to investigate and troubleshoot AWS Elemental
  MediaStore problems by analyzing container creation, policies,
  object operations, CORS, lifecycle policies, access logging,
  metric policies, CDN integration, origin endpoints, throttling,
  and following structured runbooks.
  Activate when: container creation failures, policy errors, put/get
  object failures, CORS issues, lifecycle policy problems, logging
  configuration, metric policy errors, CDN integration issues,
  origin endpoint problems, throttling, or the user says something
  is wrong with MediaStore.
compatibility: >
  Requires AWS CLI or SDK access with mediastore, mediastore-data,
  cloudfront, cloudwatch, iam, and s3 permissions.
---

# AWS Elemental MediaStore Diagnostics

## When to use

Any MediaStore investigation — container creation, policy configuration, object put/get failures, CORS issues, lifecycle policies, access logging, metric policies, CDN integration, origin endpoints, or throttling.

## Investigation workflow

### Step 1 — Collect and triage

```
aws mediastore list-containers --query 'Containers[*].{Name:Name,ARN:ARN,Status:Status,Endpoint:Endpoint,CreationTime:CreationTime}'
aws mediastore describe-container --container-name <name> --query 'Container.{Name:Name,ARN:ARN,Status:Status,Endpoint:Endpoint,AccessLoggingEnabled:AccessLoggingEnabled}'
aws mediastore get-container-policy --container-name <name>
```

### Step 2 — Domain deep dive

```
aws mediastore-data list-items --endpoint <endpoint> --path /
aws mediastore get-cors-policy --container-name <name>
aws mediastore get-lifecycle-policy --container-name <name>
```

### Step 3 — Detailed investigation

```
aws mediastore get-metric-policy --container-name <name>
aws cloudwatch get-metric-statistics --namespace AWS/MediaStore --metric-name RequestCount --dimensions Name=ContainerName,Value=<name> --start-time <start> --end-time <end> --period 300 --statistics Sum
aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventSource,AttributeValue=mediastore.amazonaws.com --max-results 20
```

Read `references/guardrails.md` before concluding on any MediaStore issue.

## Tool quick reference

| Tool / API | When to use |
|------------|-------------|
| `mediastore list-containers` | List all MediaStore containers |
| `mediastore describe-container` | Get container details and status |
| `mediastore get-container-policy` | Check container access policy |
| `mediastore-data list-items` | List objects in a container |
| `mediastore-data put-object` | Upload object to container |
| `mediastore-data get-object` | Retrieve object from container |
| `mediastore get-cors-policy` | Check CORS configuration |

## Gotchas: AWS Elemental MediaStore

- MediaStore containers are NOT S3 buckets. They use a different API (mediastore/mediastore-data) and have different capabilities. Do not use S3 CLI commands.
- MediaStore is optimized for live video origination with consistent low-latency reads. It is NOT a general-purpose storage service.
- Container policies use a different format than S3 bucket policies. They support conditions but have a simpler structure.
- MediaStore has TWO API namespaces: `mediastore` (control plane — containers, policies) and `mediastore-data` (data plane — objects). They use different endpoints.
- Object paths in MediaStore use forward slashes and must start with `/`. There are no "buckets within containers" — the path is flat with folder-like prefixes.
- CORS policies are set at the container level and apply to all objects. There is no per-object CORS configuration.
- MediaStore has lower throughput limits than S3. It is designed for streaming workloads, not bulk data transfer.

## Anti-hallucination rules

1. Always cite specific container names, endpoints, or API responses as evidence.
2. MediaStore and S3 are completely different services. Never suggest S3 CLI commands for MediaStore.
3. Container policies and S3 bucket policies have different formats. Never use S3 policy syntax.
4. MediaStore has two API namespaces (control plane and data plane). Never confuse them.
5. Spend no more than 2 minutes on any single hypothesis. Pivot if inconclusive.

## 12 runbooks

| Category | IDs | Covers |
|----------|-----|--------|
| A — Containers | A1–A2 | Container creation failures, container policy errors |
| B — Object Operations | B1–B2 | Put object failures, get object errors |
| C — Configuration | C1–C2 | CORS configuration, lifecycle policy issues |
| D — Monitoring | D1–D2 | Access logging, metric policy errors |
| E — Integration | E1–E2 | CDN integration, origin endpoint issues |
| F — Performance | F1 | Throttling issues |
| Z — Catch-All | Z1 | General troubleshooting |
