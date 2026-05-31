---
title: "G1 â€” Worker Tier SQS Issues"
description: "Diagnose worker tier SQS message processing problems"
status: active
severity: HIGH
triggers:
  - "worker tier not processing"
  - "SQS messages not consumed"
  - "worker daemon"
  - "sqsd"
  - "worker queue"
owner: devops-agent
objective: "Ensure the worker tier correctly polls and processes SQS messages"
context: "Worker tier environments use the sqsd daemon to poll an SQS queue and POST messages to http://localhost/ on the instance. The daemon manages message visibility, retry, and dead-letter queue routing. Common issues include queue permissions, daemon configuration, application endpoint errors, and message format problems."
---

## Phase 1 â€” Triage

MUST:
- Check environment health: `aws elasticbeanstalk describe-environment-health --environment-name <env-name> --attribute-names All`
- Check SQS queue attributes: `aws sqs get-queue-attributes --queue-url <queue-url> --attribute-names All`
- Check approximate message count: `aws sqs get-queue-attributes --queue-url <queue-url> --attribute-names ApproximateNumberOfMessages,ApproximateNumberOfMessagesNotVisible`
- Check worker configuration: `aws elasticbeanstalk describe-configuration-settings --application-name <app-name> --environment-name <env-name> --query 'ConfigurationSettings[0].OptionSettings[?Namespace==`aws:elasticbeanstalk:sqsd`]'`
- List recent events: `aws elasticbeanstalk describe-events --environment-name <env-name> --severity ERROR --max-records 20`

SHOULD:
- Verify the instance profile has SQS permissions: `aws iam list-attached-role-policies --role-name <instance-role>`
- Check the sqsd daemon logs via log retrieval
- Verify the application is listening on the configured HTTP path and port

MAY:
- Send a test message to the queue: `aws sqs send-message --queue-url <queue-url> --message-body "test"`
- Check CloudWatch metrics for SQS: `aws cloudwatch get-metric-statistics --namespace AWS/SQS --metric-name NumberOfMessagesReceived --dimensions Name=QueueName,Value=<queue-name> --start-time <start> --end-time <end> --period 300 --statistics Sum`

## Phase 2 â€” Remediate

MUST:
- Ensure the instance profile has `sqs:ReceiveMessage`, `sqs:DeleteMessage`, `sqs:ChangeMessageVisibility` permissions
- Verify the application responds with HTTP 200 to POST requests on the configured path
- Fix the SQS queue URL in the worker configuration if incorrect

SHOULD:
- Set appropriate visibility timeout (should be longer than message processing time)
- Configure a dead-letter queue for messages that fail repeatedly
- Set the HTTP path to match the application's message processing endpoint

MAY:
- Configure periodic tasks using cron.yaml for scheduled message processing
- Adjust the daemon's InactivityTimeout and VisibilityTimeout settings

## Common Issues

- symptoms: "Messages accumulating in queue â€” not being processed"
  diagnosis: "sqsd daemon is not running, queue URL is wrong, or SQS permissions are missing."
  resolution: "Check daemon logs. Verify queue URL and IAM permissions for SQS operations."

- symptoms: "Messages processed but immediately reappear in queue"
  diagnosis: "Application returns non-200 HTTP status, so sqsd does not delete the message."
  resolution: "Fix the application to return HTTP 200 on successful processing."

- symptoms: "Worker tier cannot access the SQS queue"
  diagnosis: "Instance profile lacks SQS permissions or the queue is in a different region/account."
  resolution: "Add SQS permissions to the instance role. Ensure queue is in the same region."

## Output Format

```yaml
root_cause: "worker_sqs_issue â€” <specific_cause>"
evidence:
  - type: queue_attributes
    content: "<SQS queue metrics and configuration>"
  - type: worker_config
    content: "<sqsd daemon configuration>"
  - type: instance_health
    content: "<worker instance health>"
severity: HIGH
mitigation:
  immediate: "Fix SQS permissions, queue URL, or application endpoint"
  long_term: "Configure dead-letter queue and monitor queue depth"
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
