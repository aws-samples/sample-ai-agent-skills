---
title: "C1 — No Internet Access (Fargate)"
description: "Diagnose Fargate task internet connectivity failures"
status: active
severity: HIGH
triggers:
  - "Can't reach external API"
  - "Connection timed out from Fargate"
  - "No internet in Fargate task"
owner: devops-agent
objective: "Restore internet access for Fargate tasks"
context: "Fargate tasks in private subnets need NAT gateway for internet. In public subnets, they need assignPublicIp=ENABLED. VPC endpoints can provide AWS service access without internet."
---

## Phase 1 — Triage

MUST:
- Check task networking: awsvpcConfiguration in the service/run-task config
- Determine if subnets are public or private (check route table for 0.0.0.0/0 → IGW vs NAT)
- If public subnet: verify assignPublicIp=ENABLED
- If private subnet: verify NAT gateway exists and route table has 0.0.0.0/0 → NAT

SHOULD:
- Check security group outbound rules allow the required traffic
- Verify NACL allows outbound traffic and inbound return traffic
- Check if VPC endpoints could replace internet access for AWS services

MAY:
- Test connectivity from a bastion in the same subnet
- Check DNS resolution from the task

## Common Issues

- symptoms: "Fargate task in private subnet, no NAT gateway"
  diagnosis: "Private subnet has no route to internet. Fargate tasks don't get public IPs in private subnets."
  resolution: "Add NAT gateway in public subnet, add 0.0.0.0/0 → NAT route to private subnet."

- symptoms: "Fargate task in public subnet but assignPublicIp=DISABLED"
  diagnosis: "Public subnet tasks need assignPublicIp=ENABLED to get a public IP."
  resolution: "Set assignPublicIp=ENABLED in awsvpcConfiguration."


## Safety Ratings

```
safety_ratings:
  - "Inspect/Describe resources: GREEN - Read-only API calls, no state change"
  - "Delete/Remove resources: RED - Potentially irreversible, requires confirmation"
  - "Network configuration changes: YELLOW - May affect connectivity"
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

1. If service desired count was changed, restore original: `aws ecs update-service --cluster <cluster> --service <service> --desired-count <original>`
2. If network configuration was changed, update service with original network settings: `aws ecs update-service --cluster <cluster> --service <service> --network-configuration <original>`
## Output Format

```yaml
root_cause: "no_internet_fargate — <detail>"
evidence:
  - type: vpc
    content: "<subnet type, NAT/IGW configuration, assignPublicIp setting>"
severity: HIGH
mitigation:
  immediate: "Add NAT gateway or enable assignPublicIp"
  long_term: "Use VPC endpoints for AWS services, standardize networking for ECS"
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
