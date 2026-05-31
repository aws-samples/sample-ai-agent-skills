---
title: "A1 — Cluster Creation Failures"
description: "Diagnose EKS Anywhere cluster creation failures"
status: active
severity: CRITICAL
triggers: ["cluster creation failed", "eksctl anywhere create cluster error", "cannot create cluster"]
owner: devops-agent
objective: "Resolve EKS Anywhere cluster creation issues across all supported providers"
context: "EKS Anywhere creates clusters using Cluster API. Creation involves bootstrap cluster, provider validation, machine provisioning, and component installation."
---

## Phase 1 — Triage

MUST:
- Check eksctl-anywhere version: `eksctl anywhere version`
- Validate cluster spec: `eksctl anywhere generate clusterconfig <cluster-name> --provider <provider>`
- Review creation logs in the terminal output and bootstrap cluster logs
- Check Docker is running (required for bootstrap cluster): `docker info`

SHOULD:
- Verify provider credentials (vSphere vCenter, bare metal BMC, CloudStack, Nutanix)
- Check network connectivity to provider endpoints
- Validate cluster spec YAML against the schema: `eksctl anywhere validate cluster-config -f <cluster-spec.yaml>`

MAY:
- Review previous creation attempts in the local eksctl-anywhere logs directory
- Check available resources on the target infrastructure (CPU, memory, storage)

## Phase 2 — Remediate

MUST:
- Fix cluster spec validation errors (invalid fields, missing required values)
- Resolve provider credential issues (expired tokens, insufficient permissions)
- Ensure Docker daemon is running and has sufficient resources for bootstrap cluster

SHOULD:
- Clean up failed bootstrap cluster: `kind delete cluster --name <bootstrap-cluster-name>`
- Verify DNS resolution for provider endpoints and control plane endpoint
- Ensure control plane endpoint IP is available and not in use

MAY:
- Increase Docker resource limits for large clusters
- Pre-pull required container images to speed up creation

## Safety Ratings

```
safety_ratings:
  - "eksctl anywhere version: GREEN — read-only version check"
  - "eksctl anywhere create cluster: YELLOW — creates infrastructure resources, potentially costly"
  - "kind delete cluster: YELLOW — deletes bootstrap cluster, may lose creation state"
  - "docker info: GREEN — read-only Docker status check"
```

## Escalation Conditions

- Bootstrap cluster creation fails repeatedly with Docker resource errors
- Provider API returns persistent authentication failures after credential refresh
- Cluster creation hangs for more than 60 minutes without progress
- CAPI controllers crash-loop in the bootstrap cluster

## Data Sensitivity

- `eksctl anywhere version`: LOW — returns version metadata only
- `cluster-spec.yaml`: HIGH — may contain provider credentials, endpoint URLs, and network configuration
- `eksctl anywhere create cluster logs`: MEDIUM — contain infrastructure details and provisioning steps
- `docker info`: LOW — returns Docker daemon configuration

## Prohibited Actions

- NEVER share cluster spec files containing provider credentials in public channels
- NEVER force-delete a bootstrap cluster while creation is still in progress
- NEVER reuse a control plane endpoint IP that is already assigned to another cluster
- NEVER skip provider prerequisite validation to speed up creation

## Phase 3 — Rollback

1. If creation failed mid-way, clean up the bootstrap cluster: `kind delete cluster --name <bootstrap-cluster-name>`
2. Clean up any partially created infrastructure resources on the provider (VMs, load balancers, DHCP leases)
3. For vSphere: remove orphaned VMs from the target resource pool and folder
4. For bare metal: power off and reset any machines that were partially provisioned via BMC
5. Remove local state files: `rm -rf <cluster-name>/`

## Common Issues

- symptoms: "Bootstrap cluster fails to start — Docker daemon not running or insufficient resources"
  diagnosis: "Docker is not running or has insufficient memory/CPU allocated."
  resolution: "Start Docker daemon. Allocate at least 4 CPU and 8GB RAM to Docker."

- symptoms: "Provider validation fails — cannot connect to vCenter/BMC endpoint"
  diagnosis: "Network connectivity or credential issue with the infrastructure provider."
  resolution: "Verify endpoint URL, credentials, and network path. Check firewall rules."

## Output Format

```yaml
root_cause: "cluster_creation — <specific_cause>"
evidence:
  - type: creation_error
    content: "<error details from eksctl-anywhere output>"
severity: CRITICAL
mitigation:
  immediate: "Fix creation blocker"
  long_term: "Validate cluster specs in CI before creation"
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
  - command: "describe-cluster"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"
  - command: "kubectl get configmap aws-auth"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"
  - command: "kubectl get pods"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"

## Prohibited Actions

prohibited_actions:
  - "NEVER suggest system:masters group for troubleshooting"
  - "NEVER suggest disabling RBAC or Pod Security Standards"
  - "NEVER suggest running pods as root to fix permissions"
