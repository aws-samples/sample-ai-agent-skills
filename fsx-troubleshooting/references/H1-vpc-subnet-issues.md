---
title: "H1 — VPC/Subnet Networking Issues"
description: "Diagnose VPC and subnet configuration problems affecting FSx connectivity"
status: active
severity: HIGH
triggers:
  - "VPC connectivity"
  - "subnet issue"
  - "cannot mount"
  - "network unreachable"
  - "ENI issue"
  - "security group"
owner: devops-agent
objective: "Identify and resolve VPC and subnet networking issues preventing FSx access"
context: "FSx file systems are deployed in VPC subnets with managed ENIs. Connectivity issues stem from security group rules, subnet routing, DNS resolution, VPC peering/transit gateway configuration, or ENI problems. Required ports differ by file system type."
---

## Phase 1 — Triage

MUST:
- Describe the file system for network configuration: `aws fsx describe-file-systems --file-system-ids <fs-id>`
- Check security group rules: `aws ec2 describe-security-groups --group-ids <sg-id>`
- Verify subnet configuration: `aws ec2 describe-subnets --subnet-ids <subnet-id>`
- Verify DNS resolution of the file system DNS name from the client

SHOULD:
- Check VPC route tables: `aws ec2 describe-route-tables --filters Name=association.subnet-id,Values=<subnet-id>`
- Verify ENIs are healthy: `aws ec2 describe-network-interfaces --filters Name=description,Values="*fsx*"`
- Check VPC peering or transit gateway if client is in a different VPC
- Verify NACLs are not blocking required ports: `aws ec2 describe-network-acls --filters Name=association.subnet-id,Values=<subnet-id>`

MAY:
- Test connectivity with telnet/nc to the file system port from the client
- Check VPC flow logs for dropped packets
- Verify DNS settings in VPC DHCP options

## Phase 2 — Remediate

MUST:
- Add security group rules for required ports: Lustre (988), Windows (445), ONTAP (111, 635, 2049, 4046, 11104, 11105), OpenZFS (111, 2049)
- Ensure client and FSx are in the same VPC or have proper routing (peering/TGW)
- Verify DNS resolution works from the client subnet

SHOULD:
- Use security group self-referencing rules for FSx internal communication
- Ensure NACLs allow required ports in both directions
- Document network architecture and required ports

MAY:
- Enable VPC flow logs for troubleshooting
- Set up network monitoring for FSx ENIs

## Common Issues

- symptoms: "Mount command hangs or times out"
  diagnosis: "Security group does not allow the required port from the client subnet."
  resolution: "Add inbound rule for the file system type port from the client security group or CIDR."

- symptoms: "DNS name does not resolve"
  diagnosis: "VPC DNS resolution is disabled or DHCP options are misconfigured."
  resolution: "Enable DNS resolution and DNS hostnames in the VPC. Verify DHCP options."

- symptoms: "Client in different VPC cannot access FSx"
  diagnosis: "VPC peering or transit gateway route is missing or security group does not allow cross-VPC traffic."
  resolution: "Configure VPC peering/TGW with proper routes. Update security group to allow the remote CIDR."

## Output Format

```yaml
root_cause: "vpc_subnet — <specific_cause>"
evidence:
  - type: network_config
    content: "<security groups, subnets, route tables>"
  - type: connectivity
    content: "<DNS resolution and port connectivity tests>"
  - type: eni_status
    content: "<FSx ENI health and configuration>"
severity: HIGH
mitigation:
  immediate: "Fix security group rules or routing configuration"
  long_term: "Document network requirements and automate with IaC"
```


---

## Safety Ratings

| Aspect | Rating |
|--------|--------|
| Service Impact | HIGH — FSx provides shared file storage for compute workloads, databases, and analytics |
| Data Risk | CRITICAL — File systems contain production data, application state, and potentially irreplaceable datasets |
| Recovery Complexity | HIGH — File system recreation requires backup restoration and client remounting across all consumers |
| Blast Radius | HIGH — File system outages affect all connected compute instances and dependent applications |

## Escalation Conditions

- File system enters FAILED or MISCONFIGURED state with no clear remediation path
- Multi-AZ failover not completing or standby file server unavailable during AZ event
- Data corruption detected on file system volumes or inconsistent backup state
- KMS key disabled or deleted causing file system inaccessibility
- Storage capacity at 100% utilization with no ability to scale (quota limits reached)
- Backup or restore operations failing on business-critical file systems

## Data Sensitivity

- File systems may contain regulated data (financial records, healthcare data, PII)
- Backup contents mirror production data sensitivity classifications
- Active Directory integration credentials reveal domain architecture and trust relationships
- NFS export policies and NTFS ACLs expose organizational access control structure
- ONTAP SVM configurations reveal multi-tenant data isolation boundaries

## Prohibited Actions

- NEVER suggest deleting file systems without confirming backups exist and are restorable
- NEVER suggest disabling encryption or using unencrypted file systems for any workload
- NEVER suggest reducing storage capacity — this operation is irreversible on all FSx types
- NEVER modify Active Directory integration on production Windows FSx without maintenance window
- NEVER remove or modify KMS key policies without verifying all dependent file systems
- NEVER change security group rules that could disconnect active file system clients

## Phase 3 — Rollback

- If file system configuration was changed: revert settings and verify client connectivity is restored
- If storage scaling failed: wait for pending operations to complete, verify file system returns to AVAILABLE
- If backup was deleted: check AWS Backup vault for retained copies, recreate from most recent available backup
- If security group was modified: restore previous rules immediately and verify all client mounts are functional
- If KMS key was affected: re-enable key or cancel deletion, verify file system recovers automatically
- Document all rollback actions with file system ID, affected clients, and data integrity verification steps

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
  - "NEVER suggest disabling encryption at rest"
  - "NEVER suggest removing AD integration without review"
  - "NEVER suggest public subnet placement"
