---
title: "E2 — Bootstrap Failures"
description: "Diagnose Bottlerocket bootstrap and first-boot configuration failures"
status: active
severity: CRITICAL
triggers: ["bootstrap failed", "instance not joining cluster", "first boot error", "user data not applied"]
owner: devops-agent
objective: "Resolve Bottlerocket bootstrap failures preventing instances from joining ECS/EKS clusters"
context: "Bottlerocket bootstrap reads TOML user data, applies settings, and starts the container agent. Failures here prevent cluster registration."
---

## Phase 1 — Triage

MUST:
- Check console output for bootstrap errors: `aws ec2 get-console-output --instance-id <instance-id> --latest`
- Verify instance state: `aws ec2 describe-instances --instance-ids <instance-id> --query 'Reservations[0].Instances[0].{State:State.Name,LaunchTime:LaunchTime,StateReason:StateReason}'`
- Check if instance reached SSM: `aws ssm describe-instance-information --filters "Key=InstanceIds,Values=<instance-id>"`

SHOULD:
- Verify user data in launch template: `aws ec2 describe-launch-template-versions --launch-template-id <id> --versions '$Latest' --query 'LaunchTemplateVersions[0].LaunchTemplateData.UserData'`
- Check IMDS accessibility (required for bootstrap): verify instance metadata service is enabled

MAY:
- Compare with a working instance's user data and configuration

## Phase 2 — Remediate

MUST:
- Fix TOML user data syntax — validate with a TOML parser before applying
- Ensure required settings are present: cluster-name, api-server endpoint, cluster-certificate (for EKS)
- Verify IMDS is enabled (IMDSv2 is supported): `aws ec2 modify-instance-metadata-options --instance-id <id> --http-endpoint enabled`

SHOULD:
- For EKS: verify the cluster API server endpoint is reachable from the instance subnet
- For ECS: verify the ECS cluster exists and the instance can reach ECS endpoints

MAY:
- Enable admin container in user data for debugging future bootstrap failures

## Safety Ratings

```
safety_ratings:
  - "ec2 get-console-output: GREEN — read-only boot log retrieval"
  - "ec2 describe-launch-template-versions: GREEN — read-only template inspection"
  - "ec2 modify-instance-metadata-options: YELLOW — changes IMDS configuration"
  - "Terminate and relaunch: RED — destroys instance and any local state"
```

## Escalation Conditions

- Bootstrap fails consistently across multiple instance launches with the same template
- Console output shows kernel panic or dm-verity errors
- Bootstrap succeeds but agent fails to register with the cluster
- IMDS is disabled by organization policy and cannot be re-enabled

## Data Sensitivity

- `aws ec2 get-console-output`: MEDIUM — may contain bootstrap configuration details
- Launch template user data: HIGH — contains cluster certificates and API endpoints
- `aws ssm describe-instance-information`: LOW — returns connectivity status
- IMDS configuration: LOW — returns metadata service settings

## Prohibited Actions

- NEVER decode and share user data containing cluster certificates in logs or tickets
- NEVER disable IMDSv2 enforcement to work around bootstrap issues
- NEVER modify a production launch template without testing in a staging environment first
- NEVER assume bootstrap succeeded without verifying cluster registration

## Phase 3 — Rollback

1. Revert launch template to previous working version: `aws ec2 modify-launch-template --launch-template-id <id> --default-version <previous>`
2. Terminate failed instances: `aws ec2 terminate-instances --instance-ids <id>`
3. If auto scaling group was updated, roll back: `aws autoscaling update-auto-scaling-group --auto-scaling-group-name <name> --launch-template Version=<previous>`
4. Verify new instances bootstrap successfully with the reverted template
5. Document the bootstrap failure cause for the post-mortem

## Common Issues

- symptoms: "Instance launches but never appears in ECS cluster or EKS node list"
  diagnosis: "User data TOML is malformed or missing required cluster settings."
  resolution: "Validate TOML syntax. Ensure [settings.kubernetes] or [settings.ecs] section has all required fields."

- symptoms: "Console output shows 'failed to apply settings from user data'"
  diagnosis: "User data contains invalid setting keys or values for the Bottlerocket variant."
  resolution: "Check setting paths match the variant (ECS vs EKS). Remove unsupported settings."

## Output Format

```yaml
root_cause: "bootstrap — <specific_cause>"
evidence:
  - type: bootstrap_error
    content: "<error details>"
severity: CRITICAL
mitigation:
  immediate: "Fix user data and relaunch"
  long_term: "Validate user data TOML in CI/CD before deployment"
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
