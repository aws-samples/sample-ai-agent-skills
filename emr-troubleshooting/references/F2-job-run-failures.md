---
title: "F2 — EMR on EKS Job Run Failures"
description: "Diagnose EMR on EKS job run failures"
status: active
severity: HIGH
triggers:
  - "job run failed"
  - "EMR on EKS job error"
  - "Spark on EKS failure"
  - "pod scheduling failed"
owner: devops-agent
objective: "Identify and resolve EMR on EKS job run failures"
context: "EMR on EKS job runs execute Spark applications on EKS pods. Failures can stem from execution role permissions, pod scheduling issues, resource limits, S3 access, or Spark configuration errors. Debugging requires checking both EMR and EKS logs."
---

## Phase 1 — Triage

MUST:
- Check job run status: `aws emr-containers describe-job-run --virtual-cluster-id <vc-id> --id <job-id>`
- Check execution role permissions
- Check EKS pod status: `kubectl get pods -n <emr-namespace>`
- Review pod events: `kubectl describe pod <pod-name> -n <emr-namespace>`

SHOULD:
- Check Spark driver and executor logs
- Verify S3 access from the execution role
- Check for pod resource limits and requests
- Review EKS node capacity

MAY:
- Check for node affinity or taint issues
- Review Spark configuration overrides

## Phase 2 — Remediate

MUST:
- Fix execution role IAM permissions (S3, CloudWatch Logs)
- Ensure EKS nodes have sufficient resources
- Fix Spark configuration errors

SHOULD:
- Configure appropriate resource requests and limits
- Set up CloudWatch logging for job runs
- Test with a simple Spark job first

MAY:
- Implement job run monitoring and alerting
- Create job run templates

## Common Issues

- symptoms: "Pod stuck in Pending state"
  diagnosis: "Insufficient EKS node resources or scheduling constraints."
  resolution: "Scale EKS nodes. Check resource requests. Remove restrictive node selectors."

- symptoms: "Job fails with S3 access denied"
  diagnosis: "Execution role lacks S3 permissions."
  resolution: "Add S3 permissions to the execution role. Verify IRSA configuration."

## Safety Ratings

| Phase 2 Action | safety_rating | Rationale |
|---|---|---|
| Fix execution role IAM permissions | YELLOW | Changes role permissions — review scope |
| Scale EKS nodes for capacity | YELLOW | Cluster scaling — cost increase |
| Fix Spark configuration errors | GREEN | Configuration fix — applied on next run |
| Configure resource requests and limits | GREEN | Pod configuration — non-destructive |
| Set up CloudWatch logging | GREEN | Monitoring — non-destructive |

## Escalation Conditions

- Cluster runs production Spark jobs
- Fix requires cluster termination and recreation
- Job run failure affects production data pipeline SLA
- EKS node scaling impacts other workloads on the shared cluster

## Data Sensitivity

- **Classification: HIGH**
- Execution role permissions reveal data access boundaries for Spark jobs
- Pod logs may contain data processing details and query results
- EKS pod events expose scheduling and resource allocation patterns
- S3 access patterns reveal data pipeline input/output locations

## Prohibited Actions

- **NEVER** suggest terminating a cluster with running steps without saving state
- **NEVER** suggest disabling encryption on a cluster or its storage
- **NEVER** suggest granting overly permissive IAM policies to execution roles
- **NEVER** suggest deleting EKS pods for running job runs without confirmation

## Phase 3 — Rollback

| State-Changing Action | Rollback Step |
|---|---|
| Modified execution role permissions | Revert IAM policy to previous version |
| Scaled EKS nodes | Scale down EKS nodes if no longer needed |
| Changed Spark configuration overrides | Revert to previous Spark configuration |
| Modified pod resource requests/limits | Revert to previous resource settings |

## Output Format

```yaml
root_cause: "job_run_failure — <specific_cause>"
evidence:
  - type: job_status
    content: "<job run status and error>"
  - type: pod_events
    content: "<EKS pod events>"
severity: HIGH
mitigation:
  immediate: "Fix the specific job run failure"
  long_term: "Create validated job templates and monitoring"
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
