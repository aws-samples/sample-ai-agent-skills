---
name: gamelift-diagnostics
version: "1.0.0"
last_updated: "2025-04-12"
description: >
  Use this skill to investigate and troubleshoot Amazon GameLift
  problems by analyzing fleet management, game sessions, matchmaking,
  builds, server processes, scaling, queues, and following structured
  runbooks.
  Activate when: fleet creation failures, scaling issues, game session
  placement errors, matchmaking failures, build upload errors, script
  deployment issues, server process crashes, health check failures,
  Realtime Servers problems, FlexMatch configuration errors, queue
  issues, multi-region fleet problems, Spot fleet interruptions,
  or the user says something is wrong with GameLift.
compatibility: >
  Requires AWS CLI or SDK access with gamelift, iam, ec2, s3,
  cloudwatch, and cloudtrail permissions.
---

# Amazon GameLift Diagnostics

## When to use

Any Amazon GameLift investigation — fleet creation, scaling, game session placement, matchmaking, build uploads, server processes, health checks, Realtime Servers, FlexMatch, queues, multi-region fleets, or Spot fleet management.

## Investigation workflow

### Step 1 — Collect and triage

```
aws gamelift describe-fleet-attributes --query 'FleetAttributes[*].{Id:FleetId,Name:Name,Status:Status,Type:FleetType}'
aws gamelift describe-fleet-events --fleet-id <fleet-id> --start-time <start> --end-time <end>
aws gamelift describe-game-session-queues --query 'GameSessionQueues[*].{Name:Name,Destinations:Destinations}'
```

### Step 2 — Domain deep dive

```
aws gamelift describe-fleet-capacity --fleet-id <fleet-id>
aws gamelift describe-instances --fleet-id <fleet-id> --query 'Instances[*].{Id:InstanceId,Status:Status,Type:Type}'
aws gamelift describe-matchmaking-configurations --query 'Configurations[*].{Name:Name,Status:ConfigurationStatus}'
```

### Step 3 — Detailed investigation

```
aws cloudwatch get-metric-statistics --namespace AWS/GameLift --metric-name ActiveGameSessions --dimensions Name=FleetId,Value=<fleet-id> --start-time <start> --end-time <end> --period 300 --statistics Sum
aws gamelift describe-game-sessions --fleet-id <fleet-id> --status-filter ACTIVE
aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventSource,AttributeValue=gamelift.amazonaws.com --max-results 20
```

Read `references/guardrails.md` before concluding on any GameLift issue.

## Tool quick reference

| Tool / API | When to use |
|------------|-------------|
| `gamelift describe-fleet-attributes` | Get fleet configuration |
| `gamelift describe-fleet-events` | Get fleet event history |
| `gamelift describe-fleet-capacity` | Check fleet capacity |
| `gamelift describe-instances` | List fleet instances |
| `gamelift describe-game-sessions` | List game sessions |
| `gamelift describe-matchmaking-configurations` | Check matchmaking config |
| `gamelift describe-game-session-queues` | Check queue configuration |

## Gotchas: Amazon GameLift

- GameLift has TWO fleet types: Managed EC2 (GameLift manages instances) and Anywhere (you manage your own compute). They have very different troubleshooting paths.
- Builds are for custom game servers (compiled executables). Scripts are for Realtime Servers (JavaScript). They are NOT interchangeable.
- FlexMatch is the matchmaking service. It can work with GameLift queues (standalone mode) or with GameLift-hosted game sessions (integrated mode).
- Game session placement uses queues that can span multiple fleets and regions. Placement failures may be due to capacity, not configuration.
- Spot fleets can be interrupted by AWS with 2-minute warning. Game servers must handle graceful shutdown. On-Demand fleets are not interrupted.
- Server processes must call the GameLift Server SDK to report health and accept game sessions. Missing SDK calls cause health check failures.
- Fleet scaling has auto-scaling policies (target-based or rule-based) and manual scaling. Auto-scaling reacts to metrics, not predictions.

## Anti-hallucination rules

1. Always cite specific fleet IDs, build IDs, or game session IDs as evidence.
2. Managed EC2 fleets and Anywhere fleets have DIFFERENT capabilities. Never conflate them.
3. Builds (compiled executables) and Scripts (Realtime Servers) are separate. Never mix them.
4. FlexMatch standalone mode and integrated mode have different requirements. Never confuse them.
5. Spend no more than 2 minutes on any single hypothesis. Pivot if inconclusive.

## 14 runbooks

| Category | IDs | Covers |
|----------|-----|--------|
| A — Fleets | A1-A2 | Fleet creation failures, fleet scaling issues |
| B — Sessions | B1-B2 | Game session placement, matchmaking failures |
| C — Builds | C1-C2 | Build upload errors, script deployment |
| D — Server | D1-D2 | Server process crashes, health check failures |
| E — Realtime | E1-E2 | Realtime Servers, FlexMatch configuration |
| F — Queues | F1-F2 | Queue configuration, multi-region fleets |
| G — Spot | G1 | Spot fleet interruptions |
| Z — Catch-All | Z1 | General troubleshooting |
