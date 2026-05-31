---
title: "E2 — Placement Group Issues"
description: "Diagnose placement group configuration and capacity issues for EFA"
status: active
severity: MEDIUM
triggers: ["placement group error", "insufficient capacity", "cannot launch in placement group", "cluster placement"]
owner: devops-agent
objective: "Resolve placement group issues that affect EFA performance and instance launches"
context: "Cluster placement groups ensure instances are on the same network spine for minimal latency. They are critical for EFA performance but may have capacity constraints."
---

## Phase 1 — Triage

MUST:
- Check placement group status: `aws ec2 describe-placement-groups --group-names <group> --query 'PlacementGroups[0].{Strategy:Strategy,State:State,GroupId:GroupId}'`
- Verify instances are in the placement group: `aws ec2 describe-instances --instance-ids <id1> <id2> --query 'Reservations[*].Instances[*].{Id:InstanceId,PlacementGroup:Placement.GroupName,AZ:Placement.AvailabilityZone}'`
- Check for capacity errors in recent launches: `aws ec2 describe-instance-status --instance-ids <id> --query 'InstanceStatuses[0].Events'`

SHOULD:
- Verify all instances use the same instance type within the cluster placement group
- Check if the placement group has reached capacity limits

MAY:
- Review launch history for InsufficientInstanceCapacity errors in CloudTrail

## Phase 2 — Remediate

MUST:
- Create a cluster placement group if none exists: `aws ec2 create-placement-group --group-name <name> --strategy cluster`
- Launch instances into the placement group: include `--placement GroupName=<name>` in run-instances
- For capacity errors, try a different AZ or instance type

SHOULD:
- Use a single instance type within a cluster placement group for best results
- Launch all instances simultaneously to maximize placement success

MAY:
- Use capacity reservations for guaranteed placement: `aws ec2 create-capacity-reservation --instance-type <type> --placement GroupName=<name>`

## Safety Ratings

```
safety_ratings:
  - "ec2 describe-placement-groups: GREEN — read-only placement group query"
  - "ec2 create-placement-group: GREEN — creates empty placement group"
  - "ec2 run-instances with placement: YELLOW — launches instances with cost"
  - "ec2 create-capacity-reservation: RED — creates reserved capacity with cost commitment"
```

## Escalation Conditions

- Cluster placement group has insufficient capacity for the required number of instances
- Placement group spans multiple racks despite cluster strategy (AWS infrastructure issue)
- Cannot launch the required instance type in any AZ with cluster placement
- Capacity reservation is needed but budget approval is required

## Data Sensitivity

- Placement group configuration: LOW — group names and strategies
- Instance placement details: LOW — AZ and group assignments
- Capacity reservation details: MEDIUM — reveals infrastructure planning
- CloudTrail launch events: MEDIUM — reveals IAM principals and launch parameters

## Prohibited Actions

- NEVER delete a placement group with running instances — terminate instances first
- NEVER mix instance types in a cluster placement group — it reduces placement quality
- NEVER launch EFA workloads without a placement group and claim full performance
- NEVER create spread placement groups for EFA — only cluster strategy provides the needed proximity

## Phase 3 — Rollback

1. If instances were launched in the wrong placement group, terminate and relaunch
2. Delete unused placement groups: `aws ec2 delete-placement-group --group-name <name>`
3. If capacity reservations were created, cancel if not needed: `aws ec2 cancel-capacity-reservation --capacity-reservation-id <id>`
4. Revert launch templates to previous placement group configuration
5. Verify EFA performance after placement changes

## Common Issues

- symptoms: "InsufficientInstanceCapacity when launching into cluster placement group"
  diagnosis: "The AZ does not have enough capacity for the requested instance type in a cluster."
  resolution: "Try a different AZ. Launch fewer instances. Use a different instance type. Consider capacity reservations."

- symptoms: "EFA performance is poor despite instances being in a placement group"
  diagnosis: "Instances were launched at different times and may not be on the same network spine."
  resolution: "Terminate and relaunch all instances simultaneously into a new cluster placement group."

## Output Format

```yaml
root_cause: "placement_group — <specific_cause>"
evidence:
  - type: placement_error
    content: "<error details>"
severity: MEDIUM
mitigation:
  immediate: "Fix placement group configuration"
  long_term: "Use capacity reservations for critical workloads"
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
