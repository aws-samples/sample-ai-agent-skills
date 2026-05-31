---
name: mediaconvert-diagnostics
version: "1.0.0"
last_updated: "2025-04-12"
description: >
  Use this skill to investigate and troubleshoot AWS Elemental MediaConvert
  problems by analyzing job failures, input/output configurations, queue
  management, encoding settings, IAM roles, performance, and DRM
  configurations using structured runbooks.
  Activate when: job failures, stuck jobs, output errors, input file issues,
  codec support problems, output format errors, quality settings, queue
  management, reserved pricing, S3 input/output issues, IAM role problems,
  encoding speed, parallel job issues, content protection, key management,
  or the user says something is wrong with MediaConvert without naming
  specific symptoms.
compatibility: >
  Requires AWS CLI or SDK access with MediaConvert, S3, IAM, CloudWatch,
  CloudTrail, and optionally KMS permissions.
---

# MediaConvert Diagnostics

## When to use

Any AWS Elemental MediaConvert investigation where the console alone is insufficient — job failure analysis, encoding configuration, queue management, input/output troubleshooting, performance optimization, or DRM configuration.

## Investigation workflow

### Step 1 — Collect and triage

```
aws mediaconvert describe-endpoints
aws mediaconvert list-jobs --status ERROR --max-results 10 --endpoint-url <endpoint>
aws mediaconvert list-jobs --status COMPLETE --max-results 5 --endpoint-url <endpoint>
aws mediaconvert list-queues --endpoint-url <endpoint>
aws cloudwatch get-metric-statistics --namespace AWS/MediaConvert --metric-name JobsErrored --start-time <start> --end-time <end> --period 3600 --statistics Sum
```

### Step 2 — Domain deep dive

```
aws mediaconvert get-job --id <job-id> --endpoint-url <endpoint>
aws mediaconvert get-job-template --name <template-name> --endpoint-url <endpoint>
aws mediaconvert list-presets --endpoint-url <endpoint>
aws s3 ls s3://<input-bucket>/<path>/
aws iam get-role --role-name <mediaconvert-role>
```

### Step 3 — Detailed investigation

```
aws mediaconvert get-queue --name <queue-name> --endpoint-url <endpoint>
aws cloudwatch get-metric-statistics --namespace AWS/MediaConvert --metric-name JobsCompleted --start-time <start> --end-time <end> --period 3600 --statistics Sum
aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventSource,AttributeValue=mediaconvert.amazonaws.com --max-results 10
aws kms describe-key --key-id <key-id>
```

Read `references/guardrails.md` before concluding on any MediaConvert issue.

## Tool quick reference

| Tool / API | When to use |
|------------|-------------|
| `describe-endpoints` | Get account-specific endpoint |
| `get-job` | Job details and error messages |
| `list-jobs` | Job history by status |
| `list-queues` | Queue configuration |
| `get-job-template` | Job template settings |
| `list-presets` | Output presets |
| `get-queue` | Queue details and pricing |
| `s3 ls` | Verify input/output files |
| `iam get-role` | MediaConvert service role |

## Gotchas: MediaConvert

- Account-specific endpoint: MediaConvert uses account-specific endpoints. You must call describe-endpoints first and use the returned URL for all subsequent API calls.
- Job errors are in the job details: The error message and error code in the get-job response contain the specific failure reason. Always check this first.
- IAM role is required: MediaConvert needs an IAM role with S3 read access for inputs and S3 write access for outputs. The role must have a trust policy for mediaconvert.amazonaws.com.
- S3 bucket policy: The S3 bucket must allow the MediaConvert role to read inputs and write outputs. Cross-account access requires explicit bucket policy.
- Input file format: MediaConvert supports specific input containers and codecs. Not all video formats are supported. Check the supported codecs documentation.
- Output groups: Each job can have multiple output groups (File, HLS, DASH, CMAF, MS Smooth). Each group can have multiple outputs with different settings.
- Queue types: On-demand queues (pay per minute) and reserved queues (committed pricing). Reserved queues have a fixed number of reserved transcode slots (RTS).
- Accelerated transcoding: Available for specific codecs and resolutions. Costs more but significantly faster. Not all jobs qualify.
- DRM: Supports SPEKE (Secure Packager and Encoder Key Exchange) for content protection. Requires a key provider URL.
- Job priority: Jobs within a queue are processed by priority (1-50, default 0). Higher priority jobs are processed first.

## Anti-hallucination rules

1. Always use the account-specific endpoint from describe-endpoints for all API calls.
2. Always check the job error message in get-job before diagnosing — it contains the specific failure reason.
3. Never assume all video codecs are supported — verify against MediaConvert's supported formats.
4. Reserved queues have fixed RTS capacity — they do not auto-scale.
5. Accelerated transcoding is not available for all codecs and resolutions.
6. Spend no more than 2 minutes on any single hypothesis. Pivot if inconclusive.

## 20 runbooks

| Category | IDs | Covers |
|----------|-----|--------|
| A — Jobs | A1-A3 | Job failures, stuck jobs, output errors |
| B — Input | B1-B2 | Input file issues, codec support |
| C — Output | C1-C2 | Output format, quality settings |
| D — Queue | D1-D2 | Queue management, reserved pricing |
| E — Integration | E1-E2 | S3 input/output, IAM roles |
| F — Performance | F1-F2 | Encoding speed, parallel jobs |
| G — DRM | G1-G2 | Content protection, key management |
| Z — Catch-All | Z1 | General troubleshooting |
