# MediaConvert Diagnostics Skill

Agent skill for investigating and troubleshooting AWS Elemental MediaConvert problems using structured runbooks, anti-hallucination guardrails, and systematic investigation workflows.

---

## What This Skill Covers

Structured troubleshooting for AWS Elemental MediaConvert when the console alone isn't enough — job failure analysis, input/output configuration, encoding settings, queue management, IAM integration, performance optimization, and DRM/content protection.

### Activate When

- Job failures or error states
- Jobs stuck in PROGRESSING state
- Output file errors or quality issues
- Input file format or codec problems
- Output format configuration issues
- Quality settings not producing expected results
- Queue management or capacity problems
- Reserved pricing configuration
- S3 input/output access issues
- IAM role permission errors
- Slow encoding speed
- Parallel job throughput issues
- DRM/content protection failures
- Key management problems

---

## Skill Structure

```
mediaconvert-troubleshooting/
├── SKILL.md
├── README.md
└── references/
    ├── A1-job-failures.md
    ├── A2-stuck-jobs.md
    ├── A3-output-errors.md
    ├── B1-input-file-issues.md
    ├── B2-codec-support.md
    ├── C1-output-format.md
    ├── C2-quality-settings.md
    ├── D1-queue-management.md
    ├── D2-reserved-pricing.md
    ├── E1-s3-input-output.md
    ├── E2-iam-roles.md
    ├── F1-encoding-speed.md
    ├── F2-parallel-jobs.md
    ├── G1-content-protection.md
    ├── G2-key-management.md
    ├── Z1-general-troubleshooting.md
    ├── guardrails.md
    └── hallucination-patterns.yaml
```

---

## Runbook Library (20 SOPs)

| Category | IDs | Covers |
|----------|-----|--------|
| **A — Jobs** | A1–A3 | Job failures, stuck jobs, output errors |
| **B — Input** | B1–B2 | Input file issues, codec support |
| **C — Output** | C1–C2 | Output format, quality settings |
| **D — Queue** | D1–D2 | Queue management, reserved pricing |
| **E — Integration** | E1–E2 | S3 input/output, IAM roles |
| **F — Performance** | F1–F2 | Encoding speed, parallel jobs |
| **G — DRM** | G1–G2 | Content protection, key management |
| **Z — Catch-All** | Z1 | General troubleshooting |

---

## Guardrails Summary

12 guardrails in `references/guardrails.md` covering account-specific endpoints, job error messages, IAM role requirements, S3 access, codec support, output group types, queue types, accelerated transcoding, DRM/SPEKE, job priority, audio track selection, and resolution/bitrate relationships.

---

## Investigation Workflow

1. **Triage** — Get endpoint, check job status and errors, review queue state
2. **Deep Dive** — Examine job settings, input/output files, IAM role, templates
3. **Detailed** — Inspect CloudWatch metrics, CloudTrail events, KMS keys

---

## Prerequisites

- AWS CLI v2 configured with appropriate credentials
- Permissions: `mediaconvert:*`, `s3:GetObject`, `s3:PutObject`, `iam:GetRole`, `cloudwatch:GetMetricStatistics`, `cloudtrail:LookupEvents`
- Account-specific MediaConvert endpoint URL

---

## Usage Examples

```
# Get account endpoint
aws mediaconvert describe-endpoints

# List failed jobs
aws mediaconvert list-jobs --status ERROR --max-results 10 --endpoint-url https://abc123.mediaconvert.us-east-1.amazonaws.com

# Get job details
aws mediaconvert get-job --id 1234567890123-abc123 --endpoint-url https://abc123.mediaconvert.us-east-1.amazonaws.com

# List queues
aws mediaconvert list-queues --endpoint-url https://abc123.mediaconvert.us-east-1.amazonaws.com
```

---

## License

MIT-0
