---
title: "B1 — Session Manager Failures"
description: "Diagnose Session Manager session failures and connectivity issues"
status: active
severity: HIGH
triggers:
  - "session failed"
  - "cannot start session"
  - "Session Manager not working"
  - "TargetNotConnected"
owner: devops-agent
objective: "Identify why Session Manager sessions are failing and restore access"
context: "Session Manager requires SSM Agent, ssmmessages VPC endpoint or internet access, IAM permissions for the caller, and the Session Manager plugin for CLI usage. No inbound ports are needed."
---

## Phase 1 — Triage

MUST:
- Check instance is managed: `aws ssm get-connection-status --target <instance-id>`
- Verify instance has ssmmessages access: `aws ec2 describe-vpc-endpoints --filters Name=vpc-id,Values=<vpc-id> Name=service-name,Values=com.amazonaws.<region>.ssmmessages`
- Check caller IAM permissions for ssm:StartSession
- List active sessions: `aws ssm describe-sessions --state Active`

SHOULD:
- Check Session Manager preferences: `aws ssm get-document --name SSM-SessionManagerRunShell --query 'Content' --output text`
- Verify KMS key access if session encryption is configured
- Check CloudTrail for session events: `aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventName,AttributeValue=StartSession --max-results 10`

MAY:
- Check Session Manager plugin version on local machine
- Verify session logging configuration (S3/CloudWatch)

## Phase 2 — Remediate

MUST:
- Ensure ssmmessages VPC endpoint exists or outbound internet access is available
- Grant ssm:StartSession permission to the caller
- Verify SSM Agent is running and connected

SHOULD:
- Install or update Session Manager plugin locally
- Configure session logging for audit compliance
- Set session idle timeout and max duration

MAY:
- Configure KMS encryption for session data
- Set up IAM policies to restrict session access by tag

## Common Issues

- symptoms: "TargetNotConnected error"
  diagnosis: "Instance SSM Agent is not connected. Missing ssmmessages endpoint."
  resolution: "Create ssmmessages VPC endpoint or ensure outbound HTTPS access."

- symptoms: "AccessDeniedException on StartSession"
  diagnosis: "Caller lacks ssm:StartSession permission."
  resolution: "Add ssm:StartSession to the caller's IAM policy."

- symptoms: "Session starts but immediately disconnects"
  diagnosis: "Session Manager plugin outdated or network instability."
  resolution: "Update Session Manager plugin. Check network stability."

## Output Format

```yaml
root_cause: "session_failure — <specific_cause>"
evidence:
  - type: connection_status
    content: "<get-connection-status output>"
  - type: vpc_endpoints
    content: "<ssmmessages endpoint status>"
severity: HIGH
mitigation:
  immediate: "Fix ssmmessages endpoint or IAM permissions"
  long_term: "Configure session logging and monitoring"
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
