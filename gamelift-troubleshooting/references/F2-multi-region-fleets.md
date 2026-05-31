---
title: "F2 — Multi-Region Fleets"
description: "Diagnose multi-region fleet configuration and deployment issues"
status: active
severity: HIGH
triggers: ["multi-region", "cross-region fleet", "global fleet", "region failover"]
owner: devops-agent
objective: "Resolve multi-region fleet deployment and management issues"
context: "Multi-region deployments use queues with fleets in different regions. Each region needs its own fleet, build, and configuration. Queues route based on latency and availability."
---

## Phase 1 — Triage

MUST:
- Check fleets across regions: `aws gamelift describe-fleet-attributes --query 'FleetAttributes[*].{Id:FleetId,Name:Name,Status:Status,Region:FleetId}'` (run per region)
- Verify queue has multi-region destinations: `aws gamelift describe-game-session-queues --names <queue-name>`
- Check fleet status in each region

SHOULD:
- Verify builds are uploaded in each target region
- Check player latency policies in the queue
- Verify fleet capacity in each region

MAY:
- Compare fleet configurations across regions for consistency
- Check CloudWatch metrics per region for utilization

## Phase 2 — Remediate

MUST:
- Upload build to each target region: `aws gamelift upload-build --name <name> --build-root <path> --region <region>`
- Create fleet in each region with consistent configuration
- Add all regional fleets to the queue: `aws gamelift update-game-session-queue --name <queue-name> --destinations <fleet-arns>`

SHOULD:
- Configure player latency policies for optimal routing
- Set up auto-scaling in each region independently
- Use priority configuration to prefer lower-cost regions

MAY:
- Implement fleet aliases for easier fleet replacement
- Use CloudFormation StackSets for consistent multi-region deployment

## Safety Ratings

```
safety_ratings:
  - "describe-fleet-attributes / describe-game-session-queues: GREEN — read-only inspection"
  - "upload-build (per region): YELLOW — uploads server binary"
  - "create-fleet (per region): YELLOW — creates instances, incurs costs"
  - "update-game-session-queue: YELLOW — changes routing behavior"
```

## Escalation Conditions

- Build upload fails in specific regions
- Fleet creation fails due to instance type unavailability in a region
- Queue routing not respecting latency policies
- Cross-region cost optimization requires architecture review

## Data Sensitivity

- `aws gamelift describe-fleet-attributes`: LOW — fleet metadata
- `aws gamelift describe-game-session-queues`: LOW — queue configuration
- Player latency data: LOW — network measurements
- Regional fleet configurations: LOW — infrastructure details

## Prohibited Actions

- NEVER deploy to regions without understanding data residency requirements
- NEVER remove regions from queue during peak hours without failover plan
- NEVER use different build versions across regions without testing
- NEVER ignore regional cost differences when adding fleet destinations

## Phase 3 — Rollback

1. If fleet was created in wrong region: delete fleet in wrong region
2. If queue destinations were modified: restore original destination list
3. If latency policies were changed: restore original policies
4. If build was uploaded to wrong region: delete build in wrong region
5. Document multi-region changes and update deployment runbook

## Common Issues

- symptoms: "Players in Europe experience high latency despite having an EU fleet"
  diagnosis: "Queue player latency policy is not configured, or EU fleet is not in the queue destinations."
  resolution: "Add EU fleet to queue destinations. Configure player latency policy to prefer nearby regions. Ensure client sends latency data with placement requests."

## Output Format

```yaml
root_cause: "multi_region — <specific_cause>"
evidence:
  - type: region_error
    content: "<error details>"
severity: HIGH
mitigation:
  immediate: "Fix multi-region configuration"
  long_term: "Implement automated multi-region deployment with consistent configuration"
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
