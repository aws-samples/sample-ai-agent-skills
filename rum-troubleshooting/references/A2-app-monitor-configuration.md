---
title: "A2 — App Monitor Configuration"
description: "Diagnose app monitor configuration issues"
status: active
severity: MEDIUM
triggers: ["app monitor settings wrong", "update monitor failed", "configuration issue"]
owner: devops-agent
objective: "Resolve app monitor configuration and update issues"
context: "App monitor configuration includes domain, session sample rate, telemetry types, custom events, session replay, and allowed domains."
---

## Phase 1 — Triage

MUST:
- Get app monitor details: `aws rum get-app-monitor --name <name>`
- Check app monitor state — should be ACTIVE
- Review configuration: session sample rate, telemetry types, domain, allowed domains

SHOULD:
- Verify the domain matches the application's actual domain
- Check if custom events are enabled if expected
- Verify session replay configuration

MAY:
- Compare current configuration against IaC templates
- Review recent update-app-monitor calls in CloudTrail

## Phase 2 — Remediate

MUST:
- Update configuration as needed: `aws rum update-app-monitor --name <name> --app-monitor-configuration '{"SessionSampleRate":<rate>,"Telemetries":["performance","errors","http"]}'`
- Ensure domain configuration matches the application
- Verify the app monitor returns to ACTIVE state after updates

SHOULD:
- Adjust session sample rate based on traffic and budget
- Enable or disable telemetry types as needed
- Configure allowed domains for multi-domain applications

MAY:
- Enable custom events for business metric tracking
- Configure session replay settings

## Common Issues

- symptoms: "App monitor is ACTIVE but not collecting expected data types"
  diagnosis: "Telemetry types may not include the expected data category."
  resolution: "Check telemetry configuration: `aws rum get-app-monitor --name <name>`. Update to include needed types: `aws rum update-app-monitor --name <name> --app-monitor-configuration '{\"Telemetries\":[\"performance\",\"errors\",\"http\"]}'`"

## Safety Ratings

```
safety_ratings:
  - "get-app-monitor: GREEN — read-only monitor details"
  - "update-app-monitor: YELLOW — modifies monitor configuration, may affect data collection"
  - "cloudtrail lookup-events: GREEN — read-only audit log review"
```

## Escalation Conditions

- App monitor stuck in non-ACTIVE state after configuration update
- Configuration changes not taking effect in collected data
- Domain configuration changes needed for a production application
- Session sample rate changes causing unexpected cost impact

## Data Sensitivity

- `aws rum get-app-monitor`: MEDIUM — reveals application domain and monitoring configuration
- `aws rum update-app-monitor`: MEDIUM — modifies data collection scope

## Prohibited Actions

- NEVER change the domain on a production app monitor without updating the JavaScript snippet
- NEVER increase session sample rate to 1.0 without cost analysis
- NEVER disable telemetry types that feed downstream alerting
- NEVER enable session replay without privacy review

## Phase 3 — Rollback

1. Revert configuration changes: `aws rum update-app-monitor --name <name> --app-monitor-configuration <previous-config>`
2. Verify data collection resumes with correct telemetry types
3. Confirm session sample rate is at the intended value
4. Document all configuration changes and rollback actions
5. Verify the JavaScript snippet still works with the updated configuration

## Output Format

```yaml
root_cause: "app_monitor_configuration — <specific_cause>"
evidence:
  - type: configuration_issue
    content: "<configuration details>"
severity: MEDIUM
mitigation:
  immediate: "Fix app monitor configuration"
  long_term: "Manage app monitor configuration via IaC"
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
