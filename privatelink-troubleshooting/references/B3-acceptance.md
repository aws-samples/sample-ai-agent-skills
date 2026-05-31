---
title: "B3 — Connection Acceptance"
description: "Diagnose VPC endpoint service connection acceptance issues"
status: active
severity: MEDIUM
triggers:
  - "pending acceptance"
  - "connection not accepted"
  - "auto-accept"
  - "reject connection"
  - "acceptance required"
owner: devops-agent
objective: "Resolve endpoint service connection acceptance issues"
context: "By default, endpoint service connections require manual acceptance by the provider. Connections in pendingAcceptance state do not pass traffic. Auto-accept can be enabled. AWS service endpoints are automatically accepted. Rejected connections must be recreated."
---

## Phase 1 — Triage

MUST:
- Check connection status: `aws ec2 describe-vpc-endpoint-connections --filters Name=vpc-endpoint-id,Values=<endpoint-id>`
- Check endpoint service acceptance setting: `aws ec2 describe-vpc-endpoint-service-configurations --service-ids <service-id> --query 'ServiceConfigurations[*].AcceptanceRequired'`
- Verify the endpoint consumer account is in allowed principals: `aws ec2 describe-vpc-endpoint-service-permissions --service-id <service-id>`

SHOULD:
- Check CloudTrail for acceptance/rejection events
- Verify the endpoint is not expired (connections can expire)
- Check if the endpoint was previously rejected

MAY:
- Review the endpoint service's acceptance workflow
- Check for automation that handles acceptance

## Phase 2 — Remediate

MUST:
- Accept pending connections (provider side): `aws ec2 accept-vpc-endpoint-connections --service-id <service-id> --vpc-endpoint-ids <endpoint-id>`
- Add consumer account to allowed principals if missing: `aws ec2 modify-vpc-endpoint-service-permissions --service-id <service-id> --add-allowed-principals arn:aws:iam::<account-id>:root`

SHOULD:
- Enable auto-accept for trusted accounts: `aws ec2 modify-vpc-endpoint-service-configuration --service-id <service-id> --no-acceptance-required`
- Set up notifications for pending connection requests

MAY:
- Implement Lambda-based auto-acceptance with validation logic
- Document the acceptance process for operations teams

## Common Issues

- symptoms: "Endpoint stuck in pendingAcceptance for hours"
  diagnosis: "Manual acceptance required and provider hasn't accepted."
  resolution: "Provider must accept the connection. Or enable auto-accept on the service."

- symptoms: "Endpoint was rejected"
  diagnosis: "Provider rejected the connection. Consumer account may not be authorized."
  resolution: "Add consumer to allowed principals. Delete rejected endpoint and recreate."

- symptoms: "Auto-accept enabled but connection still pending"
  diagnosis: "Consumer account not in allowed principals list. Auto-accept only works for allowed principals."
  resolution: "Add the consumer account to allowed principals list."

## Output Format

```yaml
root_cause: "acceptance — <specific_cause>"
evidence:
  - type: connection_status
    content: "<connection state and details>"
  - type: acceptance_config
    content: "<acceptance required setting>"
  - type: allowed_principals
    content: "<allowed principals list>"
severity: MEDIUM
mitigation:
  immediate: "Accept pending connection or fix permissions"
  long_term: "Configure auto-accept and allowed principals"
```


## Safety Ratings
- Overall Risk: HIGH
- Data Exposure Risk: MEDIUM — acceptance controls who can access endpoint services
- Service Disruption Risk: LOW — acceptance issues delay but don't disrupt existing connections
- Compliance Impact: HIGH — acceptance controls enforce access authorization

## Escalation Conditions
- Unauthorized connections accepted
- Legitimate connections stuck in pending acceptance
- Auto-accept settings modified without authorization
- Acceptance notifications not being delivered
- Acceptance process fails for authorized accounts

## Data Sensitivity
- Acceptance configurations — CONFIDENTIAL
- Authorized account lists — CONFIDENTIAL
- Connection request details — INTERNAL
- Acceptance logs — INTERNAL
- Never expose acceptance configurations externally

## Prohibited Actions
- Do NOT auto-accept connections from unknown accounts
- Do NOT bypass acceptance review processes
- Do NOT accept connections without verifying the requesting account
- Do NOT modify acceptance settings without security review
- Do NOT ignore pending acceptance requests
- Do NOT share acceptance configurations externally

## Phase 3 — Rollback
- Reject unauthorized connections
- Restore previous acceptance configurations
- Revert auto-accept settings
- Verify authorized connections after changes
- Document acceptance changes and security review
- Update acceptance management procedures

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
