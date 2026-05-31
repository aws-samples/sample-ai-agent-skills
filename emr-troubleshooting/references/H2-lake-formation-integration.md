---
title: "H2 — Lake Formation Integration"
description: "Diagnose EMR and Lake Formation integration issues"
status: active
severity: HIGH
triggers:
  - "Lake Formation EMR error"
  - "LF integration failed"
  - "EMR LF access denied"
  - "fine-grained access EMR"
owner: devops-agent
objective: "Identify and resolve EMR-Lake Formation integration issues"
context: "EMR integrates with Lake Formation for fine-grained access control on data lake tables. This requires a security configuration with LF enabled, proper IAM roles, and LF permissions. LF integration is set at cluster creation and cannot be changed."
---

## Phase 1 — Triage

MUST:
- Check security configuration for LF: `aws emr describe-security-configuration --name <config-name>`
- Verify LF permissions for the EMR execution role
- Check IAM role configuration for LF integration
- Verify the cluster was launched with LF-enabled security configuration

SHOULD:
- Check LF data lake settings
- Verify the EMR instance profile has lakeformation:GetDataAccess
- Test table access with a simple query
- Check for IAMAllowedPrincipals on target tables

MAY:
- Review LF permission grants for the EMR role
- Check CloudTrail for LF access events

## Phase 2 — Remediate

MUST:
- Create security configuration with LF enabled
- Grant LF permissions to the EMR execution role
- Ensure IAM role includes lakeformation:GetDataAccess

SHOULD:
- Launch a new cluster with LF-enabled security configuration
- Test access before running production workloads
- Document LF-EMR integration requirements

MAY:
- Set up LF permission monitoring for EMR roles
- Create standardized LF-EMR security configurations

## Common Issues

- symptoms: "Access denied when querying LF-managed table from EMR"
  diagnosis: "EMR role lacks LF permissions or security config doesn't enable LF."
  resolution: "Grant LF SELECT permission. Verify security configuration enables LF."

- symptoms: "LF integration not working on existing cluster"
  diagnosis: "Security configuration is immutable after cluster creation."
  resolution: "Create a new cluster with LF-enabled security configuration."

## Safety Ratings

| Phase 2 Action | safety_rating | Rationale |
|---|---|---|
| Grant LF permissions to EMR execution role | YELLOW | Changes data access permissions — review scope |
| Create security configuration with LF enabled | GREEN | New configuration — non-destructive |
| Launch new cluster with LF-enabled config | RED | Requires new cluster — LF config is immutable |
| Test access before production workloads | GREEN | Verification — non-destructive |
| Set up LF permission monitoring | GREEN | Monitoring — non-destructive |

## Escalation Conditions

- Cluster runs production Spark jobs
- Fix requires cluster termination and recreation
- LF integration failure blocks access to governed data lake tables
- Permission changes affect fine-grained access control for sensitive data

## Data Sensitivity

- **Classification: HIGH**
- LF permissions reveal fine-grained data access control policies
- Security configuration exposes data governance architecture
- LF data lake settings reveal organizational data classification
- IAMAllowedPrincipals configuration reveals legacy access patterns

## Prohibited Actions

- **NEVER** suggest terminating a cluster with running steps without saving state
- **NEVER** suggest disabling encryption on a cluster or its storage
- **NEVER** suggest granting Super permissions in Lake Formation to bypass access controls
- **NEVER** suggest removing LF governance to work around permission errors

## Phase 3 — Rollback

| State-Changing Action | Rollback Step |
|---|---|
| Granted LF permissions to EMR role | Revoke LF permissions if they were not intended |
| Created new security configuration | Delete the security configuration if not needed |
| Launched new cluster with LF config | Terminate the new cluster if LF integration is incorrect |

## Output Format

```yaml
root_cause: "lf_integration — <specific_cause>"
evidence:
  - type: security_config
    content: "<LF security configuration>"
  - type: lf_permissions
    content: "<LF permissions for EMR role>"
severity: HIGH
mitigation:
  immediate: "Fix LF permissions or create new cluster with LF config"
  long_term: "Standardize LF-EMR integration patterns"
```

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
  - "NEVER suggest disabling encryption"
  - "NEVER suggest public subnet placement for clusters"
  - "NEVER suggest removing Kerberos authentication"
