---
title: "B2 — Network Image Pull Failures"
description: "Diagnose image pull failures caused by network/VPC connectivity issues"
status: active
severity: HIGH
triggers:
  - "CannotPullContainerError"
  - "i/o timeout"
  - "TLS handshake timeout"
  - "dial tcp: lookup"
owner: devops-agent
objective: "Fix network connectivity for ECR/registry image pulls"
context: "Fargate tasks need internet access (NAT gateway) or VPC endpoints (ECR API + ECR DKR + S3) to pull images. EC2 launch type needs the container instance to have internet or VPC endpoints."
---

## Phase 1 — Triage

MUST:
- Check if the task is in a private subnet without NAT or VPC endpoints
- For Fargate: verify assignPublicIp setting and subnet type
- Check for VPC endpoints: `aws ec2 describe-vpc-endpoints --filters Name=vpc-id,Values=<vpc-id>`
- Need: com.amazonaws.region.ecr.api, com.amazonaws.region.ecr.dkr, com.amazonaws.region.s3

SHOULD:
- Check security group on the task allows outbound HTTPS (443)
- Verify NACL allows outbound 443 and inbound ephemeral ports
- Check if DNS resolution works in the VPC

MAY:
- Test connectivity from a bastion in the same subnet
- Check if the registry is a third-party registry (Docker Hub, etc.) requiring different endpoints

## Common Issues

- symptoms: "Fargate task in private subnet, no NAT, no VPC endpoints"
  diagnosis: "No path to ECR. Fargate in private subnet needs NAT or VPC endpoints."
  resolution: "Add NAT gateway or create VPC endpoints for ECR (api + dkr) and S3."

- symptoms: "VPC endpoints exist but image pull still fails"
  diagnosis: "Security group on VPC endpoint doesn't allow inbound 443 from task subnet."
  resolution: "Add inbound TCP 443 from the VPC CIDR to the endpoint security group."


## Safety Ratings

```
safety_ratings:
  - "Inspect/Describe resources: GREEN - Read-only API calls, no state change"
  - "Deploy changes: YELLOW - Affects live traffic, use canary/staged deployment"
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
2. If deployment is stuck, force a new deployment with the last known good task definition: `aws ecs update-service --cluster <cluster> --service <service> --task-definition <last-good> --force-new-deployment`
3. If network configuration was changed, update service with original network settings: `aws ecs update-service --cluster <cluster> --service <service> --network-configuration <original>`
## Output Format

```yaml
root_cause: "network_image_pull — <detail>"
evidence:
  - type: vpc
    content: "<subnet, NAT, VPC endpoint configuration>"
severity: HIGH
mitigation:
  immediate: "Add NAT gateway or VPC endpoints for ECR"
  long_term: "Standardize VPC endpoint deployment for ECS clusters"
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
