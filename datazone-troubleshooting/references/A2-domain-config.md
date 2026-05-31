# A2 — DataZone Domain Configuration Issues

## Symptoms
- Domain settings not applying
- Blueprint enablement failures
- SSO configuration issues

## Triage Commands

```bash
aws datazone get-domain --identifier <domain-id>
aws datazone list-environment-blueprints --domain-identifier <domain-id> --max-results 20
```

## Common Causes
1. **Blueprint not enabled** — Must enable blueprints before creating environments
2. **SSO not configured** — IAM Identity Center integration issues
3. **Service role permissions** — Role missing permissions for Glue/Redshift/Athena
4. **Domain not active** — Configuration changes need AVAILABLE domain
5. **Cross-account setup** — RAM sharing not configured

## Resolution
- Enable blueprints: `aws datazone put-environment-blueprint-configuration --domain-identifier <id> --environment-blueprint-identifier <blueprint-id> --enabled-regions <regions>`
- Configure SSO through IAM Identity Center for portal access
- Verify service role has permissions for target data sources
- Ensure domain is in AVAILABLE state before configuration changes


---

## Safety Ratings

| Dimension | Rating |
|-----------|--------|
| Data Sensitivity | HIGH — Domain configuration controls data access and governance |
| Blast Radius | HIGH — Blueprint and SSO changes affect all projects and users |
| Rollback Complexity | MEDIUM — Configuration changes may require re-enabling blueprints |
| Compliance Scope | HIGH — SSO and blueprint configuration affect access governance |

## Escalation Conditions

- Escalate to identity team if SSO configuration issues block user access
- Escalate to security if service role permissions are insufficient or overly broad
- Escalate to operations if blueprint enablement failures block environment creation

## Data Sensitivity

- **SSO configuration** controls user authentication — security-critical
- **Service role permissions** define data source access scope
- **Blueprint configurations** determine available environment types

## PROHIBITED Actions

- Do NOT modify SSO configuration without identity team coordination
- Do NOT grant service roles broader permissions than required for enabled blueprints
- Do NOT disable blueprints that have active environments without migration planning

## Phase 3 — Rollback

- Restore previous blueprint configuration
- Revert SSO settings if they were modified
- Restore service role permissions to previous state
- Verify all projects and environments are functional after rollback

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
