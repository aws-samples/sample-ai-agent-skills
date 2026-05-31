---
title: "C2 — Supply Planning Issues"
description: "Diagnose supply planning and replenishment issues"
status: active
severity: HIGH
triggers: ["supply planning error", "replenishment failure", "supply plan inaccurate"]
owner: devops-agent
objective: "Resolve supply planning and replenishment issues in Supply Chain"
context: "Supply planning depends on demand forecasts, inventory data, and supplier information. Requires complete data across all inputs."
---

## Phase 1 — Triage

MUST:
- Verify demand planning data is available and current
- Check inventory visibility data: `aws scn list-data-lake-datasets --instance-id <instance-id> --namespace asc`
- Verify supplier and lead time data is ingested

SHOULD:
- Check BOM (Bill of Materials) data completeness
- Verify supply planning configuration parameters

MAY:
- Review supply plan output for anomalies
- Compare supply plan with manual calculations

## Phase 2 — Remediate

MUST:
- Ensure all required input datasets are populated (demand, inventory, suppliers, BOM)
- Fix data quality issues in input datasets
- Re-run supply planning after data corrections

SHOULD:
- Validate BOM relationships and lead times
- Implement supply planning parameter tuning

MAY:
- Set up supply plan comparison dashboards
- Implement automated supply plan validation

## Safety Ratings

```
safety_ratings:
  - "list-data-lake-datasets: GREEN — read-only dataset enumeration"
  - "get-data-lake-dataset: GREEN — read-only dataset details"
  - "Supply plan re-execution: YELLOW — overwrites existing plans"
  - "BOM data modification: YELLOW — affects all downstream planning"
```

## Escalation Conditions

- Supply planning produces infeasible plans
- BOM data inconsistencies affect multiple product lines
- Lead time data is significantly inaccurate
- Supply plan conflicts with demand forecast

## Data Sensitivity

- Supply planning data: HIGH — contains proprietary supply chain strategy
- BOM data: HIGH — contains product composition and supplier relationships
- Supplier data: HIGH — contains vendor pricing and lead times
- Supply plan output: HIGH — contains business-critical procurement plans

## Prohibited Actions

- NEVER share supply plans or BOM data outside authorized teams
- NEVER modify BOM data without engineering team approval
- NEVER override supply plans without business justification
- NEVER ignore supply plan warnings about data quality

## Phase 3 — Rollback

1. If supply planning configuration was changed: restore previous parameters
2. If BOM data was modified: revert to previous version
3. If supply plan was regenerated with incorrect data: flag as invalid
4. Notify procurement team of any supply plan changes
5. Document all changes for audit trail

## Common Issues

- symptoms: "Supply plan recommends zero replenishment"
  diagnosis: "Missing inventory or demand data."
  resolution: "Verify inventory visibility data and demand forecasts are current. Check data integration flow status."

- symptoms: "Supply plan shows unrealistic lead times"
  diagnosis: "Supplier lead time data is missing or incorrect."
  resolution: "Verify supplier data in the data lake. Update lead time values from supplier agreements."

## Output Format

```yaml
root_cause: "supply_planning — <specific_cause>"
evidence:
  - type: planning_error
    content: "<error details>"
severity: HIGH
mitigation:
  immediate: "Fix input data and re-run planning"
  long_term: "Implement data completeness monitoring"
```

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
