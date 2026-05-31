---
title: "D2 — Credential Management"
description: "Diagnose credential delivery and rotation issues on external instances"
status: active
severity: CRITICAL
triggers: ["credentials expired", "credential rotation failed", "cannot access AWS services", "token expired"]
owner: devops-agent
objective: "Resolve credential management issues on ECS Anywhere external instances"
context: "ECS Anywhere delivers credentials via SSM Agent. Credentials are temporary and automatically rotated. SSM Agent must be running for rotation."
---

## Phase 1 — Triage

MUST:
- Check SSM Agent status: `aws ssm get-connection-status --target <mi-instance-id>`
- Check ECS Agent connectivity: `aws ecs describe-container-instances --cluster <cluster-name> --container-instances <instance-arn> --query 'containerInstances[*].{Agent:agentConnected,Status:status}'`
- Verify tasks can access AWS services: check task stopped reasons for credential errors

SHOULD:
- Check ECS Agent logs for credential rotation errors
- Verify the instance's SSM role has sts:AssumeRole permission
- Check if credential endpoint is accessible from containers

MAY:
- Check the ECS Agent credential proxy is running on the instance
- Verify the task role trust policy allows ecs-tasks.amazonaws.com
- Test credential delivery by running a simple task that calls STS GetCallerIdentity

## Phase 2 — Remediate

MUST:
- Restart SSM Agent if credential rotation has stopped
- Restart ECS Agent if credential proxy is not functioning
- Ensure SSM Agent can reach STS and SSM endpoints

SHOULD:
- Verify the ECS Agent credential proxy endpoint (169.254.170.2) is accessible from containers
- Check that no iptables rules block the credential proxy
- Ensure task role trust policy is correct

MAY:
- Implement credential monitoring to detect rotation failures early
- Set up alerts for SSM Agent disconnections

## Common Issues

- symptoms: "Container gets 'ExpiredTokenException' when calling AWS APIs"
  diagnosis: "Credential rotation failed — SSM Agent is disconnected or ECS Agent credential proxy is down."
  resolution: "Restart SSM Agent and ECS Agent. Verify connectivity to AWS endpoints."

- symptoms: "Task role credentials not available in container"
  diagnosis: "ECS Agent credential proxy (169.254.170.2:51679) is not accessible from the container."
  resolution: "Verify ECS Agent is running. Check iptables rules. Ensure bridge network allows access to 169.254.170.2."

## Safety Ratings

```
safety_ratings:
  - "ssm get-connection-status: GREEN — read-only"
  - "ecs describe-container-instances: GREEN — read-only"
  - "ssm send-command: RED — executes commands on remote instances"
  - "Restarting SSM/ECS Agent: YELLOW — briefly interrupts credential delivery"
```

## Escalation Conditions

- Credential rotation failure causes production task failures
- SSM Agent cannot be restarted remotely (instance fully disconnected)
- Credential proxy endpoint blocked by network configuration that cannot be changed
- Multiple instances experience simultaneous credential failures

## Data Sensitivity

- `aws ssm get-connection-status`: LOW — connection status only
- `aws ecs describe-container-instances`: MEDIUM — reveals instance status
- Credential proxy logs: HIGH — may contain credential metadata
- STS GetCallerIdentity output: MEDIUM — reveals assumed role ARN

## Prohibited Actions

- NEVER manually set AWS credentials (access keys) on external instances
- NEVER bypass the ECS Agent credential proxy with hardcoded credentials
- NEVER expose the credential proxy endpoint (169.254.170.2) outside the instance
- NEVER disable credential rotation — it is a security requirement

## Phase 3 — Rollback

1. If SSM Agent was updated and broke credential delivery, downgrade
2. If iptables rules were changed, restore previous rules
3. If ECS Agent was updated, downgrade to previous version
4. Verify credential delivery works after rollback
5. Document the credential management architecture

## Output Format

```yaml
root_cause: "credential_management — <specific_cause>"
evidence:
  - type: credential_error
    content: "<credential status and agent details>"
severity: CRITICAL
mitigation:
  immediate: "Restore SSM/ECS Agent and credential delivery"
  long_term: "Implement credential monitoring and automated recovery"
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
