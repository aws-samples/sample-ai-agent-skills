---
title: "C1 â€” EFS Security Group Issues"
description: "Diagnose security group misconfigurations blocking EFS access"
status: active
severity: HIGH
triggers:
  - "security group"
  - "port 2049"
  - "NFS blocked"
  - "connection refused"
owner: devops-agent
objective: "Identify and fix security group rules blocking NFS traffic to EFS"
context: "EFS uses NFS protocol on TCP port 2049. Both the mount target security group (inbound) and the client security group (outbound) must allow this traffic. Security group drops are silent â€” no error message, just timeout."
---

## Phase 1 â€” Triage

MUST:
- Get mount target security groups: `aws efs describe-mount-target-security-groups --mount-target-id <mt-id>`
- Check inbound rules on mount target SG: `aws ec2 describe-security-groups --group-ids <sg-id> --query 'SecurityGroups[].IpPermissions'`
- Check outbound rules on client instance SG
- Test connectivity: `nc -zv <mount-target-ip> 2049`

SHOULD:
- Verify the source in the inbound rule matches the client's SG or CIDR
- Check if the SG was recently modified: `aws cloudtrail lookup-events --lookup-attributes AttributeKey=ResourceType,AttributeValue=AWS::EC2::SecurityGroup`

MAY:
- Check VPC flow logs for rejected traffic
- Verify NACLs are not blocking port 2049

## Phase 2 â€” Remediate

MUST:
- Add inbound TCP 2049 rule to mount target SG: `aws ec2 authorize-security-group-ingress --group-id <mt-sg-id> --protocol tcp --port 2049 --source-group <client-sg-id>`
- Verify client SG allows outbound to port 2049

SHOULD:
- Use security group references instead of CIDR for better maintainability
- Document the required SG rules for EFS access

MAY:
- Create a dedicated security group for EFS mount targets

## Common Issues

- symptoms: "Mount times out with no error message"
  diagnosis: "Security group is silently dropping NFS packets."
  resolution: "Add inbound TCP 2049 rule to mount target SG from client SG."

- symptoms: "Mount works from some instances but not others"
  diagnosis: "Different instances have different security groups."
  resolution: "Ensure all client security groups are allowed in the mount target SG inbound rules."

- symptoms: "Mount stopped working after security group change"
  diagnosis: "Someone removed or modified the NFS inbound rule."
  resolution: "Check CloudTrail for SG modifications. Re-add the rule."

## Output Format

```yaml
root_cause: "security_group_block â€” <specific_cause>"
evidence:
  - type: mount_target_sg
    content: "<inbound rules>"
  - type: client_sg
    content: "<outbound rules>"
severity: HIGH
mitigation:
  immediate: "Fix security group rules"
  long_term: "Use infrastructure as code to manage SG rules and prevent drift"
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
