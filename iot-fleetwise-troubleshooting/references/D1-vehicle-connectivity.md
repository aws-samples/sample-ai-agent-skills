---
title: "D1 — Vehicle Connectivity"
description: "Diagnose vehicle-to-cloud connectivity failures and MQTT communication issues"
status: active
severity: HIGH
triggers: ["vehicle not connected", "vehicle offline", "MQTT connection failed", "vehicle cannot reach cloud"]
owner: devops-agent
objective: "Resolve vehicle connectivity issues between edge agent and AWS IoT Core"
context: "Vehicles connect to AWS IoT Core via MQTT using X.509 certificates. The edge agent manages the connection. Connectivity issues can be certificate, network, or endpoint related."
---

## Phase 1 — Triage

MUST:
- Check vehicle exists: `aws iotfleetwise get-vehicle --vehicle-name <name>`
- Verify IoT thing connectivity (requires fleet indexing): `aws iot search-index --query-string "thingName:<vehicle-name> AND connectivity.connected:true"`
- Check certificate status for the vehicle's IoT thing: `aws iot list-thing-principals --thing-name <vehicle-name>`

SHOULD:
- Verify certificate is ACTIVE: `aws iot describe-certificate --certificate-id <cert-id> --query 'certificateDescription.status'`
- Check IoT policy attached to certificate allows required MQTT operations
- Verify IoT endpoint: `aws iot describe-endpoint --endpoint-type iot:Data-ATS`

MAY:
- Review IoT Core logs for connection errors: `aws logs filter-log-events --log-group-name AWSIotLogsV2 --filter-pattern "<vehicle-name>"`
- Check edge agent configuration file on the vehicle
- Test network connectivity from vehicle to IoT endpoint on port 8883

## Phase 2 — Remediate

MUST:
- Ensure vehicle's IoT certificate is ACTIVE
- Verify IoT policy allows iot:Connect, iot:Publish, iot:Subscribe, iot:Receive for FleetWise topics
- Confirm edge agent configuration has correct endpoint URL and certificate paths

SHOULD:
- Verify TLS certificate chain is complete (root CA, device cert, private key)
- Check vehicle's network connectivity (cellular, Wi-Fi, ethernet)
- Ensure edge agent is configured with correct vehicle name matching the FleetWise vehicle

MAY:
- Rotate vehicle certificate if compromised: create new cert, attach to thing, deactivate old cert
- Update edge agent to latest version for connectivity improvements
- Configure connection retry and backoff settings in edge agent

## Safety Ratings

```
safety_ratings:
  - "get-vehicle: GREEN — read-only inspection"
  - "search-index: GREEN — read-only connectivity check"
  - "list-thing-principals: GREEN — read-only certificate listing"
  - "update-certificate --new-status ACTIVE: YELLOW — activates authentication credential"
  - "update-certificate --new-status REVOKED: RED — permanently revokes certificate"
```

## Escalation Conditions

- Fleet-wide connectivity loss indicating infrastructure or certificate authority issue
- Vehicle certificate compromised requiring emergency rotation
- Edge agent firmware bug causing connectivity failures across vehicle model
- Network infrastructure changes blocking MQTT traffic to AWS endpoints

## Data Sensitivity

- `aws iotfleetwise get-vehicle`: LOW — returns vehicle metadata
- `aws iot list-thing-principals`: MEDIUM — returns certificate ARNs
- `aws iot describe-certificate`: LOW — returns certificate metadata, not private key
- Edge agent configuration: HIGH — contains endpoint and credential paths

## Prohibited Actions

- NEVER revoke a vehicle certificate without a plan to provision a replacement
- NEVER share vehicle private keys or certificate files in unsecured channels
- NEVER disable TLS verification on the edge agent for troubleshooting
- NEVER modify IoT policies to allow wildcard (*) topics in production

## Phase 3 — Rollback

1. If certificate was deactivated, reactivate: `aws iot update-certificate --certificate-id <id> --new-status ACTIVE`
2. If IoT policy was modified, restore previous version
3. If edge agent configuration was changed, restore from backup
4. Verify vehicle reconnects and resumes data collection
5. Monitor connectivity for 30 minutes to confirm stability

## Common Issues

- symptoms: "Vehicle shows as disconnected in fleet indexing"
  diagnosis: "Edge agent is not running, certificate is inactive, or network connectivity is lost."
  resolution: "Check edge agent process on vehicle. Verify certificate status. Test network connectivity to IoT endpoint."

- symptoms: "Edge agent connects but immediately disconnects"
  diagnosis: "IoT policy does not allow the client ID used by the edge agent, or another client is using the same client ID."
  resolution: "Verify IoT policy allows the client ID (typically the vehicle name). Ensure no duplicate connections with the same client ID."

## Output Format

```yaml
root_cause: "vehicle_connectivity — <specific_cause>"
evidence:
  - type: connectivity_error
    content: "<error details>"
severity: HIGH
mitigation:
  immediate: "Restore vehicle connectivity"
  long_term: "Implement connectivity monitoring and alerting"
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
