---
title: "B2 — Port Forwarding"
description: "Diagnose Session Manager port forwarding issues"
status: active
severity: MEDIUM
triggers:
  - "port forwarding not working"
  - "StartPortForwardingSession"
  - "tunnel failed"
  - "cannot connect to remote port"
owner: devops-agent
objective: "Identify why port forwarding through Session Manager is failing"
context: "Port forwarding uses Session Manager to create a tunnel from a local port to a remote port on the instance. It requires the Session Manager plugin, ssmmessages endpoint access, and appropriate IAM permissions including ssm:StartSession with the AWS-StartPortForwardingSession document."
---

## Phase 1 — Triage

MUST:
- Verify instance is managed and connected: `aws ssm get-connection-status --target <instance-id>`
- Check caller has permission for port forwarding document: verify IAM policy allows `ssm:StartSession` with condition on `ssm:SessionDocumentAccessCheck` for `AWS-StartPortForwardingSession`
- Verify Session Manager plugin is installed locally
- Check if the target port is listening on the instance: `aws ssm send-command --instance-ids <instance-id> --document-name AWS-RunShellScript --parameters commands=["ss -tlnp | grep <port>"]`

SHOULD:
- Check for active sessions that may conflict: `aws ssm describe-sessions --state Active --filters key=Target,value=<instance-id>`
- Verify security group allows the target service (internal to instance, not inbound from internet)
- Check Session Manager preferences for any restrictions

MAY:
- Test basic session connectivity first before port forwarding
- Check local firewall rules on the client machine

## Phase 2 — Remediate

MUST:
- Ensure Session Manager plugin is installed and up to date
- Grant IAM permission for AWS-StartPortForwardingSession document
- Verify target service is running and listening on the specified port

SHOULD:
- Use the correct port forwarding command syntax
- Test with a simple session first to isolate the issue

MAY:
- Configure port forwarding to remote hosts (AWS-StartPortForwardingSessionToRemoteHost)
- Set up IAM policies to restrict which ports can be forwarded

## Common Issues

- symptoms: "Plugin not found error"
  diagnosis: "Session Manager plugin not installed on local machine."
  resolution: "Install the Session Manager plugin from AWS documentation."

- symptoms: "Connection refused on local port"
  diagnosis: "Target service not running on the specified port."
  resolution: "Verify the service is running and listening on the target port."

- symptoms: "AccessDeniedException for port forwarding"
  diagnosis: "IAM policy does not allow the port forwarding session document."
  resolution: "Add ssm:StartSession permission with AWS-StartPortForwardingSession document condition."

## Output Format

```yaml
root_cause: "port_forwarding — <specific_cause>"
evidence:
  - type: connection_status
    content: "<instance connection status>"
  - type: target_port
    content: "<port listening status>"
severity: MEDIUM
mitigation:
  immediate: "Fix plugin installation or IAM permissions"
  long_term: "Document port forwarding procedures and IAM policies"
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
