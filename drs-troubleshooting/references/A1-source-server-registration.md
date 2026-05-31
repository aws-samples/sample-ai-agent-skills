---
title: "A1 — Source Server Registration"
description: "Diagnose source server registration failures in DRS"
status: active
severity: HIGH
triggers:
  - "source server not appearing"
  - "server not registered"
  - "DRS no servers"
owner: devops-agent
objective: "Identify and resolve source server registration issues"
context: "Source servers must have the AWS Replication Agent installed and network connectivity to DRS endpoints. Registration failures are typically caused by agent installation issues, network connectivity, or IAM credential problems."
---

## Phase 1 — Triage

MUST:
- List all source servers: `aws drs describe-source-servers`
- Check agent installation status on the source server (agent logs at /var/log/aws-replication-agent/ on Linux)
- Verify outbound TCP 443 connectivity from source to DRS endpoint
- Verify outbound TCP 1500 connectivity from source to replication server

SHOULD:
- Check IAM credentials used by the agent: `aws sts get-caller-identity` (from source server)
- Verify the staging area subnet exists and has proper routing
- Check CloudTrail for InitializeService events: `aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventName,AttributeValue=InitializeService`

MAY:
- Review agent installer logs for errors
- Test DNS resolution of DRS endpoints from source server

## Phase 2 — Remediate

MUST:
- Ensure agent has valid AWS credentials with drs:* permissions
- Verify network path from source to DRS endpoints (TCP 443) and replication server (TCP 1500)
- Reinstall agent if installation is corrupted

SHOULD:
- Configure proxy settings if source server uses a proxy for outbound access
- Verify staging area subnet security groups allow inbound TCP 1500

MAY:
- Use VPC endpoints for DRS if direct internet access is not available

## Common Issues

- symptoms: "Agent installed but server not appearing in DRS console"
  diagnosis: "Network connectivity or IAM credential issue preventing registration."
  resolution: "Verify TCP 443 to DRS endpoint and IAM credentials. Check agent logs."

- symptoms: "Agent installation fails with permission error"
  diagnosis: "IAM user/role lacks required DRS permissions."
  resolution: "Attach AWSElasticDisasterRecoveryAgentInstallationPolicy to the IAM entity."

## Output Format

```yaml
root_cause: "source_server_registration — <specific_cause>"
evidence:
  - type: agent_status
    content: "<agent installation and connectivity status>"
  - type: iam_permissions
    content: "<IAM credential verification>"
severity: HIGH
mitigation:
  immediate: "Fix connectivity or credential issue"
  long_term: "Automate agent deployment with validated network paths"
```


---

## Safety Ratings

| Dimension | Rating |
|-----------|--------|
| Data Sensitivity | HIGH — Source server registration involves IAM credentials and network configuration |
| Blast Radius | LOW — Registration failures don't affect existing replication |
| Rollback Complexity | LOW — Agent can be reinstalled; server can be re-registered |
| Compliance Scope | HIGH — Source server registration is the foundation of DR readiness |

## Escalation Conditions

- Escalate to security if IAM credentials used by the agent are overly permissive
- Escalate to network team if connectivity issues prevent registration
- Escalate to DR coordinator if registration failures delay DR readiness

## Data Sensitivity

- **IAM credentials** on source servers are authentication secrets — rotate regularly
- **Network configuration** reveals infrastructure topology
- **Agent logs** may contain server details

## PROHIBITED Actions

- Do NOT use root or admin IAM credentials for the replication agent
- Do NOT leave TCP 443/1500 open to 0.0.0.0/0 — scope to DRS endpoints
- Do NOT install the agent on servers without authorization from server owners

## Phase 3 — Rollback

- Uninstall the agent from the source server if registration was unauthorized
- Revoke IAM credentials used by the agent if they were compromised
- Remove the source server from DRS: `aws drs disconnect-from-service --source-server-id <id>`

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
