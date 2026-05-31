---
title: "B1 — Flink Job Failures"
description: "Diagnose Apache Flink job failures in Managed Flink applications"
status: active
severity: HIGH
triggers: ["Flink job failed", "job exception", "task failure", "operator error"]
owner: devops-agent
objective: "Resolve Flink job failures including operator errors, serialization issues, and out-of-memory errors"
context: "Flink jobs consist of operators connected in a DAG. Failures can occur at any operator due to code bugs, data issues, or resource exhaustion."
---

## Phase 1 — Triage

MUST:
- Check application status: `aws kinesisanalyticsv2 describe-application --application-name <app-name>`
- Review CloudWatch Logs for exceptions: `aws logs filter-log-events --log-group-name /aws/kinesis-analytics/<app-name> --filter-pattern "Exception" --start-time <epoch-ms> --limit 50`
- Check downtime metric: `aws cloudwatch get-metric-statistics --namespace AWS/KinesisAnalytics --metric-name downtime --dimensions Name=Application,Value=<app-name> --start-time <start> --end-time <end> --period 300 --statistics Maximum`

SHOULD:
- Check heap memory usage: `aws cloudwatch get-metric-statistics --namespace AWS/KinesisAnalytics --metric-name heapMemoryUtilization --dimensions Name=Application,Value=<app-name> --start-time <start> --end-time <end> --period 300 --statistics Maximum`
- Review number of restarts metric

MAY:
- Check thread count and GC metrics for resource exhaustion
- Review operator-level metrics for bottleneck identification

## Phase 2 — Remediate

MUST:
- Fix code bugs identified in exception logs
- Resolve serialization errors (ensure all types are serializable)
- Address out-of-memory errors by increasing KPUs or optimizing state

SHOULD:
- Add error handling for malformed input records
- Configure dead-letter queues for unprocessable records

MAY:
- Optimize operator chain for better resource utilization
- Implement custom metrics for application-specific monitoring

## Safety Ratings

```
safety_ratings:
  - "describe-application: GREEN — read-only application details"
  - "logs filter-log-events: GREEN — read-only log review"
  - "cloudwatch get-metric-statistics: GREEN — read-only metrics"
  - "update-application: YELLOW — modifies application configuration"
```

## Escalation Conditions

- Job fails with internal Flink errors not related to user code
- Out-of-memory errors persist after increasing KPUs
- Job failure causes data loss in downstream systems
- Failure pattern is intermittent and difficult to reproduce

## Data Sensitivity

- `aws logs filter-log-events`: HIGH — may contain application data in exception messages
- `aws cloudwatch get-metric-statistics`: LOW — aggregated metrics only
- `aws kinesisanalyticsv2 describe-application`: MEDIUM — reveals application configuration

## Prohibited Actions

- NEVER restart a failing application without diagnosing the root cause
- NEVER increase KPUs as a first response to job failures
- NEVER disable checkpointing to avoid checkpoint-related failures
- NEVER ignore serialization errors — they indicate data model issues

## Phase 3 — Rollback

1. Stop the failing application: `aws kinesisanalyticsv2 stop-application --application-name <name>`
2. Restore previous code version from S3 if code change caused the failure
3. Restore from a known-good snapshot: `aws kinesisanalyticsv2 start-application --application-name <name> --run-configuration '{"FlinkRunConfiguration":{"AllowNonRestoredState":false}}'`
4. Verify application is processing data correctly after rollback
5. Document the failure, root cause, and fix for future reference

## Common Issues

- symptoms: "Application restarts repeatedly with 'java.io.NotSerializableException'"
  diagnosis: "A class used in Flink operators is not serializable."
  resolution: "Ensure all classes used in map/filter/process functions implement Serializable. Use transient for non-serializable fields."

## Output Format

```yaml
root_cause: "flink_job — <specific_cause>"
evidence:
  - type: job_exception
    content: "<exception details>"
severity: HIGH
mitigation:
  immediate: "Fix job failure"
  long_term: "Add error handling"
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
