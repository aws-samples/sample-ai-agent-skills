# Amazon Transcribe Diagnostics Skill

Agent skill for investigating and troubleshooting Amazon Transcribe problems using structured runbooks, anti-hallucination guardrails, and systematic investigation workflows.

---

## What This Skill Covers

Structured troubleshooting for Amazon Transcribe — batch transcription, streaming, custom vocabulary, medical transcription, call analytics, subtitle generation, and content redaction.

### Activate When

- Transcription job failures
- Streaming transcription errors
- Custom vocabulary issues
- Poor transcription accuracy
- Medical transcription problems
- Call analytics failures
- Subtitle generation issues
- Content redaction problems
- General Transcribe errors

---

## Skill Structure

```
transcribe-troubleshooting/
├── SKILL.md
├── README.md
└── references/
    ├── A1-job-failures.md
    ├── A2-audio-format.md
    ├── B1-streaming-setup.md
    ├── B2-streaming-errors.md
    ├── C1-vocabulary-creation.md
    ├── C2-vocabulary-accuracy.md
    ├── D1-medical-transcription.md
    ├── D2-medical-vocabulary.md
    ├── E1-call-analytics.md
    ├── E2-channel-config.md
    ├── F1-content-redaction.md
    ├── F2-subtitles.md
    ├── Z1-general-troubleshooting.md
    ├── guardrails.md
    └── hallucination-patterns.yaml
```

---

## Runbook Library (14 SOPs)

| Category | IDs | Covers |
|----------|-----|--------|
| **A — Batch Jobs** | A1–A2 | Job failures, audio format issues |
| **B — Streaming** | B1–B2 | Streaming setup, streaming errors |
| **C — Custom Vocabulary** | C1–C2 | Vocabulary creation, vocabulary accuracy |
| **D — Medical** | D1–D2 | Medical transcription, medical vocabulary |
| **E — Call Analytics** | E1–E2 | Call analytics jobs, channel configuration |
| **F — Features** | F1–F2 | Content redaction, subtitles |
| **Z — Catch-All** | Z1 | General troubleshooting |

---

## Prerequisites

- AWS CLI v2 configured with appropriate credentials
- Permissions: `transcribe:*`, `s3:GetObject`, `s3:PutObject`
- S3 bucket with audio files for transcription

---

## License

MIT-0
