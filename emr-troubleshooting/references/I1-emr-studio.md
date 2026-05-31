---
title: "I1 — EMR Studio Issues"
description: "Diagnose EMR Studio access and configuration problems"
status: active
severity: MEDIUM
triggers:
  - "EMR Studio error"
  - "Studio access denied"
  - "workspace creation failed"
  - "Studio connectivity"
owner: devops-agent
objective: "Identify and resolve EMR Studio issues"
context: "EMR Studio provides a web-based IDE for data engineering. It requires IAM Identity Center (SSO) or IAM authentication, VPC configuration, S3 storage, and service role. Workspaces connect to EMR clusters or EMR Serverless applications."
---

## Phase 1 — Triage

MUST:
- Check Studio configuration: `aws emr describe-studio --studio-id <id>`
- Verify IAM Identity Center or IAM authentication setup
- Check Studio service role permissions
- Verify VPC and subnet configuration for Studio

SHOULD:
- Check workspace status and connectivity
- Verify S3 storage bucket for Studio
- Check for network connectivity to EMR clusters
- Review user/group session mappings

MAY:
- Check browser compatibility
- Review Studio access logs

## Phase 2 — Remediate

MUST:
- Fix authentication configuration (SSO or IAM)
- Ensure service role has required permissions
- Verify VPC connectivity between Studio and clusters

SHOULD:
- Configure appropriate session policies for users
- Test workspace creation and cluster attachment
- Document Studio setup requirements

MAY:
- Set up Studio monitoring
- Create user onboarding documentation

## Common Issues

- symptoms: "Cannot access EMR Studio"
  diagnosis: "IAM Identity Center not configured or user not assigned."
  resolution: "Configure IAM Identity Center. Assign users/groups to the Studio."

- symptoms: "Workspace cannot connect to cluster"
  diagnosis: "VPC connectivity issue between Studio and cluster."
  resolution: "Verify Studio and cluster are in the same VPC or have connectivity."

## Safety Ratings

| Phase 2 Action | safety_rating | Rationale |
|---|---|---|
| Fix authentication configuration (SSO or IAM) | YELLOW | Changes access control — verify user mappings |
| Ensure service role has required permissions | YELLOW | Changes role permissions — review scope |
| Verify VPC connectivity between Studio and clusters | GREEN | Verification — read-only |
| Configure session policies for users | YELLOW | Changes user permissions — review scope |
| Test workspace creation and cluster attachment | GREEN | Testing — non-destructive |

## Escalation Conditions

- Cluster runs production Spark jobs
- Fix requires cluster termination and recreation
- Studio access failure blocks data engineering team productivity
- IAM Identity Center changes affect other AWS services

## Data Sensitivity

- **Classification: HIGH**
- Studio configuration reveals IDE access patterns and user mappings
- IAM Identity Center settings expose organizational identity management
- VPC connectivity reveals network architecture between Studio and clusters
- S3 storage bucket may contain notebook code and data analysis results

## Prohibited Actions

- **NEVER** suggest terminating a cluster with running steps without saving state
- **NEVER** suggest disabling encryption on a cluster or its storage
- **NEVER** suggest granting admin access to all Studio users to resolve access issues
- **NEVER** suggest disabling authentication on EMR Studio

## Phase 3 — Rollback

| State-Changing Action | Rollback Step |
|---|---|
| Modified authentication configuration | Revert SSO/IAM settings to previous configuration |
| Updated service role permissions | Revert role policy to previous version |
| Changed session policies | Revert session policies to previous mappings |

## Output Format

```yaml
root_cause: "emr_studio — <specific_cause>"
evidence:
  - type: studio_config
    content: "<Studio configuration>"
  - type: connectivity
    content: "<workspace connectivity status>"
severity: MEDIUM
mitigation:
  immediate: "Fix Studio configuration or connectivity"
  long_term: "Standardize Studio setup and user management"
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
