---
title: "B1 — Remote Desktop Failures"
description: "Diagnose Fleet Manager remote desktop connection failures"
status: active
severity: HIGH
triggers: ["remote desktop not working", "RDP via Fleet Manager failed", "cannot connect remote desktop"]
owner: devops-agent
objective: "Resolve Fleet Manager remote desktop connection issues"
context: "Fleet Manager remote desktop uses Session Manager tunneling. It requires SSM Agent 3.0+, proper IAM permissions, and does NOT require inbound security group rules."
---

## Phase 1 — Triage

MUST:
- Verify SSM Agent version is 3.0+: `aws ssm describe-instance-information --filters Key=InstanceIds,Values=<instance-id> --query 'InstanceInformationList[*].{Id:InstanceId,Agent:AgentVersion,Platform:PlatformType}'`
- Check connection status: `aws ssm get-connection-status --target <instance-id>`
- Verify IAM permissions include `ssm:StartSession` with the `AWS-StartPortForwardingSession` document

SHOULD:
- Check if Session Manager plugin is installed on the client machine
- Verify the instance OS supports remote desktop (Windows RDP or Linux with DCV)

MAY:
- Review Session Manager preferences: `aws ssm get-document --name SSM-SessionManagerRunShell`
- Check CloudTrail for StartSession errors

## Phase 2 — Remediate

MUST:
- Update SSM Agent to 3.0+ if outdated
- Ensure IAM policy allows `ssm:StartSession` for the target instance
- For Windows: verify RDP service is running on the instance

SHOULD:
- Install Session Manager plugin on the client if missing
- For Linux: install and configure NICE DCV for graphical remote desktop

MAY:
- Configure Session Manager logging for audit compliance

## Common Issues

- symptoms: "Remote desktop button grayed out in Fleet Manager"
  diagnosis: "SSM Agent version below 3.0 or instance platform not supported."
  resolution: "Update SSM Agent to 3.0+. Verify instance is Windows or Linux with DCV."

- symptoms: "Connection times out when starting remote desktop"
  diagnosis: "SSM connectivity issue — agent cannot reach SSM endpoints."
  resolution: "Verify VPC endpoints or NAT gateway. Check security group outbound rules for HTTPS (443)."

## Safety Ratings

```
safety_ratings:
  - "describe-instance-information: GREEN — read-only agent version check"
  - "get-connection-status: GREEN — read-only connectivity check"
  - "ssm:StartSession: YELLOW — establishes interactive session to instance"
  - "send-command to update agent: YELLOW — modifies agent software"
```

## Escalation Conditions

- Remote desktop fails with internal errors after all prerequisites are met
- Session Manager sessions start but remote desktop rendering fails
- Multiple instances in the same VPC experience simultaneous remote desktop failures
- Remote desktop works from one client but not another with identical IAM permissions

## Data Sensitivity

- `aws ssm describe-instance-information`: LOW — instance metadata only
- `aws ssm get-connection-status`: LOW — connectivity status only
- `aws ssm start-session`: HIGH — provides interactive access to the instance
- `aws cloudtrail lookup-events`: MEDIUM — reveals session history and user identities

## Prohibited Actions

- NEVER open inbound RDP (3389) or SSH (22) ports to troubleshoot Fleet Manager remote desktop
- NEVER disable Windows Firewall on the instance as a troubleshooting step
- NEVER share Session Manager session tokens or connection details
- NEVER use remote desktop to access production instances without change management approval

## Phase 3 — Rollback

1. If SSM Agent was updated and remote desktop broke, downgrade to previous version
2. If Session Manager preferences were modified, restore previous configuration
3. If IAM policy changes were made, revert to the previous policy version
4. Terminate any orphaned sessions: `aws ssm terminate-session --session-id <session-id>`
5. Document the issue and resolution for the operations team

## Output Format

```yaml
root_cause: "remote_desktop_failure — <specific_cause>"
evidence:
  - type: connection_error
    content: "<error details>"
severity: HIGH
mitigation:
  immediate: "Restore remote desktop connectivity"
  long_term: "Standardize SSM Agent version across fleet"
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
  - "NEVER suggest overly broad SSM document permissions"
  - "NEVER suggest disabling session logging"
  - "NEVER suggest sharing SSM documents publicly"
