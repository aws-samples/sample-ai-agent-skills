---
title: "H1 â€” Log Retrieval Issues"
description: "Diagnose problems retrieving logs from Elastic Beanstalk environments"
status: active
severity: LOW
triggers:
  - "cannot retrieve logs"
  - "log retrieval failed"
  - "tail logs empty"
  - "bundle logs"
  - "request-environment-info"
owner: devops-agent
objective: "Successfully retrieve application and system logs from the environment"
context: "Beanstalk supports two log retrieval methods: tail logs (last 100 lines from key log files) and bundle logs (full log archive uploaded to S3). Logs are retrieved via request-environment-info followed by retrieve-environment-info. Log retrieval requires the health agent and proper S3 permissions."
---

## Phase 1 â€” Triage

MUST:
- Request tail logs: `aws elasticbeanstalk request-environment-info --environment-name <env-name> --info-type tail`
- Retrieve tail logs (wait a few seconds): `aws elasticbeanstalk retrieve-environment-info --environment-name <env-name> --info-type tail`
- Check environment status: `aws elasticbeanstalk describe-environments --environment-names <env-name>`
- Verify instances are running: `aws elasticbeanstalk describe-instances-health --environment-name <env-name>`

SHOULD:
- Request full bundle logs: `aws elasticbeanstalk request-environment-info --environment-name <env-name> --info-type bundle`
- Check the S3 bucket for log bundles: `aws s3 ls s3://elasticbeanstalk-<region>-<account-id>/resources/environments/logs/`
- Verify instance profile has S3 write permissions

MAY:
- SSH to instances directly to check logs if retrieval fails
- Check if the health agent is running on instances

## Phase 2 â€” Remediate

MUST:
- Ensure instances are running and reachable
- Verify the instance profile has S3 permissions for log upload
- Wait sufficient time between request and retrieve calls

SHOULD:
- Enable CloudWatch Logs streaming for persistent log access
- Configure log rotation to prevent disk space issues on instances

MAY:
- Use SSM Session Manager to access instances without SSH keys
- Configure custom log file paths in the Beanstalk log configuration

## Common Issues

- symptoms: "retrieve-environment-info returns empty results"
  diagnosis: "Logs were not yet uploaded. The request is asynchronous."
  resolution: "Wait 10-30 seconds after request-environment-info before calling retrieve."

- symptoms: "Log retrieval fails â€” S3 permission denied"
  diagnosis: "Instance profile lacks S3 write permissions to the Beanstalk bucket."
  resolution: "Add S3 write permissions to the instance profile for the elasticbeanstalk-* bucket."

- symptoms: "Only partial logs returned â€” missing application logs"
  diagnosis: "Application logs are not in the default log paths monitored by Beanstalk."
  resolution: "Configure custom log paths in .ebextensions or use CloudWatch Logs agent."

## Output Format

```yaml
root_cause: "log_retrieval_issue â€” <specific_cause>"
evidence:
  - type: environment_status
    content: "<environment and instance status>"
  - type: logs
    content: "<retrieved log content or error>"
severity: LOW
mitigation:
  immediate: "Fix log retrieval permissions or wait for async upload"
  long_term: "Enable CloudWatch Logs streaming for reliable log access"
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
