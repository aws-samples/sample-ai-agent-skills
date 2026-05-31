---
title: "C1 — VM Discovery Failures"
description: "Diagnose virtual machine discovery failures"
status: active
severity: MEDIUM
triggers: ["VM not discovered", "missing VM", "VM not in list", "discovery not working"]
owner: devops-agent
objective: "Resolve VM discovery issues in Backup Gateway"
context: "VM discovery is periodic. New VMs may take time to appear. Discovery depends on hypervisor connection and vCenter permissions."
---

## Phase 1 — Triage

MUST:
- List discovered VMs: `aws backup-gateway list-virtual-machines --query 'VirtualMachines[*].{Name:Name,Arn:ResourceArn,Path:Path,HypervisorId:HypervisorId}'`
- Verify hypervisor connection is active: `aws backup-gateway get-hypervisor --hypervisor-arn <arn>`
- Confirm the VM exists in vCenter

SHOULD:
- Check if the VM was recently created (discovery delay)
- Verify the vCenter user has read permissions on the VM's folder/datacenter

MAY:
- Check the VM's location in the vCenter inventory hierarchy
- Review if the VM is in a resource pool or folder the service account can access

## Phase 2 — Remediate

MUST:
- Ensure hypervisor connection is ONLINE
- Verify vCenter service account has ReadOnly permission at the datacenter level
- Wait for the next discovery cycle if the VM was recently created

SHOULD:
- Grant permissions at the datacenter level rather than individual VM level
- Verify the VM is not in a restricted folder

MAY:
- Trigger a manual test by updating the hypervisor connection
- Check if VM templates are being excluded from discovery

## Safety Ratings

```
safety_ratings:
  - "backup-gateway list-virtual-machines: GREEN — read-only VM listing"
  - "backup-gateway get-hypervisor: GREEN — read-only hypervisor status"
  - "vCenter permission verification: GREEN — read-only permission check"
```

## Escalation Conditions

- VMs are not discovered after 24 hours with an active hypervisor connection
- vCenter permissions require VMware admin changes
- Discovery fails for all VMs, not just specific ones
- VM discovery worked previously but stopped

## Data Sensitivity

- `aws backup-gateway list-virtual-machines`: MEDIUM — reveals VM inventory and paths
- vCenter inventory: MEDIUM — reveals infrastructure topology
- VM paths: LOW — reveals folder structure in vCenter

## Prohibited Actions

- NEVER assume a VM doesn't exist because it's not in the discovered list
- NEVER grant vCenter administrator permissions to fix discovery issues
- NEVER modify VM configurations to make them discoverable
- NEVER delete and re-add the hypervisor to force discovery without exhausting other options

## Phase 3 — Rollback

1. If vCenter permissions were modified, restore original permission assignments
2. If the hypervisor connection was updated, verify it reconnects successfully
3. Verify previously discovered VMs are still in the list
4. Check that backup schedules for existing VMs are not disrupted
5. Document the discovery issue and resolution

## Common Issues

- symptoms: "New VM created yesterday does not appear in list-virtual-machines"
  diagnosis: "VM discovery is periodic. New VMs may take time to appear."
  resolution: "Wait for the next discovery cycle. Verify hypervisor connection is ONLINE and vCenter user has ReadOnly permission on the VM's datacenter."

## Output Format

```yaml
root_cause: "vm_discovery — <specific_cause>"
evidence:
  - type: discovery_error
    content: "<error details>"
severity: MEDIUM
mitigation:
  immediate: "Fix discovery prerequisites"
  long_term: "Ensure broad vCenter permissions for discovery"
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
  - "NEVER suggest disabling vault lock"
  - "NEVER suggest deleting backup plans without review"
  - "NEVER suggest removing cross-account backup copies"
