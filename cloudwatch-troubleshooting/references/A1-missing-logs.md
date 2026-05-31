---
title: "A1 — Missing Logs"
description: "Diagnose why logs are not appearing in a CloudWatch log group"
status: active
severity: HIGH
triggers:
  - "logs not appearing"
  - "missing log events"
  - "log group empty"
  - "no logs in CloudWatch"
  - "application logs missing"
owner: devops-agent
objective: "Identify why expected log events are not being delivered to the CloudWatch log group"
context: "Missing logs are caused by IAM permission issues, incorrect log group names, CloudWatch Agent misconfiguration, VPC endpoint missing, log group in wrong region, or the application not writing to stdout/stderr (for container workloads). Lambda functions auto-create log groups but need logs:CreateLogGroup and logs:PutLogEvents permissions."
---

## Phase 1 — Triage

MUST:
- Verify the log group exists: `aws logs describe-log-groups --log-group-name-prefix <name>`
- Check the log group region matches the resource region
- Verify IAM role/policy allows `logs:CreateLogGroup`, `logs:CreateLogStream`, `logs:PutLogEvents`
- Check if log events exist in a different time range: `aws logs filter-log-events --log-group-name <name> --start-time <epoch-ms> --limit 5`

SHOULD:
- Check CloudWatch Agent status on the instance (if applicable): `sudo /opt/aws/amazon-cloudwatch-agent/bin/amazon-cloudwatch-agent-ctl -a status`
- Verify VPC endpoints exist if the resource is in a private subnet: `aws ec2 describe-vpc-endpoints --filters Name=service-name,Values=com.amazonaws.<region>.logs`
- Check CloudTrail for PutLogEvents failures

MAY:
- Check if the application is writing to the expected log file path
- Verify container logging driver configuration (awslogs driver for ECS)

## Phase 2 — Remediate

MUST:
- Create the log group if missing: `aws logs create-log-group --log-group-name <name>`
- Fix IAM permissions to include CloudWatch Logs actions
- Ensure resource is in the same region as the expected log group

SHOULD:
- Add VPC endpoint for CloudWatch Logs if in a private subnet
- Restart the CloudWatch Agent after configuration changes
- Verify the awslogs log driver configuration for ECS tasks

MAY:
- Enable CloudWatch Agent debug logging for further investigation

## Common Issues

- symptoms: "Lambda function logs not appearing"
  diagnosis: "Lambda execution role missing logs:CreateLogGroup or logs:PutLogEvents permission."
  resolution: "Attach AWSLambdaBasicExecutionRole managed policy or add logs permissions to the execution role."

- symptoms: "ECS container logs missing"
  diagnosis: "Task definition not using awslogs log driver, or task execution role missing logs permissions."
  resolution: "Set logDriver to awslogs in container definition. Ensure task execution role has logs permissions."

- symptoms: "EC2 application logs not appearing"
  diagnosis: "CloudWatch Agent not running or misconfigured log file path."
  resolution: "Check agent status and verify the log file path in the agent configuration JSON."

## Safety Ratings

| Phase 2 Action | Rating | Rationale |
|---|---|---|
| Create log group | GREEN | New resource creation |
| Fix IAM permissions | YELLOW | IAM changes have broad impact potential |
| Add VPC endpoint for CloudWatch Logs | GREEN | Additive network configuration |
| Restart CloudWatch Agent | YELLOW | Brief interruption in log collection |

## Escalation Conditions

- Production application logs missing — escalate immediately to application team
- Lambda function logs missing affecting debugging — escalate to development team
- IAM permission changes required — escalate to IAM administrator
- VPC endpoint creation in production VPCs — escalate to network team

## Data Sensitivity

| Triage Command | Sensitivity | Rationale |
|---|---|---|
| `aws logs describe-log-groups` | MEDIUM | Exposes log group names and configuration |
| `aws logs filter-log-events` | HIGH | Exposes actual log content |
| `aws ec2 describe-vpc-endpoints` | LOW | Network configuration |

## Prohibited Actions

- **NEVER** suggest deleting log groups with retention data without confirming data is no longer needed
- **NEVER** suggest disabling alarms in production without approval
- **NEVER** suggest removing metric filters that feed production alarms
- **NEVER** delete log groups that are actively receiving logs from production services

## Phase 3 — Rollback

| Remediation Action | Rollback Step |
|---|---|
| Created log group | Delete: `aws logs delete-log-group` (caution: data loss) |
| Fixed IAM permissions | Revert IAM policy changes |
| Added VPC endpoint | Delete: `aws ec2 delete-vpc-endpoints` |
| Restarted CloudWatch Agent | No rollback needed; agent restarts automatically |

## Output Format

```yaml
root_cause: "missing_logs — <specific_cause>"
evidence:
  - type: log_group_config
    content: "<log group existence and configuration>"
  - type: iam_permissions
    content: "<relevant IAM policy>"
severity: HIGH
mitigation:
  immediate: "Fix IAM permissions, log group, or agent configuration"
  long_term: "Automate log group creation and IAM policy validation"
```

## Safety Ratings

safety_ratings:
  - "Phase 1 triage commands (describe/get/list): GREEN — read-only"
  - "Phase 2 configuration changes: YELLOW — state-changing but recoverable"
  - "Phase 2 resource deletion or security changes: RED — destructive or irreversible"

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
  - "NEVER suggest deleting alarms without review"
  - "NEVER reduce log retention below compliance requirements"
  - "NEVER disable detailed monitoring in production"
