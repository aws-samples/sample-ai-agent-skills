# Amazon Rekognition Diagnostics Skill

Agent skill for investigating and troubleshooting Amazon Rekognition problems using structured runbooks, anti-hallucination guardrails, and systematic investigation workflows.

---

## What This Skill Covers

Structured troubleshooting for Amazon Rekognition — image/video analysis, face detection and collections, custom labels, content moderation, streaming video analysis, and text detection.

### Activate When

- Image analysis errors or unexpected results
- Video analysis job failures
- Face detection or collection issues
- Custom Labels training or inference failures
- Content moderation inaccuracies
- Streaming video processor errors
- API throttling or permission errors
- General Rekognition errors

---

## Skill Structure

```
rekognition-troubleshooting/
├── SKILL.md
├── README.md
└── references/
    ├── A1-label-detection.md
    ├── A2-text-detection.md
    ├── B1-face-analysis.md
    ├── B2-face-collections.md
    ├── C1-video-jobs.md
    ├── C2-video-face-detection.md
    ├── D1-custom-labels-project.md
    ├── D2-custom-labels-training.md
    ├── D3-custom-labels-inference.md
    ├── E1-image-moderation.md
    ├── E2-video-moderation.md
    ├── F1-stream-processor.md
    ├── G1-iam-throttling.md
    ├── Z1-general-troubleshooting.md
    ├── guardrails.md
    └── hallucination-patterns.yaml
```

---

## Runbook Library (14 SOPs)

| Category | IDs | Covers |
|----------|-----|--------|
| **A — Image Analysis** | A1–A2 | Label detection, text detection |
| **B — Face Detection** | B1–B2 | Face analysis, face collections |
| **C — Video Analysis** | C1–C2 | Async video jobs, video face detection |
| **D — Custom Labels** | D1–D3 | Project creation, training, inference |
| **E — Content Moderation** | E1–E2 | Image moderation, video moderation |
| **F — Streaming** | F1 | Stream processor issues |
| **G — General** | G1 | IAM and throttling |
| **Z — Catch-All** | Z1 | General troubleshooting |

---

## Prerequisites

- AWS CLI v2 configured with appropriate credentials
- Permissions: `rekognition:*`, `s3:GetObject`, `kinesisvideo:*` (for streaming), `cloudwatch:GetMetricStatistics`
- S3 bucket with images/videos for analysis

---

## License

MIT-0
