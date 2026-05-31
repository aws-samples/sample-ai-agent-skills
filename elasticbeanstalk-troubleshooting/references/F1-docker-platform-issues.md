---
title: "F1 â€” Docker Platform Issues"
description: "Diagnose Docker platform deployment and runtime problems"
status: active
severity: MEDIUM
triggers:
  - "docker deployment failed"
  - "Dockerfile error"
  - "docker build failed"
  - "Dockerrun.aws.json"
  - "container not starting"
owner: devops-agent
objective: "Resolve Docker platform deployment and runtime issues"
context: "Elastic Beanstalk Docker platform supports single-container Docker (Dockerfile or Dockerrun.aws.json v1) and ECS-managed multi-container Docker (Dockerrun.aws.json v3). Docker platform configuration differs significantly from non-Docker platforms. Image pull failures, Dockerfile errors, and port mapping issues are common."
---

## Phase 1 â€” Triage

MUST:
- Check environment status and events: `aws elasticbeanstalk describe-events --environment-name <env-name> --severity ERROR --max-records 20`
- Check environment health: `aws elasticbeanstalk describe-environment-health --environment-name <env-name> --attribute-names All`
- Verify platform is Docker: `aws elasticbeanstalk describe-environments --environment-names <env-name> --query 'Environments[0].PlatformArn'`
- Retrieve logs for Docker errors: `aws elasticbeanstalk request-environment-info --environment-name <env-name> --info-type tail` then `aws elasticbeanstalk retrieve-environment-info --environment-name <env-name> --info-type tail`

SHOULD:
- Check if Dockerrun.aws.json or Dockerfile is present in the source bundle
- Verify ECR image accessibility: `aws ecr describe-images --repository-name <repo-name> --image-ids imageTag=<tag>`
- Check instance profile has ECR pull permissions: `aws iam list-attached-role-policies --role-name <instance-role>`

MAY:
- Check Docker daemon logs on instances via log bundle
- Verify port mapping matches the application's listening port

## Phase 2 â€” Remediate

MUST:
- Fix Dockerfile syntax or build errors
- Ensure Dockerrun.aws.json format matches the platform version (v1 for single, v3 for ECS)
- Grant ECR pull permissions to the instance profile if using private ECR images

SHOULD:
- Expose the correct port in the Dockerfile (EXPOSE directive)
- Configure the container port in Beanstalk to match the application port
- Test Docker builds locally before deploying to Beanstalk

MAY:
- Use ECR lifecycle policies to manage image retention
- Configure Docker compose for local development parity

## Common Issues

- symptoms: "Docker image pull failed â€” access denied"
  diagnosis: "Instance profile lacks ECR permissions or the image repository is in a different account/region."
  resolution: "Add AmazonEC2ContainerRegistryReadOnly policy to the instance role. For cross-account, configure ECR repository policy."

- symptoms: "Container starts but health check fails"
  diagnosis: "Container port does not match the Beanstalk health check port or the application is not ready."
  resolution: "Ensure EXPOSE port in Dockerfile matches the Port option in aws:elasticbeanstalk:application:environment."

- symptoms: "Dockerfile build fails during deployment"
  diagnosis: "Dockerfile has syntax errors, missing dependencies, or network issues during build."
  resolution: "Test the Dockerfile locally. Ensure build dependencies are available. Check NAT gateway for private subnets."

## Output Format

```yaml
root_cause: "docker_platform_issue â€” <specific_cause>"
evidence:
  - type: events
    content: "<deployment error events>"
  - type: logs
    content: "<Docker daemon and application logs>"
  - type: configuration
    content: "<Dockerrun.aws.json or Dockerfile details>"
severity: MEDIUM
mitigation:
  immediate: "Fix Docker configuration and redeploy"
  long_term: "Test Docker builds locally and maintain ECR permissions"
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
