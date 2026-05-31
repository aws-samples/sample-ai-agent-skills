---
title: "Z1 — General MWAA Troubleshooting (Catch-All)"
description: "Fallback SOP for MWAA issues that do not match any specific runbook"
status: active
severity: MEDIUM
triggers:
  - ".*"
owner: devops-agent
objective: "Systematically investigate an unknown MWAA issue, classify the failure domain, and match to an existing SOP or escalate"
context: "This SOP is invoked when symptoms don't match any of the specific runbooks. It provides a broad, methodical investigation that narrows the failure domain step by step."
---

## Phase 1 — Triage

MUST:
- List all environments: `aws mwaa list-environments`
- Get environment details: `aws mwaa get-environment --name <env-name>`
- Check environment status: `aws mwaa get-environment --name <env-name> --query 'Environment.{Status:Status,StatusReason:StatusReason,AirflowVersion:AirflowVersion}'`
- Check execution role: `aws mwaa get-environment --name <env-name> --query 'Environment.ExecutionRoleArn'`
- Check S3 configuration: `aws mwaa get-environment --name <env-name> --query 'Environment.{SourceBucketArn:SourceBucketArn,DagS3Path:DagS3Path}'`
- Check logging configuration: `aws mwaa get-environment --name <env-name> --query 'Environment.LoggingConfiguration'`

SHOULD:
- Check recent CloudTrail events: `aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventSource,AttributeValue=airflow.amazonaws.com --max-results 20`
- Check CloudWatch metrics: `aws cloudwatch get-metric-statistics --namespace AWS/MWAA --metric-name RunningTasks --dimensions Name=Environment,Value=<env-name> --start-time <start> --end-time <end> --period 300 --statistics Average`
- Check VPC configuration: `aws mwaa get-environment --name <env-name> --query 'Environment.NetworkConfiguration'`
- Check error logs across all components: `aws logs filter-log-events --log-group-name airflow-<env-name>-Scheduler --filter-pattern "ERROR" --start-time <epoch-ms>`

## Phase 2 — Classify

Based on triage results, classify into a failure domain:
- Environment creation/update/version issues → Environment (A1-A3)
- DAG import, task, or scheduling failures → DAGs (B1-B3)
- VPC or web server access issues → Connectivity (C1-C2)
- Worker scaling or queue backlog → Performance (D1-D2)
- Plugin or requirements problems → Plugins (E1-E2)
- Log delivery or missing logs → Logging (F1-F2)
- IAM or secrets backend issues → Security (G1-G2)
- S3 bucket or connection problems → Integration (H1-H2)

If classified: switch to the specific SOP immediately.
If unclassified: continue to Phase 3.

## Phase 3 — Deep Investigation

MUST:
- Check all environment configurations systematically
- Review CloudTrail for recent configuration changes
- Verify IAM execution role permissions match requirements
- Test DAG sync and task execution via CLI token

SHOULD:
- Check AWS Health Dashboard for MWAA service events
- Compare with a known-good environment configuration
- Review all CloudWatch log groups for errors

## Phase 4 — Report

MUST:
- State the investigation path taken
- State root cause if identified, or "unclassified" with best hypothesis
- List all evidence collected
- Recommend next steps

## Output Format

```yaml
root_cause: "<identified_cause OR unclassified>"
failure_domain: "<environment|dags|connectivity|performance|plugins|logging|security|integration|unknown>"
investigation_path: "environment status → DAGs → connectivity → <domain_classification>"
evidence:
  - type: environment_config
    content: "<environment configuration summary>"
  - type: logs
    content: "<relevant error logs>"
  - type: cloudtrail
    content: "<relevant events>"
severity: MEDIUM
mitigation:
  immediate: "<specific action if root cause found, or escalate>"
  long_term: "Implement monitoring for the identified failure pattern"
```


## Safety Ratings
- Overall Risk: MEDIUM
- Data Exposure Risk: MEDIUM — general troubleshooting may involve inspecting sensitive configurations
- Service Disruption Risk: LOW — diagnostic commands are read-only
- Compliance Impact: LOW — general troubleshooting follows standard operational procedures

## Escalation Conditions
- Issue cannot be resolved using standard troubleshooting procedures
- Environment enters an unrecoverable state
- Multiple unrelated issues occur simultaneously indicating platform problem
- AWS Health Dashboard shows MWAA service event
- Issue requires AWS Support engagement

## Data Sensitivity
- Environment configuration — INTERNAL
- CloudWatch metrics and logs — INTERNAL
- CloudTrail events — INTERNAL
- Error messages and stack traces — INTERNAL (may contain CONFIDENTIAL data)
- Never expose AWS account details in external communications

## Prohibited Actions
- Do NOT make changes to production environments without a change management process
- Do NOT run destructive commands during initial diagnosis
- Do NOT share environment details in public forums without sanitizing
- Do NOT bypass established escalation procedures
- Do NOT attempt fixes that are outside the scope of MWAA service boundaries
- Do NOT ignore recurring issues — document patterns for root cause analysis

## Phase 3 — Rollback
- Revert any diagnostic changes made during troubleshooting
- Restore environment to last known good configuration if changes were applied
- Document all troubleshooting steps taken and their outcomes
- Verify environment health after troubleshooting session
- Update runbooks with new findings from the troubleshooting session
- Close out any temporary access granted for troubleshooting

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
  - "NEVER suggest disabling webserver authentication"
  - "NEVER suggest public network access mode for production"
  - "NEVER suggest overly broad Airflow execution role"
