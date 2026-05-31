---
title: "C3 — Health Check Failures"
description: "Diagnose and resolve CodeDeploy health check failures during deployment"
status: active
severity: HIGH
triggers:
  - "health check failed"
  - "unhealthy"
  - "minimum healthy hosts"
  - "ValidateService"
  - "ELB health"
owner: devops-agent
objective: "Identify why health checks are failing during deployment and resolve the issue"
context: "CodeDeploy health checks determine if instances are healthy after deployment. For EC2, the ValidateService lifecycle hook runs health checks. For ELB-integrated deployments, instances must pass ELB health checks to be considered healthy. The minimumHealthyHosts setting controls how many instances must remain healthy during deployment."
---

## Phase 1 — Triage

MUST:
- Get deployment target status: `aws codedeploy get-deployment-target --deployment-id <deployment-id> --target-id <instance-id>`
- Check deployment configuration: `aws codedeploy get-deployment-config --deployment-config-name <config-name>`
- Check if ELB is associated: `aws codedeploy get-deployment-group --application-name <app> --deployment-group-name <dg> --query 'deploymentGroupInfo.loadBalancerInfo'`
- If ELB associated: check target group health: `aws elbv2 describe-target-health --target-group-arn <tg-arn>`

SHOULD:
- Check ValidateService hook script output on the instance
- Verify the application is actually running after deployment
- Check application logs on the target instance for startup errors

MAY:
- Check ELB health check configuration (path, interval, thresholds): `aws elbv2 describe-target-groups --target-group-arns <tg-arn>`
- Review security group rules for health check traffic
- Check if the application port changed after deployment

## Phase 2 — Remediate

MUST:
- If ValidateService hook fails: fix the health check script (check URL, port, timeout)
- If ELB health check fails: verify the application is listening on the expected port and path
- If minimumHealthyHosts too restrictive: adjust the deployment configuration
- If application not starting: check application logs and fix startup issues

SHOULD:
- Ensure health check endpoints return 200 quickly (within health check timeout)
- Configure appropriate health check intervals and thresholds in the ELB
- Use a dedicated health check endpoint that validates application readiness

MAY:
- Implement a warm-up period before health checks start
- Use custom deployment configurations with appropriate healthy host percentages
- Add detailed health check logging for future debugging

## Common Issues

- symptoms: "Deployment stuck — minimumHealthyHosts constraint not met"
  diagnosis: "Too many instances are unhealthy, and the deployment cannot proceed without violating the minimum healthy hosts constraint."
  resolution: "Lower minimumHealthyHosts percentage or fix the health check failures on existing instances."

- symptoms: "ValidateService hook failed — connection refused"
  diagnosis: "Application is not running or not listening on the expected port after deployment."
  resolution: "Check ApplicationStart hook. Verify the application started successfully. Check port binding."

- symptoms: "ELB health check failing after deployment"
  diagnosis: "Application health check endpoint returns non-200 or times out."
  resolution: "Verify health check path exists and returns 200. Check security groups allow health check traffic from ELB."

- symptoms: "Instances deregistered from target group during deployment"
  diagnosis: "CodeDeploy deregisters instances from the ELB during deployment. If re-registration fails, instances stay out of service."
  resolution: "Check BlockTraffic and AllowTraffic lifecycle events. Verify instance re-registration succeeds."

## Output Format

```yaml
root_cause: "Health check failure — <specific_cause>"
evidence:
  - type: deployment_target
    content: "<instance health check status>"
  - type: elb_health
    content: "<target group health details>"
severity: HIGH
mitigation:
  immediate: "Fix health check failures and redeploy"
  long_term: "Implement robust health check endpoints, tune deployment configuration"
```


## Safety Ratings

| Phase 2 Action | safety_rating |
|---|---|
| If ValidateService hook fails: fix the health check script | YELLOW |
| If ELB health check fails: verify application is listening on expected port and path | GREEN |
| If minimumHealthyHosts too restrictive: adjust the deployment configuration | YELLOW |
| If application not starting: check application logs and fix startup issues | YELLOW |
| Ensure health check endpoints return 200 quickly | GREEN |
| Configure appropriate health check intervals and thresholds in the ELB | YELLOW |
| Use a dedicated health check endpoint | GREEN |
| Implement a warm-up period before health checks start | GREEN |
| Use custom deployment configurations | YELLOW |
| Add detailed health check logging | GREEN |

## Escalation Conditions

- Fix requires modifying production pipeline
- Fix requires changing deployment strategy

## Data Sensitivity

- **HIGH**: build logs (may contain secrets), artifact contents
- Health check configurations may reveal application endpoints and infrastructure topology

## Prohibited Actions

- NEVER suggest storing secrets in buildspec.yml
- NEVER suggest skipping approval stages in production pipelines

## Phase 3 — Rollback

1. If minimumHealthyHosts was changed, restore the previous deployment configuration
2. If ELB health check settings were modified, restore previous interval, threshold, and path settings
3. If the ValidateService hook script was changed, revert in source control and redeploy
4. If deployment configuration was changed, restore the previous configuration name in the deployment group
5. Verify rollback by deploying and confirming health checks pass with the restored configuration

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
