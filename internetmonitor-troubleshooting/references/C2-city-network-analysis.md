---
title: "C2 — City-Network Analysis"
description: "Diagnose city-network level analysis and coverage issues"
status: active
severity: MEDIUM
triggers: ["city-network missing", "ASN not monitored", "location not covered", "city-network data"]
owner: devops-agent
objective: "Resolve city-network analysis coverage and data issues"
context: "City-networks are combinations of client city and ASN (network provider). MaxCityNetworksToMonitor controls coverage."
---

## Phase 1 — Triage

MUST:
- Check MaxCityNetworksToMonitor setting: `aws internetmonitor get-monitor --monitor-name <name> --query 'MaxCityNetworksToMonitor'`
- Query top locations: `aws internetmonitor start-query --monitor-name <name> --query-type TOP_LOCATIONS --filter-parameters '{"StartTime":"<time>","EndTime":"<time>"}'`
- Verify the expected city-network appears in monitored data

SHOULD:
- Check if the city-network has sufficient traffic volume to be included
- Review the traffic distribution across city-networks
- Verify the ASN (network provider) is correctly identified

MAY:
- Query detailed data for a specific location: `aws internetmonitor start-query --monitor-name <name> --query-type TOP_LOCATION_DETAILS --filter-parameters '{"StartTime":"<time>","EndTime":"<time>","City":"<city>","Country":"<country>"}'`
- Compare city-network coverage against application access logs

## Phase 2 — Remediate

MUST:
- If city-network is missing due to low MaxCityNetworksToMonitor, evaluate increasing the value
- Document which city-networks are and are not covered
- Verify that critical business locations are within the monitored set

SHOULD:
- Increase MaxCityNetworksToMonitor if critical locations are excluded: `aws internetmonitor update-monitor --monitor-name <name> --max-city-networks-to-monitor <value>`
- Review cost implications before increasing city-network coverage

MAY:
- Create separate monitors for different resource groups to optimize coverage
- Set up periodic reviews of city-network coverage

## Common Issues

- symptoms: "Expected city-network not appearing in top locations query"
  diagnosis: "City-network may have insufficient traffic or MaxCityNetworksToMonitor is too low."
  resolution: "Check MaxCityNetworksToMonitor value. Internet Monitor prioritizes city-networks by traffic volume. Low-traffic locations may be excluded when the limit is reached."

## Safety Ratings

```
safety_ratings:
  - "get-monitor: GREEN — read-only monitor details"
  - "start-query TOP_LOCATIONS: GREEN — read-only location query"
  - "start-query TOP_LOCATION_DETAILS: GREEN — read-only detailed location query"
  - "update-monitor max-city-networks: YELLOW — changes monitoring scope and cost"
```

## Escalation Conditions

- Critical business locations consistently excluded from monitoring despite adequate traffic
- MaxCityNetworksToMonitor at maximum but coverage still insufficient
- City-network data shows anomalies that do not match known traffic patterns
- ASN identification appears incorrect for known network providers

## Data Sensitivity

- `aws internetmonitor start-query TOP_LOCATIONS`: MEDIUM — reveals geographic traffic distribution
- `aws internetmonitor start-query TOP_LOCATION_DETAILS`: MEDIUM — reveals detailed per-location traffic data
- `aws internetmonitor get-monitor`: LOW — returns monitor configuration

## Prohibited Actions

- NEVER increase MaxCityNetworksToMonitor to maximum without cost analysis
- NEVER assume a missing city-network means no users in that location
- NEVER share city-network traffic distribution data without authorization
- NEVER reduce MaxCityNetworksToMonitor on a production monitor without impact analysis

## Phase 3 — Rollback

1. If MaxCityNetworksToMonitor was changed, revert to previous value: `aws internetmonitor update-monitor --monitor-name <name> --max-city-networks-to-monitor <previous-value>`
2. If additional monitors were created, delete them if not needed
3. Verify city-network coverage returns to expected state after changes
4. Document the coverage analysis and any changes made
5. Confirm cost impact of any configuration changes

## Output Format

```yaml
root_cause: "city_network_analysis — <specific_cause>"
evidence:
  - type: coverage_analysis
    content: "<city-network coverage details>"
severity: MEDIUM
mitigation:
  immediate: "Adjust city-network coverage settings"
  long_term: "Implement periodic coverage reviews aligned with traffic patterns"
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
