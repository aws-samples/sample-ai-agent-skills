---
title: "A1 — SSM Agent Not Connecting"
description: "Diagnose why an EC2 instance SSM Agent is not connecting to Systems Manager"
status: active
severity: HIGH
triggers:
  - "SSM Agent not connecting"
  - "instance not managed"
  - "ConnectionLost"
  - "PingStatus offline"
  - "not showing in Fleet Manager"
owner: devops-agent
objective: "Identify why the SSM Agent cannot connect and restore managed instance status"
context: "SSM Agent requires installation, IAM instance profile with AmazonSSMManagedInstanceCore, and outbound HTTPS access to three SSM endpoints. Missing any component causes connection failure. Amazon Linux 2 and Windows Server 2016+ have the agent pre-installed."
---

## Phase 1 — Triage

MUST:
- Check instance managed status: `aws ssm describe-instance-information --filters Key=InstanceIds,Values=<instance-id>`
- Check connection status: `aws ssm get-connection-status --target <instance-id>`
- Verify IAM instance profile: `aws ec2 describe-instances --instance-ids <instance-id> --query 'Reservations[*].Instances[*].{IAMRole:IamInstanceProfile.Arn,State:State.Name,SubnetId:SubnetId,VpcId:VpcId}'`
- Check IAM role policies: `aws iam list-attached-role-policies --role-name <role-name>`

SHOULD:
- Check if instance is running: `aws ec2 describe-instance-status --instance-ids <instance-id>`
- Verify security group outbound rules: `aws ec2 describe-security-groups --group-ids <sg-id> --query 'SecurityGroups[*].IpPermissionsEgress'`
- Check for VPC endpoints: `aws ec2 describe-vpc-endpoints --filters Name=vpc-id,Values=<vpc-id> --query 'VpcEndpoints[*].{Service:ServiceName,State:State}'`
- Check route table for internet access: `aws ec2 describe-route-tables --filters Name=association.subnet-id,Values=<subnet-id>`

MAY:
- Check SSM Agent logs via Run Command (if partially working): `aws ssm send-command --instance-ids <instance-id> --document-name AWS-RunShellScript --parameters commands=["cat /var/log/amazon/ssm/amazon-ssm-agent.log | tail -50"]`
- Verify NACLs are not blocking outbound HTTPS: `aws ec2 describe-network-acls --filters Name=association.subnet-id,Values=<subnet-id>`

## Phase 2 — Remediate

MUST:
- Attach IAM instance profile with AmazonSSMManagedInstanceCore if missing
- Ensure outbound HTTPS (443) to ssm, ssmmessages, and ec2messages endpoints
- Verify SSM Agent is installed and running on the instance

SHOULD:
- Create VPC endpoints for private subnets without internet access
- Update SSM Agent to the latest version
- Restart SSM Agent after configuration changes

MAY:
- Enable SSM Agent auto-update via State Manager association
- Configure CloudWatch alarms for SSM Agent connectivity

## Common Issues

- symptoms: "Instance not in describe-instance-information output"
  diagnosis: "Missing IAM instance profile or incorrect policy."
  resolution: "Attach instance profile with AmazonSSMManagedInstanceCore policy."

- symptoms: "PingStatus shows ConnectionLost"
  diagnosis: "Agent was connected but lost connectivity. Network or agent issue."
  resolution: "Check security groups, NACLs, route tables. Verify agent is running."

- symptoms: "Instance in private subnet, no VPC endpoints"
  diagnosis: "No outbound path to SSM endpoints."
  resolution: "Create VPC endpoints for ssm, ssmmessages, and ec2messages."

## Output Format

```yaml
root_cause: "ssm_agent_not_connecting — <specific_cause>"
evidence:
  - type: instance_info
    content: "<describe-instance-information output>"
  - type: iam_role
    content: "<IAM role and policies>"
  - type: network
    content: "<VPC endpoint or route table status>"
severity: HIGH
mitigation:
  immediate: "Fix IAM role or network connectivity"
  long_term: "Implement SSM Agent auto-update and connectivity monitoring"
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

- Parameter Store may contain secrets, API keys, and credentials
- Command output may contain system configuration details
- Session logs may capture sensitive terminal activity
- Classification: HIGHLY CONFIDENTIAL - secrets management and system access data

## Prohibited Actions

- NEVER execute untested commands via Run Command on production instances
- NEVER modify patch baselines during active patching windows
- NEVER delete SSM parameters containing production secrets without rotation
- NEVER deregister managed instances without confirming no active sessions

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
  - "NEVER suggest overly broad SSM document permissions"
  - "NEVER suggest disabling session logging"
  - "NEVER suggest sharing SSM documents publicly"
