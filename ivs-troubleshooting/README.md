# IVS Diagnostics Skill

Agent skill for investigating and troubleshooting Amazon Interactive Video Service (IVS) problems using structured runbooks, anti-hallucination guardrails, and systematic investigation workflows.

---

## What This Skill Covers

Structured troubleshooting for Amazon IVS — channel creation, streaming, playback, chat, recording, low-latency optimization, and real-time stages.

### Activate When

- Channel creation or configuration failures
- Stream key issues
- Streaming or ingest failures
- Playback errors or quality issues
- Chat room or messaging problems
- Recording configuration or S3 issues
- Low-latency optimization
- Real-time stage or participant issues

---

## Skill Structure

```
ivs-troubleshooting/
├── SKILL.md
├── README.md
└── references/
    ├── A1-channel-creation.md
    ├── A2-stream-keys.md
    ├── B1-streaming-failures.md
    ├── B2-ingest-issues.md
    ├── C1-playback-errors.md
    ├── C2-playback-quality.md
    ├── D1-chat-rooms.md
    ├── D2-chat-messaging.md
    ├── E1-auto-recording.md
    ├── E2-recording-s3.md
    ├── F1-low-latency.md
    ├── G1-stages.md
    ├── G2-participant-issues.md
    ├── Z1-general-troubleshooting.md
    ├── guardrails.md
    └── hallucination-patterns.yaml
```

---

## Prerequisites

- AWS CLI v2 with `ivs:*`, `ivschat:*`, `ivsrealtime:*`, `s3:*`, `cloudwatch:GetMetricStatistics`

---

## License

MIT-0
