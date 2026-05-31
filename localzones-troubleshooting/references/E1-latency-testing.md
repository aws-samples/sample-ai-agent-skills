---
title: "E1 — Latency Testing"
description: "Diagnose latency testing and performance measurement issues in Local Zones"
status: active
severity: MEDIUM
triggers: ["latency too high", "Local Zone not faster", "performance testing", "latency measurement"]
owner: devops-agent
objective: "Resolve latency testing issues and validate Local Zone performance benefits"
context: "Local Zones provide lower latency to end users in the metro area. Latency benefits depend on user proximity, network path, and application architecture. Testing must compare Local Zone vs parent region latency."
---

## Phase 1 — Triage

MUST:
- Measure latency from client to Local Zone instance: `ping -c 20 <local-zone-instance-ip>`
- Measure latency from client to parent region instance: `ping -c 20 <region-instance-ip>`
- Check network path: `traceroute <local-zone-instance-ip>`

SHOULD:
- Test application-level latency (HTTP request/response time)
- Measure latency from Local Zone to parent region services (S3, DynamoDB)
- Check CloudWatch NetworkIn/NetworkOut metrics for the instance

MAY:
- Use iperf3 for throughput testing between zones
- Run latency tests at different times of day
- Compare latency from multiple client locations

## Phase 2 — Remediate

MUST:
- Verify client is geographically close to the Local Zone metro area
- Ensure application components that need low latency are co-located in the Local Zone
- Check that DNS resolution returns Local Zone endpoint, not parent region

SHOULD:
- Move latency-sensitive data stores to Local Zone (EBS, local RDS)
- Minimize calls from Local Zone to parent region services

MAY:
- Implement caching in Local Zone to reduce parent region round trips
- Use Route 53 latency-based routing to direct users to nearest endpoint

## Safety Ratings

```
safety_ratings:
  - "ping/traceroute: GREEN — read-only network diagnostics"
  - "CloudWatch metrics: GREEN — read-only performance data"
  - "iperf3 testing: GREEN — network measurement tool"
  - "Route 53 routing changes: YELLOW — affects DNS resolution for all users"
```

## Escalation Conditions

- Local Zone latency is higher than parent region indicating routing issues
- Network path shows unexpected hops through distant locations
- Application latency does not improve despite infrastructure being in Local Zone
- ISP routing issues causing traffic to bypass Local Zone

## Data Sensitivity

- Latency measurements: LOW — network performance data
- Traceroute output: MEDIUM — reveals network topology and hop addresses
- Instance IP addresses: MEDIUM — infrastructure endpoint information
- Application performance data: LOW — aggregate timing metrics

## Prohibited Actions

- NEVER run sustained high-bandwidth tests without coordinating with network team
- NEVER publish internal network topology from traceroute results
- NEVER conclude Local Zone is not beneficial based on a single test from one location
- NEVER modify DNS routing during production hours without a rollback plan

## Phase 3 — Rollback

1. If DNS routing was changed for testing, revert to previous routing configuration
2. If test instances were launched, terminate them after testing
3. If Route 53 records were modified, restore previous record values
4. Document test results for future reference

## Common Issues

- symptoms: "Latency to Local Zone is same as parent region"
  diagnosis: "Client is not in the Local Zone metro area or network routing bypasses the zone."
  resolution: "Verify client location. Check traceroute for routing path. Test from a client in the metro area."

- symptoms: "Application latency improved but database calls are slow"
  diagnosis: "Application is in Local Zone but database is in parent region."
  resolution: "Deploy RDS in Local Zone or implement caching to reduce parent region database calls."

## Output Format

```yaml
root_cause: "latency_testing — <specific_cause>"
evidence:
  - type: latency_data
    content: "<latency measurements and network path>"
severity: MEDIUM
mitigation:
  immediate: "Verify test methodology and client location"
  long_term: "Implement continuous latency monitoring"
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
