# G1 — DataZone Data Quality Issues

## Symptoms
- Data quality rules not executing
- Quality results not appearing
- Quality gate blocking publishing

## Triage Commands

```bash
aws datazone get-asset --domain-identifier <domain-id> --identifier <asset-id>
# Check Glue Data Quality for execution
aws glue get-data-quality-result --result-id <result-id>
aws glue list-data-quality-results --max-results 10
```

## Common Causes
1. **Glue Data Quality not configured** — Rules execute via Glue
2. **IAM permissions** — Role can't execute Glue Data Quality
3. **Rule syntax error** — DQDL (Data Quality Definition Language) syntax wrong
4. **Data source inaccessible** — Glue can't access underlying data
5. **Quality threshold** — Results below configured threshold

## Resolution
- Configure Glue Data Quality rules for the underlying data source
- Verify IAM role has glue:StartDataQualityRulesetEvaluationRun permission
- Validate DQDL rule syntax
- Ensure Glue can access the data (S3 permissions, Glue catalog)
- Review quality results and adjust thresholds if appropriate


---

## Safety Ratings

| Dimension | Rating |
|-----------|--------|
| Data Sensitivity | HIGH — Data quality rules execute against actual data sources |
| Blast Radius | MEDIUM — Quality gate failures block asset publishing |
| Rollback Complexity | LOW — Rules can be updated; quality gates can be adjusted |
| Compliance Scope | HIGH — Data quality is a compliance requirement for many frameworks |

## Escalation Conditions

- Escalate to data owners if quality results indicate data integrity issues
- Escalate to compliance if quality gates are blocking required asset publishing
- Escalate to data platform team if Glue Data Quality execution fails

## Data Sensitivity

- **Quality results** may reveal data content patterns — restrict access
- **DQDL rules** define data expectations — operational data
- **Glue job execution** accesses actual data sources

## PROHIBITED Actions

- Do NOT lower quality thresholds to bypass publishing gates without data owner approval
- Do NOT disable quality checks for compliance-required data assets
- Do NOT grant Glue Data Quality roles broader permissions than required

## Phase 3 — Rollback

- Restore previous quality rule definitions
- Revert quality threshold changes
- Re-run quality evaluation with corrected rules
- Document quality gate overrides for audit trail

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
