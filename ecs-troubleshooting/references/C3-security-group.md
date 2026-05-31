---
title: "C3 — Security Group Issues"
description: "Diagnose ECS task connectivity failures caused by security group misconfiguration"
status: active
severity: HIGH
triggers:
  - "Connection refused"
  - "Connection timed out"
  - "Task can't reach service"
owner: devops-agent
objective: "Fix security group rules for ECS task connectivity"
context: "In awsvpc mode, each task gets its own ENI and security group. The task SG must allow inbound from the load balancer SG (if using ALB/NLB) and outbound to required services. Bridge mode uses the container instance's SG."
---

## Phase 1 — Triage

MUST:
- Identify the task's security group (from awsvpcConfiguration or container instance)
- Check inbound rules: does the SG allow traffic from the load balancer or clients?
- Check outbound rules: does the SG allow traffic to downstream services?
- Verify the load balancer SG allows outbound to the task SG

SHOULD:
- Check if the task needs to reach AWS services (ECR, Secrets Manager, CloudWatch)
- Verify outbound HTTPS (443) is allowed for AWS API calls
- Check if the SG was recently modified

MAY:
- Use VPC flow logs to confirm SG is the blocking point
- Check if NACLs are also blocking

## Common Issues

- symptoms: "ALB health checks failing, task is healthy"
  diagnosis: "Task SG doesn't allow inbound from ALB SG on the health check port."
  resolution: "Add inbound rule: TCP/<port> from ALB security group."

- symptoms: "Task can't push logs to CloudWatch"
  diagnosis: "Task SG outbound doesn't allow HTTPS (443) to CloudWatch endpoint."
  resolution: "Add outbound rule for TCP 443 or use VPC endpoint for CloudWatch."


## Safety Ratings

```
safety_ratings:
  - "Inspect/Describe resources: GREEN - Read-only API calls, no state change"
  - "Review logs and metrics: GREEN - Read-only observability data access"
  - "Network configuration changes: YELLOW - May affect connectivity"
  - "Certificate/TLS changes: RED - May cause downtime if misconfigured"
```

## Escalation Conditions

- Service handles production traffic or is business-critical
- Fix requires modifying the task definition for a running service
- Resolution involves changing capacity provider or cluster configuration
- Multiple services affected suggesting cluster-level or account-level issue
- Deployment rollback has been triggered by the circuit breaker

## Data Sensitivity

HIGH - ECS task definitions may contain secrets ARNs, environment variables with credentials, and sensitive configuration. Container logs may capture application data including PII. Service Connect and service discovery configurations may expose internal architecture.

## Prohibited Actions

- NEVER suggest stopping all tasks in a service to fix a deployment issue - this causes a full outage
- NEVER suggest removing or modifying the task execution role while tasks are running
- NEVER recommend deleting a service without first scaling desired count to 0 and draining connections
- NEVER suggest modifying a running task definition - create a new revision instead
- NEVER recommend disabling the deployment circuit breaker on a production service

## Phase 3 - Rollback

1. If task definition was updated, roll back to previous revision: `aws ecs update-service --cluster <cluster> --service <service> --task-definition <previous-revision>`
2. If service desired count was changed, restore original: `aws ecs update-service --cluster <cluster> --service <service> --desired-count <original>`
3. If network configuration was changed, update service with original network settings: `aws ecs update-service --cluster <cluster> --service <service> --network-configuration <original>`
4. If health check settings were modified, restore original target group health check configuration
5. If log configuration was changed, create a new task definition revision with the original log driver settings
## Output Format

```yaml
root_cause: "security_group — <missing rule>"
evidence:
  - type: security_group
    content: "<SG ID and missing rule detail>"
severity: HIGH
mitigation:
  immediate: "Add the required SG rule"
  long_term: "Use IaC for SG management, reference SGs by ID for ALB→task rules"
```

## Escalation Conditions

escalation_conditions:
  - "Remediation requires modifying IAM policies in a production account"
  - "Remediation requires disabling a security control even temporarily"
  - "Root cause cannot be identified after 3 hypothesis pivots"
  - "Blast radius affects more than one account or region"
  - "Issue involves potential data loss or exposure"

## Data Sensitivity

data_sensitivity:
  - command: "describe-tasks"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"
  - command: "describe-services"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"
  - command: "execute-command"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"

## Prohibited Actions

prohibited_actions:
  - "NEVER suggest running containers as root to fix permission issues"
  - "NEVER suggest disabling task networking"
  - "NEVER remove IAM task role to troubleshoot"
