---
title: "E2 — Query Result Reuse"
description: "Diagnose and configure Athena query result reuse for cost savings"
status: active
severity: LOW
triggers:
  - "result reuse"
  - "cache"
  - "repeated queries"
  - "same query"
owner: devops-agent
objective: "Configure and troubleshoot query result reuse to avoid redundant data scanning"
context: "Athena query result reuse returns cached results for identical queries, avoiding re-scanning data. Available in engine v3 workgroups."
---

## Phase 1 — Triage

MUST:
- Check workgroup result reuse setting: `aws athena get-work-group --work-group <workgroup-name> --query 'WorkGroup.Configuration.ResultConfiguration'`
- Verify engine version is v3: `aws athena get-work-group --work-group <workgroup-name> --query 'WorkGroup.Configuration.EngineVersion'`
- Check if query is eligible for reuse (no non-deterministic functions)
- Verify query execution shows reuse: `aws athena get-query-execution --query-execution-id <id> --query 'QueryExecution.Statistics'`

SHOULD:
- Check the maximum age setting for cached results
- Verify the query text is identical (whitespace and case matter)
- Check if underlying data has changed since the cached result

MAY:
- Monitor cache hit rate across the workgroup
- Compare costs with and without result reuse

## Phase 2 — Remediate

MUST:
- If not enabled: enable result reuse in workgroup configuration
- If engine v2: upgrade to engine v3 (result reuse requires v3)
- If queries not matching: ensure identical query text for reuse
- If stale results: adjust maximum age or disable for specific queries

SHOULD:
- Set appropriate max age based on data freshness requirements
- Document which queries benefit from result reuse

MAY:
- Use query result reuse with scheduled reports for maximum savings
- Combine with CTAS materialization for complex repeated queries

## Common Issues

- symptoms: "Result reuse not working despite being enabled"
  diagnosis: "Query contains non-deterministic functions (NOW(), RANDOM()) or workgroup is on engine v2."
  resolution: "Remove non-deterministic functions or upgrade to engine v3."

## Output Format

```yaml
root_cause: "Query result reuse — <specific_issue>"
evidence:
  - type: workgroup_config
    content: "<result reuse settings>"
severity: LOW
mitigation:
  immediate: "Enable result reuse and verify query eligibility"
  long_term: "Design queries for reuse compatibility and monitor cache hit rates"
```


## Safety Ratings

| Phase 2 Action | safety_rating | Rationale |
|---|---|---|
| Enable result reuse in workgroup | GREEN | Performance optimization — non-destructive |
| Upgrade to engine v3 | YELLOW | May change query behavior for existing workloads |
| Ensure identical query text for reuse | GREEN | Query modification guidance only |
| Adjust maximum age for cached results | GREEN | Cache configuration — non-destructive |

## Escalation Conditions

- Query accesses sensitive data (PII, financial, healthcare records)
- Fix requires modifying workgroup settings that affect multiple teams
- Engine version upgrade may change query behavior
- Cached results may serve stale data if max age is too long

## Data Sensitivity

- Cached query results may contain sensitive data — ensure result location is properly secured
- Result reuse means sensitive query results persist longer in S3
- Cache configuration affects data freshness for compliance-sensitive queries

## Prohibited Actions

- NEVER suggest scanning entire tables without LIMIT to debug — this can incur massive costs and expose sensitive data
- NEVER enable result reuse for queries that must always return real-time data
- NEVER set excessively long cache max age for compliance-sensitive data

## Phase 3 — Rollback

- If result reuse was enabled incorrectly: disable in workgroup configuration
- If engine version was upgraded: revert using `aws athena update-work-group` with previous version
- If max age was set incorrectly: adjust to appropriate value
- Document all workgroup changes for audit trail

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
  - "NEVER suggest disabling query result encryption"
  - "NEVER suggest overly broad S3 access for workgroups"
  - "NEVER suggest disabling workgroup enforcement"
