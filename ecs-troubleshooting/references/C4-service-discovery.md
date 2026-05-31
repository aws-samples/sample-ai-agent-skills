---
title: "C4 — Service Discovery Issues"
description: "Diagnose ECS service discovery (Cloud Map) failures"
status: active
severity: MEDIUM
triggers:
  - "Service discovery not resolving"
  - "Cloud Map registration failed"
  - "DNS SRV record not found"
owner: devops-agent
objective: "Fix service discovery registration and DNS resolution"
context: "ECS service discovery uses AWS Cloud Map. Tasks register DNS records (A or SRV) when they start. Health checks (Cloud Map or Route 53) determine if the record is returned. Cloud Map health checks are separate from ELB health checks."
---

## Phase 1 — Triage

MUST:
- Check service discovery configuration on the ECS service
- Verify Cloud Map namespace and service exist: `aws servicediscovery list-services`
- Check registered instances: `aws servicediscovery list-instances --service-id <id>`
- Test DNS resolution of the service discovery name

SHOULD:
- Check Cloud Map health check configuration (custom health check vs Route 53)
- Verify tasks are registering successfully (check service events)
- Check if deregistration delay is causing stale records

MAY:
- Check Route 53 private hosted zone for the namespace
- Verify the VPC is associated with the private hosted zone

## Common Issues

- symptoms: "DNS resolves but returns unhealthy instance IPs"
  diagnosis: "Cloud Map health check failing but DNS TTL hasn't expired."
  resolution: "Fix health check or reduce DNS TTL. Check health check configuration."

- symptoms: "Service discovery name doesn't resolve"
  diagnosis: "Namespace not associated with the VPC, or no healthy instances registered."
  resolution: "Verify VPC association and check that tasks are registering and passing health checks."


## Safety Ratings

```
safety_ratings:
  - "Inspect/Describe resources: GREEN - Read-only API calls, no state change"
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
3. If health check settings were modified, restore original target group health check configuration
## Output Format

```yaml
root_cause: "service_discovery — <detail>"
evidence:
  - type: cloud_map
    content: "<namespace, service, and registration state>"
severity: MEDIUM
mitigation:
  immediate: "Fix health checks or VPC association"
  long_term: "Monitor service discovery health, use appropriate TTL values"
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
