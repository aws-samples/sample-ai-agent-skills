---
title: "B1 — Health Event Analysis"
description: "Diagnose and analyze Internet Monitor health events"
status: active
severity: HIGH
triggers: ["health event detected", "performance degradation", "internet issue detected"]
owner: devops-agent
objective: "Analyze health events to determine impact and root cause of internet performance issues"
context: "Health events are generated when availability or performance drops below configured thresholds for specific city-networks."
---

## Phase 1 — Triage

MUST:
- List active health events: `aws internetmonitor list-health-events --monitor-name <name> --event-status ACTIVE`
- Get health event details: `aws internetmonitor get-health-event --monitor-name <name> --event-id <event-id>`
- Identify affected city-networks, ASNs, and the scope of impact

SHOULD:
- Check if the event correlates with a global internet event: `aws internetmonitor list-internet-events`
- Review the event timeline (StartedAt, EndedAt, LastUpdatedAt)
- Check impacted locations and client networks

MAY:
- Query historical data for the affected city-networks: `aws internetmonitor start-query --monitor-name <name> --query-type MEASUREMENTS --filter-parameters '{"StartTime":"<time>","EndTime":"<time>"}'`
- Cross-reference with CloudWatch metrics for the monitored resources

## Phase 2 — Remediate

MUST:
- Document the affected city-networks, ASNs, and impact percentages
- Determine if the issue is ISP-side (internet path) or origin-side (AWS infrastructure)
- Communicate impact to stakeholders with evidence from the health event

SHOULD:
- If ISP-side: monitor for resolution and document for post-incident review
- If origin-side: investigate CloudFront or VPC health independently
- Set up CloudWatch alarms for health event notifications if not already configured

MAY:
- Consider traffic routing changes (Route 53 failover) if impact is severe and prolonged
- Engage AWS Support if the issue appears to be on the AWS network side

## Common Issues

- symptoms: "Health event shows availability drop but application metrics look normal"
  diagnosis: "The health event may affect a small city-network with low traffic volume."
  resolution: "Check the impactedLocations in the health event. Low-traffic city-networks may trigger events without noticeable application impact."

## Safety Ratings

```
safety_ratings:
  - "list-health-events: GREEN — read-only health event enumeration"
  - "get-health-event: GREEN — read-only health event details"
  - "list-internet-events: GREEN — read-only global internet event listing"
  - "start-query: GREEN — initiates read-only data query"
```

## Escalation Conditions

- Health event affects more than 10% of total application traffic
- Availability score drops below 90% for a major city-network
- Health event persists for more than 4 hours without improvement
- Multiple monitors show correlated health events suggesting widespread issue

## Data Sensitivity

- `aws internetmonitor list-health-events`: LOW — returns event metadata
- `aws internetmonitor get-health-event`: MEDIUM — reveals traffic patterns and affected locations
- `aws internetmonitor start-query`: MEDIUM — returns detailed traffic measurements

## Prohibited Actions

- NEVER claim a health event indicates an application bug without corroborating evidence
- NEVER dismiss a health event as a false positive without analyzing the affected city-networks
- NEVER modify monitor thresholds during an active health event to suppress alerts
- NEVER delete health event data or modify the monitor during active incident investigation

## Phase 3 — Rollback

1. If threshold changes were made during investigation, revert to previous values
2. If CloudWatch alarms were modified, restore original alarm configurations
3. If traffic routing changes were made, verify they can be safely reverted after the event resolves
4. Document the health event timeline, root cause, and resolution for post-incident review
5. Verify the health event has resolved and monitor scores have returned to baseline

## Output Format

```yaml
root_cause: "health_event — <specific_cause>"
evidence:
  - type: health_event
    content: "<event details with affected city-networks>"
severity: HIGH
mitigation:
  immediate: "Monitor event progression and communicate impact"
  long_term: "Set up automated alerting and traffic failover for affected paths"
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
