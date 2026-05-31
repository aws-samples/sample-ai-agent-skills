---
title: "C2 — Bare Metal Provider Issues"
description: "Diagnose bare metal (Tinkerbell) provider problems in EKS Anywhere"
status: active
severity: HIGH
triggers: ["bare metal provisioning failed", "Tinkerbell error", "PXE boot failed", "BMC connection error", "hardware inventory issue"]
owner: devops-agent
objective: "Resolve bare metal provisioning issues including Tinkerbell stack, BMC connectivity, and PXE boot failures"
context: "Bare metal provider uses Tinkerbell for machine provisioning via PXE boot. Requires BMC/IPMI credentials, hardware inventory, and network infrastructure supporting PXE."
---

## Phase 1 — Triage

MUST:
- Check TinkerbellDatacenterConfig: `kubectl get tinkerbelldatacenterconfigs -A -o yaml --kubeconfig <mgmt-kubeconfig>`
- Check Tinkerbell stack health: `kubectl get pods -n eksa-system -l app.kubernetes.io/part-of=tinkerbell --kubeconfig <mgmt-kubeconfig>`
- Review hardware inventory: `kubectl get hardware -A --kubeconfig <mgmt-kubeconfig>`

SHOULD:
- Check BMC connectivity: `ipmitool -I lanplus -H <bmc-ip> -U <user> -P <pass> chassis status`
- Review Tinkerbell workflow status: `kubectl get workflows -A --kubeconfig <mgmt-kubeconfig>`
- Check Boots (PXE/DHCP) logs: `kubectl logs -n eksa-system deployment/boots --tail=200 --kubeconfig <mgmt-kubeconfig>`

MAY:
- Verify DHCP relay configuration if machines are on different VLANs
- Check network switch port configuration for PXE boot support

## Phase 2 — Remediate

MUST:
- Fix BMC credential issues in hardware inventory resources
- Ensure Tinkerbell stack pods are running and healthy
- Verify network supports PXE boot (DHCP, TFTP, HTTP)

SHOULD:
- Update hardware inventory with correct MAC addresses and BMC IPs
- Verify OS image URLs are accessible from the provisioning network

MAY:
- Restart Tinkerbell components if they are in a degraded state
- Re-create hardware inventory resources with corrected information

## Safety Ratings

```
safety_ratings:
  - "kubectl get hardware/workflows: GREEN — read-only status check"
  - "kubectl logs tinkerbell pods: GREEN — read-only log retrieval"
  - "ipmitool chassis status: GREEN — read-only BMC query"
  - "ipmitool chassis power on/off: RED — controls physical machine power state"
```

## Escalation Conditions

- Tinkerbell stack pods crash-loop and cannot be recovered
- BMC firmware issues prevent IPMI communication
- PXE boot fails due to network infrastructure (DHCP relay, VLAN) issues requiring network team
- Multiple hardware entries have incorrect MAC addresses or BMC credentials

## Data Sensitivity

- `kubectl get hardware`: HIGH — contains BMC credentials, MAC addresses, and IP assignments
- `kubectl get tinkerbelldatacenterconfigs`: MEDIUM — contains network configuration
- `ipmitool commands`: HIGH — require BMC credentials and control physical hardware
- `Tinkerbell logs`: MEDIUM — may contain provisioning details and network information

## Prohibited Actions

- NEVER power cycle production machines via BMC without explicit approval
- NEVER delete hardware inventory resources for machines that are actively running workloads
- NEVER expose BMC credentials in logs, support bundles, or public channels
- NEVER modify Tinkerbell workflows for machines that are mid-provisioning

## Phase 3 — Rollback

1. If a machine was provisioned incorrectly, power it off via BMC: `ipmitool -I lanplus -H <bmc-ip> -U <user> -P <pass> chassis power off`
2. Delete the corresponding Machine and Hardware resources from the management cluster
3. Re-create hardware inventory with corrected information
4. If Tinkerbell stack was modified, restore from the original deployment manifests
5. Verify remaining bare metal nodes are healthy and workloads are running

## Common Issues

- symptoms: "Machine stuck in Provisioning — PXE boot timeout"
  diagnosis: "Network does not support PXE boot or DHCP relay is misconfigured."
  resolution: "Verify DHCP relay points to Tinkerbell Boots service. Check VLAN and switch port configuration."

## Output Format

```yaml
root_cause: "bare_metal_provider — <specific_cause>"
evidence:
  - type: tinkerbell_error
    content: "<error details from Tinkerbell logs or BMC>"
severity: HIGH
mitigation:
  immediate: "Fix provisioning blocker"
  long_term: "Validate hardware inventory and network before provisioning"
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
