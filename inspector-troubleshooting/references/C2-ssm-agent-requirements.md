---
title: "C2 — SSM Agent Requirements"
description: "Diagnose SSM agent issues affecting Inspector EC2 scanning"
status: active
severity: HIGH
triggers:
  - "SSM agent not running"
  - "instance not managed"
  - "SSM connectivity"
  - "agent version"
owner: devops-agent
objective: "Ensure SSM agent is properly configured for Inspector EC2 scanning"
context: "Inspector EC2 scanning depends on SSM agent to inventory installed packages. The agent must be running, the instance must have an IAM role with SSM permissions, and network connectivity to SSM endpoints must exist (internet, NAT, or VPC endpoints)."
---

## Phase 1 — Triage

MUST:
- Check SSM managed instances: `aws ssm describe-instance-information --query 'InstanceInformationList[*].{Id:InstanceId,PingStatus:PingStatus,AgentVersion:AgentVersion,Platform:PlatformType,LastPing:LastPingDateTime}'`
- Check specific instance: `aws ssm describe-instance-information --filters Key=InstanceIds,Values=<instance-id>`
- Check instance IAM role: `aws ec2 describe-iam-instance-profile-associations --filters Name=instance-id,Values=<instance-id>`
- Verify IAM role has SSM policy: `aws iam list-attached-role-policies --role-name <role-name>`

SHOULD:
- Check if SSM VPC endpoints exist (for private subnets): `aws ec2 describe-vpc-endpoints --filters Name=service-name,Values=com.amazonaws.<region>.ssm`
- Verify instance has outbound internet access or NAT gateway
- Check SSM agent version is current

MAY:
- Check SSM agent logs on the instance: `/var/log/amazon/ssm/` (Linux) or `%PROGRAMDATA%\Amazon\SSM\Logs` (Windows)
- Verify instance metadata service (IMDS) is accessible

## Phase 2 — Remediate

MUST:
- Install SSM agent if missing (pre-installed on Amazon Linux 2, Windows Server)
- Attach IAM role with AmazonSSMManagedInstanceCore managed policy
- Ensure network path to SSM endpoints exists

SHOULD:
- Use SSM Quick Setup for automated agent deployment
- Create VPC endpoints for SSM in private subnets (ssm, ssmmessages, ec2messages)
- Update SSM agent to latest version

MAY:
- Use AWS Config rule `ec2-instance-managed-by-systems-manager` for compliance
- Set up hybrid activation for on-premises instances

## Common Issues

- symptoms: "Instance not appearing in SSM Fleet Manager"
  diagnosis: "SSM agent not installed, not running, or no IAM role."
  resolution: "Install agent, start service, attach IAM role with SSM permissions."

- symptoms: "SSM agent running but PingStatus is ConnectionLost"
  diagnosis: "Network connectivity to SSM endpoints lost."
  resolution: "Check security groups, NACLs, route tables. Add VPC endpoints for private subnets."

- symptoms: "Old SSM agent version"
  diagnosis: "Agent auto-update not configured."
  resolution: "Update agent: `aws ssm send-command --document-name AWS-UpdateSSMAgent --targets Key=InstanceIds,Values=<instance-id>`"

## Safety Ratings

| Phase 2 Action | Rating | Rationale |
|---|---|---|
| Install SSM agent | YELLOW | Agent installation on running instances |
| Attach IAM role with SSM permissions | YELLOW | IAM changes affect instance capabilities |
| Create VPC endpoints for SSM | GREEN | Additive network configuration |
| Update SSM agent | GREEN | Agent update, minimal disruption |

## Escalation Conditions

- SSM agent deployment across production fleet — escalate to operations team
- IAM role changes on production instances — escalate to operations team
- VPC endpoint creation in production VPCs — escalate to network team
- SSM connectivity issues in private subnets — escalate to network team

## Data Sensitivity

| Triage Command | Sensitivity | Rationale |
|---|---|---|
| `aws ssm describe-instance-information` | MEDIUM | Exposes instance agent status, platform, and version |
| `aws ec2 describe-iam-instance-profile-associations` | MEDIUM | Exposes IAM role associations |
| `aws iam list-attached-role-policies` | MEDIUM | Exposes IAM policies |
| `aws ec2 describe-vpc-endpoints` | LOW | Network configuration |

## Prohibited Actions

- **NEVER** suggest disabling Inspector scanning as a troubleshooting step
- **NEVER** suggest suppressing all findings without individual review
- **NEVER** remove SSM agent from instances as a troubleshooting step
- **NEVER** grant overly broad IAM permissions (use AmazonSSMManagedInstanceCore)

## Phase 3 — Rollback

| Remediation Action | Rollback Step |
|---|---|
| Installed SSM agent | Uninstall SSM agent (not recommended for production) |
| Attached IAM role | Detach IAM role (caution: may affect other SSM functionality) |
| Created VPC endpoints | Delete VPC endpoints: `aws ec2 delete-vpc-endpoints` |
| Updated SSM agent | Cannot downgrade; deploy previous AMI if critical |

## Output Format

```yaml
root_cause: "ssm_agent — <specific_cause>"
evidence:
  - type: ssm_status
    content: "<SSM agent status and version>"
  - type: iam_role
    content: "<IAM role and policies>"
  - type: network
    content: "<network connectivity to SSM>"
severity: HIGH
mitigation:
  immediate: "Fix SSM agent, IAM role, or network connectivity"
  long_term: "Automate SSM agent deployment and monitoring"
```

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
  - "NEVER suggest disabling scanning to reduce findings"
  - "NEVER suggest ignoring critical vulnerabilities"
  - "NEVER suggest removing SSM agent to avoid scans"
