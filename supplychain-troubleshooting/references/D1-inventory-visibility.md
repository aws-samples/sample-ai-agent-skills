---
title: "D1 — Inventory Visibility"
description: "Diagnose inventory visibility and tracking issues"
status: active
severity: HIGH
triggers: ["inventory not visible", "inventory discrepancy", "stock level incorrect", "inventory data missing"]
owner: devops-agent
objective: "Resolve inventory visibility issues in Supply Chain"
context: "Inventory visibility aggregates data from multiple sources with eventual consistency. Discrepancies may reflect timing differences."
---

## Phase 1 — Triage

MUST:
- Check inventory datasets: `aws scn list-data-lake-datasets --instance-id <instance-id> --namespace asc`
- Verify data integration flows for inventory sources are running
- Check last successful flow execution timestamp

SHOULD:
- Compare inventory data in Supply Chain with source systems
- Verify data integration flow frequency matches business requirements

MAY:
- Check for data transformation errors in integration flows
- Review inventory data quality metrics

## Phase 2 — Remediate

MUST:
- Fix data integration flows for inventory data sources
- Ensure source data format matches expected schema
- Re-run integration flows after fixing issues

SHOULD:
- Increase flow execution frequency for near-real-time visibility
- Implement data reconciliation checks

MAY:
- Set up inventory discrepancy alerts
- Build inventory data quality dashboards

## Safety Ratings

```
safety_ratings:
  - "list-data-lake-datasets: GREEN — read-only dataset enumeration"
  - "get-data-lake-dataset: GREEN — read-only dataset details"
  - "Data integration re-execution: YELLOW — updates inventory data"
  - "Flow frequency changes: YELLOW — affects data freshness and costs"
```

## Escalation Conditions

- Inventory discrepancies exceed business tolerance thresholds
- Multiple data sources show conflicting inventory levels
- Data integration flows fail consistently for critical inventory sources
- Inventory visibility gaps affect order fulfillment

## Data Sensitivity

- Inventory data: HIGH — contains proprietary stock levels and locations
- Warehouse data: HIGH — contains facility and logistics information
- Integration flow configuration: MEDIUM — contains source system details

## Prohibited Actions

- NEVER manually adjust inventory data in the data lake without audit trail
- NEVER assume inventory discrepancies are errors without checking timing
- NEVER disable data integration flows without understanding downstream impact
- NEVER share inventory data outside authorized supply chain teams

## Phase 3 — Rollback

1. If data integration flows were modified: revert to previous configuration
2. If inventory data was manually corrected: document and verify corrections
3. If flow frequency was changed: restore previous schedule
4. Verify inventory visibility is consistent after changes
5. Notify supply chain operations team of any data changes

## Common Issues

- symptoms: "Inventory levels show zero for items known to be in stock"
  diagnosis: "Data integration flow for the inventory source has not run or failed."
  resolution: "Check flow execution status. Fix any errors and re-run the flow."

- symptoms: "Inventory levels don't match source warehouse system"
  diagnosis: "Data propagation delay or flow execution frequency too low."
  resolution: "Check last flow execution time. Increase frequency if needed. Allow for propagation delay."

## Output Format

```yaml
root_cause: "inventory_visibility — <specific_cause>"
evidence:
  - type: visibility_error
    content: "<error details>"
severity: HIGH
mitigation:
  immediate: "Fix data integration and refresh inventory"
  long_term: "Implement reconciliation monitoring"
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
