---
title: "Z1 — General Troubleshooting"
description: "General ECS Anywhere troubleshooting for issues not covered by specific runbooks"
status: active
severity: MEDIUM
triggers: ["ECS Anywhere error", "external instance issue", "general ECS Anywhere problem", "unknown ECS Anywhere error"]
owner: devops-agent
objective: "Systematically diagnose and resolve general ECS Anywhere issues"
context: "Catch-all runbook for ECS Anywhere issues. Follows systematic elimination: registration → agent → task → networking → credentials."
---

## Phase 1 — Triage

MUST:
- Check cluster health: `aws ecs describe-clusters --clusters <cluster-name> --include ATTACHMENTS,SETTINGS,STATISTICS`
- Check external instance status: `aws ecs list-container-instances --cluster <cluster-name> --status ACTIVE`
- Check service health: `aws ecs describe-services --cluster <cluster-name> --services <service-name> --query 'services[*].{Status:status,Desired:desiredCount,Running:runningCount,Events:events[:5]}'`

SHOULD:
- Verify SSM Agent connectivity for all external instances
- Check ECS Agent connectivity for all instances
- Review CloudTrail for recent ECS API errors: `aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventSource,AttributeValue=ecs.amazonaws.com --max-results 20`

MAY:
- Check AWS Health Dashboard for ECS service issues
- Compare current state with a known-good configuration
- Review recent changes that may have caused the issue

## Phase 2 — Remediate

MUST:
- Address the specific error identified in triage
- Verify all agents (SSM, ECS) are connected and healthy
- Ensure task definitions use EXTERNAL launch type

SHOULD:
- Test the fix with a non-production service first
- Monitor the cluster after applying the fix
- Document the root cause and resolution

MAY:
- Set up comprehensive monitoring and alerting
- Implement automated health checks for external instances
- Create a runbook for the specific issue if recurring

## Common Issues

- symptoms: "Everything was working but suddenly stopped"
  diagnosis: "SSM Agent disconnection, credential expiry, or network change."
  resolution: "Check SSM Agent status. Verify network connectivity. Restart agents if needed."

- symptoms: "New tasks fail but existing tasks continue running"
  diagnosis: "Image pull failure, credential rotation issue, or resource exhaustion."
  resolution: "Check ECR connectivity. Verify credentials. Check available resources on instances."

## Safety Ratings

```
safety_ratings:
  - "ecs describe-clusters: GREEN — read-only"
  - "ecs list-container-instances: GREEN — read-only"
  - "ecs describe-services: GREEN — read-only"
  - "cloudtrail lookup-events: GREEN — read-only"
```

## Escalation Conditions

- Issue cannot be diagnosed with available logs and metrics
- Problem affects multiple services or instances simultaneously
- Issue appears to be an AWS service problem
- Resolution requires infrastructure changes beyond the team's scope

## Data Sensitivity

- `aws ecs describe-clusters`: LOW — cluster metadata
- `aws ecs describe-services`: MEDIUM — service configuration and events
- `aws cloudtrail lookup-events`: MEDIUM — API call history
- Instance logs: HIGH — may contain application data and credentials

## Prohibited Actions

- NEVER deregister external instances as a troubleshooting step without backup plan
- NEVER modify production services without a rollback plan
- NEVER assume the issue is with AWS — verify configuration first
- NEVER share instance logs externally without redacting sensitive data

## Phase 3 — Rollback

1. If any configuration changes were made during troubleshooting, revert them
2. If services were modified, restore previous configuration
3. If agents were restarted, verify they reconnected successfully
4. Verify the system returns to its pre-troubleshooting state
5. Document all changes made and their outcomes

## Output Format

```yaml
root_cause: "general — <specific_cause>"
evidence:
  - type: diagnostic_finding
    content: "<finding details>"
severity: MEDIUM
mitigation:
  immediate: "Apply targeted fix"
  long_term: "Implement monitoring and automated recovery"
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
