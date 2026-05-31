---
title: "A2 — Managed Instance Registration"
description: "Diagnose managed instance registration failures in Systems Manager"
status: active
severity: HIGH
triggers:
  - "instance not registering"
  - "managed instance registration"
  - "Fleet Manager missing instance"
  - "instance not appearing"
owner: devops-agent
objective: "Identify why an instance is not registering as a managed instance and fix registration"
context: "Managed instance registration requires SSM Agent, IAM instance profile, and network connectivity. New instances may take up to 5 minutes to appear. The instance must be in running state with a valid IAM role."
---

## Phase 1 — Triage

MUST:
- List all managed instances: `aws ssm describe-instance-information --query 'InstanceInformationList[*].{Id:InstanceId,Ping:PingStatus,Agent:AgentVersion,Platform:PlatformType}'`
- Check instance state and IAM profile: `aws ec2 describe-instances --instance-ids <instance-id> --query 'Reservations[*].Instances[*].{State:State.Name,IAM:IamInstanceProfile.Arn,LaunchTime:LaunchTime}'`
- Verify IAM role has SSM permissions: `aws iam list-attached-role-policies --role-name <role-name>`
- Check instance profile association: `aws ec2 describe-iam-instance-profile-associations --filters Name=instance-id,Values=<instance-id>`

SHOULD:
- Check if instance was recently launched (allow 5 minutes): `aws ec2 describe-instances --instance-ids <instance-id> --query 'Reservations[*].Instances[*].LaunchTime'`
- Verify SSM Agent version compatibility: `aws ssm describe-instance-information --filters Key=InstanceIds,Values=<instance-id> --query 'InstanceInformationList[*].AgentVersion'`
- Check CloudTrail for registration events: `aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventName,AttributeValue=UpdateInstanceInformation --max-results 10`

MAY:
- Check for instance metadata service (IMDSv2) issues: `aws ec2 describe-instances --instance-ids <instance-id> --query 'Reservations[*].Instances[*].MetadataOptions'`
- Verify no conflicting SSM Agent configurations

## Phase 2 — Remediate

MUST:
- Attach correct IAM instance profile if missing
- Ensure SSM Agent is installed and running
- Verify network path to SSM endpoints

SHOULD:
- Wait 5 minutes after instance launch before investigating
- Update SSM Agent to latest version
- Check instance metadata service accessibility

MAY:
- Use EC2 launch templates with pre-configured IAM roles
- Implement automated registration verification

## Common Issues

- symptoms: "Instance running but not in Fleet Manager"
  diagnosis: "Missing IAM instance profile or SSM Agent not installed."
  resolution: "Attach AmazonSSMManagedInstanceCore role and verify agent installation."

- symptoms: "Instance appears then disappears from managed instances"
  diagnosis: "Intermittent network connectivity to SSM endpoints."
  resolution: "Check security groups, NACLs, and route tables for stable HTTPS outbound."

- symptoms: "IMDSv2 required but agent cannot get credentials"
  diagnosis: "SSM Agent version too old to support IMDSv2 hop limit."
  resolution: "Update SSM Agent to latest version. Set metadata hop limit to 2 if using containers."

## Output Format

```yaml
root_cause: "managed_instance_registration — <specific_cause>"
evidence:
  - type: instance_state
    content: "<EC2 instance details>"
  - type: iam_profile
    content: "<IAM instance profile and policies>"
severity: HIGH
mitigation:
  immediate: "Fix IAM role attachment or agent installation"
  long_term: "Use launch templates with pre-configured SSM access"
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
