---
title: "C1 — vSphere Provider Issues"
description: "Diagnose vSphere infrastructure provider problems in EKS Anywhere"
status: active
severity: HIGH
triggers: ["vSphere error", "vCenter connection failed", "VM template not found", "resource pool issue", "govc error"]
owner: devops-agent
objective: "Resolve vSphere-specific provisioning and connectivity issues"
context: "vSphere provider uses vCenter API to clone VMs from templates. Requires specific permissions, resource pools, folders, networks, and datastores."
---

## Phase 1 — Triage

MUST:
- Check VSphereDatacenterConfig: `kubectl get vspheredatacenterconfigs -A -o yaml --kubeconfig <mgmt-kubeconfig>`
- Check VSphereMachineConfig: `kubectl get vspheremachineconfigs -A -o yaml --kubeconfig <mgmt-kubeconfig>`
- Review CAPV controller logs: `kubectl logs -n capv-system deployment/capv-controller-manager --tail=200 --kubeconfig <mgmt-kubeconfig>`

SHOULD:
- Verify vCenter connectivity: `govc about -u <vcenter-url>`
- Check VM template exists: `govc vm.info <template-path>`
- Verify resource pool capacity: `govc pool.info <resource-pool-path>`

MAY:
- Check vCenter task history for failed VM operations
- Verify network and datastore accessibility from the resource pool

## Phase 2 — Remediate

MUST:
- Fix vCenter credential issues (expired password, insufficient permissions)
- Import correct VM template matching the target Kubernetes version
- Ensure resource pool has sufficient CPU, memory, and storage

SHOULD:
- Verify vCenter permissions include VM creation, cloning, and network assignment
- Check that the specified folder, network, and datastore exist and are accessible

MAY:
- Create dedicated resource pool for EKS Anywhere to isolate resources
- Pre-clone templates to reduce provisioning time

## Safety Ratings

```
safety_ratings:
  - "kubectl get vspheredatacenterconfigs: GREEN — read-only configuration check"
  - "govc about/vm.info/pool.info: GREEN — read-only vCenter queries"
  - "kubectl logs capv-controller: GREEN — read-only log retrieval"
  - "govc vm.clone: YELLOW — creates new VMs consuming vSphere resources"
```

## Escalation Conditions

- vCenter API consistently returns 503 or connection timeout errors
- VM template corruption prevents cloning operations
- Resource pool at capacity with no ability to expand
- vCenter permissions require vSphere administrator intervention

## Data Sensitivity

- `kubectl get vspheredatacenterconfigs`: MEDIUM — contains vCenter URL and datacenter details
- `kubectl get vspheremachineconfigs`: MEDIUM — contains resource pool, template, and network paths
- `govc commands`: MEDIUM — require vCenter credentials in environment or URL
- `CAPV controller logs`: HIGH — may contain vCenter API calls with credential details

## Prohibited Actions

- NEVER store vCenter credentials in plain text in cluster spec files committed to Git
- NEVER modify vSphere templates that are in use by running clusters
- NEVER delete resource pools or folders containing active EKS Anywhere VMs
- NEVER change vCenter network settings on running cluster VMs

## Phase 3 — Rollback

1. If VMs were created incorrectly, power off and delete them from vCenter: `govc vm.destroy <vm-path>`
2. Clean up orphaned disks on the datastore
3. If resource pool was modified, restore original CPU/memory reservations
4. Verify remaining cluster VMs are healthy and accessible
5. Re-import correct VM template if the wrong version was used

## Common Issues

- symptoms: "CAPV controller reports 'template not found' during provisioning"
  diagnosis: "VM template path in VSphereMachineConfig does not match actual vCenter inventory."
  resolution: "Verify template path with govc and update VSphereMachineConfig."

## Output Format

```yaml
root_cause: "vsphere_provider — <specific_cause>"
evidence:
  - type: vsphere_error
    content: "<error details from CAPV controller or govc>"
severity: HIGH
mitigation:
  immediate: "Fix vSphere configuration"
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
