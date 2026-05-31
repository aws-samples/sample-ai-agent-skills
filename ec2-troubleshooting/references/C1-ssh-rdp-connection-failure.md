---
title: "C1 — SSH/RDP Connection Failure"
description: "Diagnose inability to connect to EC2 instances via SSH (Linux) or RDP (Windows)"
status: active
severity: HIGH
triggers:
  - "Connection timed out"
  - "Connection refused"
  - "Permission denied.*publickey"
  - "No route to host"
  - "port 22.*unreachable"
  - "port 3389.*unreachable"
owner: devops-agent
objective: "Identify the connectivity blocker and restore SSH/RDP access"
context: "SSH/RDP failures have a layered diagnosis path: instance state → status checks → security group → NACL → route table → OS-level firewall → SSH/RDP service → key/credentials. Work from outside in."
---

## Phase 1 — Triage (outside-in)

MUST:
- Verify instance is running: `aws ec2 describe-instances --instance-ids <id>` — state must be 'running'
- Check status checks: `aws ec2 describe-instance-status --instance-ids <id>` — both must pass
- Check security group allows inbound on port 22 (SSH) or 3389 (RDP) from your source IP
- Check NACL allows inbound on port 22/3389 AND outbound on ephemeral ports (1024-65535)
- Check route table has route to your source (IGW for public, VPN/DX for private)
- Verify instance has a public IP or you're connecting via private network

SHOULD:
- Use VPC Reachability Analyzer: `aws ec2 create-network-insights-path` to test connectivity
- Check system log for OS-level issues: `aws ec2 get-console-output --instance-id <id>`
- Verify the correct key pair was used (Linux SSH)
- Check if SSM Session Manager works as alternative access method

MAY:
- Check VPC flow logs for REJECT entries on port 22/3389
- Test with EC2 Instance Connect (Linux) as alternative

## Phase 2 — Remediate

Based on the layer where the block is found:

- Security group: Add inbound rule for SSH/RDP from your IP
- NACL: Add allow rule for inbound port 22/3389 and outbound ephemeral ports
- No public IP: Attach EIP or use SSM Session Manager / bastion host
- OS firewall (iptables/Windows Firewall): Fix via SSM or rescue instance
- SSH service down: Fix via SSM, serial console, or rescue instance
- Wrong key pair: Use EC2 Instance Connect or rescue instance to add correct key

## Common Issues

- symptoms: "Connection timed out (no response at all)"
  diagnosis: "Traffic not reaching the instance. Check: security group, NACL, route table, public IP, source IP."
  resolution: "Work through the network layers outside-in. Most common: SG missing inbound rule for your IP."

- symptoms: "Connection refused (immediate rejection)"
  diagnosis: "Traffic reaches the instance but SSH/RDP service is not listening. Instance is reachable but service is down."
  resolution: "Check system log for service errors. Use SSM to restart sshd/RDP service."

- symptoms: "Permission denied (publickey)"
  diagnosis: "SSH key mismatch. Wrong key pair, wrong username, or authorized_keys file corrupted."
  resolution: "Verify correct username (ec2-user, ubuntu, admin, centos). Use EC2 Instance Connect or rescue instance to fix authorized_keys."

## Safety Ratings
Rate each remediation action in Phase 2:
- GREEN: read-only diagnostic commands (describe-*, get-*, list-*)
- YELLOW: state-changing but recoverable (modify security group, restart instance, modify parameter)
- RED: destructive or irreversible (terminate instance, delete volume, modify key infrastructure)

Format as:
```
safety_ratings:
  - "describe-instances, describe-security-groups, describe-network-acls: GREEN — read-only"
  - "get-console-output, get-console-screenshot: GREEN — read-only diagnostics"
  - "VPC Reachability Analyzer: GREEN — read-only network path analysis"
  - "Add security group inbound rule for SSH/RDP: YELLOW — opens network access, recoverable"
  - "Add NACL rule: YELLOW — affects all instances in subnet, recoverable"
  - "Attach Elastic IP: YELLOW — may change public IP, recoverable by disassociating"
  - "Fix OS firewall via rescue instance: YELLOW — requires stop+start, careful disk manipulation"
  - "Modify authorized_keys via rescue instance: YELLOW — requires stop+start, disk attach/detach"
```

## Escalation Conditions
- Instance is in a production Auto Scaling group and cannot be stopped
- All access methods fail (SSH, RDP, SSM, Serial Console)
- Security group change would expose instance to broader network than intended
- Instance is in a private subnet with no bastion host or SSM access configured
- Multiple instances in the same subnet are simultaneously unreachable
- Fix requires modifying shared NACL affecting other production workloads

## Data Sensitivity
- HIGH: get-console-output (may contain credentials in boot logs, SSH key fingerprints)
- HIGH: describe-instances (reveals public/private IPs, security groups, key pair names, IAM roles)
- MEDIUM: VPC flow logs (reveals traffic patterns, source/destination IPs)
- MEDIUM: describe-security-groups (reveals network access rules and allowed CIDRs)

## Prohibited Actions
- NEVER suggest opening 0.0.0.0/0 on security groups to fix SSH/RDP connectivity
- NEVER suggest disabling the OS firewall entirely as a fix
- NEVER suggest sharing SSH private keys or embedding them in scripts
- NEVER suggest enabling password authentication for SSH as a workaround
- NEVER suggest modifying NACLs to allow all traffic (0.0.0.0/0 all ports) as a fix

## Phase 3 — Rollback
- If security group rule was added: remove the rule with `revoke-security-group-ingress`
- If NACL rule was added: remove the rule with `delete-network-acl-entry`
- If Elastic IP was attached: disassociate and release with `disassociate-address` and `release-address`
- If OS firewall was modified via rescue instance: re-attach volume to rescue instance and revert changes
- If authorized_keys was modified: restore from backup or re-attach to rescue instance to fix

## Output Format

```yaml
root_cause: "<sg_block|nacl_block|no_public_ip|route_missing|service_down|key_mismatch|os_firewall> — <detail>"
evidence:
  - type: network_check
    content: "<specific layer where block was found>"
severity: HIGH
mitigation:
  immediate: "Fix the blocking layer"
  long_term: "Use SSM Session Manager as backup access, configure VPC flow logs"
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
  - command: "describe-instances"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"
  - command: "get-console-output"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"
  - command: "ssm send-command"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"

## Prohibited Actions

prohibited_actions:
  - "NEVER suggest 0.0.0.0/0 inbound security group rules as a fix"
  - "NEVER suggest disabling instance metadata service"
  - "NEVER terminate instances without confirmation"
