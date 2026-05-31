---
title: "C2 — Streaming Latency Issues"
description: "Diagnose AppStream 2.0 streaming latency problems"
status: active
severity: MEDIUM
triggers:
  - "streaming slow"
  - "high latency"
  - "laggy session"
  - "poor streaming quality"
owner: devops-agent
objective: "Identify and resolve streaming latency issues"
context: "Streaming latency depends on network distance, bandwidth, protocol (NICE DCV), instance type, and application demands. UDP 8433 provides better streaming than HTTPS-only. Network quality between user and AppStream region is critical."
---

## Phase 1 — Triage

MUST:
- Check the fleet's region relative to user location
- Verify UDP 8433 is allowed (provides better streaming than HTTPS-only)
- Check instance type (GPU instances for graphics-heavy apps)
- Check CloudWatch session metrics: `aws cloudwatch get-metric-statistics --namespace AWS/AppStream --metric-name ActualCapacity --dimensions Name=Fleet,Value=<fleet-name> --start-time <start> --end-time <end> --period 300 --statistics Average`

SHOULD:
- Verify the user's network bandwidth and latency to the AppStream region
- Check if the user is behind a proxy that may add latency
- Verify the application's resource requirements match the instance type

MAY:
- Test with the AppStream client (may perform better than browser)
- Check if other users in the same location experience similar latency

## Phase 2 — Remediate

MUST:
- Ensure UDP 8433 is allowed through firewalls for optimal streaming
- Use an AppStream region closest to the users
- Use appropriate instance type for the application workload

SHOULD:
- Use GPU instance types for graphics-intensive applications
- Recommend the AppStream client for better performance
- Ensure minimum 5 Mbps bandwidth per user

MAY:
- Configure streaming quality settings on the fleet
- Set up AppStream in multiple regions for global users

## Common Issues

- symptoms: "Streaming laggy and choppy"
  diagnosis: "UDP 8433 blocked — falling back to HTTPS-only streaming."
  resolution: "Allow UDP 8433 outbound to AppStream streaming endpoints."

- symptoms: "Graphics-heavy application slow"
  diagnosis: "Instance type lacks GPU for graphics rendering."
  resolution: "Use a Graphics or GraphicsPro instance type."

- symptoms: "Latency varies by user location"
  diagnosis: "Users far from the AppStream region experience higher latency."
  resolution: "Deploy fleets in regions closer to users."

## Output Format

```yaml
root_cause: "streaming_latency — <specific_cause>"
evidence:
  - type: network_config
    content: "<UDP 8433 and HTTPS connectivity>"
  - type: instance_type
    content: "<fleet instance type>"
  - type: user_location
    content: "<user distance from AppStream region>"
severity: MEDIUM
mitigation:
  immediate: "Fix network configuration for optimal streaming"
  long_term: "Deploy in regions closest to users with appropriate instance types"
```


## Safety Ratings

| Phase 2 Action | safety_rating | Rationale |
|---|---|---|
| Ensure UDP 8433 allowed | YELLOW | Firewall/network change |
| Use AppStream region closest to users | GREEN | Recommendation |
| Use appropriate instance type | YELLOW | Fleet configuration change |
| Use GPU instance types | YELLOW | Fleet configuration change |
| Recommend AppStream client | GREEN | Client recommendation |

## Escalation Conditions

- Fix requires fleet rebuild affecting active users

## Data Sensitivity

- **Network configuration**: UDP/HTTPS connectivity details — INTERNAL
- **Instance type**: Fleet compute configuration — INTERNAL
- **User location**: Geographic proximity data — INTERNAL

## Prohibited Actions

- NEVER suggest deleting fleet with active sessions
- NEVER change instance types on a fleet with active sessions
- NEVER modify network rules without testing impact on streaming quality

## Phase 3 — Rollback

1. Revert firewall/network changes for UDP 8433
2. Revert instance type changes if fleet was modified
3. Restore previous streaming quality settings
4. Verify streaming sessions are functioning normally
5. Collect user feedback on streaming quality after rollback

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
