---
title: "D1 — Linux Subscription Discovery"
description: "Diagnose Linux subscription discovery issues"
status: active
severity: MEDIUM
triggers: ["linux subscription", "RHEL discovery", "SUSE discovery", "linux license tracking"]
owner: devops-agent
objective: "Resolve Linux subscription discovery and tracking issues"
context: "Linux subscription discovery tracks RHEL, SUSE, and Ubuntu Pro subscriptions via SSM inventory. Requires SSM Agent on managed instances."
---

## Phase 1 — Triage

MUST:
- Check Linux subscription settings: `aws license-manager-linux-subscriptions get-service-settings`
- List discovered subscriptions: `aws license-manager-linux-subscriptions list-linux-subscriptions`
- Verify SSM Agent status on instances: `aws ssm describe-instance-information --query 'InstanceInformationList[*].{Id:InstanceId,PingStatus:PingStatus,Platform:PlatformName}'`

SHOULD:
- Check SSM inventory for software data: `aws ssm list-inventory-entries --instance-id <id> --type-name AWS:Application`
- Verify IAM role for SSM on instances

MAY:
- Check for instances not managed by SSM
- Review discovery schedule and last run time

## Phase 2 — Remediate

MUST:
- Enable Linux subscription discovery: `aws license-manager-linux-subscriptions update-service-settings --linux-subscriptions-discovery Enabled --linux-subscriptions-discovery-settings '{"SourceRegions":["us-east-1","us-west-2"],"OrganizationIntegration":"Enabled"}'`
- Ensure SSM Agent is installed and running on all Linux instances
- Verify IAM instance profile includes AmazonSSMManagedInstanceCore policy

SHOULD:
- Configure discovery for all regions with Linux instances
- Enable organization integration for cross-account discovery

MAY:
- Set up CloudWatch dashboards for subscription tracking
- Automate SSM Agent installation via user data or State Manager

## Safety Ratings

```
safety_ratings:
  - "license-manager-linux-subscriptions get-service-settings: GREEN — read-only check"
  - "license-manager-linux-subscriptions list-linux-subscriptions: GREEN — read-only listing"
  - "ssm describe-instance-information: GREEN — read-only instance check"
  - "license-manager-linux-subscriptions update-service-settings: YELLOW — enables discovery service"
```

## Escalation Conditions

- SSM Agent cannot be installed on critical instances
- Discovery misidentifies subscription types
- Organization integration fails across member accounts
- Large number of unmanaged instances discovered

## Data Sensitivity

- `aws ssm describe-instance-information`: MEDIUM — reveals instance details and OS
- Linux subscription data: MEDIUM — reveals software licensing posture
- Instance inventory: MEDIUM — reveals installed software
- Organization discovery: MEDIUM — reveals cross-account infrastructure

## Prohibited Actions

- NEVER disable Linux subscription discovery during an audit
- NEVER ignore discovered subscription discrepancies
- NEVER install SSM Agent on instances without proper authorization
- NEVER share subscription discovery data outside compliance team

## Phase 3 — Rollback

1. Disable discovery if enabled in error: update service settings with discovery Disabled
2. Remove SSM Agent from instances if installed incorrectly
3. Revert organization integration settings if misconfigured
4. Verify no compliance data was lost during changes
5. Document discovery configuration changes

## Common Issues

- symptoms: "Linux subscriptions list is empty despite running RHEL instances"
  diagnosis: "SSM Agent not installed or not reporting inventory on the instances."
  resolution: "Install SSM Agent, attach AmazonSSMManagedInstanceCore IAM policy, and wait for inventory collection."

## Output Format

```yaml
root_cause: "linux_subscription_discovery — <specific_cause>"
evidence:
  - type: discovery_error
    content: "<error details>"
severity: MEDIUM
mitigation:
  immediate: "Fix discovery configuration"
  long_term: "Automate SSM Agent deployment and monitoring"
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
  - "NEVER suggest overly broad IAM permissions as a fix"
  - "NEVER suggest disabling security controls for troubleshooting"
  - "NEVER suggest deleting resources without confirmation"
