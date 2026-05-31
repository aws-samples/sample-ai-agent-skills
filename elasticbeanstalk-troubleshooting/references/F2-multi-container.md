---
title: "F2 â€” Multi-Container Docker Issues"
description: "Diagnose ECS-managed multi-container Docker platform problems"
status: active
severity: MEDIUM
triggers:
  - "multi-container docker"
  - "Dockerrun.aws.json v3"
  - "ECS task"
  - "multiple containers"
  - "docker compose"
owner: devops-agent
objective: "Resolve multi-container Docker deployment and orchestration issues"
context: "Multi-container Docker on Beanstalk uses Amazon ECS to manage containers. It requires Dockerrun.aws.json v3 (ECS task definition format). Each instance runs an ECS agent that manages container lifecycle. Issues often involve task definition errors, container linking, volume mounting, and ECS agent communication."
---

## Phase 1 â€” Triage

MUST:
- Check environment events: `aws elasticbeanstalk describe-events --environment-name <env-name> --severity ERROR --max-records 20`
- Check ECS cluster status: `aws ecs list-clusters`
- List ECS tasks: `aws ecs list-tasks --cluster <cluster-name>`
- Describe task failures: `aws ecs describe-tasks --cluster <cluster-name> --tasks <task-arns>`
- Retrieve logs: `aws elasticbeanstalk request-environment-info --environment-name <env-name> --info-type tail`

SHOULD:
- Check ECS task definition: `aws ecs describe-task-definition --task-definition <task-def>`
- Verify container images are accessible from the instances
- Check ECS agent status on instances

MAY:
- Check ECS service events: `aws ecs describe-services --cluster <cluster-name> --services <service-name>`
- Review container-level CloudWatch logs if configured

## Phase 2 â€” Remediate

MUST:
- Fix Dockerrun.aws.json v3 format to match ECS task definition schema
- Ensure all container images are accessible (ECR permissions, Docker Hub rate limits)
- Fix container port mappings and links

SHOULD:
- Test the task definition locally using `docker compose` or ECS local
- Configure proper container health checks in the task definition
- Set appropriate memory and CPU limits for each container

MAY:
- Use CloudWatch Logs for container-level logging
- Configure service discovery for inter-container communication

## Common Issues

- symptoms: "ECS task fails to start â€” essential container exited"
  diagnosis: "An essential container in the task definition crashed or failed to start."
  resolution: "Check container logs for the failing container. Fix the application error or configuration."

- symptoms: "Dockerrun.aws.json validation error"
  diagnosis: "The v3 format does not match the ECS task definition schema."
  resolution: "Validate the Dockerrun.aws.json against the ECS task definition JSON schema."

- symptoms: "Container cannot communicate with other containers"
  diagnosis: "Container links or network configuration is incorrect."
  resolution: "Use container links in the task definition or configure the awsvpc network mode."

## Output Format

```yaml
root_cause: "multi_container_issue â€” <specific_cause>"
evidence:
  - type: ecs_tasks
    content: "<ECS task status and failure reasons>"
  - type: task_definition
    content: "<task definition configuration>"
  - type: logs
    content: "<container logs>"
severity: MEDIUM
mitigation:
  immediate: "Fix task definition and container configuration"
  long_term: "Test multi-container setups locally before deploying"
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
