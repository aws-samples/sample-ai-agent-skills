# Amazon GameLift Diagnostics Skill

Agent skill for investigating and troubleshooting Amazon GameLift problems using structured runbooks, anti-hallucination guardrails, and systematic investigation workflows.

---

## What This Skill Covers

Structured troubleshooting for Amazon GameLift — fleet management, game session placement, matchmaking, build uploads, server processes, health checks, Realtime Servers, FlexMatch, queues, multi-region fleets, and Spot fleet management.

### Activate When

- Fleet creation failures
- Fleet scaling issues
- Game session placement errors
- Matchmaking failures
- Build upload errors
- Script deployment issues
- Server process crashes
- Health check failures
- Realtime Servers problems
- FlexMatch configuration errors
- Queue configuration issues
- Multi-region fleet problems
- Spot fleet interruptions
- General GameLift errors

---

## Skill Structure

```
gamelift-troubleshooting/
├── SKILL.md
├── README.md
└── references/
    ├── A1-fleet-creation-failures.md
    ├── A2-fleet-scaling-issues.md
    ├── B1-game-session-placement.md
    ├── B2-matchmaking-failures.md
    ├── C1-build-upload-errors.md
    ├── C2-script-deployment.md
    ├── D1-server-process-crashes.md
    ├── D2-health-check-failures.md
    ├── E1-realtime-servers.md
    ├── E2-flexmatch-configuration.md
    ├── F1-queue-configuration.md
    ├── F2-multi-region-fleets.md
    ├── G1-spot-fleet-interruptions.md
    ├── Z1-general-troubleshooting.md
    ├── guardrails.md
    └── hallucination-patterns.yaml
```

---

## Runbook Library (14 SOPs)

| Category | IDs | Covers |
|----------|-----|--------|
| **A — Fleets** | A1–A2 | Creation failures, scaling issues |
| **B — Sessions** | B1–B2 | Game session placement, matchmaking |
| **C — Builds** | C1–C2 | Build upload, script deployment |
| **D — Server** | D1–D2 | Process crashes, health checks |
| **E — Realtime** | E1–E2 | Realtime Servers, FlexMatch |
| **F — Queues** | F1–F2 | Queue configuration, multi-region |
| **G — Spot** | G1 | Spot fleet interruptions |
| **Z — Catch-All** | Z1 | General troubleshooting |

---

## Prerequisites

- AWS CLI v2 with `gamelift:*`, `iam:Get*`, `ec2:Describe*`, `s3:Get*`, `cloudwatch:GetMetricStatistics` permissions

---

## License

MIT-0
