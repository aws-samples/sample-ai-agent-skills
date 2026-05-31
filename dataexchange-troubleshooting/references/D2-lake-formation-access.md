---
title: "D2 — Lake Formation Access"
description: "Diagnose Lake Formation data permission access issues via Data Exchange"
status: active
severity: HIGH
triggers: ["Lake Formation access denied", "cannot query shared table", "LF-tag permission error", "Glue catalog not visible"]
owner: devops-agent
objective: "Resolve Lake Formation data permission access issues including RAM invitations, LF grants, and Athena/Spectrum queries"
context: "Lake Formation data permissions in Data Exchange use AWS RAM for cross-account sharing. Subscribers must accept RAM invitations, create resource links, and configure Lake Formation grants."
---

## Phase 1 — Triage

MUST:
- Check RAM resource share invitations: `aws ram get-resource-share-invitations --query 'resourceShareInvitations[?status==`PENDING`]'`
- List Lake Formation permissions: `aws lakeformation list-permissions --principal '{"DataLakePrincipalIdentifier":"<role-arn>"}'`
- Check shared databases in Glue catalog: `aws glue get-databases --query 'DatabaseList[*].{Name:Name,CatalogId:CatalogId}'`

SHOULD:
- Verify resource links exist for shared databases/tables
- Check Lake Formation data lake settings for cross-account access
- Verify Athena workgroup configuration for querying shared data

MAY:
- Test query access via Athena: `aws athena start-query-execution --query-string "SELECT * FROM shared_db.table LIMIT 1"`
- Check Lake Formation audit logs for access denied events

## Phase 2 — Remediate

MUST:
- Accept pending RAM resource share invitation
- Create resource links for shared databases and tables in the local Glue catalog
- Grant Lake Formation permissions to querying principals

SHOULD:
- Configure Lake Formation to use IAM-based access control (not legacy Glue permissions)
- Verify the subscriber account is registered as a Lake Formation administrator

MAY:
- Set up LF-tag-based access control for fine-grained permissions
- Configure Athena query result encryption for shared data

## Safety Ratings

```
safety_ratings:
  - "ram get-resource-share-invitations: GREEN — read-only RAM check"
  - "lakeformation list-permissions: GREEN — read-only permission listing"
  - "glue get-databases: GREEN — read-only catalog query"
  - "lakeformation grant-permissions: YELLOW — modifies data access permissions"
```

## Escalation Conditions

- RAM invitation expired and provider must re-send
- Lake Formation administrator access required but not available
- Cross-account Lake Formation sharing requires organization-level configuration
- Shared tables use unsupported formats or partitioning schemes

## Data Sensitivity

- `aws ram get-resource-share-invitations`: MEDIUM — reveals cross-account sharing details
- `aws lakeformation list-permissions`: MEDIUM — reveals data access grants
- `aws glue get-databases`: LOW — returns database metadata
- `Athena query results`: HIGH — contains actual shared data

## Prohibited Actions

- NEVER grant Lake Formation permissions to overly broad principals (e.g., IAMAllowedPrincipals)
- NEVER disable Lake Formation access control to bypass permission issues
- NEVER create resource links to untrusted external catalogs
- NEVER query shared data without complying with the provider's data usage terms

## Phase 3 — Rollback

1. If permissions were granted incorrectly, revoke: `aws lakeformation revoke-permissions`
2. If resource links were created to wrong databases, delete them from Glue catalog
3. If RAM invitation was accepted incorrectly, contact provider to revoke
4. Verify no unauthorized data access occurred
5. Restore Lake Formation settings to previous state

## Common Issues

- symptoms: "Athena query fails — access denied on shared table"
  diagnosis: "Lake Formation permissions not granted to the querying role."
  resolution: "Grant SELECT permission via Lake Formation for the shared database/table."

## Output Format

```yaml
root_cause: "lake_formation_access — <specific_cause>"
evidence:
  - type: lakeformation_error
    content: "<error details>"
severity: HIGH
mitigation:
  immediate: "Fix Lake Formation permissions"
  long_term: "Document LF permission setup and use LF-tags for scalable access"
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
