---
title: "C1 — Coverage Gaps"
description: "Identify and resolve Inspector scanning coverage gaps"
status: active
severity: HIGH
triggers:
  - "coverage gap"
  - "resources not covered"
  - "incomplete scanning"
  - "missing coverage"
owner: devops-agent
objective: "Identify resources not being scanned and close coverage gaps"
context: "Coverage gaps occur when resources exist but are not being scanned by Inspector. For EC2, this usually means missing SSM agent. For ECR, it means basic scanning instead of enhanced. For Lambda, it means unsupported runtimes. Use coverage statistics to identify gaps."
---

## Phase 1 — Triage

MUST:
- Check overall coverage statistics: `aws inspector2 list-coverage-statistics`
- List uncovered resources: `aws inspector2 list-coverage --filter-criteria '{"scanStatusCode":[{"comparison":"EQUALS","value":"INACTIVE"}]}'`
- Check coverage by resource type: `aws inspector2 list-coverage-statistics --filter-criteria '{"resourceType":[{"comparison":"EQUALS","value":"AWS_EC2_INSTANCE"}]}'`
- Check account scanning status: `aws inspector2 batch-get-account-status`

SHOULD:
- Identify specific uncovered EC2 instances and check SSM status
- Check ECR repositories not using enhanced scanning
- Verify Lambda functions with unsupported runtimes

MAY:
- Compare total resource count with covered resource count
- Check for tag-based exclusion rules limiting coverage

## Phase 2 — Remediate

MUST:
- Deploy SSM agent to uncovered EC2 instances
- Enable enhanced scanning for ECR repositories
- Update Lambda functions to supported runtimes

SHOULD:
- Set up automated SSM agent deployment for new instances
- Configure default scanning for new ECR repositories
- Monitor coverage statistics regularly

MAY:
- Create CloudWatch alarms on coverage percentage
- Use AWS Config rules to enforce SSM agent installation

## Common Issues

- symptoms: "50% of EC2 instances not covered"
  diagnosis: "Half the fleet missing SSM agent or IAM role."
  resolution: "Deploy SSM agent and IAM role. Use SSM Quick Setup for automation."

- symptoms: "New instances not automatically covered"
  diagnosis: "AMI doesn't include SSM agent or launch template lacks IAM role."
  resolution: "Update AMI with SSM agent. Add IAM role to launch templates."

## Safety Ratings

| Phase 2 Action | Rating | Rationale |
|---|---|---|
| Deploy SSM agent to uncovered instances | YELLOW | Agent installation on running instances |
| Enable enhanced scanning for ECR | YELLOW | Changes scanning mode |
| Update Lambda runtimes | YELLOW | Runtime changes may affect function behavior |
| Set up automated SSM deployment | GREEN | Automation configuration |

## Escalation Conditions

- Large coverage gaps (>30% of resources uncovered) — escalate to security operations
- Production instances not covered — escalate immediately to security team
- Coverage gaps in compliance-critical environments — escalate to compliance team
- Scanning configuration changes affecting organization — escalate to delegated admin

## Data Sensitivity

| Triage Command | Sensitivity | Rationale |
|---|---|---|
| `aws inspector2 list-coverage-statistics` | MEDIUM | Exposes coverage metrics |
| `aws inspector2 list-coverage` (INACTIVE) | MEDIUM | Exposes uncovered resource details |
| `aws inspector2 batch-get-account-status` | MEDIUM | Exposes scanning status |

## Prohibited Actions

- **NEVER** suggest disabling Inspector scanning as a troubleshooting step
- **NEVER** suggest suppressing all findings without individual review
- **NEVER** exclude production resources from scanning to improve coverage percentages
- **NEVER** ignore coverage gaps in internet-facing resources

## Phase 3 — Rollback

| Remediation Action | Rollback Step |
|---|---|
| Deployed SSM agent | Uninstall SSM agent (not recommended) |
| Enabled enhanced ECR scanning | Revert to basic scanning |
| Updated Lambda runtimes | Revert to previous runtime |

## Output Format

```yaml
root_cause: "coverage_gap — <specific_cause>"
evidence:
  - type: coverage_stats
    content: "<coverage statistics by resource type>"
  - type: uncovered_resources
    content: "<list of uncovered resources>"
severity: HIGH
mitigation:
  immediate: "Fix coverage for identified resources"
  long_term: "Automate coverage enforcement for new resources"
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
  - "NEVER suggest disabling scanning to reduce findings"
  - "NEVER suggest ignoring critical vulnerabilities"
  - "NEVER suggest removing SSM agent to avoid scans"
