# F1 — DataZone Business Catalog and Glossary Issues

## Symptoms
- Glossary term creation fails
- Search not returning expected results
- Glossary terms not associated with assets

## Triage Commands

```bash
aws datazone search --domain-identifier <domain-id> --search-scope GLOSSARY_TERM --search-text "<term>"
aws datazone search --domain-identifier <domain-id> --search-scope ASSET --search-text "<search>"
```

## Common Causes
1. **Glossary not created** — Must create glossary before terms
2. **Term name conflict** — Duplicate term names in same glossary
3. **Search index lag** — New terms take time to appear in search
4. **Association missing** — Terms not linked to assets
5. **Glossary scope** — Glossary is domain-scoped, not project-scoped

## Resolution
- Create glossary first, then add terms within it
- Use unique term names within each glossary
- Wait for search index to update after creating terms
- Associate terms with assets during publishing or editing
- Glossaries are visible across all projects in the domain
- Use glossary terms for consistent business terminology


---

## Safety Ratings

| Dimension | Rating |
|-----------|--------|
| Data Sensitivity | MEDIUM — Glossary terms define business vocabulary for data assets |
| Blast Radius | LOW — Glossary changes affect metadata, not data access |
| Rollback Complexity | LOW — Terms can be updated or recreated |
| Compliance Scope | MEDIUM — Consistent business terminology supports data governance |

## Escalation Conditions

- Escalate to data governance if glossary terms are inconsistent or misleading
- Escalate to data stewards if term definitions need business review

## Data Sensitivity

- **Glossary terms** define business vocabulary — domain-scoped visibility
- **Term associations** with assets reveal data classification

## PROHIBITED Actions

- Do NOT delete glossary terms that are associated with published assets
- Do NOT create duplicate or conflicting term definitions
- Do NOT modify term definitions without data steward approval

## Phase 3 — Rollback

- Restore previous term definitions
- Re-associate terms with assets if associations were broken
- Notify data stewards of any glossary changes

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
