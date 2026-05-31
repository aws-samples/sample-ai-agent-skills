---
title: "Z1 — General Troubleshooting"
description: "Catch-all runbook for Bottlerocket issues not covered by specific runbooks"
status: active
severity: MEDIUM
triggers: ["bottlerocket issue", "bottlerocket error", "bottlerocket problem", "general bottlerocket"]
owner: devops-agent
objective: "Provide a systematic approach for diagnosing unclassified Bottlerocket issues"
context: "Use this runbook when the issue does not clearly match a specific category. Follow the triage steps to classify and route to the appropriate runbook."
---

## Phase 1 — Triage

MUST:
- Identify the Bottlerocket variant and version: `aws ssm start-session --target <instance-id> --document-name AWS-StartInteractiveCommand --parameters command="apiclient get os"`
- Check overall instance health: `aws ec2 describe-instance-status --instance-ids <instance-id>`
- Verify SSM connectivity: `aws ssm describe-instance-information --filters "Key=InstanceIds,Values=<instance-id>"`

SHOULD:
- Collect comprehensive logs: `logdog`
- Check console output for errors: `aws ec2 get-console-output --instance-id <instance-id> --latest`
- Review recent CloudTrail events: `aws cloudtrail lookup-events --lookup-attributes AttributeKey=ResourceName,AttributeValue=<instance-id> --max-results 20`

MAY:
- Compare instance configuration with a known-good instance
- Check AWS Health Dashboard for regional service issues

## Phase 2 — Remediate

MUST:
- Route to the appropriate specific runbook based on triage findings
- If AMI/update issue → A1 or A2
- If ECS agent issue → B1; if kubelet issue → B2
- If access issue → C1 or C2
- If networking/storage → D1 or D2
- If settings/bootstrap → E1 or E2
- If security → F1

SHOULD:
- Document the symptoms and triage findings before routing
- Check if the issue is a known Bottlerocket bug in GitHub issues

MAY:
- Open a Bottlerocket GitHub issue if the problem appears to be an OS bug

## Safety Ratings

```
safety_ratings:
  - "apiclient get os: GREEN — read-only OS version query"
  - "ec2 describe-instance-status: GREEN — read-only health check"
  - "logdog: GREEN — read-only log collection"
  - "ec2 get-console-output: GREEN — read-only boot log retrieval"
```

## Escalation Conditions

- Issue cannot be classified into any specific runbook category
- Multiple symptoms spanning different categories simultaneously
- Suspected Bottlerocket OS bug not documented in known issues
- Instance is completely unreachable via SSM and console output shows no useful information

## Data Sensitivity

- `apiclient get os`: LOW — returns OS version information
- `logdog` output: MEDIUM — comprehensive system logs may contain hostnames and IPs
- `aws cloudtrail lookup-events`: MEDIUM — reveals API actions and IAM principals
- `aws ec2 get-console-output`: MEDIUM — may contain boot-time configuration

## Prohibited Actions

- NEVER terminate an instance as a first troubleshooting step — always collect logs first
- NEVER apply multiple changes simultaneously — change one thing at a time
- NEVER skip log collection before making changes
- NEVER assume the issue is Bottlerocket-specific without checking AWS service health

## Phase 3 — Rollback

1. Revert any changes made during investigation
2. If instance was rebooted, verify workloads recovered
3. If settings were modified, restore original values via apiclient
4. Document all changes made and their outcomes
5. If instance is unrecoverable, terminate and let auto scaling replace

## Common Issues

- symptoms: "Instance is running but not responding to any management commands"
  diagnosis: "Control container may be down, or instance may be in a kernel panic state."
  resolution: "Check console output. Try stop/start. If unrecoverable, terminate and relaunch."

- symptoms: "Bottlerocket instance works initially but fails after some time"
  diagnosis: "Possible disk space exhaustion, memory pressure, or container runtime issues."
  resolution: "Check disk usage (D2), memory metrics in CloudWatch, and container runtime logs."

## Output Format

```yaml
root_cause: "general — <specific_cause>"
evidence:
  - type: diagnostic_finding
    content: "<finding details>"
severity: MEDIUM
mitigation:
  immediate: "Route to specific runbook"
  long_term: "Implement monitoring for early detection"
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
