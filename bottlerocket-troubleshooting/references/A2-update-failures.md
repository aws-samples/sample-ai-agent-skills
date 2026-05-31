---
title: "A2 — Update Failures"
description: "Diagnose Bottlerocket OS update and rollback failures"
status: active
severity: HIGH
triggers: ["update failed", "apiclient update error", "rollback failed", "stuck on old version", "reboot loop"]
owner: devops-agent
objective: "Resolve Bottlerocket OS update failures and ensure instances run the target version"
context: "Bottlerocket uses A/B partition updates. The inactive partition is updated, then the system reboots into it. Rollback swaps back to the previous partition."
---

## Phase 1 — Triage

MUST:
- Check current OS version: `aws ssm start-session --target <instance-id> --document-name AWS-StartInteractiveCommand --parameters command="apiclient get os.version-lock"`
- Check update status: `aws ssm start-session --target <instance-id> --document-name AWS-StartInteractiveCommand --parameters command="apiclient update check"`
- Verify instance connectivity: `aws ssm describe-instance-information --filters "Key=InstanceIds,Values=<instance-id>" --query 'InstanceInformationList[0].{PingStatus:PingStatus,LastPing:LastPingDateTime}'`

SHOULD:
- Check console output for boot errors: `aws ec2 get-console-output --instance-id <instance-id> --latest`
- Verify the update repository is reachable (requires internet or VPC endpoint)

MAY:
- Collect full logs: `aws ssm start-session --target <instance-id> --document-name AWS-StartInteractiveCommand --parameters command="logdog"`

## Phase 2 — Remediate

MUST:
- If update check fails, verify network connectivity to update repository
- Apply update with explicit steps: `apiclient update apply` then `apiclient reboot`
- If stuck after reboot, initiate rollback: `apiclient update revert` then `apiclient reboot`

SHOULD:
- For managed node groups, use EKS node group update: `aws eks update-nodegroup-version --cluster-name <cluster> --nodegroup-name <nodegroup>`
- Verify the target version is available before applying

MAY:
- Use SSM Run Command for fleet-wide updates: `aws ssm send-command --document-name AWS-RunShellScript --targets "Key=tag:OS,Values=Bottlerocket" --parameters commands=["apiclient update check"]`

## Safety Ratings

```
safety_ratings:
  - "apiclient update check: GREEN — read-only check for available updates"
  - "apiclient update apply: YELLOW — writes to inactive partition, requires reboot"
  - "apiclient reboot: RED — causes instance reboot and temporary workload disruption"
  - "apiclient update revert: RED — rolls back to previous OS version, requires reboot"
```

## Escalation Conditions

- Instance enters reboot loop after update apply
- Rollback fails and instance is unreachable via SSM
- Managed node group update stuck in degraded state for over 30 minutes
- Update causes ECS agent or kubelet to fail on the new version

## Data Sensitivity

- `apiclient get os.version-lock`: LOW — returns version information only
- `apiclient update check`: LOW — checks for available updates
- `aws ec2 get-console-output`: MEDIUM — may contain boot-time secrets or configuration
- `logdog` output: MEDIUM — contains system logs that may include IP addresses and hostnames

## Prohibited Actions

- NEVER apply updates to all instances simultaneously — use rolling updates
- NEVER skip the reboot step after apiclient update apply
- NEVER force-terminate an instance during an active update process
- NEVER disable the update lock without understanding the version pinning reason

## Phase 3 — Rollback

1. Revert the update: `apiclient update revert` then `apiclient reboot`
2. If instance is unreachable, stop and start the instance: `aws ec2 stop-instances --instance-ids <id>` then `aws ec2 start-instances --instance-ids <id>`
3. If stop/start fails, terminate and let auto scaling replace: `aws ec2 terminate-instances --instance-ids <id>`
4. For managed node groups, roll back the node group version: `aws eks update-nodegroup-version --cluster-name <cluster> --nodegroup-name <nodegroup> --release-version <previous>`
5. Pin the version to prevent re-update: `apiclient set os.version-lock=<known-good-version>`

## Common Issues

- symptoms: "apiclient update apply succeeds but instance boots into old version"
  diagnosis: "Reboot was not performed after update apply, or the boot partition selection failed."
  resolution: "Run apiclient reboot after update apply. Check console output for boot partition errors."

- symptoms: "Update check returns no available updates but instance is outdated"
  diagnosis: "Version lock is set or the update repository is unreachable."
  resolution: "Check apiclient get os.version-lock and verify network access to updates.bottlerocket.aws."

## Output Format

```yaml
root_cause: "update_failure — <specific_cause>"
evidence:
  - type: update_error
    content: "<error details>"
severity: HIGH
mitigation:
  immediate: "Rollback or fix update process"
  long_term: "Implement rolling update strategy with health checks"
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
