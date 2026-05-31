---
title: "C1 — Networking Issues"
description: "Diagnose networking issues on ECS Anywhere external instances"
status: active
severity: HIGH
triggers: ["network error", "cannot connect", "DNS failure", "port conflict", "connectivity issue"]
owner: devops-agent
objective: "Resolve networking issues on ECS Anywhere external instances"
context: "External instances use host or bridge network mode only. They need outbound HTTPS to AWS endpoints and may need connectivity to other services."
---

## Phase 1 — Triage

MUST:
- Check task network mode: `aws ecs describe-task-definition --task-definition <task-def> --query 'taskDefinition.networkMode'`
- Verify instance connectivity to AWS endpoints (via SSM): `aws ssm send-command --instance-ids <mi-id> --document-name AWS-RunShellScript --parameters commands=["curl -s https://ecs.<region>.amazonaws.com/"]`
- Check for port conflicts in host mode: `aws ssm send-command --instance-ids <mi-id> --document-name AWS-RunShellScript --parameters commands=["netstat -tlnp"]`

SHOULD:
- Verify DNS resolution works on the instance
- Check firewall rules allow required outbound connections
- Verify container port mappings in bridge mode

MAY:
- Test connectivity between containers on different instances
- Check if proxy configuration is needed and correct
- Verify MTU settings are not causing packet fragmentation

## Phase 2 — Remediate

MUST:
- Fix network mode to host or bridge (not awsvpc)
- Resolve port conflicts in host mode by changing container ports or switching to bridge mode
- Ensure outbound HTTPS (443) to ECS, SSM, ECR, S3, and CloudWatch Logs endpoints

SHOULD:
- Configure DNS on the instance to resolve AWS endpoints
- Set up firewall rules for required AWS IP ranges
- Use bridge mode with port mappings to avoid port conflicts

MAY:
- Set up VPN or Direct Connect for hybrid connectivity
- Implement service mesh for inter-service communication
- Configure Docker network settings for custom bridge networks

## Common Issues

- symptoms: "Container cannot reach external services or AWS APIs"
  diagnosis: "Firewall blocks outbound HTTPS or DNS resolution fails."
  resolution: "Allow outbound HTTPS (443) to required endpoints. Verify DNS configuration."

- symptoms: "Port conflict prevents task from starting in host mode"
  diagnosis: "Another container or host process is using the same port."
  resolution: "Switch to bridge mode with port mappings, or change the container port."

## Safety Ratings

```
safety_ratings:
  - "ecs describe-task-definition: GREEN — read-only"
  - "ssm send-command: RED — executes commands on remote instances"
  - "ecs register-task-definition: YELLOW — creates new task definition"
  - "Firewall rule changes: RED — affects network security"
```

## Escalation Conditions

- Network infrastructure changes require approval from network team
- VPN or Direct Connect setup needed for hybrid connectivity
- Firewall rules cannot be modified due to security policies
- DNS resolution issues require infrastructure-level changes

## Data Sensitivity

- `aws ecs describe-task-definition`: MEDIUM — reveals port mappings and network config
- `aws ssm send-command`: HIGH — executes commands revealing network state
- Network configuration: MEDIUM — reveals firewall rules and routing
- Port scan results: MEDIUM — reveals running services

## Prohibited Actions

- NEVER use awsvpc network mode on external instances — it is not supported
- NEVER open inbound ports from the internet without security review
- NEVER disable firewalls entirely to troubleshoot connectivity
- NEVER hardcode IP addresses for AWS endpoints — use DNS names

## Phase 3 — Rollback

1. If network mode was changed in task definition, revert to previous version
2. If firewall rules were modified, restore previous rules
3. If DNS configuration was changed, restore previous settings
4. Verify networking works correctly after rollback
5. Document the networking requirements and configuration

## Output Format

```yaml
root_cause: "networking — <specific_cause>"
evidence:
  - type: network_error
    content: "<connectivity test results>"
severity: HIGH
mitigation:
  immediate: "Fix network configuration"
  long_term: "Document network requirements and automate connectivity checks"
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
