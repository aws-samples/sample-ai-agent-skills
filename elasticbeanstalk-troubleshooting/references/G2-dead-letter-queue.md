---
title: "G2 â€” Dead Letter Queue Issues"
description: "Diagnose dead letter queue configuration and message handling problems"
status: active
severity: MEDIUM
triggers:
  - "dead letter queue"
  - "DLQ"
  - "failed messages"
  - "message retry"
  - "poison messages"
owner: devops-agent
objective: "Configure and troubleshoot dead letter queue for failed message handling"
context: "The worker tier can route messages that fail processing (after max retries) to a dead letter queue (DLQ). The DLQ must be configured on the source SQS queue as a redrive policy. Messages in the DLQ need manual investigation and reprocessing."
---

## Phase 1 â€” Triage

MUST:
- Check source queue redrive policy: `aws sqs get-queue-attributes --queue-url <queue-url> --attribute-names RedrivePolicy`
- Check DLQ message count: `aws sqs get-queue-attributes --queue-url <dlq-url> --attribute-names ApproximateNumberOfMessages`
- Check worker configuration for max retries: `aws elasticbeanstalk describe-configuration-settings --application-name <app-name> --environment-name <env-name> --query 'ConfigurationSettings[0].OptionSettings[?Namespace==`aws:elasticbeanstalk:sqsd`]'`
- Sample messages from DLQ: `aws sqs receive-message --queue-url <dlq-url> --max-number-of-messages 5 --attribute-names All`

SHOULD:
- Check if the DLQ exists and is accessible
- Verify the maxReceiveCount in the redrive policy matches expected retry behavior
- Check CloudWatch alarms for DLQ message count

MAY:
- Check DLQ CloudWatch metrics: `aws cloudwatch get-metric-statistics --namespace AWS/SQS --metric-name ApproximateNumberOfMessagesVisible --dimensions Name=QueueName,Value=<dlq-name> --start-time <start> --end-time <end> --period 300 --statistics Maximum`

## Phase 2 â€” Remediate

MUST:
- Configure a redrive policy on the source queue if DLQ is not set up
- Investigate failed messages in the DLQ to identify the failure pattern
- Fix the application to handle the failing message format

SHOULD:
- Set up CloudWatch alarms on DLQ message count for early detection
- Implement a DLQ consumer to process or archive failed messages
- Set appropriate maxReceiveCount (typically 3-5 retries)

MAY:
- Use SQS DLQ redrive to move messages back to the source queue after fixing the issue
- Implement message validation before processing to reject invalid messages early

## Common Issues

- symptoms: "Messages going to DLQ after one attempt"
  diagnosis: "maxReceiveCount is set to 1 or visibility timeout is too short."
  resolution: "Increase maxReceiveCount in the redrive policy. Ensure visibility timeout exceeds processing time."

- symptoms: "DLQ not receiving failed messages"
  diagnosis: "Redrive policy is not configured on the source queue."
  resolution: "Set the redrive policy: `aws sqs set-queue-attributes --queue-url <queue-url> --attributes '{\"RedrivePolicy\":\"{\\\"deadLetterTargetArn\\\":\\\"<dlq-arn>\\\",\\\"maxReceiveCount\\\":\\\"5\\\"}\"}'`"

- symptoms: "DLQ filling up rapidly"
  diagnosis: "Application is consistently failing to process certain message types."
  resolution: "Investigate the common pattern in DLQ messages. Fix the application or filter invalid messages."

## Output Format

```yaml
root_cause: "dlq_issue â€” <specific_cause>"
evidence:
  - type: redrive_policy
    content: "<source queue redrive policy>"
  - type: dlq_messages
    content: "<sample messages from DLQ>"
  - type: worker_config
    content: "<retry and timeout configuration>"
severity: MEDIUM
mitigation:
  immediate: "Fix DLQ configuration or application message handling"
  long_term: "Monitor DLQ depth and implement automated DLQ processing"
```

---

## Safety Ratings

| Aspect | Rating |
|--------|--------|
| Service Impact | HIGH — Elastic Beanstalk manages full application stacks including compute, networking, and load balancing |
| Data Risk | MEDIUM — Environment changes can affect application availability and data connections |
| Recovery Complexity | MEDIUM — Environments can be rebuilt but require correct configuration and deployment artifacts |
| Blast Radius | HIGH — Environment-level changes affect all instances and the application endpoint |

## Escalation Conditions

- Environment health degraded to Red with customer-facing impact
- Deployment failure with no successful rollback available
- Auto-scaling events failing causing capacity shortage under load
- Platform update causing application incompatibility in production
- CloudFormation stack in UPDATE_ROLLBACK_FAILED state requiring manual intervention
- Load balancer health checks failing across all instances

## Data Sensitivity

- Environment variables may contain database credentials, API keys, and secrets
- Instance profile roles may grant broad AWS service access
- Application source bundles in S3 contain proprietary application code
- Log files may contain PII, authentication tokens, or business data
- .ebextensions may contain infrastructure secrets and custom configurations

## PROHIBITED Actions

- NEVER suggest terminating a production environment without confirming blue-green swap is ready
- NEVER modify environment variables containing secrets without secure handling procedures
- NEVER force a platform update on production without testing on a staging environment first
- NEVER delete application versions that are currently deployed to any environment
- NEVER change the load balancer type on a running environment (requires environment rebuild)
- NEVER disable enhanced health reporting on production environments

## Phase 3 — Rollback

- If deployment failed: use ws elasticbeanstalk abort-environment-update then redeploy last known good version
- If environment configuration change caused issues: apply previous saved configuration via update-environment
- If platform update broke the application: rebuild environment on previous platform version
- If auto-scaling changes caused instability: revert min/max/desired counts to previous values
- If .ebextensions changes caused failures: redeploy previous application version with working extensions
- Document all rollback actions with environment ID, version label, and configuration template name
## Safety Ratings

safety_ratings:
  - "Phase 1 triage commands (describe/get/list): GREEN â€” read-only"
  - "Phase 2 configuration changes: YELLOW â€” state-changing but recoverable"
  - "Phase 2 resource deletion or security changes: RED â€” destructive or irreversible"

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
