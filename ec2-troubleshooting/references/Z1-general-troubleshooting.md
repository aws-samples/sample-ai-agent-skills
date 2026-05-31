---
title: "Z1 — General EC2 Troubleshooting (Catch-All)"
description: "Fallback SOP for EC2 issues that do not match any specific runbook"
status: active
severity: MEDIUM
triggers:
  - ".*"
owner: devops-agent
objective: "Systematically investigate an unknown EC2 issue, classify the failure domain, and match to an existing SOP or escalate"
context: "This SOP is invoked when symptoms don't match any of the specific runbooks. It provides a broad, methodical investigation that narrows the failure domain step by step."
---

## Phase 1 — Triage

MUST:
- Get instance details: `aws ec2 describe-instances --instance-ids <id>`
- Check instance state (running, stopped, terminated, pending, stopping)
- Check status checks: `aws ec2 describe-instance-status --instance-ids <id>`
- Get system log: `aws ec2 get-console-output --instance-id <id>`
- Get screenshot: `aws ec2 get-console-screenshot --instance-id <id>`
- Check for scheduled events in instance status

SHOULD:
- Check CloudWatch metrics: CPUUtilization, StatusCheckFailed, NetworkIn/Out
- Check CloudTrail for recent API calls affecting the instance
- If SSM available: run basic diagnostics (uptime, free -m, df -h, dmesg | tail)

## Phase 2 — Classify

Based on triage results, classify into a failure domain:
- Instance not running → Check state transitions (I1-I3)
- Status check failing → System (B1) or Instance (B2) status check
- Cannot connect → Connectivity (C1-C5)
- Performance degraded → Performance (D1-D5)
- Storage issues → Storage (E1-E4)
- Boot failure → Boot/Init (F1-F4)
- Network issues → Networking (G1-G4)
- Permission errors → IAM/Security (H1-H3)
- Instance type issues → Instance Types (J1-J3)
- Maintenance → Maintenance (K1-K3)

If classified: switch to the specific SOP immediately.
If unclassified: continue to Phase 3.

## Phase 3 — Deep Investigation

MUST:
- Check all CloudWatch metrics for anomalies
- Review system log thoroughly for any error patterns
- Check VPC configuration (SGs, NACLs, routes)
- Check EBS volume states
- Review recent CloudTrail events for changes

SHOULD:
- Use VPC Reachability Analyzer if connectivity is involved
- Check AWS Health Dashboard for service events
- Compare with a known-good instance of the same type

## Phase 4 — Report

MUST:
- State the investigation path taken
- State root cause if identified, or "unclassified" with best hypothesis
- List all evidence collected
- Recommend next steps

## Safety Ratings
Rate each remediation action in Phase 2:
- GREEN: read-only diagnostic commands (describe-*, get-*, list-*)
- YELLOW: state-changing but recoverable (modify security group, restart instance, modify parameter)
- RED: destructive or irreversible (terminate instance, delete volume, modify key infrastructure)

Format as:
```
safety_ratings:
  - "describe-instances, describe-instance-status: GREEN — read-only"
  - "get-console-output, get-console-screenshot: GREEN — read-only diagnostics"
  - "CloudWatch metrics review: GREEN — read-only"
  - "CloudTrail event review: GREEN — read-only"
  - "SSM basic diagnostics (uptime, free, df, dmesg): GREEN — read-only"
  - "VPC Reachability Analyzer: GREEN — read-only path analysis"
  - "Any remediation action: varies by specific SOP — classify first before acting"
```

## Escalation Conditions
- Issue cannot be classified into any specific failure domain after thorough investigation
- Multiple failure domains are involved simultaneously
- Instance is completely unresponsive with no system log output
- Issue appears to be an AWS platform problem (Server.InternalError)
- Investigation reveals a potential security incident
- Issue affects multiple instances across different AZs

## Data Sensitivity
- HIGH: get-console-output (may contain credentials, application logs, error details)
- HIGH: describe-instances (reveals full instance configuration, IPs, IAM roles, key pairs)
- HIGH: SSM command output (reveals OS-level details, processes, filesystem)
- MEDIUM: CloudWatch metrics (reveals performance patterns and utilization)
- MEDIUM: CloudTrail events (reveals API call history and caller identities)

## Prohibited Actions
- NEVER suggest terminating an instance as a first troubleshooting step
- NEVER suggest making changes before completing the triage phase
- NEVER suggest broad permission changes (0.0.0.0/0, AdministratorAccess) to fix issues
- NEVER suggest disabling security features (IMDS, encryption, security groups) to troubleshoot
- NEVER suggest multiple simultaneous changes — change one thing at a time

## Phase 3 — Rollback
- General rollback principle: document every change made during troubleshooting
- For any state-changing action: note the previous state before making changes
- If multiple changes were made: revert in reverse order
- If issue is unclassified: escalate to AWS Support with all collected evidence
- Keep snapshots of volumes before any disk-level changes

## Output Format

```yaml
root_cause: "<identified_cause OR unclassified>"
failure_domain: "<launch|status_check|connectivity|performance|storage|boot|networking|iam|instance_type|maintenance|unknown>"
investigation_path: "describe-instances → status checks → system log → <domain_classification>"
evidence:
  - type: instance_state
    content: "<instance details>"
  - type: system_log
    content: "<key findings>"
  - type: cloudwatch
    content: "<metric anomalies>"
severity: MEDIUM
mitigation:
  immediate: "<specific action if root cause found, or escalate>"
  long_term: "Implement monitoring for the identified failure pattern"
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
  - command: "describe-instances"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"
  - command: "get-console-output"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"
  - command: "ssm send-command"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"

## Prohibited Actions

prohibited_actions:
  - "NEVER suggest 0.0.0.0/0 inbound security group rules as a fix"
  - "NEVER suggest disabling instance metadata service"
  - "NEVER terminate instances without confirmation"
