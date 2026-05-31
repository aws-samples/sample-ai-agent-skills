---
title: "G2 — SSH Access Issues"
description: "Diagnose SSH access problems to ParallelCluster head and compute nodes"
status: active
severity: MEDIUM
triggers:
  - "SSH access"
  - "cannot SSH"
  - "connection refused"
  - "key pair"
owner: devops-agent
objective: "Resolve SSH access issues to ParallelCluster nodes"
context: "SSH access to the head node uses the key pair specified in the cluster configuration. Compute nodes are accessible from the head node. Direct SSH to compute nodes from outside requires additional configuration."
---

## Phase 1 — Triage

MUST:
- Try SSH via pcluster: `pcluster ssh --cluster-name <cluster-name>`
- Check head node security group allows SSH (port 22): `aws ec2 describe-security-groups --group-ids <sg-id> --query 'SecurityGroups[*].IpPermissions[?FromPort==`22`]'`
- Verify head node is running: `aws ec2 describe-instances --filters "Name=tag:parallelcluster:node-type,Values=HeadNode" "Name=tag:parallelcluster:cluster-name,Values=<cluster-name>"`
- Check key pair matches: verify the key pair in cluster config matches your local key

SHOULD:
- Check if head node has a public IP (if in public subnet)
- Verify NACLs allow SSH traffic
- Check for bastion host or VPN if head node is in private subnet

MAY:
- Use SSM Session Manager as alternative to SSH
- Check for host key changes if getting warnings

## Phase 2 — Remediate

MUST:
- If security group blocking: add inbound SSH rule for your IP
- If wrong key: use the key pair specified in the cluster configuration
- If private subnet: use bastion host, VPN, or SSM Session Manager

SHOULD:
- Restrict SSH access to specific IP ranges
- Use SSM Session Manager for keyless access
- Configure SSH config for easier access

MAY:
- Implement SSH key rotation
- Set up audit logging for SSH access

## Common Issues

- symptoms: "Connection timed out when SSH to head node"
  diagnosis: "Security group not allowing SSH or head node in private subnet without bastion."
  resolution: "Add SSH inbound rule. Use bastion host or SSM for private subnet access."

- symptoms: "Permission denied (publickey)"
  diagnosis: "Wrong SSH key or wrong username."
  resolution: "Use the key pair from cluster config. Default user: ec2-user (AL2) or ubuntu."

## Output Format

```yaml
root_cause: "ssh_access — <specific_cause>"
evidence:
  - type: security_group
    content: "<SSH inbound rules>"
  - type: instance_status
    content: "<head node status>"
severity: MEDIUM
mitigation:
  immediate: "Fix security group or key pair"
  long_term: "Implement SSM Session Manager for secure access"
```


## Safety Ratings
- Overall Risk: HIGH
- Data Exposure Risk: HIGH — SSH access provides direct access to cluster nodes
- Service Disruption Risk: MEDIUM — SSH issues prevent cluster administration
- Compliance Impact: HIGH — SSH access controls are critical for security compliance

## Escalation Conditions
- SSH access to head node completely fails
- SSH key compromise detected
- Unauthorized SSH access attempts detected
- SSH configuration changes lock out administrators
- SSH access from unexpected IP addresses

## Data Sensitivity
- SSH private keys — RESTRICTED
- SSH configurations — INTERNAL
- Access logs — CONFIDENTIAL
- Authorized keys lists — CONFIDENTIAL
- Never expose SSH private keys or share them between users

## Prohibited Actions
- Do NOT allow SSH access from 0.0.0.0/0
- Do NOT share SSH private keys between users
- Do NOT disable SSH key-based authentication in favor of passwords
- Do NOT ignore unauthorized SSH access attempts
- Do NOT store SSH keys in unencrypted locations
- Do NOT grant SSH access to compute nodes without justification

## Phase 3 — Rollback
- Rotate SSH keys if compromise is suspected
- Restore previous SSH configurations
- Revert security group changes that affected SSH access
- Re-enable SSH access if accidentally disabled
- Verify SSH connectivity after changes
- Document SSH access issues and update key management procedures

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
