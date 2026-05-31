---
title: "A1 — Gateway Deployment Failures"
description: "Diagnose Storage Gateway VM deployment and initial setup failures"
status: active
severity: HIGH
triggers:
  - "gateway deployment failed"
  - "cannot deploy gateway"
  - "gateway VM not starting"
  - "OVA import failed"
owner: devops-agent
objective: "Identify why a Storage Gateway VM cannot be deployed and resolve the issue"
context: "Storage Gateway is deployed as a VM on VMware ESXi, Microsoft Hyper-V, Linux KVM, or as an EC2 instance. Deployment failures include OVA/VHD import issues, insufficient hypervisor resources, network configuration problems, and disk allocation failures."
---

## Phase 1 — Triage

MUST:
- Verify hypervisor compatibility: ESXi 6.5+, Hyper-V 2012 R2+, KVM with QEMU
- Check VM resource allocation: minimum 4 vCPUs, 16 GB RAM, 80 GB root disk
- Verify network configuration: IP address, DNS, default gateway
- Check disk allocation: root disk + cache disk (150 GB min) + upload buffer (150 GB min)

SHOULD:
- Verify OVA/VHD image was downloaded correctly (check file size/checksum)
- Check hypervisor storage has sufficient space for VM disks
- Verify VM network adapter is connected to correct network/VLAN

MAY:
- Check hypervisor logs for VM creation errors
- Verify BIOS/UEFI settings if VM fails to boot

## Phase 2 — Remediate

MUST:
- Allocate minimum required resources for the gateway type
- Configure network with valid IP, DNS, and gateway
- Allocate separate disks for cache and upload buffer

SHOULD:
- Use SSD-backed storage for cache and upload buffer disks
- For EC2 deployment, use appropriate instance type (m5.xlarge or larger)
- Configure NTP for time synchronization

MAY:
- Deploy in a private subnet with VPC endpoints for enhanced security
- Use CloudFormation/Terraform for repeatable deployments

## Common Issues

- symptoms: "VM fails to boot after OVA import"
  diagnosis: "Hypervisor version incompatible or insufficient resources."
  resolution: "Verify hypervisor version. Allocate minimum 4 vCPUs and 16 GB RAM."

- symptoms: "Gateway VM has no network connectivity"
  diagnosis: "Network adapter not connected or VLAN misconfigured."
  resolution: "Check VM network adapter settings. Verify VLAN and IP configuration."

- symptoms: "Disk allocation errors during setup"
  diagnosis: "Insufficient hypervisor storage or wrong disk type."
  resolution: "Ensure hypervisor has space. Use thick-provisioned disks for cache."

## Output Format

```yaml
root_cause: "deployment_failure — <specific_cause>"
evidence:
  - type: vm_resources
    content: "<CPU, RAM, disk allocation>"
  - type: network_config
    content: "<IP, DNS, gateway settings>"
  - type: hypervisor
    content: "<hypervisor type and version>"
severity: HIGH
mitigation:
  immediate: "Fix the specific deployment blocker"
  long_term: "Document deployment requirements and automate with IaC"
```


## Safety Ratings

| Phase 2 Action | Rating | Rationale |
|---|---|---|
| Investigate and collect diagnostic data | GREEN | Read-only analysis; no state change |
| Modify configuration settings | YELLOW | Configuration change; reversible with backup |
| Restart or reboot components | YELLOW | Service interruption; recoverable |
| Delete or remove resources | RED | Potentially destructive; requires confirmation and backup |

## Escalation Conditions

- Production environment impact detected or imminent
- Multiple failed remediation attempts (>2 cycles without resolution)
- Security-related root cause identified (unauthorized access, data exposure)
- Issue requires cross-service or cross-account coordination
- Customer-facing service degradation exceeding SLA thresholds
- Regulatory or compliance implications discovered during investigation

## Data Sensitivity

- Gateway volumes contain production application data
- Tape archives may contain regulated backup data
- File shares expose S3 bucket contents to on-premises systems
- Classification: CONFIDENTIAL - production data in transit and at rest

## Prohibited Actions

- NEVER shut down or reboot gateway VMs during active data transfers
- NEVER delete volumes or tapes containing unverified backup data
- NEVER modify cache disk configuration on production gateways without maintenance window
- NEVER change iSCSI targets or NFS/SMB shares during business hours without approval

## Phase 3 -- Rollback

MUST:
- Document all changes made during Phase 2 remediation
- Verify rollback procedures exist before applying any fix
- Confirm service health after rollback using the same diagnostic steps from Phase 1
- Notify stakeholders if rollback is initiated in production

SHOULD:
- Maintain a timeline of all actions taken for post-incident review
- Capture before/after configuration snapshots
- Test rollback in non-production environment when possible
- Update runbook with lessons learned after incident resolution

MAY:
- Automate rollback procedures for frequently occurring issues
- Create CloudWatch alarms to detect regression after rollback

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
  - "NEVER suggest overly broad IAM permissions as a fix"
  - "NEVER suggest disabling security controls for troubleshooting"
  - "NEVER suggest deleting resources without confirmation"
