---
title: "B3 â€” Immutable vs Rolling Deployment Issues"
description: "Diagnose issues specific to immutable and rolling deployment strategies"
status: active
severity: MEDIUM
triggers:
  - "immutable deployment"
  - "rolling deployment"
  - "rolling with additional batch"
  - "traffic splitting"
  - "deployment strategy"
owner: devops-agent
objective: "Resolve deployment strategy-specific issues and select the appropriate deployment policy"
context: "Each deployment policy has different behavior: all-at-once (fastest, downtime), rolling (batch updates, reduced capacity), rolling with additional batch (maintains capacity), immutable (new ASG, safe rollback), traffic splitting (canary). Understanding the differences is critical for diagnosis."
---

## Phase 1 â€” Triage

MUST:
- Check current deployment policy: `aws elasticbeanstalk describe-configuration-settings --application-name <app-name> --environment-name <env-name> --query 'ConfigurationSettings[0].OptionSettings[?Namespace==`aws:elasticbeanstalk:command`]'`
- Check environment health during deployment: `aws elasticbeanstalk describe-environment-health --environment-name <env-name> --attribute-names All`
- Check per-instance health: `aws elasticbeanstalk describe-instances-health --environment-name <env-name>`
- List ASGs (check for temporary immutable ASG): `aws autoscaling describe-auto-scaling-groups --query 'AutoScalingGroups[?contains(Tags[?Key==`elasticbeanstalk:environment-name`].Value, `<env-name>`)]'`
- List recent events: `aws elasticbeanstalk describe-events --environment-name <env-name> --max-records 20`

SHOULD:
- Check batch size and percentage settings for rolling deployments
- Verify health check configuration matches application startup time
- Check for orphaned temporary ASGs from failed immutable deployments

MAY:
- Check ELB target group health: `aws elbv2 describe-target-health --target-group-arn <tg-arn>`
- Review CloudWatch metrics for deployment duration trends

## Phase 2 â€” Remediate

MUST:
- For stuck immutable deployments, check and clean up the temporary ASG
- For rolling deployment failures, ensure batch size allows enough healthy instances
- Adjust health check grace period to accommodate application startup time

SHOULD:
- Use immutable for production environments requiring safe rollback
- Use rolling with additional batch to maintain full capacity during updates
- Set appropriate batch size (percentage or fixed) for rolling deployments

MAY:
- Switch to traffic splitting for canary-style deployments
- Use blue/green (CNAME swap) for the safest deployment with instant rollback

## Common Issues

- symptoms: "Immutable deployment stuck â€” temporary ASG instances not becoming healthy"
  diagnosis: "New instances fail health checks. The temporary ASG will be terminated after timeout."
  resolution: "Check application startup logs on new instances. Fix health check or application issues."

- symptoms: "Rolling deployment causes service degradation"
  diagnosis: "Batch size is too large, leaving insufficient healthy instances to handle traffic."
  resolution: "Reduce batch size percentage or use rolling with additional batch to maintain capacity."

- symptoms: "Orphaned temporary ASG after failed immutable deployment"
  diagnosis: "Cleanup of the temporary ASG failed after an immutable deployment failure."
  resolution: "Manually terminate the orphaned ASG instances and delete the ASG."

## Output Format

```yaml
root_cause: "deployment_strategy_issue â€” <specific_cause>"
evidence:
  - type: deployment_config
    content: "<deployment policy and batch settings>"
  - type: instance_health
    content: "<per-instance health during deployment>"
  - type: asg
    content: "<ASG status including temporary ASGs>"
severity: MEDIUM
mitigation:
  immediate: "Fix the deployment strategy configuration or clean up stuck resources"
  long_term: "Select appropriate deployment policy based on availability and rollback requirements"
```

---

## Safety Ratings

| Aspect | Rating |
|--------|--------|
| Service Impact | HIGH — Elastic Beanstalk manages full application stacks including compute, networking, and load balancing |
| Data Risk | MEDIUM — Environment changes can affect application availability and data connections |
| Recovery Complexity | MEDIUM — Environments can be rebuilt but require correct configuration and deployment artifacts |
| Blast Radius | HIGH — Environment-level changes affect all instances and the application endpoint |

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

## Phase 3 — Rollback

- If deployment failed: use ws elasticbeanstalk abort-environment-update then redeploy last known good version
- If environment configuration change caused issues: apply previous saved configuration via update-environment
- If platform update broke the application: rebuild environment on previous platform version
- If auto-scaling changes caused instability: revert min/max/desired counts to previous values
- If .ebextensions changes caused failures: redeploy previous application version with working extensions
- Document all rollback actions with environment ID, version label, and configuration template name
## Safety Ratings

safety_ratings:
  - "Phase 1 triage commands (describe/get/list): GREEN â€” read-only"
  - "Phase 2 configuration changes: YELLOW â€” state-changing but recoverable"
  - "Phase 2 resource deletion or security changes: RED â€” destructive or irreversible"

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
