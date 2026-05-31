---
title: "C4 — RDS Protection Issues"
description: "Diagnose GuardDuty RDS protection and database threat detection issues"
status: active
severity: MEDIUM
triggers:
  - "RDS protection"
  - "RDS findings"
  - "database threats"
  - "RDS login activity"
  - "RDS suspicious access"
owner: devops-agent
objective: "Identify and resolve RDS protection issues for database threat detection"
context: "GuardDuty RDS Protection monitors RDS login activity to detect suspicious access patterns such as brute force attacks, anomalous login behavior, and access from known malicious IPs. It supports Aurora MySQL and Aurora PostgreSQL databases."
---

## Phase 1 — Triage

MUST:
- Check RDS protection status: `aws guardduty get-detector --detector-id <detector-id>` (check RDS_LOGIN_EVENTS feature)
- List RDS-related findings: `aws guardduty list-findings --detector-id <detector-id> --finding-criteria '{"Criterion":{"type":{"Prefix":"CredentialAccess:RDS"}}}' --max-results 10`
- Verify Aurora databases exist: `aws rds describe-db-clusters --query 'DBClusters[?Engine==`aurora-mysql` || Engine==`aurora-postgresql`]'`
- Check the detector is ENABLED

SHOULD:
- Verify the Aurora cluster has login activity monitoring supported
- Check organization configuration for RDS protection
- Review the service-linked role permissions

MAY:
- Check Aurora cluster connection logs for correlation
- Verify Aurora cluster security group configuration

## Phase 2 — Remediate

MUST:
- Enable RDS protection if disabled: `aws guardduty update-detector --detector-id <detector-id> --features '[{"Name":"RDS_LOGIN_EVENTS","Status":"ENABLED"}]'`
- Verify the feature status changed to ENABLED

SHOULD:
- Enable RDS protection across all member accounts
- Monitor RDS finding generation after enabling

MAY:
- Set up automated response for RDS credential access findings
- Review Aurora database access patterns and security groups

## Common Issues

- symptoms: "No RDS findings despite Aurora databases running"
  diagnosis: "RDS protection is not enabled, or the databases are not Aurora MySQL/PostgreSQL."
  resolution: "Enable RDS_LOGIN_EVENTS feature. Verify databases are supported Aurora engines."

- symptoms: "RDS protection enabled but no findings for non-Aurora RDS instances"
  diagnosis: "RDS protection only supports Aurora MySQL and Aurora PostgreSQL. Standard RDS instances are not monitored."
  resolution: "RDS protection is limited to Aurora. Use other security controls for non-Aurora databases."

## Safety Ratings

| Phase 2 Action | Rating | Rationale |
|---|---|---|
| Enable RDS protection | YELLOW | Configuration change; reversible |
| Enable across member accounts | YELLOW | Org-wide change; reversible |

## Escalation Conditions

- Detector configuration changes
- RDS protection enablement/disablement

## Data Sensitivity

| Triage Command | Sensitivity | Rationale |
|---|---|---|
| `get-detector` (RDS feature) | LOW | Feature status |
| `list-findings` (RDS) | HIGH | Database threat details |
| `describe-db-clusters` | MEDIUM | Database configuration |

## Prohibited Actions

- NEVER suggest disabling GuardDuty detector
- NEVER suggest suppressing all findings
- NEVER suggest disabling S3/EKS/Lambda protection

## Phase 3 — Rollback

| Remediation | Rollback Step |
|---|---|
| RDS protection enablement | Disable via `update-detector` (not recommended) |

## Output Format

```yaml
root_cause: "rds_protection — <specific_cause>"
evidence:
  - type: detector_features
    content: "<RDS protection status>"
  - type: findings
    content: "<RDS-related findings>"
  - type: databases
    content: "<Aurora cluster details>"
severity: MEDIUM
mitigation:
  immediate: "Enable RDS protection for Aurora databases"
  long_term: "Monitor RDS finding patterns and secure database access"
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
  - command: "get-findings"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"
  - command: "list-detectors"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"
  - command: "get-threat-intel-set"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"

## Prohibited Actions

prohibited_actions:
  - "NEVER suggest disabling GuardDuty to stop alerts"
  - "NEVER suggest archiving findings without investigation"
  - "NEVER expose finding details in public logs"
