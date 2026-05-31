---
title: "A1 — MQTT Connection Failures"
description: "Diagnose why devices cannot connect to AWS IoT Core via MQTT"
status: active
severity: HIGH
triggers:
  - "MQTT connection failed"
  - "device not connecting"
  - "connection refused"
  - "TLS handshake failed"
  - "connection timeout"
owner: devops-agent
objective: "Identify why MQTT connections are failing and resolve the issue"
context: "MQTT connection failures occur due to: wrong endpoint, certificate issues, TLS version mismatch, IoT policy missing iot:Connect, client ID conflicts, or network/firewall blocking port 8883."
---

## Phase 1 — Triage

MUST:
- Verify IoT endpoint: `aws iot describe-endpoint --endpoint-type iot:Data-ATS`
- Check certificate status: `aws iot describe-certificate --certificate-id <cert-id> --query 'certificateDescription.{Status:status,CertArn:certificateArn}'`
- Verify certificate is attached to a thing: `aws iot list-thing-principals --thing-name <thing-name>`
- Check IoT policy allows iot:Connect: `aws iot list-attached-policies --target <cert-arn>` then `aws iot get-policy --policy-name <name>`
- Check CloudWatch metric: `aws cloudwatch get-metric-statistics --namespace AWS/IoT --metric-name Connect.AuthError --start-time <start> --end-time <end> --period 300 --statistics Sum`

SHOULD:
- Verify client ID in policy matches device client ID
- Check if port 8883 (MQTT) or 443 (MQTT over WSS) is accessible
- Verify CA certificate is the Amazon Root CA (ATS endpoint)
- Check IoT logs: `aws logs filter-log-events --log-group-name AWSIotLogsV2 --filter-pattern "CONNECT"`

MAY:
- Test connection with mosquitto_pub/sub client
- Check for client ID conflicts (two devices using same client ID)
- Verify TLS 1.2 support on device

## Phase 2 — Remediate

MUST:
- Activate certificate if INACTIVE: `aws iot update-certificate --certificate-id <cert-id> --new-status ACTIVE`
- Attach certificate to thing: `aws iot attach-thing-principal --thing-name <thing> --principal <cert-arn>`
- Attach policy to certificate: `aws iot attach-policy --policy-name <policy> --target <cert-arn>`
- Ensure policy allows: `{"Effect":"Allow","Action":"iot:Connect","Resource":"arn:aws:iot:<region>:<account>:client/<client-id>"}`

SHOULD:
- Use ATS endpoint (iot:Data-ATS) instead of legacy endpoint
- Ensure unique client IDs per device
- Use Amazon Root CA 1 for ATS endpoints

MAY:
- Implement connection retry with exponential backoff
- Set up CloudWatch alarms on Connect.AuthError metric

## Output Format

```yaml
root_cause: "mqtt_connection_failure — <specific_cause>"
evidence:
  - type: certificate_status
    content: "<certificate state>"
  - type: policy_check
    content: "<iot:Connect permission>"
  - type: endpoint
    content: "<IoT endpoint>"
severity: HIGH
mitigation:
  immediate: "Fix certificate, policy, or endpoint configuration"
  long_term: "Implement connection monitoring and automated certificate management"
```

## Safety Ratings
```
safety_ratings:
  - "Triage commands (describe/list/get): GREEN — read-only"
  - "Configuration changes: YELLOW — recoverable"
  - "Delete/terminate operations: RED — irreversible"
```

## Escalation Conditions
- "Resource serves production workload"
- "Fix requires deleting or recreating resources"
- "Fix involves security or encryption changes"
- "Blast radius affects multiple dependent services"

## Data Sensitivity
- HIGH: credentials, PII, security configurations
- MEDIUM: resource configurations, metrics
- LOW: status information, error messages

## Prohibited Actions
- Never revoke or delete device certificates without confirming device decommission
- Never modify IoT policies on production devices without testing in a staging environment
- Never delete thing groups that contain active production devices
- Never alter MQTT topic rules that route critical telemetry data
- Never disable custom authorizers serving active device connections

## Phase 3 — Rollback
- "Document current state before any changes"
- "If fix worsens the issue, revert to documented state"
- Reattach previous certificates if device connectivity is lost after changes
- Restore previous IoT policy versions if devices lose permissions
- Revert rule engine action configurations if message routing breaks

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
