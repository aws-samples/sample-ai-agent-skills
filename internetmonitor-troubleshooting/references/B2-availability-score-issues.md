---
title: "B2 — Availability Score Issues"
description: "Diagnose availability score anomalies and unexpected values"
status: active
severity: HIGH
triggers: ["low availability score", "availability dropped", "score not updating", "unexpected availability"]
owner: devops-agent
objective: "Investigate and resolve availability score anomalies in Internet Monitor"
context: "Availability scores measure the percentage of traffic completing successfully per city-network. Scores below threshold trigger health events."
---

## Phase 1 — Triage

MUST:
- Get current monitor status: `aws internetmonitor get-monitor --monitor-name <name>`
- Check ProcessingStatus — INSUFFICIENT_DATA means not enough traffic for scoring
- List recent health events related to availability: `aws internetmonitor list-health-events --monitor-name <name> --event-status ACTIVE`

SHOULD:
- Query availability measurements: `aws internetmonitor start-query --monitor-name <name> --query-type MEASUREMENTS`
- Check if the score issue is global or limited to specific city-networks
- Verify the availability threshold configuration

MAY:
- Compare current scores against historical baseline
- Check if monitored resources (CloudFront/VPC) have had recent changes

## Phase 2 — Remediate

MUST:
- Identify whether the score drop is due to internet path issues or origin issues
- Document affected city-networks and their individual availability scores
- Verify the score is not a data artifact from low traffic volume

SHOULD:
- If INSUFFICIENT_DATA: verify resources have active traffic flowing through them
- If score drop is real: correlate with ISP outages or internet events
- Review CloudFront or VPC health metrics independently

MAY:
- Adjust availability threshold if current threshold generates excessive noise
- Configure CloudWatch alarms for availability score monitoring

## Common Issues

- symptoms: "Availability score shows 0% but application is working fine"
  diagnosis: "Monitor may have INSUFFICIENT_DATA or no traffic flowing through monitored resources."
  resolution: "Check ProcessingStatus field. Verify resources are attached and have active traffic. A score of 0% with no traffic is a data gap, not an outage."

## Safety Ratings

```
safety_ratings:
  - "get-monitor: GREEN — read-only monitor details"
  - "list-health-events: GREEN — read-only event listing"
  - "start-query: GREEN — read-only data query"
  - "update-monitor threshold: YELLOW — changes alerting behavior"
```

## Escalation Conditions

- Availability score below 90% for more than 1 hour across multiple city-networks
- Score shows 0% or INSUFFICIENT_DATA despite confirmed active traffic
- Availability score discrepancy between Internet Monitor and application-level metrics
- Persistent score degradation with no corresponding internet events

## Data Sensitivity

- `aws internetmonitor get-monitor`: LOW — returns monitor configuration
- `aws internetmonitor start-query`: MEDIUM — reveals traffic patterns and availability data
- `aws internetmonitor list-health-events`: LOW — returns event metadata

## Prohibited Actions

- NEVER interpret a low availability score as server downtime — it measures internet path success
- NEVER change availability thresholds during an active incident to suppress alerts
- NEVER dismiss low scores without investigating the affected city-networks
- NEVER report availability scores as application SLA metrics without proper context

## Phase 3 — Rollback

1. If thresholds were changed, revert to previous values: `aws internetmonitor update-monitor --monitor-name <name> --health-events-config <previous-config>`
2. If resources were modified during investigation, restore original resource list
3. Verify scores return to expected ranges after any configuration changes
4. Document the score anomaly investigation and findings
5. Confirm health event alerting is functioning correctly

## Output Format

```yaml
root_cause: "availability_score — <specific_cause>"
evidence:
  - type: score_analysis
    content: "<score details and affected city-networks>"
severity: HIGH
mitigation:
  immediate: "Identify and communicate score anomaly cause"
  long_term: "Set up automated monitoring for score deviations"
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
