---
title: "F1 — CloudWatch Logs Integration"
description: "Diagnose CloudTrail to CloudWatch Logs delivery issues"
status: active
severity: MEDIUM
triggers:
  - "CloudWatch Logs not receiving"
  - "log group empty"
  - "CW integration broken"
  - "near-real-time events"
owner: devops-agent
objective: "Restore CloudTrail event delivery to CloudWatch Logs"
context: "CloudTrail can deliver events to CloudWatch Logs for near-real-time monitoring (1-2 min delay). This requires a log group, an IAM role, and proper configuration on the trail. Issues include role permissions, log group deletion, or configuration errors."
---

## Phase 1 — Triage

MUST:
- Check trail CW Logs configuration: `aws cloudtrail get-trail --name <trail-name> --query 'Trail.{LogGroup:CloudWatchLogsLogGroupArn,Role:CloudWatchLogsRoleArn}'`
- Check trail status for CW delivery: `aws cloudtrail get-trail-status --name <trail-name> --query '{LatestCloudWatchLogsDeliveryTime:LatestCloudWatchLogsDeliveryTime,LatestCloudWatchLogsDeliveryError:LatestCloudWatchLogsDeliveryError}'`
- Verify log group exists: `aws logs describe-log-groups --log-group-name-prefix <log-group-name>`
- Check IAM role: `aws iam get-role --role-name <cw-role-name>`

SHOULD:
- Verify IAM role trust policy allows CloudTrail
- Check IAM role has logs:CreateLogStream and logs:PutLogEvents permissions
- Verify log group retention settings

MAY:
- Check CloudWatch Logs metric filters and alarms
- Review log group subscription filters

## Phase 2 — Remediate

MUST:
- Create log group if missing: `aws logs create-log-group --log-group-name <name>`
- Create/fix IAM role with CloudTrail trust and CW Logs permissions
- Update trail: `aws cloudtrail update-trail --name <trail-name> --cloud-watch-logs-log-group-arn <arn> --cloud-watch-logs-role-arn <role-arn>`

SHOULD:
- Set log group retention to control costs
- Create metric filters for critical events
- Set up CloudWatch alarms on critical patterns

MAY:
- Configure subscription filters for real-time processing
- Set up cross-account log delivery

## Common Issues

- symptoms: "LatestCloudWatchLogsDeliveryError: Access Denied"
  diagnosis: "IAM role lacks permissions to write to CloudWatch Logs."
  resolution: "Update IAM role with logs:CreateLogStream and logs:PutLogEvents."

- symptoms: "Log group exists but no new events"
  diagnosis: "Trail CW Logs configuration removed or role deleted."
  resolution: "Re-configure CW Logs on the trail with valid log group and role."

## Safety Ratings

| Phase 2 Action | Rating | Rationale |
|---|---|---|
| Create log group | GREEN | New resource creation |
| Create/fix IAM role | YELLOW | IAM changes have broad impact potential |
| Update trail with CW Logs config | YELLOW | Trail configuration change |
| Set log group retention | GREEN | Cost management, does not affect delivery |

## Escalation Conditions

- CloudWatch Logs delivery broken for security monitoring trail — escalate immediately to security operations
- IAM role changes for CloudTrail integration — escalate to IAM administrator
- Log group deletion detected — escalate to security team
- Metric filters and alarms dependent on CW Logs delivery — escalate to monitoring team

## Data Sensitivity

| Triage Command | Sensitivity | Rationale |
|---|---|---|
| `aws cloudtrail get-trail` (CW Logs config) | HIGH | Exposes log group ARN and IAM role |
| `aws cloudtrail get-trail-status` (CW delivery) | HIGH | Exposes delivery status and errors |
| `aws iam get-role` | MEDIUM | Exposes IAM role configuration |

## Prohibited Actions

- **NEVER** suggest stopping or deleting CloudTrail trails
- **NEVER** suggest disabling log file validation
- **NEVER** suggest removing S3 bucket policy for log delivery
- **NEVER** delete CloudWatch log groups receiving CloudTrail events without confirming downstream dependencies

## Phase 3 — Rollback

| Remediation Action | Rollback Step |
|---|---|
| Created log group | Delete the log group: `aws logs delete-log-group` (caution: data loss) |
| Created/fixed IAM role | Revert IAM role to previous configuration |
| Updated trail CW Logs config | Remove CW Logs config: `aws cloudtrail update-trail` without CW Logs parameters |

## Output Format

```yaml
root_cause: "cloudwatch_logs — <specific_cause>"
evidence:
  - type: trail_cw_config
    content: "<CloudWatch Logs configuration>"
  - type: delivery_status
    content: "<latest delivery time and error>"
severity: MEDIUM
mitigation:
  immediate: "Fix IAM role and trail configuration"
  long_term: "Monitor CW Logs delivery and set up alerting"
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
  - command: "lookup-events"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"
  - command: "get-trail-status"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"
  - command: "describe-trails"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"

## Prohibited Actions

prohibited_actions:
  - "NEVER suggest disabling CloudTrail logging"
  - "NEVER suggest deleting trail logs"
  - "NEVER reduce logging scope in production"
