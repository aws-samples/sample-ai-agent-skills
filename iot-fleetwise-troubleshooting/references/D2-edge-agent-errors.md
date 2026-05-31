---
title: "D2 — Edge Agent Errors"
description: "Diagnose FleetWise Edge Agent (FWE) configuration, startup, and runtime errors"
status: active
severity: HIGH
triggers: ["edge agent crash", "FWE error", "edge agent not starting", "agent configuration error", "collection agent failed"]
owner: devops-agent
objective: "Resolve edge agent issues on vehicles to enable data collection"
context: "The FleetWise Edge Agent (FWE) runs on the vehicle, collects signals from vehicle networks (CAN, OBD-II), and sends data to the cloud. Configuration is local to the vehicle and not pushed from the cloud."
---

## Phase 1 — Triage

MUST:
- Verify vehicle exists in FleetWise: `aws iotfleetwise get-vehicle --vehicle-name <name>`
- Check vehicle's IoT thing connectivity: `aws iot search-index --query-string "thingName:<vehicle-name>"`
- Verify campaign is targeting the vehicle's fleet: `aws iotfleetwise get-campaign --name <campaign-name> --query '{TargetArn:targetArn,Status:status}'`

SHOULD:
- Check edge agent logs on the vehicle for error messages
- Verify edge agent configuration file has correct vehicle name, endpoint, and credential paths
- Confirm vehicle network interfaces (CAN bus, OBD-II port) are accessible to the agent

MAY:
- Check edge agent version compatibility with current FleetWise API
- Review system resource usage (CPU, memory, disk) on the vehicle
- Verify edge agent has permissions to access vehicle network interfaces

## Phase 2 — Remediate

MUST:
- Fix edge agent configuration: correct vehicle name, IoT endpoint, certificate paths
- Ensure edge agent binary is compatible with the vehicle's OS and architecture
- Verify vehicle network interfaces are available and not locked by other processes

SHOULD:
- Update edge agent to latest stable version
- Configure appropriate logging level for diagnostics
- Ensure edge agent has sufficient system resources (memory, CPU)

MAY:
- Implement edge agent health monitoring and auto-restart (systemd, watchdog)
- Configure edge agent persistent storage for offline data buffering
- Set up remote edge agent log collection for fleet-wide diagnostics

## Safety Ratings

```
safety_ratings:
  - "get-vehicle: GREEN — read-only inspection"
  - "search-index: GREEN — read-only connectivity check"
  - "get-campaign: GREEN — read-only campaign inspection"
  - "Edge agent restart: YELLOW — interrupts data collection temporarily"
  - "Edge agent configuration change: YELLOW — may affect data collection behavior"
```

## Escalation Conditions

- Edge agent crashes repeatedly indicating firmware or hardware compatibility issue
- Edge agent cannot access vehicle CAN bus due to hardware or driver issues
- Fleet-wide edge agent failures after version update
- Edge agent consuming excessive vehicle resources affecting vehicle operation

## Data Sensitivity

- `aws iotfleetwise get-vehicle`: LOW — returns vehicle metadata
- Edge agent configuration file: HIGH — contains endpoint and credential paths
- Edge agent logs: HIGH — may contain vehicle diagnostic data and signal values
- Vehicle CAN bus data: HIGH — contains real-time vehicle operational data

## Prohibited Actions

- NEVER deploy untested edge agent versions to production vehicles
- NEVER configure edge agent to collect all CAN signals without filtering — excessive bandwidth and storage
- NEVER run edge agent with root privileges unless required for CAN bus access
- NEVER ignore edge agent resource consumption — it runs on resource-constrained vehicle hardware

## Phase 3 — Rollback

1. If edge agent was updated, revert to previous version
2. If configuration was changed, restore from backup configuration file
3. Restart edge agent with restored configuration
4. Verify data collection resumes correctly
5. Monitor vehicle system resources to confirm agent is stable

## Common Issues

- symptoms: "Edge agent starts but immediately exits with configuration error"
  diagnosis: "Configuration file has incorrect vehicle name, missing certificate files, or invalid endpoint URL."
  resolution: "Verify configuration file. Ensure vehicle name matches FleetWise vehicle exactly. Check certificate files exist at configured paths."

- symptoms: "Edge agent running but no data sent to cloud"
  diagnosis: "Edge agent cannot read vehicle CAN bus, or no active campaign targets the vehicle's fleet."
  resolution: "Verify CAN interface is accessible. Check vehicle is in a fleet with an active RUNNING campaign."

## Output Format

```yaml
root_cause: "edge_agent — <specific_cause>"
evidence:
  - type: agent_error
    content: "<error details>"
severity: HIGH
mitigation:
  immediate: "Fix edge agent configuration and restart"
  long_term: "Implement edge agent fleet management and monitoring"
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
  - command: "describe-instances"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"
  - command: "get-console-output"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"
  - command: "ssm send-command"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"

## Prohibited Actions

prohibited_actions:
  - "NEVER suggest 0.0.0.0/0 inbound security group rules as a fix"
  - "NEVER suggest disabling instance metadata service"
  - "NEVER terminate instances without confirmation"
