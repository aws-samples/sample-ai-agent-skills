# MediaLive Diagnostics Skill

Agent skill for investigating and troubleshooting AWS Elemental MediaLive problems using structured runbooks, anti-hallucination guardrails, and systematic investigation workflows.

---

## What This Skill Covers

Structured troubleshooting for AWS Elemental MediaLive — channel creation, input configuration, output groups, encoding profiles, pipeline alerts, failover, and schedule actions.

### Activate When

- Channel creation or start failures
- Input configuration or connectivity issues
- Output group errors (HLS, DASH, CMAF, Archive)
- Encoding profile or codec problems
- Pipeline alerts or health issues
- Automatic or input failover failures
- Schedule action errors
- Audio/caption configuration issues

---

## Skill Structure

```
medialive-troubleshooting/
├── SKILL.md
├── README.md
└── references/
    ├── A1-channel-creation.md
    ├── A2-channel-start-stop.md
    ├── A3-channel-state.md
    ├── B1-input-config.md
    ├── B2-input-security-groups.md
    ├── B3-input-switching.md
    ├── C1-output-groups.md
    ├── C2-hls-output.md
    ├── C3-dash-cmaf-output.md
    ├── D1-video-encoding.md
    ├── D2-audio-captions.md
    ├── E1-pipeline-alerts.md
    ├── E2-health-monitoring.md
    ├── F1-automatic-failover.md
    ├── F2-input-failover.md
    ├── G1-schedule-actions.md
    ├── Z1-general-troubleshooting.md
    ├── guardrails.md
    └── hallucination-patterns.yaml
```

---

## Runbook Library

| Category | IDs | Covers |
|----------|-----|--------|
| **A — Channel** | A1–A3 | Creation failures, start/stop, state issues |
| **B — Input** | B1–B3 | Input config, security groups, switching |
| **C — Output** | C1–C3 | Output groups, HLS, DASH/CMAF |
| **D — Encoding** | D1–D2 | Video encoding, audio/captions |
| **E — Pipeline** | E1–E2 | Pipeline alerts, health monitoring |
| **F — Failover** | F1–F2 | Automatic failover, input failover |
| **G — Schedule** | G1 | Schedule actions |
| **Z — Catch-All** | Z1 | General troubleshooting |

---

## Prerequisites

- AWS CLI v2 with `medialive:*`, `s3:*`, `iam:GetRole`, `cloudwatch:GetMetricStatistics`, `cloudtrail:LookupEvents`
- MediaLive IAM role with medialive.amazonaws.com trust policy

---

## License

MIT-0
