# C2 — DataZone Asset Publishing Issues

## Symptoms
- Asset publishing fails
- Published asset not visible to other projects
- Asset revision errors

## Triage Commands

```bash
aws datazone get-asset --domain-identifier <domain-id> --identifier <asset-id>
aws datazone get-listing --domain-identifier <domain-id> --identifier <listing-id>
```

## Common Causes
1. **Missing metadata** — Required fields not populated
2. **Glossary terms** — Required glossary associations missing
3. **Data quality** — Quality gate blocking publication
4. **Approval required** — Publishing workflow requires approval
5. **Environment not connected** — No environment to validate data source

## Resolution
- Complete all required metadata before publishing
- Associate required glossary terms with the asset
- Run data quality checks and resolve failures
- Submit for approval if publishing workflow is configured
- Ensure project has an active environment connected to the data source
- Published assets create listings visible in the domain catalog


---

## Safety Ratings

| Dimension | Rating |
|-----------|--------|
| Data Sensitivity | HIGH — Publishing makes data assets discoverable across the domain |
| Blast Radius | MEDIUM — Published assets are visible to all domain users |
| Rollback Complexity | LOW — Assets can be unpublished |
| Compliance Scope | HIGH — Publishing controls data discoverability and access |

## Escalation Conditions

- Escalate to data governance if sensitive assets are published without proper classification
- Escalate to data owners if publishing fails for compliance-required assets

## Data Sensitivity

- **Published assets** are discoverable by all domain users — verify sensitivity classification
- **Listing metadata** is visible across the domain

## PROHIBITED Actions

- Do NOT publish assets containing sensitive data without proper classification and access controls
- Do NOT bypass data quality gates for compliance-required assets
- Do NOT publish assets without required glossary term associations

## Phase 3 — Rollback

- Unpublish incorrectly published assets
- Revoke any subscriptions granted to the incorrectly published asset
- Update asset metadata and republish with correct classification

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
