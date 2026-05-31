---
title: "H2 — Agent Deployment Issues"
description: "Diagnose GuardDuty security agent deployment failures"
status: active
severity: HIGH
triggers:
  - "agent deployment"
  - "agent not installed"
  - "agent failed"
  - "guardduty agent"
  - "security agent"
owner: devops-agent
objective: "Identify and resolve GuardDuty security agent deployment failures across EC2, EKS, and ECS"
context: "The GuardDuty security agent is required for runtime monitoring. Deployment methods vary by resource type: SSM for EC2, EKS add-on for EKS, and sidecar for ECS Fargate. Auto-management simplifies deployment but requires proper prerequisites."
---

## Phase 1 — Triage

MUST:
- Check coverage for unhealthy resources: `aws guardduty list-coverage --detector-id <detector-id> --filter-criteria '{"FilterCriterion":[{"CriterionKey":"COVERAGE_STATUS","FilterCondition":{"EqualsValue":"UNHEALTHY"}}]}'`
- Check the coverage issue reason for each unhealthy resource
- For EC2: verify SSM agent status: `aws ssm describe-instance-information --filters Key=InstanceIds,Values=<instance-id>`
- For EKS: check add-on status: `aws eks describe-addon --cluster-name <cluster> --addon-name aws-guardduty-agent`
- Verify auto-management settings in the detector

SHOULD:
- For EC2: check instance profile has SSM permissions: `aws iam get-instance-profile --instance-profile-name <profile>`
- For EKS: verify node IAM role permissions
- Check for VPC endpoint connectivity if instances are in private subnets

MAY:
- Check SSM command history for agent installation: `aws ssm list-commands --filters Key=DocumentName,Values=AWS-ConfigureAWSPackage`
- Review EKS add-on pod logs for deployment issues
- Check ECS task definition for sidecar container configuration

## Phase 2 — Remediate

MUST:
- For EC2: ensure SSM agent is running and instance profile has SSM permissions
- For EKS: install or update the GuardDuty EKS add-on: `aws eks create-addon --cluster-name <cluster> --addon-name aws-guardduty-agent`
- For ECS: update task definitions to include the GuardDuty sidecar
- Verify coverage status improves after deployment

SHOULD:
- Enable auto-management to simplify future deployments
- Ensure VPC endpoints exist for SSM and GuardDuty in private subnets
- Monitor agent health after deployment

MAY:
- Create automation for agent deployment on new instances
- Set up CloudWatch alarms for agent health status

## Common Issues

- symptoms: "EC2 agent deployment fails — SSM not reachable"
  diagnosis: "The EC2 instance does not have SSM agent running or lacks network connectivity to SSM endpoints."
  resolution: "Install/start SSM agent, add SSM permissions to instance profile, and ensure VPC endpoint or internet access."

- symptoms: "EKS add-on shows DEGRADED status"
  diagnosis: "The GuardDuty agent pods are failing on EKS nodes due to resource constraints or permission issues."
  resolution: "Check agent pod status with kubectl. Verify node IAM role has required permissions. Check resource limits."

- symptoms: "Auto-management enabled but agent not deploying on new instances"
  diagnosis: "New instances may lack SSM agent or instance profile, or there is a delay in auto-management."
  resolution: "Ensure AMIs include SSM agent and instance profiles have SSM permissions. Allow time for auto-management."

## Safety Ratings

| Phase 2 Action | Rating | Rationale |
|---|---|---|
| Install SSM agent | YELLOW | Agent installation; can be removed |
| Install EKS add-on | YELLOW | Add-on deployment; can be removed |
| Update ECS task definition | YELLOW | Task definition change; reversible |
| Enable auto-management | YELLOW | Configuration change; reversible |

## Escalation Conditions

- Agent deployment to production resources
- Detector configuration changes
- EKS add-on changes on production clusters

## Data Sensitivity

| Triage Command | Sensitivity | Rationale |
|---|---|---|
| `list-coverage` (UNHEALTHY) | MEDIUM | Resource details and failure reasons |
| `describe-instance-information` | MEDIUM | Instance details |
| `describe-addon` | LOW | Add-on status |

## Prohibited Actions

- NEVER suggest disabling GuardDuty detector
- NEVER suggest suppressing all findings
- NEVER suggest disabling S3/EKS/Lambda protection

## Phase 3 — Rollback

| Remediation | Rollback Step |
|---|---|
| SSM agent installation | Uninstall SSM package |
| EKS add-on installation | Delete via `delete-addon` |
| ECS task definition update | Revert to previous task definition revision |
| Auto-management enablement | Disable auto-management |

## Output Format

```yaml
root_cause: "agent_deployment — <specific_cause>"
evidence:
  - type: coverage
    content: "<unhealthy resource coverage details>"
  - type: agent_status
    content: "<agent deployment status per resource>"
  - type: prerequisites
    content: "<SSM/EKS add-on/ECS sidecar status>"
severity: HIGH
mitigation:
  immediate: "Fix prerequisites and deploy the agent"
  long_term: "Enable auto-management and ensure all new resources get the agent"
```

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
  - command: "get-findings"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"
  - command: "list-detectors"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"
  - command: "get-threat-intel-set"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"

## Prohibited Actions

prohibited_actions:
  - "NEVER suggest disabling GuardDuty to stop alerts"
  - "NEVER suggest archiving findings without investigation"
  - "NEVER expose finding details in public logs"
