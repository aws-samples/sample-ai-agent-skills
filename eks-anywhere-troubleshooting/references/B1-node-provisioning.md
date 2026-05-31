---
title: "B1 — Node Provisioning"
description: "Diagnose node provisioning failures in EKS Anywhere"
status: active
severity: HIGH
triggers: ["node not provisioning", "machine stuck provisioning", "node not joining cluster", "VM creation failed"]
owner: devops-agent
objective: "Resolve node provisioning issues across vSphere, bare metal, CloudStack, and Nutanix providers"
context: "Node provisioning is handled by CAPI provider controllers. Each provider has different provisioning mechanisms — vSphere clones VMs, bare metal uses Tinkerbell PXE boot, CloudStack deploys instances, Nutanix creates VMs via Prism."
---

## Phase 1 — Triage

MUST:
- Check machine status: `kubectl get machines -A -o wide --kubeconfig <mgmt-kubeconfig>`
- Review provider-specific machine objects: `kubectl get vspheremachines -A -o wide` or `kubectl get tinkerbellmachines -A -o wide`
- Check CAPI provider controller logs: `kubectl logs -n <provider-namespace> deployment/<provider-controller> --tail=200 --kubeconfig <mgmt-kubeconfig>`

SHOULD:
- Verify infrastructure resources are available (CPU, memory, storage, IP addresses)
- Check machine bootstrap status: `kubectl get machines -A -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.status.phase}{"\n"}{end}'`

MAY:
- Review cloud-init or ignition logs on the provisioned node (if accessible)
- Check DHCP server logs for IP assignment issues

## Phase 2 — Remediate

MUST:
- Fix provider credential or permission issues preventing VM/machine creation
- Resolve resource exhaustion (insufficient CPU, memory, storage, or IP addresses)
- Ensure VM templates or OS images match the target Kubernetes version

SHOULD:
- Verify network connectivity between provisioned nodes and the control plane endpoint
- Check that kubeadm bootstrap token has not expired

MAY:
- Manually delete stuck Machine resources to trigger re-provisioning
- Increase resource pool limits on the infrastructure provider

## Safety Ratings

```
safety_ratings:
  - "kubectl get machines/vspheremachines: GREEN — read-only status check"
  - "kubectl logs provider-controller: GREEN — read-only log retrieval"
  - "kubectl delete machine: YELLOW — triggers machine replacement, may cause workload disruption"
  - "provider resource pool changes: YELLOW — affects infrastructure capacity"
```

## Escalation Conditions

- Multiple machines stuck in Provisioning phase for more than 30 minutes
- Provider API returns persistent errors after credential and permission fixes
- Node provisioning succeeds but nodes fail to join the cluster repeatedly
- Infrastructure resource exhaustion cannot be resolved without capacity planning

## Data Sensitivity

- `kubectl get machines`: LOW — returns machine metadata and status
- `kubectl logs provider-controller`: MEDIUM — may contain provider API calls and credentials
- `cloud-init logs`: MEDIUM — may contain bootstrap tokens and cluster join information
- `provider API responses`: HIGH — may contain infrastructure credentials and network details

## Prohibited Actions

- NEVER delete control plane Machine resources to fix provisioning — this risks etcd quorum loss
- NEVER modify provider credentials in running controller pods — update the Secret resources instead
- NEVER provision nodes with mismatched Kubernetes versions
- NEVER bypass machine health checks by deleting MachineHealthCheck resources

## Phase 3 — Rollback

1. If a machine was provisioned incorrectly, delete the Machine resource: `kubectl delete machine <name> -n <namespace>` (worker nodes only)
2. Clean up orphaned infrastructure resources on the provider (VMs, disks, network interfaces)
3. For bare metal: power off incorrectly provisioned machines via BMC and reset hardware inventory
4. Verify remaining nodes are healthy: `kubectl get nodes --kubeconfig <workload-kubeconfig>`
5. Re-trigger provisioning by scaling the MachineDeployment: `kubectl scale machinedeployment <name> --replicas=<desired>`

## Common Issues

- symptoms: "Machine stuck in Provisioning — VM template not found"
  diagnosis: "The specified VM template does not exist or is inaccessible in the provider."
  resolution: "Import the correct OVA template matching the target Kubernetes version."

## Output Format

```yaml
root_cause: "node_provisioning — <specific_cause>"
evidence:
  - type: provisioning_error
    content: "<error details>"
severity: HIGH
mitigation:
  immediate: "Fix provisioning blocker"
  long_term: "Automate template management and resource monitoring"
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
