---
name: transcribe-diagnostics
version: "1.0.0"
last_updated: "2025-04-12"
description: >
  Use this skill to investigate and troubleshoot Amazon Transcribe
  problems by analyzing transcription jobs, streaming transcription,
  custom vocabulary, medical transcription, call analytics, subtitles,
  language identification, content redaction, and following structured
  runbooks. Activate when: transcription job failures, streaming errors,
  custom vocabulary issues, poor transcription accuracy, medical
  transcription problems, call analytics failures, subtitle generation
  issues, or the user says something is wrong with Transcribe.
compatibility: >
  Requires AWS CLI or SDK access with Transcribe, S3, IAM, CloudWatch,
  and CloudTrail permissions.
---

# Amazon Transcribe Diagnostics

## When to use

Any Amazon Transcribe investigation where the console alone is insufficient — batch transcription, streaming, custom vocabulary, medical transcription, call analytics, or subtitle generation.

## Investigation workflow

### Step 1 — Collect and triage

```
aws transcribe list-transcription-jobs --status FAILED --max-results 10
aws transcribe list-vocabularies
aws transcribe list-call-analytics-jobs --status FAILED --max-results 10
aws transcribe list-medical-transcription-jobs --status FAILED --max-results 10
```

### Step 2 — Domain deep dive

```
aws transcribe get-transcription-job --transcription-job-name <job-name>
aws transcribe get-vocabulary --vocabulary-name <name>
aws transcribe get-call-analytics-job --call-analytics-job-name <job-name>
```

### Step 3 — Detailed investigation

```
aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventSource,AttributeValue=transcribe.amazonaws.com --max-results 20
aws s3 cp s3://<bucket>/<output-key> - | head -100
```

Read `references/guardrails.md` before concluding on any Transcribe issue.

## Tool quick reference

| Tool / API | When to use |
|------------|-------------|
| `transcribe start-transcription-job` | Start batch transcription |
| `transcribe get-transcription-job` | Check job status |
| `transcribe list-vocabularies` | List custom vocabularies |
| `transcribe start-stream-transcription` | Start streaming |
| `transcribe start-call-analytics-job` | Start call analytics |
| `transcribe start-medical-transcription-job` | Medical transcription |

## Gotchas: Amazon Transcribe

- Audio format must match MediaFormat parameter. Supported: WAV, MP3, MP4, FLAC, OGG, AMR, WebM.
- Custom vocabulary improves recognition of specific terms. Vocabulary must be in READY state before use.
- Streaming transcription uses WebSocket or HTTP/2. Has different latency characteristics than batch.
- Medical transcription is a separate API with HIPAA compliance. Don't use standard API for medical content.
- Call Analytics requires stereo audio with separate channels per speaker.
- Content redaction replaces PII in transcripts. Only available for specific languages.
- Language identification can auto-detect language but adds processing time.
- Maximum audio duration varies: 4 hours for standard, 4 hours for medical.

## Anti-hallucination rules

1. Always cite specific job names, vocabulary names, or API responses as evidence.
2. Audio format must match MediaFormat. Never assume automatic format detection.
3. Custom vocabulary must be READY. Never assume immediate availability.
4. Medical transcription uses separate API. Never use standard API for medical.
5. Call Analytics requires stereo audio. Never assume mono works.
6. Spend no more than 2 minutes on any single hypothesis. Pivot if inconclusive.

## 14 runbooks

| Category | IDs | Covers |
|----------|-----|--------|
| A — Batch Jobs | A1-A2 | Job failures, audio format issues |
| B — Streaming | B1-B2 | Streaming setup, streaming errors |
| C — Custom Vocabulary | C1-C2 | Vocabulary creation, vocabulary accuracy |
| D — Medical | D1-D2 | Medical transcription, medical vocabulary |
| E — Call Analytics | E1-E2 | Call analytics jobs, channel configuration |
| F — Features | F1-F2 | Content redaction, subtitles |
| Z — Catch-All | Z1 | General troubleshooting |
