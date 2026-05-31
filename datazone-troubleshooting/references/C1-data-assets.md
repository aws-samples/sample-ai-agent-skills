# C1 — DataZone Data Asset Issues

## Symptoms
- Asset not appearing in catalog
- Asset metadata incomplete
- Asset search not returning results

## Triage Commands

```bash
aws datazone search --domain-identifier <domain-id> --search-scope ASSET --search-text "<search-term>"
aws datazone get-asset --domain-identifier <domain-id> --identifier <asset-id>
aws datazone list-asset-revisions --domain-identifier <domain-id> --identifier <asset-id>
```

## Common Causes
1. **Asset not published** — Asset created but not published to catalog
2. **Search index delay** — New assets take time to appear in search
3. **Metadata incomplete** — Missing required metadata fields
4. **Wrong project** — Asset in different project than expected
5. **Asset type mismatch** — Wrong asset type for the data source

## Resolution
- Publish asset to make it discoverable in catalog
- Wait a few minutes for search index to update
- Complete all required metadata fields (name, description, schema)
- Verify asset is in the correct project
- Use appropriate asset type for the data source (Glue table, Redshift table, etc.)


---

## Safety Ratings

| Dimension | Rating |
|-----------|--------|
| Data Sensitivity | HIGH — Data assets represent actual data sources with potentially sensitive content |
| Blast Radius | LOW — Asset metadata changes don't affect underlying data |
| Rollback Complexity | LOW — Assets can be updated or recreated |
| Compliance Scope | HIGH — Asset metadata and classification support data governance |

## Escalation Conditions

- Escalate to data governance if assets are missing required metadata or classifications
- Escalate to data owners if assets are incorrectly cataloged

## Data Sensitivity

- **Asset metadata** describes data content and sensitivity — handle per governance policy
- **Schema information** reveals data structure — may be sensitive
- **Asset search results** may expose data catalog contents to authorized users

## PROHIBITED Actions

- Do NOT publish assets without completing required metadata fields
- Do NOT miscategorize asset sensitivity levels
- Do NOT delete assets that have active subscriptions

## Phase 3 — Rollback

- Update asset metadata to correct values
- Unpublish incorrectly published assets
- Restore asset revisions if available

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
