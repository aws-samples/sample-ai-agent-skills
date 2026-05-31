---
title: "Z1 — General Fleet Manager Troubleshooting"
description: "Catch-all runbook for Fleet Manager issues not covered by specific runbooks"
status: active
severity: MEDIUM
triggers: ["Fleet Manager error", "SSM issue", "general Fleet Manager problem"]
owner: devops-agent
objective: "Provide systematic troubleshooting for unclassified Fleet Manager issues"
context: "This runbook covers general Fleet Manager issues that don't fit specific categories. Start here when the issue is unclear, then route to a specific runbook once the domain is identified."
---

## Phase 1 — Triage

MUST:
- List all managed nodes: `aws ssm describe-instance-information --query 'InstanceInformationList[*].{Id:InstanceId,Ping:PingStatus,Agent:AgentVersion,Platform:PlatformType,IP:IPAddress}'`
- Check SSM service health in the region: `aws health describe-events --filter eventTypeCategories=issue,services=SSM`
- Verify IAM caller identity: `aws sts get-caller-identity`

SHOULD:
- Check recent SSM API errors in CloudTrail: `aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventSource,AttributeValue=ssm.amazonaws.com --max-results 10`
- Verify the AWS console region matches the expected region

MAY:
- Check AWS Service Health Dashboard for SSM outages
- Review recent infrastructure changes that may affect SSM

## Phase 2 — Remediate

MUST:
- Identify the specific issue domain (registration, connectivity, features, permissions)
- Route to the appropriate specific runbook (A1-F1) once domain is identified
- Verify basic prerequisites: SSM Agent installed, IAM profile attached, network connectivity

SHOULD:
- Check if the issue affects a single instance or multiple instances
- Determine if the issue started after a recent change

MAY:
- Collect SSM Agent logs from affected instances for detailed analysis
- Open an AWS Support case if the issue appears to be a service-side problem

## Common Issues

- symptoms: "Fleet Manager console shows errors or is slow to load"
  diagnosis: "Browser cache issue, console bug, or SSM service degradation."
  resolution: "Clear browser cache. Check AWS Service Health Dashboard. Try a different browser."

- symptoms: "Some Fleet Manager features work but others don't"
  diagnosis: "SSM Agent version too old for newer features or missing specific IAM permissions."
  resolution: "Update SSM Agent to latest. Review IAM permissions for the specific feature."

## Safety Ratings

```
safety_ratings:
  - "describe-instance-information: GREEN — read-only node listing"
  - "health describe-events: GREEN — read-only service health check"
  - "sts get-caller-identity: GREEN — read-only identity check"
  - "cloudtrail lookup-events: GREEN — read-only audit log query"
```

## Escalation Conditions

- Multiple unrelated Fleet Manager features fail simultaneously
- Issue affects all instances in the account or region
- SSM service health shows active incidents
- Issue persists after all standard troubleshooting steps are exhausted

## Data Sensitivity

- `aws ssm describe-instance-information`: LOW — instance metadata only
- `aws cloudtrail lookup-events`: MEDIUM — reveals API activity and principal identities
- `aws sts get-caller-identity`: LOW — reveals current IAM identity
- SSM Agent logs: MEDIUM — may contain endpoint URLs, error details, and instance metadata

## Prohibited Actions

- NEVER make broad changes (restart all agents, modify all security groups) without isolating the issue
- NEVER assume a service outage without checking AWS Health Dashboard
- NEVER skip the triage phase and jump directly to remediation
- NEVER ignore the possibility that the issue is in a different region than expected

## Phase 3 — Rollback

1. If any changes were made during troubleshooting, document them
2. Revert changes in reverse order if they did not resolve the issue
3. Verify the environment is in the same state as before troubleshooting
4. If the issue is unresolved, collect all diagnostic data for escalation
5. Document the troubleshooting steps taken and results for the support case

## Output Format

```yaml
root_cause: "general_fleet_manager — <specific_cause>"
evidence:
  - type: general_error
    content: "<error details>"
severity: MEDIUM
mitigation:
  immediate: "Route to specific runbook or apply general fix"
  long_term: "Implement monitoring and alerting for Fleet Manager health"
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
