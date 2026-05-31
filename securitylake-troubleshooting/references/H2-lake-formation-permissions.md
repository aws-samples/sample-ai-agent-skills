---
title: "H2 — Lake Formation Permission Issues"
description: "Diagnose Lake Formation permission and grant problems"
status: active
severity: HIGH
triggers:
  - "Lake Formation access denied"
  - "cannot grant permissions"
  - "table access denied"
  - "database permission"
owner: devops-agent
objective: "Resolve Lake Formation permission issues for Security Lake"
context: "Lake Formation controls access to Security Lake data. Subscribers need Lake Formation grants on Glue database and tables. The Security Lake admin must be a Lake Formation admin to manage grants."
---

## Phase 1 — Triage

MUST:
- Check Lake Formation admins: `aws lakeformation get-data-lake-settings`
- List permissions for subscriber: `aws lakeformation list-permissions --principal DataLakePrincipal={DataLakePrincipalIdentifier=<role-arn>}`
- Verify grants exist for database AND tables
- Check if IAMAllowedPrincipals is removed (required for Lake Formation mode)

SHOULD:
- Verify Lake Formation is in "use Lake Formation permissions" mode (not IAM-only)
- Check for conflicting Lake Formation permissions
- Verify cross-account Lake Formation grants if subscriber is in different account

## Phase 2 — Remediate

MUST:
- Add Security Lake admin as Lake Formation admin
- Grant SELECT on database: `aws lakeformation grant-permissions --principal DataLakePrincipal={DataLakePrincipalIdentifier=<arn>} --resource Database={Name=amazon_security_lake_glue_db_<region>} --permissions DESCRIBE`
- Grant SELECT on tables for subscriber principal
- Remove IAMAllowedPrincipals from database if using Lake Formation mode

SHOULD:
- Grant permissions on specific tables, not all tables
- Use Lake Formation tags for scalable permission management

## Output Format

```yaml
root_cause: "lake-formation — <specific_cause>"
evidence:
  - type: lf_permissions
    content: "<granted permissions>"
  - type: lf_settings
    content: "<Lake Formation mode>"
severity: HIGH
mitigation:
  immediate: "Grant required Lake Formation permissions"
  long_term: "Implement Lake Formation tag-based access control"
```


## Safety Ratings

| Phase 2 Action | Rating | Rationale |
|---|---|---|
| Investigate and collect diagnostic data | GREEN | Read-only analysis; no state change |
| Modify configuration settings | YELLOW | Configuration change; reversible with backup |
| Restart or reboot components | YELLOW | Service interruption; recoverable |
| Delete or remove resources | RED | Potentially destructive; requires confirmation and backup |

## Escalation Conditions

- Production environment impact detected or imminent
- Multiple failed remediation attempts (>2 cycles without resolution)
- Security-related root cause identified (unauthorized access, data exposure)
- Issue requires cross-service or cross-account coordination
- Customer-facing service degradation exceeding SLA thresholds
- Regulatory or compliance implications discovered during investigation

## Data Sensitivity

- Security logs contain authentication and authorization events
- Network flow logs may reveal infrastructure topology
- CloudTrail data contains API activity across all services
- Classification: HIGHLY CONFIDENTIAL - aggregated security telemetry requires strict access control

## Prohibited Actions

- NEVER delete or disable Security Lake in production without CISO approval
- NEVER modify subscriber access that could disrupt security monitoring
- NEVER change data retention policies without compliance team review
- NEVER remove log sources that are required for regulatory compliance

## Phase 3 -- Rollback

MUST:
- Document all changes made during Phase 2 remediation
- Verify rollback procedures exist before applying any fix
- Confirm service health after rollback using the same diagnostic steps from Phase 1
- Notify stakeholders if rollback is initiated in production

SHOULD:
- Maintain a timeline of all actions taken for post-incident review
- Capture before/after configuration snapshots
- Test rollback in non-production environment when possible
- Update runbook with lessons learned after incident resolution

MAY:
- Automate rollback procedures for frequently occurring issues
- Create CloudWatch alarms to detect regression after rollback

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
  - "NEVER suggest disabling security standards"
  - "NEVER suppress findings without investigation"
  - "NEVER remove security integrations"
