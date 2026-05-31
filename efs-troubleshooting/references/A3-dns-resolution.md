---
title: "A3 â€” EFS DNS Resolution Issues"
description: "Diagnose DNS resolution failures for EFS mount targets"
status: active
severity: MEDIUM
triggers:
  - "DNS resolution"
  - "cannot resolve"
  - "name resolution"
  - "fs-xxxxx.efs.region.amazonaws.com"
owner: devops-agent
objective: "Identify and fix DNS resolution issues for EFS file system endpoints"
context: "EFS uses DNS names in the format fs-xxxxx.efs.region.amazonaws.com. DNS resolution requires VPC DNS resolution and DNS hostnames to be enabled. Cross-VPC access requires additional DNS configuration."
---

## Phase 1 â€” Triage

MUST:
- Check VPC DNS settings: `aws ec2 describe-vpc-attribute --vpc-id <vpc-id> --attribute enableDnsSupport`
- Check VPC DNS hostnames: `aws ec2 describe-vpc-attribute --vpc-id <vpc-id> --attribute enableDnsHostnames`
- Test DNS resolution: `nslookup <fs-id>.efs.<region>.amazonaws.com`
- Verify mount target exists: `aws efs describe-mount-targets --file-system-id <fs-id>`

SHOULD:
- Check if using a custom DNS server that doesn't forward to VPC DNS
- Verify the instance can reach the VPC DNS resolver (x.x.x.2 address)

MAY:
- Check Route 53 Resolver rules if using hybrid DNS
- Verify DNS resolution from the specific subnet

## Phase 2 â€” Remediate

MUST:
- Enable VPC DNS support and DNS hostnames if disabled
- Ensure mount targets exist (DNS only resolves if mount targets are present)

SHOULD:
- Configure custom DNS servers to forward amazonaws.com queries to VPC DNS
- Use mount target IP as fallback if DNS cannot be fixed immediately

MAY:
- Set up Route 53 Resolver endpoints for cross-VPC DNS resolution

## Common Issues

- symptoms: "nslookup fails for EFS DNS name"
  diagnosis: "VPC DNS support or DNS hostnames is disabled."
  resolution: "Enable both enableDnsSupport and enableDnsHostnames on the VPC."

- symptoms: "DNS resolves but to wrong IP"
  diagnosis: "Mount target in a different AZ is being resolved."
  resolution: "This is normal â€” EFS DNS returns the mount target IP for the instance's AZ."

- symptoms: "DNS works from one VPC but not peered VPC"
  diagnosis: "Cross-VPC DNS resolution is not configured."
  resolution: "Use Route 53 Resolver or mount target IP directly for cross-VPC access."

## Output Format

```yaml
root_cause: "dns_resolution_failure â€” <specific_cause>"
evidence:
  - type: vpc_dns_config
    content: "<VPC DNS settings>"
  - type: dns_test
    content: "<nslookup results>"
severity: MEDIUM
mitigation:
  immediate: "Enable VPC DNS or use mount target IP"
  long_term: "Ensure VPC DNS is enabled in all VPCs using EFS"
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
