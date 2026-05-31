---
title: "C2 — Control Container Issues"
description: "Diagnose Bottlerocket control container and SSM connectivity problems"
status: active
severity: HIGH
triggers: ["ssm not connecting", "control container down", "cannot reach instance", "ssm session failed"]
owner: devops-agent
objective: "Restore control container and SSM Session Manager connectivity to Bottlerocket instances"
context: "The control container runs the SSM agent and is enabled by default. It provides apiclient and logdog for diagnostics."
---

## Phase 1 — Triage

MUST:
- Check SSM connectivity: `aws ssm describe-instance-information --filters "Key=InstanceIds,Values=<instance-id>" --query 'InstanceInformationList[0].{PingStatus:PingStatus,LastPing:LastPingDateTime}'`
- Verify instance is running: `aws ec2 describe-instance-status --instance-ids <instance-id> --query 'InstanceStatuses[0].{InstanceState:InstanceState.Name,SystemStatus:SystemStatus.Status,InstanceStatus:InstanceStatus.Status}'`
- Check IAM instance profile: `aws ec2 describe-iam-instance-profile-associations --filters "Name=instance-id,Values=<instance-id>"`

SHOULD:
- Verify security group allows outbound HTTPS (443): `aws ec2 describe-security-groups --group-ids <sg-id> --query 'SecurityGroups[0].IpPermissionsEgress'`
- Check for SSM VPC endpoints if in private subnet: `aws ec2 describe-vpc-endpoints --filters "Name=service-name,Values=com.amazonaws.<region>.ssm" --query 'VpcEndpoints[*].{Id:VpcEndpointId,State:State}'`

MAY:
- Check console output for control container errors: `aws ec2 get-console-output --instance-id <instance-id> --latest`

## Phase 2 — Remediate

MUST:
- Ensure IAM role includes `AmazonSSMManagedInstanceCore` managed policy
- Verify network path to SSM endpoints (internet gateway or VPC endpoints)
- Confirm control container is enabled in settings (enabled by default)

SHOULD:
- Create VPC endpoints for ssm, ssmmessages, and ec2messages if in private subnet
- Verify route table has appropriate routes for SSM endpoint access

MAY:
- Reboot instance to restart control container: `aws ec2 reboot-instances --instance-ids <instance-id>`

## Safety Ratings

```
safety_ratings:
  - "ssm describe-instance-information: GREEN — read-only SSM status check"
  - "ec2 describe-instance-status: GREEN — read-only instance health check"
  - "ec2 describe-vpc-endpoints: GREEN — read-only VPC endpoint enumeration"
  - "ec2 reboot-instances: RED — causes instance reboot and temporary workload disruption"
```

## Escalation Conditions

- Instance is running but SSM shows no connectivity for over 15 minutes
- Control container is disabled and no other access method is available
- VPC endpoint creation requires network team approval
- Multiple instances simultaneously lose SSM connectivity

## Data Sensitivity

- `aws ssm describe-instance-information`: LOW — returns connectivity status
- `aws ec2 describe-security-groups`: MEDIUM — reveals network security configuration
- `aws ec2 describe-vpc-endpoints`: LOW — returns VPC endpoint metadata
- `aws ec2 get-console-output`: MEDIUM — may contain boot-time configuration details

## Prohibited Actions

- NEVER disable the control container — it is the primary management interface
- NEVER modify security groups to allow all inbound traffic as a troubleshooting step
- NEVER create VPC endpoints without verifying the endpoint policy restricts access appropriately
- NEVER terminate an instance just because SSM is unreachable — try reboot first

## Phase 3 — Rollback

1. If security group was modified, revert: `aws ec2 revoke-security-group-egress` / `aws ec2 authorize-security-group-egress` with original rules
2. If VPC endpoints were created incorrectly, delete: `aws ec2 delete-vpc-endpoints --vpc-endpoint-ids <id>`
3. If IAM role was modified, revert to previous policy attachment
4. If instance was rebooted, verify workloads recovered: check ECS tasks or Kubernetes pods
5. Confirm SSM connectivity is restored: `aws ssm describe-instance-information`

## Common Issues

- symptoms: "SSM shows PingStatus: ConnectionLost for Bottlerocket instance"
  diagnosis: "Instance lacks outbound HTTPS access to SSM endpoints or IAM permissions are missing."
  resolution: "Verify outbound 443 in security group, check for SSM VPC endpoints in private subnets, and confirm IAM role."

- symptoms: "New Bottlerocket instance never appears in SSM managed instances"
  diagnosis: "IAM instance profile is missing or does not include SSM permissions."
  resolution: "Attach instance profile with AmazonSSMManagedInstanceCore policy. May require instance stop/start."

## Output Format

```yaml
root_cause: "control_container — <specific_cause>"
evidence:
  - type: ssm_error
    content: "<error details>"
severity: HIGH
mitigation:
  immediate: "Restore SSM connectivity"
  long_term: "Ensure VPC endpoints and IAM are configured in IaC"
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
  - "NEVER suggest overly broad IAM permissions as a fix"
  - "NEVER suggest disabling security controls for troubleshooting"
  - "NEVER suggest deleting resources without confirmation"
