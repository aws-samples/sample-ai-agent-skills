---
title: "Z1 — General Auto Scaling Troubleshooting"
description: "Catch-all systematic investigation for Auto Scaling issues not covered by specific runbooks"
status: active
severity: MEDIUM
triggers:
  - "Auto Scaling issue"
  - "ASG problem"
  - "scaling not working"
  - "Auto Scaling error"
  - "general troubleshooting"
owner: devops-agent
objective: "Systematically investigate Auto Scaling issues using a structured diagnostic approach"
context: "This runbook provides a general-purpose investigation framework for Auto Scaling issues that don't clearly match a specific runbook. It covers ASG health, scaling activities, policy configuration, launch template validation, and systematic elimination of common causes."
---

## Phase 1 — Triage

MUST:
- Check ASG overview: `aws autoscaling describe-auto-scaling-groups --auto-scaling-group-names <asg-name>`
- Check recent scaling activities: `aws autoscaling describe-scaling-activities --auto-scaling-group-name <asg-name> --max-items 20`
- Check all scaling policies: `aws autoscaling describe-policies --auto-scaling-group-name <asg-name>`
- Check instance states: `aws autoscaling describe-auto-scaling-instances --query 'AutoScalingInstances[?AutoScalingGroupName==`<asg-name>`]'`
- Check for suspended processes: `aws autoscaling describe-auto-scaling-groups --auto-scaling-group-names <asg-name> --query 'AutoScalingGroups[*].SuspendedProcesses'`

SHOULD:
- Check CloudTrail for recent ASG changes: `aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventSource,AttributeValue=autoscaling.amazonaws.com --max-results 20`
- Check CloudWatch alarms associated with scaling policies
- Verify launch template/configuration is valid
- Check lifecycle hooks and warm pool status

MAY:
- Check AWS Health Dashboard for service issues
- Review EventBridge events for Auto Scaling
- Check service quotas for Auto Scaling limits

## Phase 2 — Remediate

MUST:
- Based on triage findings, pivot to the appropriate specific runbook:
  - Scaling issues → A1 (not scaling out), A2 (not scaling in), A3 (too aggressive), A4 (cooldown)
  - Launch issues → B1 (launch failures), B2 (template errors), B3 (AMI issues)
  - Health issues → C1 (health checks), C2 (instance refresh), C3 (warm pool)
  - Policy issues → D1 (target tracking), D2 (step scaling), D3 (predictive)
  - Lifecycle → E1 (hooks), E2 (termination policies)
  - Integration → F1 (ELB), F2 (EventBridge)
  - Mixed instances → G1 (policy), G2 (spot allocation)
- Document findings and root cause

SHOULD:
- Check for multiple concurrent issues
- Review recent changes that may have caused the issue

MAY:
- Set up comprehensive monitoring to prevent recurrence
- Create post-incident report for significant issues

## Output Format

```yaml
root_cause: "general — <specific_cause>"
evidence:
  - type: asg_status
    content: "<ASG configuration and state>"
  - type: scaling_activities
    content: "<recent scaling events>"
  - type: policies
    content: "<scaling policy configuration>"
severity: MEDIUM
general_analysis:
  asg_state: "healthy | degraded | failing"
  scaling_status: "active | blocked | misconfigured"
  specific_runbook: "<recommended runbook ID>"
mitigation:
  immediate: "Follow the identified specific runbook"
  long_term: "Implement comprehensive monitoring and alerting"
```


## Safety Ratings

| Phase 2 Action | safety_rating | Rationale |
|---|---|---|
| Pivot to appropriate specific runbook | GREEN | Diagnostic routing — no system changes |
| Document findings | GREEN | Documentation only |
| Set up comprehensive monitoring | GREEN | Monitoring — non-destructive |

## Escalation Conditions

- Fix requires modifying production ASG capacity or scaling behavior
- Fix requires changing launch template used by production instances
- Unclassified issues may indicate systemic infrastructure problems
- Investigation may reveal multiple concurrent issues

## Data Sensitivity

- General investigation may expose ASG configurations across multiple groups
- CloudTrail analysis may reveal operational patterns and access history
- Instance states reveal deployment and infrastructure health

## Prohibited Actions

- NEVER suggest setting min/max to 0 in production — this terminates all instances
- NEVER suggest deleting ASG with running instances — this causes immediate outage
- NEVER modify ASG configuration during investigation without a rollback plan
- NEVER change scaling policies as part of general troubleshooting without specific diagnosis

## Phase 3 — Rollback

- General troubleshooting is diagnostic — rollback depends on the specific runbook identified
- If any ASG settings were changed during investigation: restore original values
- Document all investigative actions for operational audit trail

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
