---
title: "E2 â€” Load Balancer Configuration Issues"
description: "Diagnose ELB/ALB configuration problems in Elastic Beanstalk environments"
status: active
severity: MEDIUM
triggers:
  - "load balancer error"
  - "ALB configuration"
  - "ELB issues"
  - "listener configuration"
  - "SSL certificate"
  - "502 bad gateway"
owner: devops-agent
objective: "Configure the load balancer correctly for the Beanstalk environment"
context: "Beanstalk web server environments use an ELB (Classic, ALB, or NLB). Configuration includes listeners, health checks, SSL certificates, security groups, and stickiness. ALB is the default for new environments. Load balancer type cannot be changed after creation."
---

## Phase 1 â€” Triage

MUST:
- Check load balancer type and config: `aws elasticbeanstalk describe-configuration-settings --application-name <app-name> --environment-name <env-name> --query 'ConfigurationSettings[0].OptionSettings[?contains(Namespace, `aws:elbv2`) || contains(Namespace, `aws:elb`)]'`
- Check environment resources for LB details: `aws elasticbeanstalk describe-environment-resources --environment-name <env-name>`
- Describe the load balancer: `aws elbv2 describe-load-balancers --load-balancer-arns <lb-arn>`
- Check listeners: `aws elbv2 describe-listeners --load-balancer-arn <lb-arn>`
- Check target health: `aws elbv2 describe-target-health --target-group-arn <tg-arn>`

SHOULD:
- Check security groups on the load balancer: `aws ec2 describe-security-groups --group-ids <sg-ids>`
- Verify SSL certificate is valid: `aws acm describe-certificate --certificate-arn <cert-arn>`
- Check for 5xx errors in ELB access logs

MAY:
- Check ELB CloudWatch metrics: `aws cloudwatch get-metric-statistics --namespace AWS/ApplicationELB --metric-name HTTPCode_ELB_5XX_Count --dimensions Name=LoadBalancer,Value=<lb-id> --start-time <start> --end-time <end> --period 300 --statistics Sum`

## Phase 2 â€” Remediate

MUST:
- Fix listener configuration (port, protocol, certificate)
- Ensure security groups allow traffic on the listener ports
- Verify instances are registered and healthy in the target group

SHOULD:
- Configure HTTPS listener with a valid ACM certificate
- Set up HTTP to HTTPS redirect
- Enable access logging for troubleshooting

MAY:
- Configure sticky sessions if the application requires session affinity
- Set up connection draining for graceful instance removal

## Common Issues

- symptoms: "502 Bad Gateway from the load balancer"
  diagnosis: "No healthy targets or application not responding on the expected port."
  resolution: "Check target health. Verify application listens on the port configured in the target group."

- symptoms: "HTTPS not working â€” certificate error"
  diagnosis: "SSL certificate not configured on the listener or certificate is invalid/expired."
  resolution: "Attach a valid ACM certificate to the HTTPS listener."

- symptoms: "Cannot change load balancer type"
  diagnosis: "Load balancer type (Classic, ALB, NLB) is set at environment creation and cannot be changed."
  resolution: "Create a new environment with the desired LB type and perform blue/green swap."

## Output Format

```yaml
root_cause: "load_balancer_issue â€” <specific_cause>"
evidence:
  - type: load_balancer
    content: "<LB type, listeners, and configuration>"
  - type: target_health
    content: "<target group health status>"
  - type: security_groups
    content: "<security group rules>"
severity: MEDIUM
mitigation:
  immediate: "Fix load balancer configuration"
  long_term: "Document LB requirements and validate during environment creation"
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
