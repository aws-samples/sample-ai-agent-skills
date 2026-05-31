---
title: "A2 — Node Not Showing in Fleet Manager"
description: "Diagnose why a managed node does not appear in Fleet Manager console"
status: active
severity: HIGH
triggers: ["node not showing", "instance missing from Fleet Manager", "cannot see instance"]
owner: devops-agent
objective: "Identify and resolve why a registered or expected node is not visible in Fleet Manager"
context: "Fleet Manager displays nodes that have successfully registered with SSM in the current region. Visibility issues can stem from region mismatch, agent status, or IAM permissions."
---

## Phase 1 — Triage

MUST:
- Check if instance exists and is running: `aws ec2 describe-instance-status --instance-ids <instance-id>`
- Query SSM for the instance: `aws ssm describe-instance-information --filters Key=InstanceIds,Values=<instance-id>`
- Verify you are in the correct region: `aws ssm describe-instance-information --region <expected-region> --filters Key=InstanceIds,Values=<instance-id>`

SHOULD:
- Check SSM Agent ping status: `aws ssm get-connection-status --target <instance-id>`
- Verify IAM instance profile is attached and has correct policies

MAY:
- Check if the instance was recently terminated and relaunched with a new ID
- Review SSM Agent logs for registration errors

## Phase 2 — Remediate

MUST:
- Ensure SSM Agent is running on the instance
- Verify IAM instance profile with `AmazonSSMManagedInstanceCore` is attached
- Confirm network path to SSM endpoints (VPC endpoints or NAT gateway)

SHOULD:
- Restart SSM Agent if ping status is stale: `aws ssm send-command --instance-ids <instance-id> --document-name AWS-RunShellScript --parameters commands=["sudo systemctl restart amazon-ssm-agent"]`
- Update SSM Agent if version is outdated

MAY:
- Enable SSM Agent verbose logging for deeper diagnostics

## Common Issues

- symptoms: "Instance is running but not in Fleet Manager"
  diagnosis: "Wrong region selected in console or SSM Agent not installed."
  resolution: "Switch to the correct region. Verify SSM Agent is installed and running."

- symptoms: "Instance was visible yesterday but disappeared"
  diagnosis: "SSM Agent stopped or instance profile was removed."
  resolution: "Check agent status and instance profile. Restart agent if needed."

## Safety Ratings

```
safety_ratings:
  - "describe-instance-information: GREEN — read-only query"
  - "get-connection-status: GREEN — read-only connectivity check"
  - "send-command restart agent: YELLOW — restarts a service on the instance"
  - "describe-instance-status: GREEN — read-only instance status"
```

## Escalation Conditions

- Node was previously visible and disappeared without any changes to IAM or network
- Multiple nodes in the same VPC simultaneously stopped appearing
- SSM Agent logs show authentication errors despite correct IAM configuration
- Instance is in a GovCloud or China region with different endpoint requirements

## Data Sensitivity

- `aws ssm describe-instance-information`: LOW — instance metadata only
- `aws ssm get-connection-status`: LOW — connectivity status only
- `aws ec2 describe-instance-status`: LOW — instance health status
- `aws ssm send-command`: MEDIUM — executes commands on the instance

## Prohibited Actions

- NEVER terminate an instance to force re-registration
- NEVER modify SSM Agent configuration files directly without backing them up
- NEVER assume a node is decommissioned just because it is not showing
- NEVER change the instance region configuration in SSM Agent without understanding the impact

## Phase 3 — Rollback

1. If SSM Agent was restarted and caused issues, check logs: `/var/log/amazon/ssm/amazon-ssm-agent.log`
2. If agent configuration was modified, restore from backup
3. If VPC endpoint changes were made, revert security group and route table changes
4. Verify the node reappears in Fleet Manager after fixes
5. Document the root cause and resolution for future reference

## Output Format

```yaml
root_cause: "node_not_showing — <specific_cause>"
evidence:
  - type: visibility_error
    content: "<error details>"
severity: HIGH
mitigation:
  immediate: "Restore node visibility"
  long_term: "Monitor SSM Agent health with CloudWatch alarms"
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
