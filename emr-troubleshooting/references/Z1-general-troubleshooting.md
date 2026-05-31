---
title: "Z1 — General EMR Troubleshooting"
description: "Catch-all systematic investigation for EMR issues"
status: active
severity: MEDIUM
triggers:
  - "EMR issue"
  - "EMR error"
  - "cluster problem"
  - "EMR not working"
  - "general troubleshooting"
owner: devops-agent
objective: "Systematically investigate EMR issues using a structured diagnostic approach"
context: "This runbook provides a general-purpose investigation framework for EMR issues that don't clearly match a specific runbook. It covers cluster health, step status, instance groups, and systematic elimination of common causes."
---

## Phase 1 — Triage

MUST:
- Check cluster status: `aws emr describe-cluster --cluster-id <id> --query 'Cluster.Status'`
- List recent steps: `aws emr list-steps --cluster-id <id> --step-states FAILED CANCELLED`
- Check instance groups: `aws emr list-instance-groups --cluster-id <id>`
- Check cluster state change reason

SHOULD:
- Check CloudTrail for recent EMR events: `aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventSource,AttributeValue=elasticmapreduce.amazonaws.com --max-results 20`
- Review S3 cluster logs
- Check CloudWatch metrics for the cluster
- Verify IAM roles and security configuration

MAY:
- Check AWS Health Dashboard for EMR service issues
- Review application-specific logs (Spark, Hive, etc.)

## Phase 2 — Remediate

MUST:
- Based on triage findings, pivot to the appropriate specific runbook:
  - Cluster lifecycle → A1 (launch), A2 (termination), A3 (bootstrap), A4 (steps)
  - Performance → B1 (Spark), B2 (YARN), B3 (shuffle)
  - Storage → C1 (S3), C2 (HDFS)
  - Networking → D1 (VPC), D2 (security groups)
  - Scaling → E1 (managed), E2 (fleet)
  - EMR on EKS → F1 (virtual cluster), F2 (job runs)
  - Serverless → G1 (applications), G2 (capacity)
  - Security → H1 (Kerberos), H2 (Lake Formation)
  - Notebooks → I1 (Studio), I2 (Jupyter)
- Document findings and root cause

SHOULD:
- Check for multiple concurrent issues
- Review recent changes

MAY:
- Set up comprehensive monitoring
- Create post-incident report

## Common Issues

- symptoms: "Cluster in WAITING state but jobs not running"
  diagnosis: "No steps submitted or step concurrency exhausted."
  resolution: "Submit steps. Check step concurrency configuration."

- symptoms: "Cluster performance degraded"
  diagnosis: "Resource contention, data skew, or insufficient capacity."
  resolution: "Check YARN resources, Spark configuration, and scaling settings."

## Safety Ratings

| Phase 2 Action | safety_rating | Rationale |
|---|---|---|
| Check cluster status and steps | GREEN | Read-only investigation |
| Check CloudTrail and CloudWatch | GREEN | Read-only investigation |
| Review S3 cluster logs | GREEN | Read-only investigation |
| Pivot to specific runbook | YELLOW | Specific runbook may include state-changing actions |
| Set up comprehensive monitoring | GREEN | Monitoring — non-destructive |

## Escalation Conditions

- Cluster runs production Spark jobs
- Fix requires cluster termination and recreation
- Issue cannot be classified after systematic investigation
- Multiple concurrent issues affect cluster stability

## Data Sensitivity

- **Classification: HIGH**
- Cluster status reveals operational state and configuration
- Step execution details may contain query plans and data access patterns
- CloudTrail events expose all EMR API calls and configurations
- S3 cluster logs may contain application output with sensitive data

## Prohibited Actions

- **NEVER** suggest terminating a cluster with running steps without saving state
- **NEVER** suggest disabling encryption on a cluster or its storage
- **NEVER** suggest making broad configuration changes during general investigation
- **NEVER** suggest restarting services on the master node without understanding impact

## Phase 3 — Rollback

| State-Changing Action | Rollback Step |
|---|---|
| General triage is read-only | No rollback needed for Phase 1 investigation |
| Pivot to specific runbook | Follow rollback steps in the specific runbook |

## Output Format

```yaml
root_cause: "general — <specific_cause>"
evidence:
  - type: cluster_status
    content: "<cluster state and configuration>"
  - type: steps
    content: "<step execution status>"
  - type: instances
    content: "<instance group status>"
severity: MEDIUM
general_analysis:
  cluster_state: "RUNNING | WAITING | TERMINATED"
  steps: "all passed | failures found"
  resources: "sufficient | constrained"
  specific_runbook: "<recommended runbook ID>"
mitigation:
  immediate: "Follow the identified specific runbook"
  long_term: "Implement comprehensive EMR monitoring and alerting"
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
