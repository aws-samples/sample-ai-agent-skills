---
title: "H1 — Kerberos Authentication Issues"
description: "Diagnose Kerberos authentication problems on EMR"
status: active
severity: HIGH
triggers:
  - "Kerberos error"
  - "authentication failed"
  - "KDC issue"
  - "kinit failed"
owner: devops-agent
objective: "Identify and resolve Kerberos authentication issues on EMR"
context: "EMR supports Kerberos authentication via a cluster-dedicated KDC or external KDC (Active Directory). Kerberos is configured through security configurations and is immutable after cluster creation. Common issues: KDC connectivity, keytab problems, cross-realm trust, and ticket expiration."
---

## Phase 1 — Triage

MUST:
- Verify security configuration includes Kerberos: `aws emr describe-security-configuration --name <config-name>`
- Check KDC status on master node (via SSH)
- Verify Kerberos realm configuration
- Check for authentication errors in application logs

SHOULD:
- Test kinit from the master node
- Verify cross-realm trust configuration (if using AD)
- Check keytab file permissions
- Review Kerberos ticket cache

MAY:
- Check KDC logs for authentication failures
- Verify DNS resolution for KDC hostname

## Phase 2 — Remediate

MUST:
- Ensure KDC is running and accessible
- Fix keytab file permissions and principals
- Verify cross-realm trust configuration

SHOULD:
- Test authentication with kinit before running applications
- Ensure DNS resolves KDC hostname correctly
- Configure appropriate ticket lifetime

MAY:
- Set up Kerberos monitoring
- Create Kerberos troubleshooting documentation

## Common Issues

- symptoms: "kinit fails with KDC unreachable"
  diagnosis: "KDC not running or network connectivity issue."
  resolution: "Verify KDC service is running. Check security groups allow KDC ports (88, 749)."

- symptoms: "Cross-realm trust not working"
  diagnosis: "Trust configuration mismatch between EMR KDC and AD."
  resolution: "Verify realm names, trust passwords, and KDC hostnames match in both configurations."

## Safety Ratings

| Phase 2 Action | safety_rating | Rationale |
|---|---|---|
| Ensure KDC is running and accessible | GREEN | Service verification — non-destructive |
| Fix keytab file permissions and principals | YELLOW | Changes authentication config — verify correctness |
| Verify cross-realm trust configuration | GREEN | Verification — read-only |
| Configure appropriate ticket lifetime | GREEN | Configuration tuning — non-destructive |
| Create new cluster with fixed Kerberos config | RED | Requires new cluster — Kerberos is immutable after creation |

## Escalation Conditions

- Cluster runs production Spark jobs
- Fix requires cluster termination and recreation
- Kerberos authentication failure blocks all user access to the cluster
- Cross-realm trust involves Active Directory managed by a separate team

## Data Sensitivity

- **Classification: HIGH**
- Kerberos configuration reveals authentication architecture and trust relationships
- Keytab files contain authentication credentials — handle with extreme care
- Cross-realm trust configuration exposes organizational identity federation
- KDC logs may contain authentication attempts and principal names

## Prohibited Actions

- **NEVER** suggest terminating a cluster with running steps without saving state
- **NEVER** suggest disabling encryption on a cluster or its storage
- **NEVER** suggest disabling Kerberos authentication to work around auth failures
- **NEVER** suggest sharing keytab files or Kerberos credentials in logs or tickets

## Phase 3 — Rollback

| State-Changing Action | Rollback Step |
|---|---|
| Fixed keytab file permissions | Revert permissions if change causes other auth failures |
| Created new cluster with fixed Kerberos | Terminate new cluster if configuration is still incorrect |
| Modified cross-realm trust | Revert trust configuration on both KDC and AD sides |

## Output Format

```yaml
root_cause: "kerberos — <specific_cause>"
evidence:
  - type: security_config
    content: "<Kerberos security configuration>"
  - type: kdc_status
    content: "<KDC status and connectivity>"
severity: HIGH
mitigation:
  immediate: "Fix Kerberos configuration (may require new cluster)"
  long_term: "Document Kerberos setup and create validated security configurations"
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
