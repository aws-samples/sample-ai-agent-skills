---
title: "Z1 — General Elastic Beanstalk Troubleshooting (Catch-All)"
description: "Fallback SOP for Elastic Beanstalk issues that do not match any specific runbook"
status: active
severity: MEDIUM
triggers:
  - ".*"
owner: devops-agent
objective: "Systematically investigate an unknown Elastic Beanstalk issue, classify the failure domain, and match to an existing SOP or escalate"
context: "This SOP is invoked when symptoms don't match any of the specific runbooks. It provides a broad, methodical investigation that narrows the failure domain step by step."
---

## Phase 1 — Triage

MUST:
- Check environment status: `aws elasticbeanstalk describe-environments --environment-names <env-name>`
- Check environment health: `aws elasticbeanstalk describe-environment-health --environment-name <env-name> --attribute-names All`
- List recent events (all severities): `aws elasticbeanstalk describe-events --environment-name <env-name> --max-records 20`
- List error events: `aws elasticbeanstalk describe-events --environment-name <env-name> --severity ERROR --max-records 10`
- Check instance health: `aws elasticbeanstalk describe-instances-health --environment-name <env-name>`
- Check CloudFormation stack status: `aws cloudformation describe-stacks --stack-name awseb-<env-id>-stack`

SHOULD:
- Get configuration settings: `aws elasticbeanstalk describe-configuration-settings --application-name <app-name> --environment-name <env-name>`
- Check environment resources: `aws elasticbeanstalk describe-environment-resources --environment-name <env-name>`
- Check CloudTrail events: `aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventSource,AttributeValue=elasticbeanstalk.amazonaws.com --max-results 20`
- Retrieve tail logs: `aws elasticbeanstalk request-environment-info --environment-name <env-name> --info-type tail`

## Phase 2 — Classify

Based on triage results, classify into a failure domain:
- Environment lifecycle issues → Environment (A1-A4)
- Deployment or version issues → Deployment (B1-B3)
- Health status problems → Health (C1-C3)
- Configuration or permissions → Configuration (D1-D3)
- Scaling or load balancer → Scaling (E1-E2)
- Docker-specific issues → Docker (F1-F2)
- Worker tier or SQS → Worker (G1-G2)
- Logging or monitoring → Logging (H1-H2)

If classified: switch to the specific SOP immediately.
If unclassified: continue to Phase 3.

## Phase 3 — Deep Investigation

MUST:
- Check CloudFormation stack events for resource-level failures
- Verify IAM roles (instance profile and service role) are correctly configured
- Check platform version status and compatibility
- Review application version and source bundle

SHOULD:
- Check for recent configuration changes that may have caused the issue
- Verify VPC, subnet, and security group configuration
- Check for service quota limits

MAY:
- Check AWS Health Dashboard for Elastic Beanstalk service events
- Review account-level Beanstalk attributes: `aws elasticbeanstalk describe-account-attributes`

## Phase 4 — Report

MUST:
- State the investigation path taken
- State root cause if identified, or "unclassified" with best hypothesis
- List all evidence collected
- Recommend next steps

## Output Format

```yaml
root_cause: "<identified_cause OR unclassified>"
failure_domain: "<environment|deployment|health|configuration|scaling|docker|worker|logging|unknown>"
investigation_path: "environment status → health → events → configuration → <domain_classification>"
evidence:
  - type: environment_status
    content: "<environment status summary>"
  - type: health
    content: "<health status and causes>"
  - type: events
    content: "<recent error events>"
  - type: cloudformation
    content: "<stack status and events>"
severity: MEDIUM
mitigation:
  immediate: "<specific action if root cause found, or escalate>"
  long_term: "Implement monitoring for the identified failure pattern"
```

---

## Safety Ratings

| Aspect | Rating |
|--------|--------|
| Service Impact | HIGH � Elastic Beanstalk manages full application stacks including compute, networking, and load balancing |
| Data Risk | MEDIUM � Environment changes can affect application availability and data connections |
| Recovery Complexity | MEDIUM � Environments can be rebuilt but require correct configuration and deployment artifacts |
| Blast Radius | HIGH � Environment-level changes affect all instances and the application endpoint |

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

## Phase 3 � Rollback

- If deployment failed: use ws elasticbeanstalk abort-environment-update then redeploy last known good version
- If environment configuration change caused issues: apply previous saved configuration via update-environment
- If platform update broke the application: rebuild environment on previous platform version
- If auto-scaling changes caused instability: revert min/max/desired counts to previous values
- If .ebextensions changes caused failures: redeploy previous application version with working extensions
- Document all rollback actions with environment ID, version label, and configuration template name
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
