---
title: "A2 â€” EFS Mount Timeout"
description: "Diagnose EFS mount operations that hang or time out"
status: active
severity: HIGH
triggers:
  - "mount timeout"
  - "mount hangs"
  - "Connection timed out"
  - "mount.nfs: Connection timed out"
owner: devops-agent
objective: "Identify network or configuration issues causing mount timeouts"
context: "Mount timeouts are almost always network-related: security groups blocking port 2049, no mount target in the AZ, NACLs blocking traffic, or routing issues in the VPC."
---

## Phase 1 â€” Triage

MUST:
- Verify mount target exists in the instance's AZ: `aws efs describe-mount-targets --file-system-id <fs-id>`
- Check security group inbound rules on mount target: `aws ec2 describe-security-groups --group-ids <sg-id>`
- Check instance security group outbound rules
- Test TCP connectivity to port 2049: `nc -zv <mount-target-ip> 2049`

SHOULD:
- Check VPC NACLs for the subnet: `aws ec2 describe-network-acls --filters Name=association.subnet-id,Values=<subnet-id>`
- Verify route table has local route for the VPC CIDR
- Check if the instance is in a private subnet without proper routing

MAY:
- Check VPC flow logs for rejected traffic to port 2049
- Verify no firewall appliance is blocking NFS traffic

## Phase 2 â€” Remediate

MUST:
- Add inbound TCP 2049 rule to mount target security group
- Create mount target in the correct AZ if missing
- Fix NACL rules if blocking NFS traffic

SHOULD:
- Use mount target DNS name instead of IP for automatic AZ resolution
- Set mount timeout options: `mount -t nfs4 -o timeo=600 <target>:/ /mnt/efs`

MAY:
- Enable VPC flow logs for ongoing network debugging

## Common Issues

- symptoms: "Mount hangs indefinitely"
  diagnosis: "Security group or NACL is silently dropping packets to port 2049."
  resolution: "Check both SG and NACL rules. SG drops are silent (no REJECT, just DROP)."

- symptoms: "Mount works from one instance but not another"
  diagnosis: "Instances are in different AZs or subnets with different security groups."
  resolution: "Verify mount target exists in the failing instance's AZ. Check its security group."

- symptoms: "Timeout after VPC peering or Transit Gateway setup"
  diagnosis: "Mount target DNS resolves to an IP not routable from the peered VPC."
  resolution: "Use the mount target IP directly or set up DNS resolution across VPCs."

## Output Format

```yaml
root_cause: "mount_timeout â€” <specific_cause>"
evidence:
  - type: network_test
    content: "<connectivity test results>"
  - type: security_groups
    content: "<SG rules analysis>"
severity: HIGH
mitigation:
  immediate: "Fix security group or create mount target in correct AZ"
  long_term: "Automate mount target provisioning across all AZs"
```

---

## Safety Ratings

| Aspect | Rating |
|--------|--------|
| Service Impact | HIGH — EFS provides shared storage for multiple compute instances |
| Data Risk | CRITICAL — File system contains application data and shared state |
| Recovery Complexity | HIGH — Data loss from file system deletion is unrecoverable without backups |
| Blast Radius | HIGH — Mount or performance issues affect all connected instances |

## Escalation Conditions

- File system unavailable or mount failures affecting multiple production instances
- Throughput degradation causing application timeouts across the fleet
- Burst credits depleted with no provisioned throughput configured
- Security group changes blocking NFS access for running workloads
- Replication lag exceeding RPO thresholds for disaster recovery
- IAM authorization failures preventing application access to file system

## Data Sensitivity

- File system contents may include application data, configuration files, and shared secrets
- Access point configurations reveal application directory structure and POSIX permissions
- Mount target IPs and DNS names are internal infrastructure identifiers
- Encryption keys (KMS) protect data at rest — key access must be tightly controlled
- EFS access logs may contain file paths revealing business logic or data organization

## PROHIBITED Actions

- NEVER suggest deleting a file system with active mount targets
- NEVER remove mount targets from availability zones with running instances
- NEVER change security group rules without verifying all connected instance groups
- NEVER disable encryption in transit on file systems handling sensitive data
- NEVER modify access point POSIX permissions without understanding application requirements
- NEVER switch throughput modes during peak usage without capacity planning

## Phase 3 — Rollback

- If mount target was removed: recreate in the same subnet with the same security groups
- If security group rules were changed: restore previous inbound TCP 2049 rules from backup
- If throughput mode was changed: revert to previous mode and monitor burst credit balance
- If access point permissions were modified: restore original POSIX UID/GID and directory path
- If lifecycle policy was changed: revert transition rules and verify IA storage class files
- Document all rollback actions with file system ID, mount target IDs, and timestamps
## Safety Ratings

safety_ratings:
  - "Phase 1 triage commands (describe/get/list): GREEN â€” read-only"
  - "Phase 2 configuration changes: YELLOW â€” state-changing but recoverable"
  - "Phase 2 resource deletion or security changes: RED â€” destructive or irreversible"

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
  - "NEVER suggest disabling encryption in transit"
  - "NEVER suggest removing IAM authorization"
  - "NEVER suggest overly broad file system policies"
