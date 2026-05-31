---
name: rekognition-diagnostics
version: "1.0.0"
last_updated: "2025-04-12"
description: >
  Use this skill to investigate and troubleshoot Amazon Rekognition
  problems by analyzing image analysis, video analysis, face detection,
  face collections, custom labels, content moderation, streaming video,
  text detection, celebrity recognition, PPE detection, and following
  structured runbooks. Activate when: image analysis errors, video
  analysis failures, face detection issues, collection management
  problems, custom labels training failures, content moderation
  inaccuracies, streaming processor errors, or the user says something
  is wrong with Rekognition without naming specific symptoms.
compatibility: >
  Requires AWS CLI or SDK access with Rekognition, S3, IAM, CloudWatch,
  CloudTrail, and optionally Kinesis Video Streams and SNS permissions.
---

# Amazon Rekognition Diagnostics

## When to use

Any Amazon Rekognition investigation where the console alone is insufficient — image/video analysis, face detection and collections, custom labels, content moderation, streaming video analysis, or text detection.

## Investigation workflow

### Step 1 — Collect and triage

```
aws rekognition list-collections
aws rekognition list-projects
aws rekognition list-stream-processors
```

### Step 2 — Domain deep dive

```
aws rekognition describe-collection --collection-id <collection-id>
aws rekognition describe-projects --project-names <project-name>
aws rekognition describe-project-versions --project-arn <project-arn>
aws rekognition get-face-detection --job-id <job-id>
```

### Step 3 — Detailed investigation

```
aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventSource,AttributeValue=rekognition.amazonaws.com --max-results 20
aws cloudwatch get-metric-statistics --namespace AWS/Rekognition --metric-name SuccessfulRequestCount --start-time <start> --end-time <end> --period 3600 --statistics Sum
```

Read `references/guardrails.md` before concluding on any Rekognition issue.

## Tool quick reference

| Tool / API | When to use |
|------------|-------------|
| `rekognition detect-faces` | Analyze faces in image |
| `rekognition detect-labels` | Detect objects/scenes |
| `rekognition detect-moderation-labels` | Content moderation |
| `rekognition start-face-detection` | Async video face detection |
| `rekognition list-collections` | List face collections |
| `rekognition describe-projects` | Custom Labels projects |
| `rekognition list-stream-processors` | Streaming processors |
| CloudWatch metrics | Monitor API calls, errors |

## Gotchas: Amazon Rekognition

- Image size limit is 5MB for S3 and 5MB base64 for direct API. Larger images must be resized.
- Video analysis is asynchronous. Start* APIs return a job ID. Use Get* APIs to retrieve results. Results available via SNS notification.
- Face collections have a default limit of face vectors. IndexFaces adds faces; DeleteFaces removes them. Collections are region-specific.
- Custom Labels requires minimum training images (varies by use case, typically 10+ per label). Training can take 30 minutes to 24 hours.
- Content moderation confidence thresholds should be tuned. Default returns all detections. Set MinConfidence to reduce false positives.
- Streaming video processors require Kinesis Video Streams input and Kinesis Data Streams output.
- API calls are throttled per account per region. Implement retry with exponential backoff.
- DetectText has a 100-word limit per image. For documents, use Amazon Textract instead.

## Anti-hallucination rules

1. Always cite specific collection IDs, project ARNs, or API responses as evidence.
2. Video analysis is async. Never assume results are immediately available.
3. Custom Labels requires training. Never assume a model works without training.
4. Image size limits are strict. Never assume larger images are accepted.
5. Face collections are region-specific. Never assume cross-region access.
6. Spend no more than 2 minutes on any single hypothesis. Pivot if inconclusive.

## 14 runbooks

| Category | IDs | Covers |
|----------|-----|--------|
| A — Image Analysis | A1-A2 | Label detection, text detection |
| B — Face Detection | B1-B2 | Face analysis, face collections |
| C — Video Analysis | C1-C2 | Async video jobs, video face detection |
| D — Custom Labels | D1-D3 | Project creation, training, inference |
| E — Content Moderation | E1-E2 | Image moderation, video moderation |
| F — Streaming | F1 | Stream processor issues |
| G — General | G1 | IAM and throttling |
| Z — Catch-All | Z1 | General troubleshooting |
