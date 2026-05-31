---
title: "A3 — Outpost Hardware Failures"
description: "Diagnose and respond to hardware component failures on AWS Outposts"
status: active
severity: CRITICAL
triggers:
  - "hardware failure"
  - "host degraded"
  - "asset unhealthy"
  - "rack hardware"
owner: devops-agent
objective: "Identify failed hardware components and coordinate replacement with AWS"
context: "AWS manages Outpost hardware but the customer must provide physical access. Hardware failures can affect compute hosts, storage drives, or networking components. AWS proactively monitors and replaces failed components."
---

## Phase 1 — Triage

MUST:
- List Outpost assets and status: `aws outposts list-assets --outpost-identifier <outpost-id>`
- Check for scheduled instance events: `aws ec2 describe-instance-status --filters "Name=event.code,Values=instance-retirement,instance-stop,system-maintenance" --query 'InstanceStatuses[?OutpostArn!=null]'`
- Check instance status checks: `aws ec2 describe-instance-status --filters "Name=outpost-arn,Values=<outpost-arn>" --query 'InstanceStatuses[*].{Id:InstanceId,System:SystemStatus.Status,Instance:InstanceStatus.Status}'`
- Review AWS Health events: `aws health describe-events --filter "services=OUTPOSTS"`

SHOULD:
- Check CloudWatch for host-level metrics anomalies
- Identify which instances are affected by the hardware failure
- Verify if AWS has already scheduled a maintenance event

MAY:
- Check AWS Personal Health Dashboard for Outpost-specific notifications
- Review historical hardware events for patterns

## Phase 2 — Remediate

MUST:
- If instances are on a degraded host, migrate them to healthy hosts (stop/start)
- Ensure AWS Support has been notified of the hardware failure
- Provide physical access to the Outpost rack for AWS technicians if replacement is needed

SHOULD:
- Migrate critical workloads to healthy hosts proactively
- Create snapshots of EBS volumes on affected hosts
- Document the failure for capacity planning

MAY:
- Implement instance placement strategies to spread across hosts
- Set up automated instance recovery for critical workloads

## Common Issues

- symptoms: "Instance status check failed — system status impaired"
  diagnosis: "Underlying host hardware has failed."
  resolution: "Stop and start the instance to migrate to a healthy host. Contact AWS Support."

- symptoms: "AWS Health notification about Outpost hardware maintenance"
  diagnosis: "AWS has detected a failing component and scheduled replacement."
  resolution: "Migrate instances before the maintenance window. Provide rack access to AWS."

## Output Format

```yaml
root_cause: "hardware — <specific_component>"
evidence:
  - type: asset_status
    content: "<asset health information>"
  - type: instance_status
    content: "<affected instance status checks>"
severity: CRITICAL
mitigation:
  immediate: "Migrate instances from degraded hosts"
  long_term: "Implement multi-host placement and automated recovery"
```


## Safety Ratings
- Overall Risk: HIGH
- Data Exposure Risk: MEDIUM — hardware failures may affect data on local storage
- Service Disruption Risk: HIGH — hardware failures directly impact workload availability
- Compliance Impact: HIGH — hardware failures may affect data residency and availability requirements

## Escalation Conditions
- Multiple hardware components fail simultaneously
- Hardware failure causes data loss on local storage
- Hardware replacement requires extended downtime
- Hardware failure affects Outpost management plane
- Physical security of failed hardware components is a concern

## Data Sensitivity
- Hardware serial numbers and configurations — INTERNAL
- Failure diagnostics and logs — INTERNAL
- Local storage data — CONFIDENTIAL
- Physical site access details — RESTRICTED
- Never expose hardware failure details that reveal site vulnerabilities

## Prohibited Actions
- Do NOT attempt hardware repairs without AWS authorization
- Do NOT remove failed hardware components without following AWS procedures
- Do NOT ignore hardware failure alerts
- Do NOT bypass physical security procedures for hardware access
- Do NOT dispose of failed hardware without proper data sanitization
- Do NOT continue running workloads on degraded hardware without risk assessment

## Phase 3 — Rollback
- Migrate workloads from failed hardware to healthy capacity
- Restore data from backups if local storage was affected
- Verify workload health after hardware replacement
- Re-enable monitoring on replaced hardware components
- Document hardware failure timeline and resolution
- Update capacity planning based on hardware reliability data

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
