---
title: "F1 — EMR on EKS Virtual Cluster Issues"
description: "Diagnose EMR on EKS virtual cluster problems"
status: active
severity: HIGH
triggers:
  - "virtual cluster error"
  - "EMR on EKS setup"
  - "virtual cluster creation failed"
  - "EKS namespace"
owner: devops-agent
objective: "Identify and resolve EMR on EKS virtual cluster issues"
context: "EMR on EKS requires an existing EKS cluster with a dedicated namespace, RBAC configuration, and virtual cluster registration. Issues stem from EKS cluster configuration, RBAC permissions, namespace setup, or virtual cluster registration failures."
---

## Phase 1 — Triage

MUST:
- List virtual clusters: `aws emr-containers list-virtual-clusters`
- Describe virtual cluster: `aws emr-containers describe-virtual-cluster --id <id>`
- Verify EKS cluster exists: `aws eks describe-cluster --name <cluster-name>`
- Check namespace exists: `kubectl get namespace <emr-namespace>`

SHOULD:
- Verify RBAC configuration: `kubectl get rolebinding -n <emr-namespace>`
- Check EMR on EKS service-linked role
- Verify EKS cluster OIDC provider
- Check for EKS cluster access issues

MAY:
- Review EKS cluster security configuration
- Check for namespace resource quotas

## Phase 2 — Remediate

MUST:
- Create EKS namespace for EMR: `kubectl create namespace <emr-namespace>`
- Configure RBAC for EMR: `eksctl create iamidentitymapping`
- Register virtual cluster: `aws emr-containers create-virtual-cluster`

SHOULD:
- Use eksctl for simplified EMR on EKS setup
- Verify OIDC provider is configured for the EKS cluster
- Test with a simple Spark job after setup

MAY:
- Set up monitoring for virtual cluster health
- Create setup automation scripts

## Common Issues

- symptoms: "Virtual cluster creation fails"
  diagnosis: "EKS namespace not configured or RBAC missing."
  resolution: "Create namespace, configure RBAC, and retry virtual cluster creation."

- symptoms: "Virtual cluster in ARRESTED state"
  diagnosis: "EKS cluster or namespace deleted or inaccessible."
  resolution: "Verify EKS cluster and namespace exist. Recreate virtual cluster if needed."

## Safety Ratings

| Phase 2 Action | safety_rating | Rationale |
|---|---|---|
| Create EKS namespace for EMR | GREEN | Namespace creation — non-destructive |
| Configure RBAC for EMR | YELLOW | Changes EKS access control — review permissions |
| Register virtual cluster | GREEN | Registration — non-destructive |
| Verify OIDC provider configuration | GREEN | Verification — read-only |
| Recreate virtual cluster | YELLOW | Deletes and recreates — verify no active job runs |

## Escalation Conditions

- Cluster runs production Spark jobs
- Fix requires cluster termination and recreation
- Virtual cluster serves production EMR on EKS workloads
- EKS cluster is shared with other teams and RBAC changes affect them

## Data Sensitivity

- **Classification: HIGH**
- EKS namespace configuration reveals multi-tenant architecture
- RBAC bindings expose access control for data processing workloads
- OIDC provider configuration reveals identity federation setup
- Virtual cluster state reveals EMR on EKS deployment patterns

## Prohibited Actions

- **NEVER** suggest terminating a cluster with running steps without saving state
- **NEVER** suggest disabling encryption on a cluster or its storage
- **NEVER** suggest deleting EKS namespaces with active workloads
- **NEVER** suggest granting cluster-admin RBAC to EMR service accounts

## Phase 3 — Rollback

| State-Changing Action | Rollback Step |
|---|---|
| Created EKS namespace | Delete the namespace if virtual cluster setup fails |
| Configured RBAC for EMR | Remove RBAC role bindings if not needed |
| Registered virtual cluster | Delete virtual cluster registration if misconfigured |

## Output Format

```yaml
root_cause: "virtual_cluster — <specific_cause>"
evidence:
  - type: virtual_cluster_status
    content: "<virtual cluster state>"
  - type: eks_config
    content: "<EKS cluster and namespace configuration>"
severity: HIGH
mitigation:
  immediate: "Fix EKS configuration and virtual cluster setup"
  long_term: "Automate EMR on EKS setup and monitoring"
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
