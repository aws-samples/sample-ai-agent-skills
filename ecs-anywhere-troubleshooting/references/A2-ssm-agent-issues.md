---
title: "A2 — SSM Agent Issues"
description: "Diagnose SSM Agent connectivity and operation issues on external instances"
status: active
severity: CRITICAL
triggers: ["SSM agent disconnected", "instance offline", "agent not responding", "credential rotation failed"]
owner: devops-agent
objective: "Resolve SSM Agent issues on ECS Anywhere external instances"
context: "SSM Agent is mandatory for ECS Anywhere. It handles registration, credential delivery, and communication. If SSM Agent fails, the instance becomes disconnected."
---

## Phase 1 — Triage

MUST:
- Check SSM connection status: `aws ssm get-connection-status --target <mi-instance-id>`
- Check instance information: `aws ssm describe-instance-information --instance-information-filter-list key=InstanceIds,valueSet=<mi-instance-id>`
- Verify ECS Agent connectivity: `aws ecs describe-container-instances --cluster <cluster-name> --container-instances <instance-arn> --query 'containerInstances[*].agentConnected'`

SHOULD:
- Check SSM Agent version on the instance
- Verify outbound HTTPS connectivity to ssm.<region>.amazonaws.com
- Check if the instance's IAM role has ssm:UpdateInstanceInformation permission

MAY:
- Check SSM Agent logs on the instance: `/var/log/amazon/ssm/amazon-ssm-agent.log`
- Verify DNS resolution for AWS endpoints from the instance
- Check if a proxy is required and configured for SSM Agent

## Phase 2 — Remediate

MUST:
- Restart SSM Agent on the instance if stopped
- Fix network connectivity to SSM endpoints (HTTPS 443)
- Update SSM Agent if version is outdated

SHOULD:
- Configure proxy settings if the instance requires a proxy for internet access
- Verify DNS resolution works for all required AWS endpoints
- Check firewall rules allow outbound HTTPS to AWS IP ranges

MAY:
- Set up SSM Agent auto-update
- Implement monitoring for SSM Agent health
- Configure SSM Agent to use VPC endpoints if available via VPN/Direct Connect

## Common Issues

- symptoms: "Instance shows 'ConnectionLost' in SSM"
  diagnosis: "SSM Agent cannot reach SSM endpoints. Network issue or agent crashed."
  resolution: "Check network connectivity. Restart SSM Agent. Verify firewall allows HTTPS to ssm.<region>.amazonaws.com."

- symptoms: "ECS Agent shows agentConnected: false but SSM shows Online"
  diagnosis: "SSM Agent is working but ECS Agent has stopped or crashed."
  resolution: "Restart ECS Agent on the instance. Check /var/log/ecs/ecs-agent.log for errors."

## Safety Ratings

```
safety_ratings:
  - "ssm get-connection-status: GREEN — read-only"
  - "ssm describe-instance-information: GREEN — read-only"
  - "ecs describe-container-instances: GREEN — read-only"
  - "ssm send-command: RED — executes commands on remote instances"
```

## Escalation Conditions

- SSM Agent cannot be restarted remotely (instance is completely disconnected)
- Network infrastructure changes required to allow AWS endpoint access
- SSM Agent crashes repeatedly after restart
- Credential rotation failure causes cascading task failures

## Data Sensitivity

- `aws ssm get-connection-status`: LOW — connection status only
- `aws ssm describe-instance-information`: MEDIUM — reveals IP addresses and platform details
- SSM Agent logs: HIGH — may contain credential rotation details and error context
- Network configuration: MEDIUM — reveals firewall rules and proxy settings

## Prohibited Actions

- NEVER uninstall SSM Agent on an active ECS Anywhere instance
- NEVER manually set AWS credentials on the instance — SSM handles credential delivery
- NEVER disable SSM Agent auto-update without a maintenance plan
- NEVER expose SSM endpoints through public-facing firewall rules

## Phase 3 — Rollback

1. If SSM Agent was updated and broke, downgrade to previous version
2. If network changes caused disconnection, revert firewall/proxy changes
3. If the instance cannot be recovered, deregister and re-register it
4. Verify SSM and ECS Agent connectivity after rollback
5. Document the SSM Agent issue and prevention measures

## Output Format

```yaml
root_cause: "ssm_agent — <specific_cause>"
evidence:
  - type: agent_status
    content: "<SSM and ECS agent status details>"
severity: CRITICAL
mitigation:
  immediate: "Restore SSM Agent connectivity"
  long_term: "Implement SSM Agent monitoring and auto-recovery"
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
  - command: "describe-tasks"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"
  - command: "describe-services"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"
  - command: "execute-command"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"

## Prohibited Actions

prohibited_actions:
  - "NEVER suggest running containers as root to fix permission issues"
  - "NEVER suggest disabling task networking"
  - "NEVER remove IAM task role to troubleshoot"
