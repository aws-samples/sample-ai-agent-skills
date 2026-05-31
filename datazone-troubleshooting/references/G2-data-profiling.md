# G2 — DataZone Data Profiling Issues

## Symptoms
- Data profiling not running
- Profile results incomplete
- Profiling errors

## Triage Commands

```bash
aws datazone get-asset --domain-identifier <domain-id> --identifier <asset-id>
aws glue list-data-quality-results --max-results 10
```

## Common Causes
1. **Profiling not enabled** — Must enable profiling for the data source
2. **Glue permissions** — Role can't run Glue profiling jobs
3. **Large dataset** — Profiling takes time for large tables
4. **Data format** — Unsupported data types for profiling
5. **Cost concerns** — Profiling runs Glue jobs with associated costs

## Resolution
- Enable data profiling in the environment configuration
- Verify IAM role has Glue job execution permissions
- For large datasets, profiling may take significant time
- Review supported data types for profiling
- Monitor Glue job costs associated with profiling runs
- Profile results include: row count, null counts, distinct values, distributions


---

## Safety Ratings

| Dimension | Rating |
|-----------|--------|
| Data Sensitivity | HIGH — Profiling analyzes actual data content and distributions |
| Blast Radius | LOW — Profiling is read-only analysis |
| Rollback Complexity | LOW — Profiling results can be regenerated |
| Compliance Scope | MEDIUM — Profile results may reveal sensitive data patterns |

## Escalation Conditions

- Escalate to data owners if profiling reveals unexpected sensitive data patterns
- Escalate to finance if profiling Glue job costs are unexpectedly high

## Data Sensitivity

- **Profile results** (distributions, null counts, distinct values) may reveal sensitive data patterns
- **Glue job execution** reads actual data — IAM role must be scoped appropriately

## PROHIBITED Actions

- Do NOT share profiling results containing sensitive data patterns outside authorized teams
- Do NOT run profiling on large datasets without estimating Glue job costs
- Do NOT grant profiling roles access beyond the specific data source

## Phase 3 — Rollback

- Delete incorrect profiling results
- Revert profiling configuration changes
- Re-run profiling with corrected settings

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
