---
title: "G1 — Latency Issues"
description: "Diagnose QLDB query and transaction latency problems"
status: active
severity: MEDIUM
triggers: ["slow queries", "high latency", "timeout"]
owner: devops-agent
objective: "Identify and resolve latency issues"
context: "Latency issues occur when: missing indexes, large transactions, OCC retries, or session pool exhausted."
---

## Phase 1 — Triage

MUST:
- Check CloudWatch CommandLatency metric
- Verify queries use indexed fields
- Check for OCC conflict retries
- Monitor session count vs limits

SHOULD:
- Check transaction size (number of documents)
- Verify connection pooling in QLDB driver
- Check for full table scans

## Phase 2 — Remediate

MUST:
- Create indexes on queried fields
- Reduce transaction size
- Implement connection pooling

SHOULD:
- Optimize PartiQL queries
- Monitor OCC conflict rates

## Output Format

```yaml
root_cause: "latency — <specific_cause>"
severity: MEDIUM
```


## Safety Ratings
- Overall Risk: MEDIUM
- Data Exposure Risk: LOW — latency is a performance concern
- Service Disruption Risk: MEDIUM — high latency affects application performance
- Compliance Impact: LOW — latency is an operational concern

## Escalation Conditions
- Query latency exceeds SLA thresholds
- Write latency affects transaction completion
- Latency spikes correlate with ledger operations
- Latency degradation without clear cause
- Latency affects application timeout configurations

## Data Sensitivity
- Latency metrics — INTERNAL
- Performance baselines — INTERNAL
- Query patterns — INTERNAL
- Transaction statistics — INTERNAL
- Never expose latency data that reveals business patterns

## Prohibited Actions
- Do NOT ignore latency degradation on production ledgers
- Do NOT run unoptimized queries that increase latency
- Do NOT bypass latency monitoring
- Do NOT accept degraded latency without investigation
- Do NOT disable latency alerting
- Do NOT modify ledger configurations without latency testing

## Phase 3 — Rollback
- Revert query changes that increased latency
- Restore previous index configurations
- Optimize queries to reduce latency
- Verify latency after changes
- Document latency issues and performance baselines
- Update latency monitoring thresholds

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
