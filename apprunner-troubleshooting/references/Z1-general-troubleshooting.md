---
title: "Z1 — General App Runner Troubleshooting (Catch-All)"
description: "Fallback SOP for App Runner issues that do not match any specific runbook"
status: active
severity: MEDIUM
triggers:
  - ".*"
owner: devops-agent
objective: "Systematically investigate an unknown App Runner issue, classify the failure domain, and match to an existing SOP or escalate"
context: "This SOP is invoked when symptoms don't match any of the specific runbooks. It provides a broad, methodical investigation that narrows the failure domain step by step."
---

## Phase 1 — Triage

MUST:
- Get service overview: `aws apprunner describe-service --service-arn <service-arn>`
- Check service status: `aws apprunner describe-service --service-arn <service-arn> --query 'Service.Status'`
- List recent operations: `aws apprunner list-operations --service-arn <service-arn> --max-results 10`
- Check error rates: `aws cloudwatch get-metric-statistics --namespace AWS/AppRunner --metric-name 5xxStatusResponses --dimensions Name=ServiceName,Value=<service-name> Name=ServiceId,Value=<service-id> --start-time <start> --end-time <end> --period 300 --statistics Sum`
- Check request count: `aws cloudwatch get-metric-statistics --namespace AWS/AppRunner --metric-name RequestCount --dimensions Name=ServiceName,Value=<service-name> Name=ServiceId,Value=<service-id> --start-time <start> --end-time <end> --period 300 --statistics Sum`
- Review application logs: `aws logs get-log-events --log-group-name /aws/apprunner/<service-name>/<service-id>/application --log-stream-name <stream>`

SHOULD:
- Check recent CloudTrail events: `aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventSource,AttributeValue=apprunner.amazonaws.com --max-results 10`
- Check source configuration: `aws apprunner describe-service --service-arn <service-arn> --query 'Service.SourceConfiguration'`
- Check health check configuration: `aws apprunner describe-service --service-arn <service-arn> --query 'Service.HealthCheckConfiguration'`
- Check network configuration: `aws apprunner describe-service --service-arn <service-arn> --query 'Service.NetworkConfiguration'`
- Check instance configuration: `aws apprunner describe-service --service-arn <service-arn> --query 'Service.InstanceConfiguration'`

## Phase 2 — Classify

Based on triage results, classify into a failure domain:
- Deployment/operation failures → Deployment (A1-A3)
- Health check failures, crashes, port issues → Runtime (B1-B3)
- Scaling or latency problems → Scaling (C1-C2)
- Custom domain, VPC, connectivity → Networking (D1-D3)
- ECR or GitHub source problems → Source (E1-E2)
- IAM or secrets issues → Security (F1-F2)
- Logging or tracing problems → Observability (G1-G2)

If classified: switch to the specific SOP immediately.
If unclassified: continue to Phase 3.

## Phase 3 — Deep Investigation

MUST:
- Check all service configurations systematically
- Review CloudTrail for recent configuration changes
- Check CloudWatch Logs for both application and service log groups
- Verify all IAM roles (access role and instance role)

SHOULD:
- Check AWS Health Dashboard for App Runner service events
- Compare with a known-good service configuration
- Check if the issue correlates with a recent deployment or configuration change

MAY:
- Check VPC connector configuration if attached
- Check custom domain status if configured
- Review auto-scaling configuration and metrics

## Phase 4 — Report

MUST:
- State the investigation path taken
- State root cause if identified, or "unclassified" with best hypothesis
- List all evidence collected
- Recommend next steps

## Output Format

```yaml
root_cause: "<identified_cause OR unclassified>"
failure_domain: "<deployment|runtime|scaling|networking|source|security|observability|unknown>"
investigation_path: "service status → operations → CloudWatch → logs → <domain_classification>"
evidence:
  - type: service_config
    content: "<service configuration summary>"
  - type: cloudwatch
    content: "<error rates and metrics>"
  - type: operations
    content: "<recent operation statuses>"
  - type: logs
    content: "<relevant log entries>"
severity: MEDIUM
mitigation:
  immediate: "<specific action if root cause found, or escalate>"
  long_term: "Implement monitoring for the identified failure pattern"
```


## Safety Ratings

| Phase 2 Action | safety_rating | Rationale |
|---|---|---|
| Get service overview | GREEN | Read-only check |
| Check service status | GREEN | Read-only check |
| List recent operations | GREEN | Read-only check |
| Check error rates | GREEN | Read-only metrics |
| Check request count | GREEN | Read-only metrics |
| Review application logs | GREEN | Read-only check |
| Check CloudTrail events | GREEN | Read-only audit |

## Escalation Conditions

- Fix requires redeployment of production service
- NEVER suggest scaling to 0 (not supported)

## Data Sensitivity

- **Service configuration**: Full service details — INTERNAL
- **CloudWatch metrics**: Error rates and request counts — INTERNAL
- **Application logs**: Application output — INTERNAL
- **CloudTrail events**: API call history — INTERNAL

## Prohibited Actions

- NEVER suggest hardcoding credentials in environment variables
- NEVER suggest scaling to 0 instances (not supported by App Runner)
- NEVER make configuration changes during initial triage phase

## Phase 3 — Rollback

1. Revert any configuration changes made during investigation
2. If a deployment was triggered, wait for automatic rollback on failure
3. Verify service returns to RUNNING status
4. Check application logs to confirm previous version is serving traffic
5. Document findings and escalate if root cause is unclassified

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
  - "NEVER suggest overly broad IAM permissions as a fix"
  - "NEVER suggest disabling security controls for troubleshooting"
  - "NEVER suggest deleting resources without confirmation"
