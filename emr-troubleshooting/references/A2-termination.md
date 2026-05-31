---
title: "A2 — Unexpected Cluster Termination"
description: "Diagnose why an EMR cluster terminated unexpectedly"
status: active
severity: CRITICAL
triggers:
  - "cluster terminated unexpectedly"
  - "TERMINATED_WITH_ERRORS"
  - "cluster disappeared"
  - "unexpected termination"
owner: devops-agent
objective: "Identify why the cluster terminated and prevent recurrence"
context: "Clusters can terminate due to step failures (action-on-failure=TERMINATE), bootstrap failures, master node failure, manual termination, auto-termination policy, or internal errors. The state change reason and CloudTrail provide key evidence."
---

## Phase 1 — Triage

MUST:
- Check termination reason: `aws emr describe-cluster --cluster-id <id> --query 'Cluster.Status.{State:State,Reason:StateChangeReason,Timeline:Timeline}'`
- Check if termination protection was enabled: `aws emr describe-cluster --cluster-id <id> --query 'Cluster.TerminationProtected'`
- Check CloudTrail for TerminateJobFlows: `aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventName,AttributeValue=TerminateJobFlows --max-results 10`
- Check step failure with TERMINATE action: `aws emr list-steps --cluster-id <id>`

SHOULD:
- Check auto-termination policy: `aws emr describe-cluster --cluster-id <id> --query 'Cluster.AutoTerminationPolicy'`
- Review bootstrap action logs in S3
- Check master node system logs
- Verify EC2 instance health events

MAY:
- Check for Spot instance interruptions
- Review CloudWatch metrics before termination

## Phase 2 — Remediate

MUST:
- Enable termination protection for production clusters
- Set step action-on-failure to CONTINUE instead of TERMINATE
- Fix the root cause (bootstrap, step, or infrastructure)

SHOULD:
- Configure auto-termination with appropriate idle timeout
- Use On-Demand instances for master and core nodes
- Set up CloudWatch alarms for cluster health

MAY:
- Implement cluster recreation automation
- Create incident response procedures for unexpected terminations

## Common Issues

- symptoms: "Cluster terminated after step failure"
  diagnosis: "Step action-on-failure set to TERMINATE_CLUSTER."
  resolution: "Change action-on-failure to CONTINUE. Fix the failing step."

- symptoms: "Cluster terminated with no apparent reason"
  diagnosis: "Auto-termination policy triggered or manual termination."
  resolution: "Check auto-termination policy. Check CloudTrail for TerminateJobFlows."

## Safety Ratings

| Phase 2 Action | safety_rating | Rationale |
|---|---|---|
| Enable termination protection | GREEN | Adds protection — non-destructive |
| Set step action-on-failure to CONTINUE | YELLOW | Changes failure behavior — may mask errors |
| Configure auto-termination with idle timeout | YELLOW | May terminate cluster if misconfigured |
| Use On-Demand instances for master/core | GREEN | Improves stability — cost increase |
| Implement cluster recreation automation | GREEN | Adds resilience — non-destructive |

## Escalation Conditions

- Cluster runs production Spark jobs
- Fix requires cluster termination and recreation
- Unexpected termination caused data loss on HDFS
- Termination affected a long-running pipeline with no checkpoint

## Data Sensitivity

- **Classification: HIGH**
- Termination reason and CloudTrail events reveal cluster operational patterns
- Step failure logs may contain query details and data access patterns
- Auto-termination policy reveals cluster usage patterns
- Spot interruption data reveals cost optimization strategy

## Prohibited Actions

- **NEVER** suggest terminating a cluster with running steps without saving state
- **NEVER** suggest disabling encryption on a cluster or its storage
- **NEVER** suggest removing termination protection on production clusters without explicit approval
- **NEVER** suggest ignoring unexpected terminations without root cause analysis

## Phase 3 — Rollback

| State-Changing Action | Rollback Step |
|---|---|
| Changed step action-on-failure to CONTINUE | Revert to TERMINATE_CLUSTER if that was the intended behavior |
| Configured auto-termination policy | Remove or adjust auto-termination policy |
| Switched to On-Demand instances | Revert to Spot if cost is a concern and stability is acceptable |

## Output Format

```yaml
root_cause: "termination — <specific_cause>"
evidence:
  - type: state_change_reason
    content: "<termination reason>"
  - type: cloudtrail
    content: "<termination event details>"
severity: CRITICAL
mitigation:
  immediate: "Fix root cause and relaunch cluster"
  long_term: "Enable termination protection and configure proper failure handling"
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
