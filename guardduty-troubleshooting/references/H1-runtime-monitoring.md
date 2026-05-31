---
title: "H1 — Runtime Monitoring"
description: "Diagnose GuardDuty runtime monitoring configuration and coverage issues"
status: active
severity: HIGH
triggers:
  - "runtime monitoring"
  - "runtime findings"
  - "runtime coverage"
  - "process monitoring"
  - "container monitoring"
owner: devops-agent
objective: "Identify and resolve runtime monitoring configuration and coverage issues"
context: "Runtime monitoring detects OS-level and container-level threats on EC2, EKS, and ECS Fargate. It requires the GuardDuty security agent deployed on target resources. Coverage status shows which resources have the agent installed and healthy."
---

## Phase 1 — Triage

MUST:
- Check runtime monitoring feature status: `aws guardduty get-detector --detector-id <detector-id>` (check RUNTIME_MONITORING feature)
- Get coverage statistics: `aws guardduty get-coverage-statistics --detector-id <detector-id> --statistics-type COUNT_BY_COVERAGE_STATUS`
- List coverage details: `aws guardduty list-coverage --detector-id <detector-id>`
- Check for UNHEALTHY resources in coverage
- Verify the detector is ENABLED

SHOULD:
- Check runtime findings: `aws guardduty list-findings --detector-id <detector-id> --finding-criteria '{"Criterion":{"type":{"Prefix":"Runtime"}}}' --max-results 10`
- Verify auto-management settings for agent deployment
- Check organization configuration for runtime monitoring: `aws guardduty describe-organization-configuration --detector-id <detector-id>`

MAY:
- Check EC2 instance SSM agent status for auto-managed deployment
- Verify EKS add-on status for EKS runtime monitoring
- Check ECS task definitions for Fargate runtime monitoring

## Phase 2 — Remediate

MUST:
- Enable runtime monitoring if disabled
- Deploy the GuardDuty agent on resources showing UNHEALTHY coverage
- Verify coverage status improves after agent deployment

SHOULD:
- Enable auto-management for agent deployment
- Configure runtime monitoring across all member accounts
- Monitor coverage statistics regularly

MAY:
- Set up CloudWatch alarms for coverage drops
- Create automation for agent deployment on new resources

## Common Issues

- symptoms: "Runtime monitoring enabled but coverage shows 0 healthy resources"
  diagnosis: "The GuardDuty agent is not deployed on any resources."
  resolution: "Enable auto-management or manually deploy the agent. For EC2, ensure SSM agent is running. For EKS, install the GuardDuty add-on."

- symptoms: "Coverage shows UNHEALTHY for specific EC2 instances"
  diagnosis: "The SSM agent is not running, the instance profile lacks SSM permissions, or the GuardDuty agent failed to install."
  resolution: "Verify SSM agent status, check instance profile permissions, and review agent installation logs."

- symptoms: "Runtime findings not generated despite healthy coverage"
  diagnosis: "Runtime findings are only generated for suspicious activity. No findings means no threats detected."
  resolution: "Healthy coverage with no findings is normal. Verify coverage is HEALTHY and the feature is working correctly."

## Safety Ratings

| Phase 2 Action | Rating | Rationale |
|---|---|---|
| Enable runtime monitoring | YELLOW | Configuration change; reversible |
| Deploy GuardDuty agent | YELLOW | Agent deployment; can be removed |
| Enable auto-management | YELLOW | Configuration change; reversible |

## Escalation Conditions

- Detector configuration changes
- Agent deployment to production resources

## Data Sensitivity

| Triage Command | Sensitivity | Rationale |
|---|---|---|
| `get-detector` (RUNTIME_MONITORING) | LOW | Feature status |
| `get-coverage-statistics` | LOW | Aggregate coverage counts |
| `list-coverage` | MEDIUM | Resource coverage details |
| `list-findings` (Runtime) | HIGH | Runtime threat details |

## Prohibited Actions

- NEVER suggest disabling GuardDuty detector
- NEVER suggest suppressing all findings
- NEVER suggest disabling S3/EKS/Lambda protection

## Phase 3 — Rollback

| Remediation | Rollback Step |
|---|---|
| Runtime monitoring enablement | Disable via `update-detector` (not recommended) |
| Agent deployment | Remove agent (delete EKS add-on, uninstall SSM package) |
| Auto-management enablement | Disable auto-management |

## Output Format

```yaml
root_cause: "runtime_monitoring — <specific_cause>"
evidence:
  - type: feature_status
    content: "<runtime monitoring feature status>"
  - type: coverage
    content: "<coverage statistics and details>"
  - type: agent_status
    content: "<agent deployment status>"
severity: HIGH
mitigation:
  immediate: "Deploy the agent and fix unhealthy coverage"
  long_term: "Enable auto-management and monitor coverage continuously"
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
