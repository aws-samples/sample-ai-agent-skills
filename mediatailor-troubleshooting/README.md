# AWS Elemental MediaTailor Diagnostics Skill

Agent skill for investigating and troubleshooting AWS Elemental MediaTailor problems using structured runbooks, anti-hallucination guardrails, and systematic investigation workflows.

---

## What This Skill Covers

Structured troubleshooting for AWS Elemental MediaTailor — server-side ad insertion, channel assembly, VAST/VMAP handling, CDN configuration, manifest manipulation, prefetch schedules, and bumper/slate configuration.

### Activate When

- Configuration creation errors
- Playback configuration failures
- Ad insertion not working
- VAST/VMAP response errors
- Channel assembly issues
- Source location errors
- CDN configuration problems
- Manifest manipulation failures
- Log configuration issues
- Prefetch schedule problems
- Bumper or slate configuration errors
- General MediaTailor errors

---

## Skill Structure

```
mediatailor-troubleshooting/
├── SKILL.md
├── README.md
└── references/
    ├── A1-configuration-creation-errors.md
    ├── A2-playback-configuration.md
    ├── B1-ad-insertion-failures.md
    ├── B2-vast-response-errors.md
    ├── C1-channel-assembly-issues.md
    ├── C2-source-location-errors.md
    ├── D1-cdn-configuration.md
    ├── D2-manifest-manipulation.md
    ├── E1-log-configuration.md
    ├── E2-prefetch-schedule-issues.md
    ├── F1-bumper-slate-configuration.md
    ├── Z1-general-troubleshooting.md
    ├── guardrails.md
    └── hallucination-patterns.yaml
```

---

## Runbook Library (12 SOPs)

| Category | IDs | Covers |
|----------|-----|--------|
| **A — Configuration** | A1–A2 | Creation errors, playback configuration |
| **B — Ad Insertion** | B1–B2 | Ad insertion failures, VAST response errors |
| **C — Channel Assembly** | C1–C2 | Channel assembly issues, source location errors |
| **D — CDN & Manifest** | D1–D2 | CDN configuration, manifest manipulation |
| **E — Logging & Prefetch** | E1–E2 | Log configuration, prefetch schedule issues |
| **F — Bumper & Slate** | F1 | Bumper/slate configuration |
| **Z — Catch-All** | Z1 | General troubleshooting |

---

## Prerequisites

- AWS CLI v2 with `mediatailor:*`, `cloudwatch:GetMetricStatistics`, `logs:FilterLogEvents`, `cloudfront:Get*`, `s3:GetObject`, `iam:Get*` permissions

---

## License

MIT-0
