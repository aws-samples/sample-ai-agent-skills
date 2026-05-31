# B2 — DataZone Environment Issues

## Symptoms
- Environment creation fails
- Environment can't connect to data source
- Environment profile errors

## Triage Commands

```bash
aws datazone list-environments --domain-identifier <domain-id> --project-identifier <project-id>
aws datazone get-environment --domain-identifier <domain-id> --identifier <env-id>
aws datazone list-environment-profiles --domain-identifier <domain-id> --max-results 20
```

## Common Causes
1. **Blueprint not enabled** — Required blueprint not enabled in domain
2. **Environment profile missing** — No profile for the blueprint type
3. **Data source inaccessible** — Glue/Redshift/Athena not reachable
4. **IAM role** — Environment role missing data source permissions
5. **VPC configuration** — Network issues for Redshift environments

## Resolution
- Enable blueprint in domain configuration first
- Create environment profile with appropriate blueprint
- Verify data source (Glue catalog, Redshift cluster) exists and is accessible
- Check environment role has permissions for target data source
- For Redshift: verify VPC, subnet, and security group configuration


---

## Safety Ratings

| Dimension | Rating |
|-----------|--------|
| Data Sensitivity | HIGH — Environments connect to data sources (Glue, Redshift, Athena) |
| Blast Radius | MEDIUM — Environment issues affect data access for project members |
| Rollback Complexity | MEDIUM — Environment recreation requires profile and blueprint configuration |
| Compliance Scope | HIGH — Environment IAM roles control data source access |

## Escalation Conditions

- Escalate to security if environment IAM roles have overly broad data source permissions
- Escalate to data platform team if data source connectivity issues arise
- Escalate to network team if VPC configuration blocks Redshift access

## Data Sensitivity

- **Environment IAM roles** have access to data sources — audit regularly
- **Data source connections** (Glue catalog, Redshift) may contain sensitive data
- **VPC configuration** for Redshift environments reveals network architecture

## PROHIBITED Actions

- Do NOT grant environment roles broader permissions than required for the specific data source
- Do NOT create environments in VPCs without proper security group configuration
- Do NOT delete environments with active subscriptions without notifying consumers

## Phase 3 — Rollback

- Delete the misconfigured environment and recreate with correct settings
- Restore environment IAM role permissions if they were modified
- Verify data source connectivity after recreation

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
