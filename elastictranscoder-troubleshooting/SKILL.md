---
name: elastictranscoder-diagnostics
version: "1.0.0"
last_updated: "2025-04-12"
description: >
  Use this skill to investigate and troubleshoot Amazon Elastic Transcoder
  problems by analyzing pipeline creation, job failures, preset errors,
  input/output format issues, notification failures, permission errors,
  watermark/thumbnail issues, and following structured runbooks.
  Activate when: pipeline creation failures, job stuck in progressing,
  preset configuration errors, media format issues, SNS notification
  problems, IAM permission errors, or the user says something is wrong
  with Elastic Transcoder.
compatibility: >
  Requires AWS CLI or SDK access with elastictranscoder, iam, s3,
  sns, and cloudwatch permissions.
---

# Amazon Elastic Transcoder Diagnostics

## When to use

Any Elastic Transcoder investigation — pipeline creation, job failures, preset configuration, input/output format errors, notification delivery, permission issues, or watermark/thumbnail problems.

## Investigation workflow

### Step 1 — Collect and triage

```
aws elastictranscoder list-pipelines --query 'Pipelines[*].{Id:Id,Name:Name,Status:Status,InputBucket:InputBucket}'
aws elastictranscoder list-jobs-by-pipeline --pipeline-id <pipeline-id> --query 'Jobs[*].{Id:Id,Status:Status,Output:Output.Status}'
aws elastictranscoder list-presets --query 'Presets[*].{Id:Id,Name:Name,Type:Type}'
```

### Step 2 — Domain deep dive

```
aws elastictranscoder read-pipeline --id <pipeline-id>
aws elastictranscoder read-job --id <job-id>
aws elastictranscoder read-preset --id <preset-id>
```

### Step 3 — Detailed investigation

```
aws s3 ls s3://<input-bucket>/<key> --human-readable
aws sns list-subscriptions-by-topic --topic-arn <topic-arn>
aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventSource,AttributeValue=elastictranscoder.amazonaws.com --max-results 20
```

Read `references/guardrails.md` before concluding on any Elastic Transcoder issue.

## Tool quick reference

| Tool / API | When to use |
|------------|-------------|
| `elastictranscoder list-pipelines` | List all pipelines |
| `elastictranscoder read-pipeline` | Get pipeline details |
| `elastictranscoder list-jobs-by-pipeline` | List jobs for a pipeline |
| `elastictranscoder read-job` | Get job details and status |
| `elastictranscoder list-presets` | List available presets |
| `elastictranscoder read-preset` | Get preset configuration |
| `elastictranscoder create-job` | Submit a transcoding job |

## Gotchas: Elastic Transcoder

- Elastic Transcoder is a LEGACY service. AWS recommends MediaConvert for new workloads. Elastic Transcoder is still supported but receives no new features.
- Pipelines are REGIONAL. Each pipeline is tied to a specific region and uses S3 buckets in that region. Cross-region bucket references fail silently.
- The service role needs s3:GetObject on the input bucket AND s3:PutObject on the output bucket. Missing either permission causes jobs to fail with AccessDenied.
- Jobs stuck in "Progressing" for extended periods usually indicate the input file is corrupted, too large, or in an unsupported codec — NOT a service outage.
- System presets cannot be modified. To customize encoding settings, create a custom preset. System preset IDs start with "1351620000001-".
- Watermarks require PNG format with alpha channel. JPEG watermarks are silently ignored or cause job failures.
- Thumbnail generation is configured per-output, not per-job. Each output can have different thumbnail settings.

## Anti-hallucination rules

1. Always cite specific pipeline IDs, job IDs, or API responses as evidence.
2. Elastic Transcoder is NOT MediaConvert. Never suggest MediaConvert APIs or features.
3. System presets are immutable. Never suggest modifying a system preset.
4. Pipeline status must be "Active" for jobs to process. Never assume pipeline is active without checking.
5. Spend no more than 2 minutes on any single hypothesis. Pivot if inconclusive.

## 12 runbooks

| Category | IDs | Covers |
|----------|-----|--------|
| A — Pipelines | A1–A2 | Creation failures, configuration |
| B — Jobs | B1–B2 | Job failures, stuck progressing |
| C — Presets | C1–C2 | Preset errors, custom preset issues |
| D — Formats | D1–D2 | Input format errors, output format issues |
| E — Notifications & Permissions | E1–E2 | Notification failures, permission errors |
| F — Media Features | F1 | Watermark and thumbnail issues |
| Z — Catch-All | Z1 | General troubleshooting |
