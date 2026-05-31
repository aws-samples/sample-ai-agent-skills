---
title: "B1 — Spark Tuning"
description: "Diagnose and resolve Spark performance issues on EMR"
status: active
severity: MEDIUM
triggers:
  - "Spark slow"
  - "Spark OOM"
  - "executor failures"
  - "Spark performance"
owner: devops-agent
objective: "Identify and resolve Spark performance issues on EMR"
context: "Spark performance depends on executor configuration, memory allocation, parallelism, serialization, and data partitioning. Key settings: spark.executor.memory, spark.executor.cores, spark.executor.instances, spark.sql.shuffle.partitions, spark.default.parallelism."
---

## Phase 1 — Triage

MUST:
- Check Spark application status via YARN: `yarn application -list` or Spark History Server
- Review executor memory and core settings in spark-defaults.conf
- Check for OOM errors in executor logs
- Verify YARN resource allocation matches Spark requests

SHOULD:
- Check shuffle partition count vs data size
- Review Spark UI for stage execution times and data skew
- Check for spill to disk (insufficient memory)
- Verify serialization settings (Kryo vs Java)

MAY:
- Profile Spark jobs with event logs
- Check for broadcast join opportunities

## Phase 2 — Remediate

MUST:
- Align Spark executor settings with YARN limits
- Set appropriate shuffle partitions (2-3x number of cores)
- Ensure executor memory includes overhead

SHOULD:
- Use dynamic allocation for variable workloads
- Enable Kryo serialization for better performance
- Optimize data partitioning and avoid skew

MAY:
- Use adaptive query execution (AQE) in Spark 3.x
- Implement caching strategies for repeated data access

## Common Issues

- symptoms: "Executor OOM errors"
  diagnosis: "Insufficient executor memory or memory overhead."
  resolution: "Increase spark.executor.memory and spark.executor.memoryOverhead."

- symptoms: "Spark job very slow with few tasks"
  diagnosis: "Too few partitions causing low parallelism."
  resolution: "Increase spark.sql.shuffle.partitions or repartition data."

## Safety Ratings

| Phase 2 Action | safety_rating | Rationale |
|---|---|---|
| Align Spark executor settings with YARN limits | GREEN | Configuration tuning — applied on next run |
| Set appropriate shuffle partitions | GREEN | Configuration tuning — non-destructive |
| Enable dynamic allocation | YELLOW | Changes resource allocation behavior at runtime |
| Enable Kryo serialization | GREEN | Performance optimization — non-destructive |
| Use adaptive query execution (AQE) | GREEN | Spark 3.x optimization — non-destructive |

## Escalation Conditions

- Cluster runs production Spark jobs
- Fix requires cluster termination and recreation
- Performance degradation affects SLA for data delivery
- Tuning changes may impact other jobs sharing the cluster

## Data Sensitivity

- **Classification: HIGH**
- Spark configuration reveals memory allocation and processing capacity
- Spark UI exposes query plans, data volumes, and execution patterns
- Executor logs may contain data samples and query details
- Performance metrics reveal workload characteristics and data scale

## Prohibited Actions

- **NEVER** suggest terminating a cluster with running steps without saving state
- **NEVER** suggest disabling encryption on a cluster or its storage
- **NEVER** suggest reducing Spark security settings (e.g., disabling SSL) for performance
- **NEVER** suggest killing running Spark applications without confirming with the user

## Phase 3 — Rollback

| State-Changing Action | Rollback Step |
|---|---|
| Changed Spark executor settings | Revert spark-defaults.conf to previous values |
| Enabled dynamic allocation | Disable dynamic allocation and set fixed executor count |
| Changed shuffle partition count | Revert spark.sql.shuffle.partitions to previous value |
| Enabled Kryo serialization | Revert to Java serialization if Kryo causes issues |

## Output Format

```yaml
root_cause: "spark_tuning — <specific_cause>"
evidence:
  - type: spark_config
    content: "<Spark configuration settings>"
  - type: performance_metrics
    content: "<execution times and resource usage>"
severity: MEDIUM
mitigation:
  immediate: "Tune Spark settings for the workload"
  long_term: "Create workload-specific Spark configuration templates"
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
  - "NEVER suggest disabling encryption"
  - "NEVER suggest public subnet placement for clusters"
  - "NEVER suggest removing Kerberos authentication"
