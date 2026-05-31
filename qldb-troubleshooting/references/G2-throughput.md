---
title: "G2 — Throughput Issues"
description: "Diagnose QLDB throughput and capacity problems"
status: active
severity: MEDIUM
triggers: ["throttling", "session limit", "OCC conflicts"]
owner: devops-agent
objective: "Identify and resolve throughput issues"
---

## Phase 1 — Triage

MUST:
- Check CloudWatch for throttling metrics
- Monitor active session count
- Check OCC conflict rate
- Verify transaction patterns for hot documents

SHOULD:
- Check Service Quotas for session limits
- Verify batch sizes are appropriate
- Monitor write throughput

## Phase 2 — Remediate

MUST:
- Reduce concurrent sessions or request limit increase
- Minimize OCC conflicts by reducing transaction scope
- Avoid hot documents (frequently updated same document)

SHOULD:
- Implement exponential backoff for retries
- Distribute writes across documents

## Output Format

```yaml
root_cause: "throughput — <specific_cause>"
severity: MEDIUM
```


## Safety Ratings
- Overall Risk: MEDIUM
- Data Exposure Risk: LOW — throughput is a performance concern
- Service Disruption Risk: MEDIUM — throughput issues affect application performance
- Compliance Impact: LOW — throughput is an operational concern

## Escalation Conditions
- Throughput drops below acceptable thresholds
- Write throughput throttling affects application
- Read throughput insufficient for query workload
- Throughput degradation during peak hours
- Throughput issues correlate with ledger size growth

## Data Sensitivity
- Throughput metrics — INTERNAL
- Performance baselines — INTERNAL
- Workload patterns — INTERNAL
- Capacity data — INTERNAL
- Never expose throughput data that reveals business volume

## Prohibited Actions
- Do NOT ignore throughput degradation on production ledgers
- Do NOT bypass throughput monitoring
- Do NOT run unoptimized queries that consume excessive throughput
- Do NOT ignore throttling warnings
- Do NOT disable throughput alerting
- Do NOT accept degraded throughput without investigation

## Phase 3 — Rollback
- Restore previous query patterns if changes caused throughput issues
- Revert application changes that increased throughput demand
- Optimize queries to reduce throughput consumption
- Verify throughput after changes
- Document throughput issues and performance baselines
- Update capacity planning based on findings

## Safety Ratings

safety_ratings:
  - "Phase 1 triage commands (describe/get/list): GREEN — read-only"
  - "Phase 2 configuration changes: YELLOW — state-changing but recoverable"
  - "Phase 2 resource deletion or security changes: RED — destructive or irreversible"

## Escalation Conditions

escalation_conditions:
  - "Remediation requires modifying IAM policies in a production account"
  - "Remediation requires disabling a security control even temporarily"
  - "Root cause cannot be identified after 3 hypothesis pivots"
  - "Blast radius affects more than one account or region"
  - "Issue involves potential data loss or exposure"

## Data Sensitivity

data_sensitivity:
  - command: "describe-* commands"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"
  - command: "get-* commands"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"
  - command: "list-* commands"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"

## Prohibited Actions

prohibited_actions:
  - "NEVER suggest overly broad IAM permissions as a fix"
  - "NEVER suggest disabling security controls for troubleshooting"
  - "NEVER suggest deleting resources without confirmation"
