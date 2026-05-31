---
title: "B2 — Job Timeout Issues"
description: "Diagnose IoT job execution timeout and stalled job problems"
status: active
severity: MEDIUM
triggers: ["job timed out", "execution TIMED_OUT", "job stuck in progress", "stalled job execution"]
owner: devops-agent
objective: "Resolve job timeout issues and configure appropriate timeout settings"
context: "IoT jobs have two timeout types: in-progress timeout (per-device execution time limit) and step timeout (per-step time limit). Missing timeouts can cause executions to run indefinitely."
---

## Phase 1 — Triage

MUST:
- Check job timeout configuration: `aws iot describe-job --job-id <job-id> --query '{TimeoutConfig:job.timeoutConfig,AbortConfig:job.abortConfig}'`
- List timed-out executions: `aws iot list-job-executions-for-job --job-id <job-id> --status TIMED_OUT`
- Check specific execution timing: `aws iot describe-job-execution --job-id <job-id> --thing-name <name> --query '{StartedAt:execution.startedAt,LastUpdatedAt:execution.lastUpdatedAt,Status:execution.status}'`

SHOULD:
- Verify device is still connected: `aws iot search-index --query-string "thingName:<name> AND connectivity.connected:true"`
- Check if device is reporting progress updates via MQTT
- Review job rollout configuration for rate limiting: `aws iot describe-job --job-id <job-id> --query 'job.jobExecutionsRolloutConfig'`

MAY:
- Check CloudWatch metrics for timeout trends: `aws cloudwatch get-metric-statistics --namespace AWS/IoT --metric-name JobsTimedOut --start-time <time> --end-time <time> --period 3600 --statistics Sum`
- Review device-side logs for processing delays

## Phase 2 — Remediate

MUST:
- Set appropriate in-progress timeout if missing: create new job with `--timeout-config inProgressTimeoutInMinutes=<minutes>`
- Ensure devices report execution status updates to prevent false timeouts
- Verify device job agent sends status updates within the timeout window

SHOULD:
- Configure step timeouts for multi-step job documents
- Set abort criteria to stop rollout if timeout rate exceeds threshold
- Configure retry criteria for TIMED_OUT executions if appropriate

MAY:
- Increase timeout values if device operations legitimately take longer
- Implement device-side heartbeat reporting during long-running operations
- Use exponential rollout to detect timeout issues early with small device groups

## Safety Ratings

```
safety_ratings:
  - "describe-job: GREEN — read-only inspection"
  - "list-job-executions-for-job: GREEN — read-only enumeration"
  - "search-index: GREEN — read-only fleet query"
  - "create-job with timeout config: YELLOW — deploys new job to devices"
  - "cancel-job: RED — stops all pending and optionally in-progress executions"
```

## Escalation Conditions

- Timeout rate exceeds 50% of target devices indicating systemic issue
- Devices become unresponsive after timeout, suggesting firmware hang
- Timeout configuration changes require modifying production job templates
- Stalled executions block subsequent jobs due to max concurrent execution limits

## Data Sensitivity

- `aws iot describe-job`: LOW — returns job configuration metadata
- `aws iot describe-job-execution`: LOW — returns execution timing and status
- `aws iot search-index`: MEDIUM — returns device connectivity and attribute data
- Device-side logs: HIGH — may contain operational data and error details

## Prohibited Actions

- NEVER set timeout to 0 or extremely low values that prevent normal execution
- NEVER force-cancel timed-out executions without checking if devices are mid-operation
- NEVER assume timeout means device failure — it may indicate slow network or large payload
- NEVER remove timeout configuration from production jobs without understanding the risk of indefinite execution

## Phase 3 — Rollback

1. If timeout was set too aggressively, cancel job: `aws iot cancel-job --job-id <job-id>`
2. Create replacement job with corrected timeout values
3. For devices stuck in TIMED_OUT, create targeted remediation job
4. If abort criteria triggered prematurely, create new job with adjusted thresholds
5. Monitor replacement job execution to confirm timeout values are appropriate

## Common Issues

- symptoms: "All executions time out after exactly the configured timeout period"
  diagnosis: "Devices are not sending status updates. The in-progress timer expires without device acknowledgment."
  resolution: "Ensure device job agent calls UpdateJobExecution with IN_PROGRESS status and step details within the timeout window."

- symptoms: "Executions time out intermittently across different devices"
  diagnosis: "Network latency or device processing speed varies. Some devices cannot complete within the timeout."
  resolution: "Increase timeout to accommodate slowest devices. Implement device-side progress reporting."

## Output Format

```yaml
root_cause: "job_timeout — <specific_cause>"
evidence:
  - type: timeout_error
    content: "<error details>"
severity: MEDIUM
mitigation:
  immediate: "Adjust timeout configuration"
  long_term: "Implement device progress reporting and adaptive timeouts"
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
