---
title: "E1 — Performance Tuning"
description: "Diagnose performance issues in M2 environments"
status: active
severity: MEDIUM
triggers: ["slow performance", "batch job slow", "transaction timeout", "high CPU", "memory pressure"]
owner: devops-agent
objective: "Identify and resolve performance bottlenecks in Mainframe Modernization environments"
context: "Performance characteristics differ between mainframe and AWS. Blu Age Java applications have JVM tuning needs. Micro Focus COBOL has runtime configuration options. Both require storage and network optimization."
---

## Phase 1 — Triage

MUST:
- Check environment metrics: `aws cloudwatch get-metric-statistics --namespace AWS/M2 --metric-name CPUUtilization --dimensions Name=EnvironmentId,Value=<env-id> --start-time <start> --end-time <end> --period 300 --statistics Average,Maximum`
- Review application logs for slow operations: `aws logs filter-log-events --log-group-name /aws/m2/<environment-name> --filter-pattern "slow OR timeout OR elapsed" --start-time <epoch-ms>`
- Check batch job execution times: `aws m2 list-batch-job-executions --application-id <app-id> --query 'batchJobExecutions[*].{JobName:jobName,Status:status,StartTime:startTime,EndTime:endTime}'`

SHOULD:
- Compare execution times with mainframe baselines
- Check memory utilization and garbage collection (Blu Age/Java)
- Review I/O patterns for dataset access bottlenecks

MAY:
- Enable detailed application profiling
- Analyze CloudWatch Logs Insights for performance patterns

## Phase 2 — Remediate

MUST:
- Identify the specific bottleneck (CPU, memory, I/O, network)
- For Blu Age: tune JVM heap size, garbage collection, and thread pools
- For Micro Focus: tune runtime configuration, file handler settings

SHOULD:
- Optimize dataset access patterns (caching, buffering)
- Right-size the environment instance type for the workload

MAY:
- Implement application-level caching for frequently accessed data
- Configure batch job parallelism for independent steps

## Common Issues

- symptoms: "Batch jobs take 3x longer than on the mainframe"
  diagnosis: "I/O-intensive jobs affected by different storage characteristics on AWS."
  resolution: "Optimize file handler buffering. Consider EBS provisioned IOPS for I/O-heavy workloads. Review dataset access patterns for optimization opportunities."

## Safety Ratings

```
safety_ratings:
  - "cloudwatch get-metric-statistics: GREEN — read-only metric retrieval"
  - "logs filter-log-events: GREEN — read-only log analysis"
  - "JVM tuning: YELLOW — changes application runtime behavior"
  - "Instance type changes: YELLOW — requires environment update, may cause downtime"
```

## Escalation Conditions

- Performance degradation exceeds SLA thresholds
- Batch job windows cannot be met with current configuration
- Transaction response times affect end-user experience
- Performance issues require environment resize (downtime)

## Data Sensitivity

- Performance metrics: LOW — aggregate resource utilization data
- Application logs: MEDIUM — may contain business data references
- JVM/runtime configuration: LOW — reveals tuning parameters
- Execution time data: LOW — reveals processing patterns

## Prohibited Actions

- NEVER change JVM or runtime settings in production without testing in non-production first
- NEVER resize environments during business-critical processing windows
- NEVER assume mainframe performance baselines directly translate to AWS
- NEVER disable logging to improve performance without understanding the trade-off

## Phase 3 — Rollback

1. Revert JVM or runtime configuration changes to previous settings
2. If environment was resized, revert to previous instance type
3. Restore file handler configuration if buffering changes caused issues
4. Verify application performance returns to pre-change baseline
5. Document performance tuning attempts and results

## Output Format

```yaml
root_cause: "performance — <specific_cause>"
evidence:
  - type: performance_metrics
    content: "<CPU, memory, I/O metrics and execution times>"
severity: MEDIUM
mitigation:
  immediate: "Address performance bottleneck"
  long_term: "Implement performance monitoring and capacity planning"
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
