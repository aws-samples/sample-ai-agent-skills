---
title: "A3 — Endpoint Status"
description: "Diagnose VPC interface endpoint status issues"
status: active
severity: HIGH
triggers:
  - "endpoint pending"
  - "endpoint failed"
  - "endpoint rejected"
  - "endpoint not available"
  - "endpoint status"
owner: devops-agent
objective: "Resolve endpoint status issues to restore connectivity"
context: "VPC endpoints go through lifecycle states: pending, pendingAcceptance, available, deleting, deleted, rejected, expired, failed. Only 'available' state passes traffic. Pending states indicate the endpoint is being created or waiting for acceptance. Failed/rejected states require investigation."
---

## Phase 1 — Triage

MUST:
- Check endpoint status: `aws ec2 describe-vpc-endpoints --vpc-endpoint-ids <endpoint-id> --query 'VpcEndpoints[*].{State:State,ServiceName:ServiceName,CreationTimestamp:CreationTimestamp}'`
- Check connection status (for custom services): `aws ec2 describe-vpc-endpoint-connections --filters Name=vpc-endpoint-id,Values=<endpoint-id>`
- Check CloudTrail for endpoint events: `aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventName,AttributeValue=CreateVpcEndpoint --max-results 10`

SHOULD:
- Check if the endpoint service is available: `aws ec2 describe-vpc-endpoint-services --service-names <service-name>`
- Verify ENIs were created: `aws ec2 describe-network-interfaces --filters Name=vpc-endpoint-id,Values=<endpoint-id>`
- Check for recent modifications: `aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventName,AttributeValue=ModifyVpcEndpoint --max-results 10`

MAY:
- Check AWS Health Dashboard for service-level issues
- Review VPC Flow Logs for traffic through endpoint ENIs

## Phase 2 — Remediate

MUST:
- For pendingAcceptance: contact the endpoint service provider to accept the connection
- For failed: delete and recreate the endpoint after fixing the root cause
- For rejected: contact the service provider to understand rejection reason
- Verify endpoint reaches "available" state before testing connectivity

SHOULD:
- Accept pending connections (provider side): `aws ec2 accept-vpc-endpoint-connections --service-id <service-id> --vpc-endpoint-ids <endpoint-id>`
- Monitor endpoint state after creation or modification

MAY:
- Enable auto-accept on the endpoint service to avoid manual acceptance
- Set up CloudWatch Events for endpoint state changes

## Common Issues

- symptoms: "Endpoint stuck in pendingAcceptance"
  diagnosis: "The endpoint service requires manual acceptance and it hasn't been accepted."
  resolution: "Provider must accept: aws ec2 accept-vpc-endpoint-connections --service-id <svc-id> --vpc-endpoint-ids <vpce-id>"

- symptoms: "Endpoint in failed state"
  diagnosis: "Endpoint creation failed due to subnet, security group, or service issue."
  resolution: "Delete the failed endpoint. Fix the root cause. Recreate."

- symptoms: "Endpoint was rejected"
  diagnosis: "The endpoint service provider rejected the connection."
  resolution: "Contact the provider. Verify your account is in the allowed principals list."

## Output Format

```yaml
root_cause: "endpoint_status — <specific_cause>"
evidence:
  - type: endpoint_state
    content: "<current endpoint state>"
  - type: connection_status
    content: "<connection acceptance status>"
  - type: cloudtrail
    content: "<relevant endpoint events>"
severity: HIGH
mitigation:
  immediate: "Accept connection or recreate endpoint"
  long_term: "Enable auto-accept and monitor endpoint lifecycle"
```


## Safety Ratings
- Overall Risk: MEDIUM
- Data Exposure Risk: LOW — endpoint status is operational data
- Service Disruption Risk: MEDIUM — unhealthy endpoints affect connectivity
- Compliance Impact: LOW — endpoint status monitoring is operational

## Escalation Conditions
- Endpoint enters failed or rejected state
- Endpoint status changes unexpectedly
- Multiple endpoints become unhealthy simultaneously
- Endpoint pending acceptance for extended period
- Endpoint status does not recover after remediation

## Data Sensitivity
- Endpoint status data — INTERNAL
- Endpoint configurations — INTERNAL
- Health check results — INTERNAL
- Status change history — INTERNAL
- Never expose endpoint status details externally

## Prohibited Actions
- Do NOT ignore unhealthy endpoint status
- Do NOT delete endpoints in pending state without investigation
- Do NOT bypass endpoint status monitoring
- Do NOT modify endpoints in transitional states
- Do NOT ignore endpoint rejection reasons
- Do NOT disable endpoint health monitoring

## Phase 3 — Rollback
- Restore previous endpoint configurations
- Re-create endpoints that entered failed state
- Revert changes that caused status degradation
- Verify endpoint health after changes
- Document status issues and resolution steps
- Update endpoint monitoring procedures

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
