---
title: "B2 — ECS/EKS on Wavelength Issues"
description: "Diagnose container orchestration problems in Wavelength Zones"
status: active
severity: HIGH
triggers:
  - "ECS wavelength"
  - "EKS wavelength"
  - "container wavelength"
  - "kubernetes wavelength"
owner: devops-agent
objective: "Resolve ECS task and EKS pod deployment issues in Wavelength Zones"
context: "ECS and EKS can run workloads in Wavelength Zones using EC2 instances. Fargate is NOT supported in Wavelength Zones. Worker nodes must use Wavelength-compatible instance types and be in Wavelength subnets."
---

## Phase 1 — Triage

MUST:
- For ECS — list services: `aws ecs list-services --cluster <cluster-name>`
- For ECS — check service events: `aws ecs describe-services --cluster <cluster-name> --services <service-name> --query 'services[*].events[:5]'`
- For EKS — check node status: `aws eks describe-nodegroup --cluster-name <cluster> --nodegroup-name <nodegroup>`
- Verify worker nodes are in Wavelength subnets: `aws ec2 describe-instances --filters "Name=availability-zone,Values=<wavelength-zone>" "Name=tag:kubernetes.io/cluster/<cluster>,Values=owned"`

SHOULD:
- Check container instance or node resources (CPU, memory)
- Verify task/pod networking configuration
- Check image pull access from Wavelength (ECR access routes through Region)

MAY:
- Review ECS Container Insights or EKS CloudWatch metrics
- Check pod logs for application-level errors

## Phase 2 — Remediate

MUST:
- If Fargate requested: switch to EC2 launch type — Fargate not supported in Wavelength
- If node launch fails: verify instance type is Wavelength-compatible
- If image pull fails: ensure routing to ECR through Region VPC

SHOULD:
- Configure EKS managed node groups with Wavelength-compatible instance types
- Use ECS capacity providers for Wavelength EC2 instances
- Ensure service discovery works across Wavelength and Region

MAY:
- Implement pod affinity/anti-affinity for Wavelength nodes
- Set up cross-zone load balancing between Wavelength and Region

## Common Issues

- symptoms: "ECS tasks cannot be placed in Wavelength"
  diagnosis: "No container instances in Wavelength subnets or Fargate requested."
  resolution: "Launch EC2 container instances in Wavelength subnets. Use EC2 launch type."

- symptoms: "EKS pods stuck in Pending on Wavelength nodes"
  diagnosis: "Insufficient resources on Wavelength worker nodes."
  resolution: "Scale Wavelength node group or reduce pod resource requests."

## Output Format

```yaml
root_cause: "container_wavelength — <specific_cause>"
evidence:
  - type: service_events
    content: "<ECS/EKS events>"
  - type: node_status
    content: "<worker node status>"
severity: HIGH
mitigation:
  immediate: "Fix container placement or node configuration"
  long_term: "Implement Wavelength-specific container orchestration patterns"
```


## Safety Ratings

| Phase 2 Action | Rating | Rationale |
|---|---|---|
| Investigate and collect diagnostic data | GREEN | Read-only analysis; no state change |
| Modify configuration settings | YELLOW | Configuration change; reversible with backup |
| Restart or reboot components | YELLOW | Service interruption; recoverable |
| Delete or remove resources | RED | Potentially destructive; requires confirmation and backup |

## Escalation Conditions

- Production environment impact detected or imminent
- Multiple failed remediation attempts (>2 cycles without resolution)
- Security-related root cause identified (unauthorized access, data exposure)
- Issue requires cross-service or cross-account coordination
- Customer-facing service degradation exceeding SLA thresholds
- Regulatory or compliance implications discovered during investigation

## Data Sensitivity

- Edge deployment configurations reveal application architecture
- Carrier IP assignments expose public-facing endpoints
- Latency data may reveal user geographic distribution
- Classification: INTERNAL - edge infrastructure topology and carrier network details

## Prohibited Actions

- NEVER terminate EC2 instances in Wavelength zones serving live edge traffic
- NEVER modify carrier gateway configurations during peak usage
- NEVER change VPC subnet associations without confirming application connectivity
- NEVER remove carrier IP addresses assigned to production workloads

## Phase 3 -- Rollback

MUST:
- Document all changes made during Phase 2 remediation
- Verify rollback procedures exist before applying any fix
- Confirm service health after rollback using the same diagnostic steps from Phase 1
- Notify stakeholders if rollback is initiated in production

SHOULD:
- Maintain a timeline of all actions taken for post-incident review
- Capture before/after configuration snapshots
- Test rollback in non-production environment when possible
- Update runbook with lessons learned after incident resolution

MAY:
- Automate rollback procedures for frequently occurring issues
- Create CloudWatch alarms to detect regression after rollback

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
