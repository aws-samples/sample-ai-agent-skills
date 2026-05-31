---
title: "E1 — Transit Gateway Registration"
description: "Diagnose transit gateway registration failures in global networks"
status: active
severity: HIGH
triggers: ["transit gateway registration failed", "TGW not registered", "register-transit-gateway error", "TGW registration pending"]
owner: devops-agent
objective: "Resolve transit gateway registration issues in AWS Network Manager global networks"
context: "Transit gateways must be explicitly registered in a global network before they can be used as core network attachments. Registration is per-region."
---

## Phase 1 — Triage

MUST:
- List TGW registrations: `aws networkmanager get-transit-gateway-registrations --global-network-id <gn-id>`
- Verify the TGW exists: `aws ec2 describe-transit-gateways --transit-gateway-ids <tgw-id>`
- Check IAM permissions for `networkmanager:RegisterTransitGateway` and `ec2:DescribeTransitGateways`

SHOULD:
- Verify the TGW is in AVAILABLE state
- Check if the TGW is already registered in another global network
- Review CloudTrail for RegisterTransitGateway errors

MAY:
- Verify the TGW is in the same account or shared via RAM
- Check TGW route tables for existing associations

## Phase 2 — Remediate

MUST:
- Fix IAM permissions if AccessDeniedException
- Register the TGW: `aws networkmanager register-transit-gateway --global-network-id <gn-id> --transit-gateway-arn <tgw-arn>`
- Wait for registration to complete (AVAILABLE state)

SHOULD:
- Verify registration state: `aws networkmanager get-transit-gateway-registrations --global-network-id <gn-id>`
- Tag the TGW for identification in Network Manager

MAY:
- Register TGWs in all regions as part of a global deployment
- Use CloudFormation for automated TGW registration

## Common Issues

- symptoms: "RegisterTransitGateway returns ResourceNotFoundException"
  diagnosis: "Global network ID is incorrect or the TGW ARN is malformed."
  resolution: "Verify the global network ID and TGW ARN. Use describe-global-networks and describe-transit-gateways to get correct values."

- symptoms: "TGW registration stuck in PENDING state"
  diagnosis: "Cross-account TGW requires RAM sharing to be accepted."
  resolution: "Accept the RAM resource share in the TGW owner account, then retry registration."

## Safety Ratings

```
safety_ratings:
  - "get-transit-gateway-registrations: GREEN — read-only enumeration"
  - "describe-transit-gateways: GREEN — read-only TGW details"
  - "register-transit-gateway: YELLOW — registers TGW in global network, no traffic impact"
  - "deregister-transit-gateway: RED — removes TGW from global network, affects management visibility"
```

## Escalation Conditions

- TGW registration fails with internal errors after fixing permissions and ARNs
- Cross-account TGW registration requires organizational RAM policy changes
- TGW is registered in another global network and cannot be moved
- Registration affects production TGW with existing VPC attachments

## Data Sensitivity

- `aws networkmanager get-transit-gateway-registrations`: LOW — returns registration metadata
- `aws ec2 describe-transit-gateways`: MEDIUM — reveals TGW configuration and ASN
- `aws cloudtrail lookup-events`: MEDIUM — reveals API call history

## Prohibited Actions

- NEVER deregister a TGW that has active core network attachments
- NEVER register a production TGW in a test global network
- NEVER assume TGW registration creates network connectivity — it only enables management
- NEVER modify TGW route tables as part of registration troubleshooting without understanding the impact

## Phase 3 — Rollback

1. If a TGW was registered in error, deregister it: `aws networkmanager deregister-transit-gateway --global-network-id <gn-id> --transit-gateway-arn <tgw-arn>`
2. Verify deregistration completed: `aws networkmanager get-transit-gateway-registrations --global-network-id <gn-id>`
3. If RAM sharing was configured for cross-account, revoke the share if no longer needed
4. Confirm the TGW's existing VPC attachments and route tables are unaffected
5. Document the rollback and notify the network team

## Output Format

```yaml
root_cause: "tgw_registration — <specific_cause>"
evidence:
  - type: registration_state
    content: "<registration details and error>"
severity: HIGH
mitigation:
  immediate: "Fix registration issue"
  long_term: "Automate TGW registration via IaC"
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
